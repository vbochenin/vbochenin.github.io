---
title: "How to change already written code: Aspects"
layout: post
author: vbochenin
tags:
  - java
  - aspects
  - annotations
categories:
  - how-to
---
## What an issue I'm trying to solve
Now I'm trying to solve some generic issue. We have a lot of idempotent methods, methods without side effects and returning the same result for the same arguments.
Additionally, we have some methods of reading data outside, but the data are rarely changed.

We may use, and we are using, caching to improve response time in this case. 
The issue is that all these methods are different, as well as the ways we are using caches (from local value holders and hash maps to EhCache).
So we would like to unify a caching approach to be able to configure different caches, their eviction, and capacity policies and ways how we store them.

Also, I wouldn't like to pollute the business code with extra supporting code.

Please compare without cache:

``` java
public Value get() {
	return readValue();
}

```

and with cache:
``` java
public Value get() {
	var value = cache.get("value");
	if (value != null) {
		return value;
	}

	value = readValue();
	cache.put("value", value);
	return value
}
```

## Changes I've made before 

First, I've unified caching interface and replaced it in already-known places:

``` java
public interface Cache<K, V> {
	V get(K key);
	void put(K key, V value);
}

public final class CacheManager {
	public static <K, V> Cache<K, V> getInstance(String region) {
		...
	}
}
```


Now code with caching looks even uglier:

``` java
public Value get() {
	var cache = CacheManager.getInstance("region"); 
	var value = cache.get("value");
	if (value != null) {
		return value;
	}

	value = readValue();
	cache.put("value", value);
	return value
}
```

Luckily, mankind invents AOP to solve this kind of task.


## Aspect oriented programming overview

AOP allows you to add actions (called `Advice`) to some points in an application (called `Join points`) specified by expressions (called `Pointcut`).
All this stuff together is called `Aspect`.

So in a few steps:

* You have a method you would like to improve. This is your `Join point`
  ``` java
	package com.example.apects;  
  
	public class Example {  
	  
	    public int calculateTwoPlusTwo() {  
	        return 4;  
	    }  
	}
  ```
- Write what you would like to do. It is your `Advice`
  ``` java
	public class ExampleAspect {  
	  
	    public int returnFive() {  
	        return 5;  
	    }  
	}

  ```
-  Specify where you would like to apply it. It is you `Pointcut`
  ``` java
    @Aspect  
	public class ExampleAspect {  
	  
	    @Pointcut("execution(public int com.example.apects.Example.calculateTwoPlusTwo())")  
	    public void returnFivePointcut() {  
	    }  
		...  
	}
  ```
-  Add bind all together. Now you have `Aspect`
  ``` java
	@Aspect  
	public class ExampleAspect {  
	  
	    @Pointcut("execution(public int com.example.apects.Example.calculateTwoPlusTwo())")  
	    public void returnFivePointcut() {  
	    }  
	  
	    @Around("returnFivePointcut()")  
	    public Object returnFive() {  
	        return 5;  
	    }  
	}
  ```

# Applying aspects for the caches problem

As you may see above, I need to know and specify all methods I would like to wrap with caching.
Bad news, there is no pattern in method names I could use to reduce the number of pointcuts.
The good news, Java has annotations and Aspects that may work with them.

So I can do something like:
- Create custom annotation
``` java
@Target({ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
public @interface Cacheable {  
    String value() default "common";  
}
```
- Write Aspect
  ``` java
@Aspect  
public class CachingAspect {  
  
    @Pointcut("@annotation(cacheable)")  
    public void cachingAnnotation(Cacheable cacheable) {}  
  
    @Around("cachingAnnotation(cacheable)")  
    public Object checkCache(ProceedingJoinPoint pjp, Cacheable cacheable) throws Throwable {  
        var cache = CacheManager.getInstance(cachable.value());  
        var value = cache.get("value");  
        if (value != null) {  
            return value;  
        }  
        value = pjp.proceed();  
        cache.put("value", value);  
        return value;  
    }  
}
  ```
- Put annotation to target method:

``` java
@Cacheble("region")
public Value get() {
	return readValue();
}
```

