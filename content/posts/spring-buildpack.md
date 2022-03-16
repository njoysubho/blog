---
title: "Spring Buildpack"
date: 2021-09-19T12:10:48+02:00
draft: true
---
Buildpacks are modular tool to create OCI compliant images for your application, without using Dockerfile. 
Before we see how Spring boot uses buildpack lets look into major components of buildpack and how they contribute in image building.
* Builder -> This is itself is an image that builds the image for your application.
* Buildpack -> A buildpack does the actual job of analysing your application code and contributes in image generation.
Each builder consists of multiple buildpacks. There are also some special buildpacks which are called Metadata buildpack, these are aggregation of different buildpacks. Any buildpack will have code for detect and build , detect script will enable the particular buildpack to participate in image building process and build script will actually have the code build images.
* Stack -> Finally there are stack, stacks provide base image to builders. Stacks can define a build image which will be used while building and run image which will be used at running the image.

Spring uses Packeto buildpack which is an implementation of CNCF buildpack spec, and has it integrated with spring boot maven plugin.

Let's see how to genarate a simple image from a spring boot application 

Below is a simple pom build config which will generate an image 

```xml
<build>
        <plugins>
                <plugin>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-maven-plugin</artifactId>
                <plugin>
        <plugins>
</build>
```
we run `mvn spring-boot:build-image` and thats it an image will be generated without any dockerfile being used.

As usual we will deep dive and try to uncover the magic inside.

When we run the mvn goal we see in console output like below

```
....
....
Pulling builder image 'docker.io/paketobuildpacks/builder:base' 100%
.....
[INFO]
[INFO]  > Running creator
[INFO]     [creator]     ===> DETECTING
[INFO]     [creator]     5 of 18 buildpacks participating
[INFO]     [creator]     paketo-buildpacks/ca-certificates   2.4.1
[INFO]     [creator]     paketo-buildpacks/bellsoft-liberica 8.6.0
[INFO]     [creator]     paketo-buildpacks/executable-jar    5.2.1
[INFO]     [creator]     paketo-buildpacks/dist-zip          4.2.1
[INFO]     [creator]     paketo-buildpacks/spring-boot       4.6.0
[INFO]     [creator]     ===> ANALYZING
[INFO]     [creator]     Restoring metadata for "paketo-buildpacks/spring-boot:helper" from app image
[INFO]     [creator]     Restoring metadata for "paketo-buildpacks/spring-boot:spring-cloud-bindings" from app image
[INFO]     [creator]     Restoring metadata for "paketo-buildpacks/spring-boot:web-application-type" from app image
[INFO]     [creator]     ===> RESTORING
[INFO]     [creator]     ===> BUILDING
[INFO]     [creator]
[INFO]     [creator]     Paketo CA Certificates Buildpack 2.4.1
[INFO]     [creator]       https://github.com/paketo-buildpacks/ca-certificates
[INFO]     [creator]       Launch Helper: Contributing to layer
[INFO]     [creator]         Creating /layers/paketo-buildpacks_ca-certificates/helper/exec.d/ca-certificates-helper
[INFO]     [creator]
[INFO]     [creator]     Paketo BellSoft Liberica Buildpack 8.6.0
[INFO]     [creator]       https://github.com/paketo-buildpacks/bellsoft-liberica
[INFO]     [creator]       Build Configuration:
[INFO]     [creator]         $BP_JVM_TYPE                 JRE             the JVM type - JDK or JRE
[INFO]     [creator]         $BP_JVM_VERSION              11.*            the Java version
[INFO]     [creator]       Launch Configuration:
[INFO]     [creator]         $BPL_HEAP_DUMP_PATH                          write heap dumps on error to this path
[INFO]     [creator]         $BPL_JVM_HEAD_ROOM           0               the headroom in memory calculation
[INFO]     [creator]         $BPL_JVM_LOADED_CLASS_COUNT  35% of classes  the number of loaded classes in memory calculation
[INFO]     [creator]         $BPL_JVM_THREAD_COUNT        250             the number of threads in memory calculation
[INFO]     [creator]         $JAVA_TOOL_OPTIONS                           the JVM launch flags
```
First notice an image `docker.io/paketobuildpacks/builder:base` is being pulled this is the builder component that we spoke earlier. 
Next we see a section called `==> Detecting` this is where each buildpack available in the builder will participate. We see the line `5 of 18 buildpacks participating` which means only 5 buildpacks' detect code has informed that they can participate in creating an image for spring-boot application. 

Here are the participating buildpacks

