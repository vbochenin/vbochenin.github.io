---
title: "Reducing heap allocations with JMH"
layout: post
author: vbochenin
tags:
  - java
  - performance
  - jmh
categories:
  - dev-journal
---
## What an issue I've tried to solve
My task was to investigate the reason why typed enumeration occupies 13 Mb per instance in a customer environment. The typed enumeration is a domain object allowing to users configure enumeration type for custom fields.
The use case is reproducible once enumeration has around 1k elements.

So I may assume the following possible causes:
- a memory leak
	- keeping in memory some stale caches values 
- storing big files in memory
	- the enumeration is uploaded by user as XML file, so some intermediate big object can be cached 
- duplicating of data (if so, then why?). 
	- multiple copies of same data sorted in cache for different keys 

## What I did to find the cause

First, I've rerun the scenario locally, made a heap dump, and analyzed it with [MemoryAnalizer](https://www.eclipse.org/mat/). 
It is visible from the report below (_Histogram -> Choose TypedEnumeration in filter -> List of object with outgoing reference -> Sort by Retained Heap_), that we are storing duplicated data for each key in `control2Options` field.

```
Class Name                                                              | Shallow Heap | Retained Heap
-------------------------------------------------------------------------------------------------------
TypedEnumeration @ 0xdedf8050                                           |           64 |     4,637,960
|- control2Options java.util.HashMap @ 0xdef01f68                       |           48 |     3,925,856
|  |- table java.util.HashMap$Node[32] @ 0xdf7d5b10                     |          144 |     3,925,808
|  |  |- [26] java.util.HashMap$Node @ 0xdf7d5ba0                       |           32 |       712,256
|  |  |- [4] java.util.HashMap$Node @ 0xdfc3b790                        |           32 |       712,256
|  |  |- [14] java.util.HashMap$Node @ 0xdf12a800                       |           32 |       356,128
|  |  |- [13] java.util.HashMap$Node @ 0xdf24eab8                       |           32 |       356,128
|  |  |- [22] java.util.HashMap$Node @ 0xdf3a9608                       |           32 |       356,128
|  |  |- [23] java.util.HashMap$Node @ 0xdfa2e9a0                       |           32 |       356,128
|  |  |- [11] java.util.HashMap$Node @ 0xdfa50ff0                       |           32 |       356,128
|  |  |- [6] java.util.HashMap$Node @ 0xdfb91928                        |           32 |       356,128
|  |  |- [5] java.util.HashMap$Node @ 0xdfd3ca10                        |           32 |       356,128
|  |  |- [0] java.util.HashMap$Node @ 0xde742630                        |           32 |         4,128
|  |  |- [19] java.util.HashMap$Node @ 0xdf38cad0                       |           32 |         4,128
|  |  |-  class java.util.HashMap$Node[] @ 0xd444d5a8                   |            0 |             0
|  |  '- Total: 12 entries                                              |              |              
|  |- <class> class java.util.HashMap @ 0xd4450dd0 System Class         |           40 |           144
|  '- Total: 2 entries                                                  |              |              
|- allOptions java.util.ArrayList @ 0xdedf8090                          |           24 |         8,040
.....
'- Total: 10 entries                                                    |              |              
TypedEnumeration @ 0xe0064a60                                           |           64 |     4,637,960
|- control2Options java.util.HashMap @ 0xe055ecb0                       |           48 |     3,925,856
|  |- table java.util.HashMap$Node[32] @ 0xe08dd030                     |          144 |     3,925,808
|  |  |- [4] java.util.HashMap$Node @ 0xe086f630                        |           32 |       712,256
|  |  |- [26] java.util.HashMap$Node @ 0xe08dd0c0                       |           32 |       712,256
|  |  |- [23] java.util.HashMap$Node @ 0xe053daf8                       |           32 |       356,128
|  |  |- [5] java.util.HashMap$Node @ 0xe084cd10                        |           32 |       356,128
|  |  |- [13] java.util.HashMap$Node @ 0xe08521f0                       |           32 |       356,128
|  |  |- [6] java.util.HashMap$Node @ 0xe08dc138                        |           32 |       356,128
|  |  |- [11] java.util.HashMap$Node @ 0xe09de780                       |           32 |       356,128
|  |  |- [14] java.util.HashMap$Node @ 0xe0a665c0                       |           32 |       356,128
|  |  |- [22] java.util.HashMap$Node @ 0xe0acc8c0                       |           32 |       356,128
|  |  |- [0] java.util.HashMap$Node @ 0xdfa1e7d0                        |           32 |         4,128
|  |  |- [19] java.util.HashMap$Node @ 0xe0aefcb0                       |           32 |         4,128
|  |  |-  class java.util.HashMap$Node[] @ 0xd444d5a8                   |            0 |             0
|  |  '- Total: 12 entries                                              |              |              
|  |- <class> class java.util.HashMap @ 0xd4450dd0 System Class         |           40 |           144
|  '- Total: 2 entries                                                  |              |              
|- allOptions java.util.ArrayList @ 0xe0064b08                          |           24 |         8,040
....
'- Total: 10 entries                                                    |              |              

```


