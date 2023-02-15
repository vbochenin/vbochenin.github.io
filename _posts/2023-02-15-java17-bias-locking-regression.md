---
title: "Java 17 migration: bias locks regression"
layout: post
author: vbochenin
tags:
  - java
  - performance
categories:
  - dev-journal
---
# What issue do I need to solve

We are about to switch from java 11 to java 17 finally, but a few performance tests failed and failed dramatically with around 50% of regression.
It was 6 seconds in java 11 vs. 9 seconds in java 17 in absolute number in some tests.
And this happens just by switching runtime.

So I've got to investigate why the tests failed and fix the issue.

# What I did to get some data

First, I've decided to run the test with a profiler. I'm using [Intellij IDEA Profiler](https://www.jetbrains.com/help/idea/profiler-intro.html) with the default settings for the smoke test.
Once I ran it, I found strange plateaus on a flame graph that was missing in the java 11 profiling report.
![[2023-02-15-plateau.png]]

![Plateaus](assets/img/posts/2023-02-15-java17-bias-locking-regression/plateau.png)


So I've decided to look closer into the method.

# Digging deeper

The method looks like this:

```
public class LazyInputStream extends InputStream{
    private InputStream is;
    ...
    
    protected synchronized InputStream getInstance() {
	   if (is == null) {
		   is = factory.create();
	   }
	   return is;
    }
    ...
	public int read() throws IOException {  
	    return getInstance().read();  
	}
} 
```

The method doesn't do any long operation. It only creates an instance of an input stream when required and uses it later.
But we are calling the method for each `read` in the external input stream. In other words, each time when we are going to read the next portion of data.
And the method is `synchronized` to guarantee a single instance of input stream creation in a multithreading environment.
And there is no this kind of plateau in java 11 profiling. I've rerun it and checked several times on different machines. 

After some googling, I've found this [JEP](https://openjdk.org/jeps/374), which says that bias locks are disabled from java 15.


# What is bias lock

Bias lock is optimization for `synchronized` in java.
It works when the method is not so concurrent, and only a single thread usually acquires a lock.
JVM raises a flag in the monitor object that some thread acquires the lock, so reacquiring and releasing the lock by the same thread is lightweight. But the lock must be revoked when another thread tries to acquire the bias lock. And the revocation is a costly operation.

So community decided to deprecate bias locks and remove them later.

Because if you really have a multithreaded application, you would like to avoid costly bias lock revocation, and if your application is not so multithreaded, you don't need the lock at all.

# How to fix it

First, let's run the tests with `-XX:+UseBiasedLocking `JVM option to turn on the bias locks.
The regression issue disappears, but the warning appears:
```
OpenJDK 64-Bit Server VM warning: Option UseBiasedLocking was deprecated in version 15.0 and will likely be removed in a future release.
```

Ok, so let's implement our lazy initialization more smartly to avoid acquiring the lock every time and use old fashion but still working [double-checked locking](https://en.wikipedia.org/wiki/Double-checked_locking).
I've found it implemented by `Suppliers.memoize` in [guava](https://github.com/google/guava) library. 

```
public class LazyInputStream extends InputStream {  

    private final Supplier<InputStream> initializer;
    
    public LazyInputStream(InputStreamFactory factory) {  
        this.initializer = Suppliers.memoize(() -> {  
            try {  
                return factory.create();  
            } catch (IOException e) {  
                throw new IllegalStateException("Failed to create input stream", e);  
            }  
        });  
    }
    
    ...
    
    protected synchronized InputStream getInstance() {
	   return initialized.get();
    }
    ...
	public int read() throws IOException {  
	    return getInstance().read();  
	}
```

The memoizing supplier uses a 2-fields variant of double-checked locking, but it is because the supplier may return `null` as a valid value.
```
public T get() {  
  if (!initialized) {          // boolean flag  
    synchronized (this) {  
      if (!initialized) {      // double check to avoid races
        T t = delegate.get();  // calling delegate to get a value
        value = t;             // and remember it
        initialized = true;    // rise the boolean flag
        return t;  
      }  
    }  
  }  
  return value;  
}
```

Once I rerun the tests, I found that regression did not disappear completely, so it looks like we have similar locks, but they are not too hot to be visible on flame graphs. 
Finally, we decided to go with turned-on bias locks and continue working on our code to improve it continuously.

# Conclusion

- You may get some performance boost only by turning on bias locks if you have something similar to us and using java 15+.
  But it is always better to rewrite problem pieces.
- Test your code performance in every java version. It will help you find regression earlier and only spend a little time looking into strange plateaus.