```
[INFO]     [creator]     paketo-buildpacks/ca-certificates   2.4.1
[INFO]     [creator]     paketo-buildpacks/bellsoft-liberica 8.6.0
[INFO]     [creator]     paketo-buildpacks/executable-jar    5.2.1
[INFO]     [creator]     paketo-buildpacks/dist-zip          4.2.1
[INFO]     [creator]     paketo-buildpacks/spring-boot       4.6.0
```
in the above `paketo-buildpacks/bellsoft-liberica` buildpack provides JDK/JRE and `paketo-buildpacks/spring-boot` actually provides spring boot dependencies and slices an application to multiple layers.
To see the full list of buildpacks available withing the builder see (https://github.com/paketo-buildpacks/base-builder/blob/main/builder.toml).

> If we notice in builder.toml, there are not 18 buildpacks mentioned, then why the above console log tells about 18 buildpacks participating? It is because some of the buildpacks are metadata buildpacks which contains more buildpacks. For example `paketo-buildpacks/java` which further contains `paketo-buildpacks/ca-certificates` , `paketo-buildpacks/bellsoft-liberica` etc.

By default packeto uses bellsoft-liberica java distribution and below are the default configuration that it uses.

```
Build Configuration:
[INFO]     [creator]         $BP_JVM_TYPE                 JRE             the JVM type - JDK or JRE
[INFO]     [creator]         $BP_JVM_VERSION              11.*            the Java version
[INFO]     [creator]       Launch Configuration:
[INFO]     [creator]         $BPL_HEAP_DUMP_PATH                          write heap dumps on error to this path
[INFO]     [creator]         $BPL_JVM_HEAD_ROOM           0               the headroom in memory calculation
[INFO]     [creator]         $BPL_JVM_LOADED_CLASS_COUNT  35% of classes  the number of loaded classes in memory calculation
[INFO]     [creator]         $BPL_JVM_THREAD_COUNT        250             the number of threads in memory calculation
[INFO]     [creator]         $JAVA_TOOL_OPTIONS                           the JVM launch flag
```
We will see how to modify this value shortly.

## Building Native image

Let's create an image containing a graal native image instead of jar. To achieve this we will do our first customization as below

```xml
<plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <configuration>
                <image>
                        <builder>paketobuildpacks/builder:tiny</builder>
                        <env>
                                <BP_NATIVE_IMAGE>true</BP_NATIVE_IMAGE>
                        </env>
                </image>
        </configuration>
</plugin>
```

Two important customizations here 

- We are using a different builder `paketobuildpacks/builder:tiny` this will help create a distroless image suitable for graal native image type artifacts.
- Second, we are passing an env variable `BP_NATIVE_IMAGE=true` which will instruct the builder to generate a graal native image.

If we see the console log now , we will see 

```
INFO]     [creator]     paketo-buildpacks/graalvm         
```
So instead of using `liberica-bellsoft` dist now builder will use graalvm dist.

Let's now change the Java version, may be we want to use jdk 17 . we can use below configuration in plugin 

```xml
<plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <configuration>
                <image>
                        <env>
                                <BP_JVM_VERSION>17</BP_JVM_VERSION>
                        </env>
                </image>
        </configuration>
</plugin>
```
We will notice in log it has fetched JDK 17 ` BellSoft Liberica JRE 17.0.0: Contributing to layer` .

We can customize the generated image name like below

```xml
<plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <configuration>
                <image>
                       <name>myimage:1.0.0</name>
                        <env>
                                <BP_JVM_VERSION>17</BP_JVM_VERSION>
                        </env>
                </image>
        </configuration>
</plugin>
```

For all the configuration see here https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-tools/spring-boot-maven-plugin/src/main/java/org/springframework/boot/maven/Image.java 

## How to change JDK 
We may want to use a different JDK than bellsoft, in order to do that there must be a buildpack available for the choice of JDK . Let's say we want to use amazon corretto we need to configure as below 

```xml
<plugin>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-maven-plugin</artifactId>
<configuration>
        <image>
                <buildpacks>
                        <buildpack>gcr.io/paketo-buildpacks/amazon-corretto:latest</buildpack>
                        <buildpack>gcr.io/paketo-buildpacks/executable-jar:latest</buildpack>
                        <buildpack>gcr.io/paketo-buildpacks/spring-boot:latest</buildpack>
                </buildpacks>
                <env>
                        <BP_JVM_VERSION>17</BP_JVM_VERSION>
                </env>
        </image>
</configuration>
</plugin>
```
> Note- It turns out when we override default buildpacks we need to provide all the buildpacks which are required. Here we only wanted to use corretto however we still need to mention spring-boot and executable-jar buildpack. This is a bit counter intuitive to me.

> Another quirk is if you do not provide the tag like `latest` it will not work. Being a docker user I initially thought without mentioning the tag will take latest automatically, however that is not the case.

So what we see when we run build -

```
[INFO]     [creator]       Corretto JDK 17.0.0: Contributing to layer
```

## JVM configuration

In order to provide memory calculation buildpack uses some defaults 

```
[INFO]     [creator]         $BPL_JVM_HEAD_ROOM           0               the headroom in memory calculation
[INFO]     [creator]         $BPL_JVM_LOADED_CLASS_COUNT  35% of classes  the number of loaded classes in memory calculation
[INFO]     [creator]         $BPL_JVM_THREAD_COUNT        250             the number of threads in memory calculation
```
There's a memory calculator https://github.com/paketo-buildpacks/libjvm/blob/main/calc/calculator.go#L29 which uses these variables to calculate heap,metaspace total jvm memory requirement.

## Buildpack vs Dockerfile

Buildpacks are opinionated and like any opinionated tech it gives us some defaults which helps to build image quickly without another piece of infrastructure code to maintain. We can get performant, secure images automatically without us need to write anything. On the other side we lose flexibility and this is where Dockerfile shines. We have more freedom to chose our base images, run any command etc. But this flexibility means we are responsible for its correctnes, performance, optimization etc.

Looking into the pros and cons we can say,if our application is trivial and not much customization is required buildpack can be a good option over using Dockerfiles.