Lets  rerun the scenario with the profiler turned on.
I've used [JMC](https://jdk.java.net/jmc/8/) and [Flight Records](https://docs.oracle.com/javacomponents/jmc-5-4/jfr-runtime-guide/about.htm#JFRUH170), and it showed me that the application occupies the extra memory during bootstrap.


![Memory allocations](/assets/img/posts/2022-10-14-when-im-using-jmh/heap-allocations.png)

Not exactly the place, but it reduced the scope significantly and after some code investigation, I found where the application is wasting memory.

Success? Nope, I still need to fix the issue and prove that our fix is fixing something.

This is where [JMH](https://github.com/openjdk/jmh) is coming to the stage.

## What we can do with the tool 

JMH is a microbenchmark framework for Java. 
It cares about memory alignments, method inlines, warmups, and all other stuff that makes benchmarks hard to write.
It also has multiple profilers you may use in your benchmarks.

```
- Profiler (org.openjdk.jmh.profile)
	|- ExternalProfiler (org.openjdk.jmh.profile)
	|	|- AbstractPerfAsmProfiler (org.openjdk.jmh.profile)
	|	|	|- DTraceAsmProfiler (org.openjdk.jmh.profile)
	|	|	|- LinuxPerfAsmProfiler (org.openjdk.jmh.profile)
	|	|	'- WinPerfAsmProfiler (org.openjdk.jmh.profile)
	|	|- AsyncProfiler (org.openjdk.jmh.profile)
	|	|- JavaFlightRecorderProfiler (org.openjdk.jmh.profile)
	|	|- LinuxPerfC2CProfiler (org.openjdk.jmh.profile)
	|	|- LinuxPerfNormProfiler (org.openjdk.jmh.profile)
	|	|- LinuxPerfProfiler (org.openjdk.jmh.profile)
	|	'- SafepointsProfiler (org.openjdk.jmh.profile)
	'- InternalProfiler (org.openjdk.jmh.profile)
		|- AsyncProfiler (org.openjdk.jmh.profile)
		|- ClassloaderProfiler (org.openjdk.jmh.profile)
		|- CompilerProfiler (org.openjdk.jmh.profile)
		|- GCProfiler (org.openjdk.jmh.profile)
		|- JavaFlightRecorderProfiler (org.openjdk.jmh.profile)
		|- PausesProfiler (org.openjdk.jmh.profile)
		'- StackProfiler (org.openjdk.jmh.profile)
```

One I've used is `GCProfiler`. 
The profiler calculates memory allocations and garbage collections during benchmark execution.
Statistics it shows, I'm interested in, is `gc.alloc.rate (MB/sec)` and `gc.alloc.rate.norm (B/op)` 
`gc.alloc.rate` shows how much memory the benchmark occupies during the trial, but `gc.alloc.rate.norm` is about how much memory was allocated during single benchmark execution.

Having investigation results in mind, I may assume that less memory the application allocated, less GC pressure and less data application is storing in cache. 

Warning: the assumption is fair only for the issue above because the investigation showed application is duplicating data it may never use.

So I've ended up with the following plan:
- write benchmarks for suspicious methods 
- run benchmarks with GCProfiler and get some numbers
- improve code or fix the issue
- run the benchmark again 
- analyze numbers
- repeat until successful 

``` java
public class TypeEnumerationAllocationBenchmark {  
  
    @Benchmark  
    public Object newObject(TypedEnumerationBenchmarkExecution execution) {  
        return new BenchmarkTypeEnumeration(...);  
    }  
  
    @Benchmark  
    public Object getAvailableOptions(TypedEnumerationBenchmarkExecution execution) {  
        return new BenchmarkTypeEnumeration(...).getAvailableOptions(null);  
    }  
  
    @Benchmark  
    public Object getAllOptions(TypedEnumerationBenchmarkExecution execution) {  
        return new BenchmarkTypeEnumeration(...).getAllSortedOptions();  
    }  
  
    @Benchmark  
    public void preLoadEnumeration(TypedEnumerationBenchmarkExecution execution, Blackhole blackhole) {  
        BenchmarkTypeEnumeration enumeration = new BenchmarkTypeEnumeration(...);  
        blackhole.consume(enumeration.getAllSortedOptions());  
        for (String controlKey : execution.controlKeys) {  
            blackhole.consume(enumeration.getAvailableOptions(controlKey));  
        }  
    }  
}
```

In parallel, I would run the simplest performance benchmark for the nominal use cases to prevent dramatic performance degradation.

``` java
public class TypeEnumerationBenchmark {  
  
    @Benchmark  
    public Object getAvailableOptions(Execution execution) {  
        return execution.enumeration.getAvailableOptions(null);  
    }  
  
    @Benchmark  
    public Object getAllOptions(Execution execution) {  
        return execution.enumeration.getAllSortedOptions();  
    }  
  
    public static void main(String[] args) throws Exception {  
        Options opt = new OptionsBuilder()  
                .parent(new CommandLineOptions(args))  
                .include(".*TypeEnumerationBenchmark.*")  
                .build();  
  
        new Runner(opt).run();  
    }  
}
```


## How does it work

JMH works with an [Annotation Processing API](https://jcp.org/en/jsr/detail?id=269) and generates multiple classes for each benchmark method (method marked with `@Benchmark` annotation).

``` java
benchmark
	jmh_generated
		benchmark.jmh_generated.TypedEnumerationBenchmarkExecution_jmhType
		benchmark.jmh_generated.TypedEnumerationBenchmarkExecution_jmhType_B1
		benchmark.jmh_generated.TypedEnumerationBenchmarkExecution_jmhType_B2
		benchmark.jmh_generated.TypedEnumerationBenchmarkExecution_jmhType_B3
		benchmark.jmh_generated.TypeEnumerationAllocationBenchmark_getAllOptions_jmhTest
		benchmark.jmh_generated.TypeEnumerationAllocationBenchmark_getAvailableOptions_jmhTest
		benchmark.jmh_generated.TypeEnumerationAllocationBenchmark_jmhType
		benchmark.jmh_generated.TypeEnumerationAllocationBenchmark_jmhType_B1
		benchmark.jmh_generated.TypeEnumerationAllocationBenchmark_jmhType_B2
		benchmark.jmh_generated.TypeEnumerationAllocationBenchmark_jmhType_B3
		benchmark.jmh_generated.TypeEnumerationAllocationBenchmark_newObject_jmhTest
		benchmark.jmh_generated.TypeEnumerationAllocationBenchmark_preLoadEnumeration_jmhTest
		benchmark.jmh_generated.TypeEnumerationBenchmark_Execution_jmhType
		benchmark.jmh_generated.TypeEnumerationBenchmark_Execution_jmhType_B1
		benchmark.jmh_generated.TypeEnumerationBenchmark_Execution_jmhType_B2
		benchmark.jmh_generated.TypeEnumerationBenchmark_Execution_jmhType_B3
		benchmark.jmh_generated.TypeEnumerationBenchmark_getAllOptions_jmhTest
		benchmark.jmh_generated.TypeEnumerationBenchmark_getAvailableOptions_jmhTest
		benchmark.jmh_generated.TypeEnumerationBenchmark_jmhType
		benchmark.jmh_generated.TypeEnumerationBenchmark_jmhType_B1
		benchmark.jmh_generated.TypeEnumerationBenchmark_jmhType_B2
		benchmark.jmh_generated.TypeEnumerationBenchmark_jmhType_B3
	benchmark.TypeEnumerationAllocationBenchmark
	benchmark.TypeEnumerationBenchmark
```

So, once you run your benchmark, it will try to find the generated classes in a classpath and use them to run the actual method and collect results.

## How to run it

There are the few ways how to run it.

The [best one](https://github.com/openjdk/jmh#preferred-usage-command-line) with more precise results is to write the `main` method, build a .jar file, and run it.
``` java
public class Benchmark {

	....

	public static void main(String[] args) throws Exception {  
	    Options opt = new OptionsBuilder()  
	            .jvmArgsAppend("-Djmh.separateClasspathJAR=true")  
	            .parent(new CommandLineOptions(args))  
	            .include(".*Benchmark.*")  
	            .build();  
	  
	    new Runner(opt).run();
	}
}
```

You may also run the `main` method from IDE, but likely your IDE will add some agent into JVM, so it makes the result less precise.

If you don't need to have precise results but rather to see performance trends, you may run the test as a JUnit test.

``` java
public class Benchmark {

	....
	@Benchmark  
	public void benchmarkMethod(...) {
		...
	}

	@Test  
	public void shouldRunBenchmark() throws Exception {  
	    Options opt = new OptionsBuilder()  
	            .include(".*Benchmark.benchmarkMethod*")  
	            .build();  
	  
	    new Runner(opt).run();  
	}
}
```
I've chosen to run from IDE because I'm interesting in memory allocations (a small side effect from IDE agents) and I'm going to use it during bugfix (I don't want to switch between IDE and console).

Once you run it you will see something like

``` bash
# JMH version: 1.35
# VM version: JDK 11.0.15, OpenJDK 64-Bit Server VM, 11.0.15+9-LTS
# VM invoker: C:\workdir\bin\amazon-correto\jdk11.0.15_9\bin\java.exe
# VM options: -ea -Didea.test.cyclic.buffer.size=1048576 -javaagent:C:\Program Files\JetBrains\IntelliJ IDEA 2022.2.1\lib\idea_rt.jar=50905:C:\Program Files\JetBrains\IntelliJ IDEA 2022.2.1\bin -Dfile.encoding=UTF-8 -Djmh.separateClasspathJAR=true
# Blackhole mode: full + dont-inline hint (auto-detected, use -Djmh.blackhole.autoDetect=false to disable)
# Warmup: 5 iterations, 10 s each
# Measurement: 5 iterations, 10 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Benchmark: benchmark.TypeEnumerationAllocationBenchmark.preLoadEnumeration
# Parameters: (controlKeysCount = 10, enumsCount = 10, prefixesCount = 1)

# Run progress: 0.00% complete, ETA 00:08:20
# Fork: 1 of 5
# Warmup Iteration   1: 100917.316 ops/s
# Warmup Iteration   2: 105528.728 ops/s
# Warmup Iteration   3: 105569.479 ops/s
# Warmup Iteration   4: 104947.364 ops/s
# Warmup Iteration   5: 105071.139 ops/s
Iteration   1: 104137.948 ops/s
                 ·gc.alloc.rate:               1900.765 MB/sec
                 ·gc.alloc.rate.norm:          20120.000 B/op
                 ·gc.churn.G1_Eden_Space:      1906.794 MB/sec
                 ·gc.churn.G1_Eden_Space.norm: 20183.824 B/op
                 ·gc.churn.G1_Old_Gen:         0.002 MB/sec
                 ·gc.churn.G1_Old_Gen.norm:    0.018 B/op
                 ·gc.count:                    33.000 counts
                 ·gc.time:                     22.000 ms

Iteration   2: 104582.692 ops/s
....

# Run complete. Total time: 00:08:22

REMEMBER: The numbers below are just data. To gain reusable insights, you need to follow up on
why the numbers are the way they are. Use profilers (see -prof, -lprof), design factorial
experiments, perform baseline and negative tests that provide experimental control, make sure
the benchmarking environment is safe on JVM/OS/HW level, ask for reviews from the domain experts.
Do not assume the numbers tell you what you want them to tell.

Benchmark                                                                            (controlKeysCount)  (enumsCount)  (prefixesCount)   Mode  Cnt        Score       Error   Units
TypeEnumerationAllocationBenchmark.preLoadEnumeration                                                10          1000                1  thrpt   10     104137.948 ±    31.242   ops/s
TypeEnumerationAllocationBenchmark.preLoadEnumeration:·gc.alloc.rate                                 10          1000                1  thrpt   10     1900.765 ±    48.356  MB/sec
TypeEnumerationAllocationBenchmark.preLoadEnumeration:·gc.alloc.rate.norm                            10          1000                1  thrpt   10  20120.000   ±    39.771    B/op
TypeEnumerationAllocationBenchmark.preLoadEnumeration:·gc.churn.G1_Eden_Space                        10          1000                1  thrpt   10     1906.794 ±    58.341  MB/sec
TypeEnumerationAllocationBenchmark.preLoadEnumeration:·gc.churn.G1_Eden_Space.norm                   10          1000                1  thrpt   10  20183.824   ± 40835.901    B/op
TypeEnumerationAllocationBenchmark.preLoadEnumeration:·gc.churn.G1_Old_Gen                           10          1000                1  thrpt   10        0.002 ±     0.027  MB/sec
TypeEnumerationAllocationBenchmark.preLoadEnumeration:·gc.churn.G1_Old_Gen.norm                      10          1000                1  thrpt   10       0.018 ±    0.012    B/op
TypeEnumerationAllocationBenchmark.preLoadEnumeration:·gc.count                                      10          1000                1  thrpt   10      33.000              counts
TypeEnumerationAllocationBenchmark.preLoadEnumeration:·gc.time                                       10          1000                1  thrpt   10      22.000                  ms
....
```
where:
- `Fork` - single benchmark execution. Every method executed several times. The number could be changed with `-f` option. Default value is `5`. `0` means no fork and could be useful during benchmark debugging.
- `Warmup Iteration` - some benchmark execution for warmup. Warmup results will not be used as benchmark result and requires mainly to eliminate some internal Java compilations and optimizations
- `Iteration #` - actual benchmark execution and intermediate results. 

More execution options you may find [here](https://github.com/guozheng/jmh-tutorial/blob/master/README.md)