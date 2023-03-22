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
After running it, I found strange plateaus on a flame graph that were missing from the Java 11 profiling report.

![Plateaus](assets/img/posts/2023-02-15-plateau.png)


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

The method doesn't perform any long operations. It just creates an instance of an input stream when needed and uses it later.
But we call the method for each `read` in the external input stream. In other words, every time we want to read the next piece of data.
And the method is `synchronised` to guarantee a single instance of input stream creation in a multi-threaded environment.
And there is no such plateau in Java 11 profiling. I've run and checked it several times on different machines. 

After some googling, I found this [JEP] (https://openjdk.org/jeps/374), which says that bias locks are disabled as of Java 15.


# What is bias lock

Bias locking is an optimisation for `synchronized` in Java.
It works when the method is not so concurrent, and usually only a single thread acquires a lock.
The JVM raises a flag in the monitor object when a thread acquires the lock, so that re-acquiring and releasing the lock by the same thread is easy. But the JVM will revoke the bias lock if another thread tries to acquire it. And revoking is a costly operation.

So the community decided to deprecate bias locks and remove them later.

After all, if you have a really multithreaded application, you want to avoid costly bias lock revocation, and if your application isn't that multithreaded, you don't need the lock at all.

# How to fix it

First, let's run the tests with `-XX:+UseBiasedLocking `JVM option to turn on the bias locks.
The regression issue disappears, but the warning appears:
```
OpenJDK 64-Bit Server VM warning: Option UseBiasedLocking was deprecated in version 15.0 and will likely be removed in a future release.
```

Ok, so let's make our lazy initialisation smarter to avoid getting the lock every time and use the old-fashioned but still working [double-checked locking](https://en.wikipedia.org/wiki/Double-checked_locking).
I found it implemented by Suppliers.memoize in the [guava](https://github.com/google/guava) library.

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

The memoising supplier uses a 2-field variant of the double-checked lock, but this is because the supplier can return `null' as a valid value.

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

When I ran the tests again, I found that the regression had not completely disappeared, so it looks like we have similar locks, but they are not too hot to be visible on the flame graphs. 
In the end, we decided to go with the bias locks turned on and keep working on our code to improve it continuously.

# Conclusion

- If you have something similar to ours and are using Java 15+, you may be able to get some performance gains just by turning on bias locks.
  But it is always better to rewrite problematic parts.
- Test the performance of your code in every version of Java. This will help you find regressions earlier and spend less time looking for strange plateaus.
