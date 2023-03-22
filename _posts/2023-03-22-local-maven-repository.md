---
title: "Using filesystem as artifact's repository"
layout: post
author: vbochenin
tags:
  - java
  - maven
  - gradle
categories:
  - dev-journal
---

## What a task I want to solve

We've got a library from another company, but we're only allowed to use it in certain projects ~~for reasons known only to the manager~~.
Basically, we don't even have the option to publish it to the artifact's repository.
Also, some of our projects use Maven; others use Gradle to build. 

As a result, we need to have an unified way to store the libraries along with our source code.

## Using `libs` folder

The easiest way is to copy the file close to the sources and push it into the Git repository.
### Step by step:
- copy a library into some folder (usually called `libs`)
    ```
    libs
        |- some-library-123.jar
    ...
    src/main
    ...
    pom.xml
    ```
- add dependecy into project:
    - for Maven: specify the dependency with `system` scope in `pom.xml`:
        ```xml
        <dependency>
            <groupId>thirdparty</groupId>
            <artifactId>some-library</artifactId>
            <version>123</version>
            <scope>system</scope>
            <systemPath>${basedir}/libs/some-library-123.jar</systemPath>
        </dependency>
        ```
    - for Gradle: use `fileTree` to specify the dependecy in `build.gradle`:
        ```groovy
        implementation fileTree(dir: 'libs', include: [ 'some-library-123.jar', ...])
        ```

So it works, but the `system' scope is deprecated, and I would like something less verbose and more consistent with other dependencies.

## Using Maven folders structure

Maven caches all dependencies in a local repository. The repository has a specific subfolders structure:
```
.m2/repository
  |- groupId
      |- artifactId
          |- version
              |- .jar file
              |- .pom file
              |- ... some other files like sha1 ...
```

But nobody said that the folder must always be in `$USER_HOME/.m2/repository`. We can specify a local repository with a filesystem URL and use the library as a regular dependency, such as:
```xml
<repository>  
  <id>libs</id>  
  <url>file://${project.basedir}/libs</url>  
</repository>
```

### Step by step:
- create a folder `libs`
- copy the library into folder with command
  ```bash
  mvn install:install-file -Dfile=library-name-0.1.jar -DgroupId=com.thirdparty-company-name -DartifactId=library-name -Dversion=0.1 -Dpackaging=jar -DlocalRepositoryPath=libs
  ```
- specify the repository pointing into the folder
    - For Maven:
      ```xml
         <repositories>
            <repository>
                <id>local-libs</id>  
                <url>file://${project.basedir}/libs</url>  
            </repository>
            ... 
        </repositories>
      ```
    - For Gradle:
      ``` groovy
          repositories {
              maven { url "file://$projectDir/libs" }  
          }
      ```
- add dependecy into project:      
    - for Maven: 
        ```xml
        <dependency>
        <groupId>com.thirdparty-company-name</groupId>
        <artifactId>library-name</artifactId>
        <version>0.1</version>
        </dependency>
        ```
    
    - for Gradle: 
        ```groovy
        implementation "com.thirdparty-company-name:library-name:0.1"
        ```

So, with some extra work in the begining, we've got:
- cleaner dependency specification;
- no coupling between the build script and the filesystem.