Looks good. I have caching separated from business code but:
1. I need to solve an issue with the cache key
	- Somehow the logic should depend on the method name and argument
2. I don't have any clue if the aspect was really applied to a method or not (just IDE highlights)

For first issue, I would use a full class and target method names and args as array. 
I would need to override `equals\hashCode` for arguments, and some arguments can be excluded. But I will care about it later.
``` java
@Aspect  
public class CachingAspect {  
  
	...
	  
	@Around("cachingAnnotation(cacheable)")  
	public Object checkCache(ProceedingJoinPoint pjp, Cacheable cacheable) throws Throwable {  
	    var cache = CacheManager.getInstance(cacheable.value());  
	    Signature signature = pjp.getSignature();  
	    if (!(signature instanceof MethodSignature)) {  
	        return pjp.proceed();
	    }  
	  
	    MethodSignature methodSignature = (MethodSignature)signature;  
	    Method method = methodSignature.getMethod();  
	    CacheKey key = new CacheKey(  
	            method.getDeclaringClass().getCanonicalName(),  
	            method.getName(),  
	            pjp.getArgs()  
	    );  
	  
	    var value = cache.get(key);  
	    if (value != null) {  
	        return value;  
	    }  
	    value = pjp.proceed();  
	  
	    cache.put(key, value);  
	    return value;  
	}  
	  
	public static class CacheKey {  
	    private final String className;  
	    private final String methodName;  
	    private final Object[] args;  
	  
	    public CacheKey(String className, String methodName, Object[] args) {  
	        this.className = className;  
	        this.methodName = methodName;  
	        this.args = args;  
	    }  
	  
	    @Override  
	    public boolean equals(Object o) {  
			...    
		}  
	  
	    @Override  
	    public int hashCode() {  
		    ...
	    }  
	}
}

```


About the second issue, I will use the AspectJ maven plugin for compile time weaving.
The plugin has a `showWeaveInfo` option to log all information about what and how is weaved.

``` xml

<build>  
    <plugins>
	    <plugin>
			<groupId>org.codehaus.mojo</groupId>  
            <artifactId>aspectj-maven-plugin</artifactId>  
            <version>1.14.0</version>  
            <configuration>
				<complianceLevel>11</complianceLevel>  
                <source>11</source>  
                <target>11</target>  
                <showWeaveInfo>true</showWeaveInfo>
            </configuration>
			<executions>
				<execution>
					<goals>
						<goal>compile</goal>  
					</goals>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
```

Once build is done, I may check logs

``` bash
[INFO] Join point 'method-call(com.example.apects.Example$Value com.example.apects.Example.getValue())'
       in Type 'com.example.apects.Example' (Example.java:7) advised by around advice from 
       'com.example.apects.CachingAspect' (CachingAspect.java:22)
[INFO] Join point 'method-execution(com.example.apects.Example$Value com.example.apects.Example.getValue())'
       in Type 'com.example.apects.Example' (Example.java:12) advised by around advice from 
       'com.example.apects.CachingAspect' (CachingAspect.java:22)
```

OK, here I may see that my join point was advised twice, so my code will also be called twice.
- the first time in method invocation point (`method-call`)
- the second time in method (`method-execution`)

So I would like to modify the pointcut to be used only in execution because I would like to cache only methods in my project.
If I cache some methods in a third-party library, I need to use a `method-call` advice to put cache-related code around method invocation.

So I will add `execution(* *.*(..))` into advice. You may read it like this: execution of any method in any class with any arguments and any modifier.

``` java
@Around(value = "execution(* *.*(..)) && cachingAnnotation(cacheable)", argNames = "pjp,cacheable")  
public Object checkCache(ProceedingJoinPoint pjp, Cacheable cacheable) throws Throwable {  
    ...
}
```

Rebuild and check logs:
``` bash
[INFO] Join point 'method-execution(com.example.apects.Example$Value com.example.apects.Example.getValue())' 
       in Type 'com.example.apects.Example' (Example.java:12) advised by around advice from 
       'com.example.apects.CachingAspect' (CachingAspect.java:22)
```


