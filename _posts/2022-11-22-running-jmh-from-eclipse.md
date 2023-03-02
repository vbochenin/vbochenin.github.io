---
title: "Running JMH benchmark from Eclipse"
layout: post
author: vbochenin
tags:
  - java
  - performance
  - jmh
  - eclipse
categories:
  - dev-journal
---
## What issue I'm trying to solve

A few months ago, we started to use [JMH](https://github.com/openjdk/jmh) in our project to test and find performance issues. 
The tool provides multiple modes and profilers, and we found this useful for our purposes. 

Intellij IDEA, which I'm using, has a useful [Intellij IDEA plugin for Java Microbenchmark Harness](https://github.com/artyushov/idea-jmh-plugin). The plugin's functionality is similar to JUnit Plugin. It simplifies benchmarks development and debugging by allowing running some benchmarks together or separately.

But... half of our team uses Eclipse as a major IDE and the IDE does not have any plugins or support for the tool.
Even if we can run it with the `main` method, it is inconvenient to change the `include` pattern and do not to forget to revert the changes before committing them into git.

So, after a bit of brainstorming, we decided to write a custom JUnit Runner with functionality to run a benchmark.

## JUnit 4 runners 

JUnit 4 has API to run any class as a test suite. 
All you need is only to:
1. write class extending `org.junit.runner.Runner` class
``` java
public class BenchmarkRunner extends Runner {
	//...
}
```
2. implement constructor and few methods
``` java
public class BenchmarkRunner extends Runner {
   public BenchmarkRunner(Class<?> benchmarkClass) {
   }

   public Description getDescription() {
	   //...
   }  

   public void run(RunNotifier notifier) {
	   //...
   }
}
```
3. add the runner to your test class
``` java
@RunWith(BenchmarkRunner.class)  
public class CustomCollectionBenchmark {
	//...
}  
```

## Implementing Benchmark JUnit runner

First, we need to provide information about tests to the JUnit engine.

JUnit runners have the `getDescription()` method for it. But how to get information about test classes and test methods? 

From JavaDoc: 
> When creating a custom runner, in addition to implementing the abstract methods here you must also provide a constructor that takes as an argument the Class containing the tests.

So, we may get the class as a constructor argument and get all information with [reflection](https://www.oracle.com/technical-resources/articles/java/javareflection.html#:~:text=Reflection%20is%20a%20feature%20in,its%20members%20and%20display%20them.)'s help.

``` java
public class BenchmarkRunner extends Runner {
	private final Class<?> benchmarkClass; // (1)  
	private final List<Method> benchmarkMethods; // (2) 
	  
	public BenchmarkRunner(Class<?> benchmarkClass) {  
	    this.benchmarkClass = benchmarkClass;  
	    this.benchmarkMethods = Arrays.stream(benchmarkClass.getDeclaredMethods())  
	            .filter(m -> m.isAnnotationPresent(Benchmark.class))  
	            .collect(Collectors.toList());  
	}
	//...
}
```
Now I have all the required information to provide a test suite description to the JUnit engine:
- Actual benchmark class
- All methods marked with `@Benchmark` in this class

``` java
public class BenchmarkRunner extends Runner {  
	//... 
    @Override  
    public Description getDescription() {  
        Description result = Description.createSuiteDescription(benchmarkClass);  
        benchmarkMethods.stream()  
                .map(m -> Description.createTestDescription(benchmarkClass, m.getName()))  
                .forEach(result::addChild);  
        return result;  
    }  
	//...
}
```

Once we run the test we may see something like this:

![Eclipse JUnit run](/assets/img/posts/2022-11-22-jmh-from-eclipse/eclips-junit-view.png)

Let's run our benchmarks. 
We need to implement the method `org.junit.runner.Runner.run(RunNotifier)` where `RunNotifier` is responsible to notify the engine about test runs.

The idea is we run sequentially one-by-one benchmark methods, each in separate `org.openjdk.jmh.runner.Runner`.

``` java
public class BenchmarkRunner extends Runner {

	...

    @Override
    public void run(RunNotifier notifier) {
        for (Method benchmarkMethod : benchmarkMethods) {
            Description testDescription = getBenchmarkMethodDescription(benchmarkMethod);
            try {
            	notifier.fireTestStarted(testDescription);
                Options opt = new OptionsBuilder()
                        .include(".*" + benchmarkClass.getName() + "." + benchmarkMethod.getName() + ".*")
                        .jvmArgsAppend("-Djmh.separateClasspathJAR=true")
                        .build();

                new org.openjdk.jmh.runner.Runner(opt).run();

                notifier.fireTestFinished(testDescription);
            } catch (Exception e) {
                notifier.fireTestFailure(new Failure(testDescription, e));
                return;
            }
        }
    }
    
    private Description getBenchmarkMethodDescription(Method benchmarkMethod) {
        return Description.createTestDescription(benchmarkClass, benchmarkMethod.getName());
    }
}
```

Options mean follow:
- `include` - benchmark we would like to include in the run.
- `jvmArgsAppend("-Djmh.separateClasspathJAR=true")` - specific option, telling JMH to build `classpath.jar` to avoid the  [The filename or extension is too long](https://stackoverflow.com/questions/65340027/jmh-1-27-fails-to-work-with-long-arguments-list-in-jvm) error

As you may see, we notify `RunNotifier` when we start and finish our test, successful or not.
Looks good, but we are running all tests even when choosing to run only a single one.

## Filtering

Our runner should implement the `org.junit.runner.manipulation.Filterable` interface to allow to JUnit engine to tell our code what tests should be run.
The interface has only a single method `void filter(Filter)` and `org.junit.runner.manipulation.Filter` argument has the `shouldRun(Description)` method we may use to know if the test requested to run.
The method doesn't return anything, so looks like we need to store filtered results and use them later.
``` java
  
public class BenchmarkRunner extends Runner implements Filterable {

	//...
  
    private List<Method> readyToRunMethods; // <= add new field to store filter result  
  
    @Override  
    public Description getDescription() {  
        Description result = Description.createSuiteDescription(benchmarkClass);  
        readyToRunMethods.stream()  // <= use the field here
                .map(this::getBenchmarkMethodDescription)  
                .forEach(result::addChild);  
        return result;  
    }  

	//...
	
    @Override  
    public void run(RunNotifier notifier) {  
        for (Method benchmarkMethod : readyToRunMethods) {  // <= and here
            //...  
        }  
    }  
  
    @Override  
    public void filter(Filter filter) throws NoTestsRemainException {  
        List<Method> filteredMethods = new ArrayList<>();  
  
        for (Method benchmarkMethod : benchmarkMethods) {  
            if (filter.shouldRun(getBenchmarkMethodDescription(benchmarkMethod))) {  
                filteredMethods.add(benchmarkMethod);  
            }  
        }  
  
        if (filteredMethods.isEmpty()) {  
            throw new NoTestsRemainException();  
        }  
  
        this.readyToRunMethods = filteredMethods;  
    }  
}
```


Now it runs only methods we ask to run.

## Conclusion

Finally, we've got a unified way to run benchmarks from any IDE during development and debugging. This really simplifies our daily life. We still run the benchmarks using the main method to reduce environmental noise and get reliable results for analysis.

_You may find the code in [GitHub](https://github.com/vbochenin/code.vbochenin.github.io/tree/main/benchmark-runner)._ 

