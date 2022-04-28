---
title: "Spring Dockerize"
date: 2022-04-16T10:22:24+02:00
draft: false
---

In my previous post we saw how to create a simple spring boot application which actually does nothing much. I also touched about how to package that as jar file and run locally. However currently in production environment docker is hugely adopted. In this post we will see how to dockerize our spring boot application. 

## First Step, just works

```docker
FROM ubuntu:18.04
ARG MAVEN_VERSION=3.8.5
ARG BASE_URL=https://downloads.apache.org/maven/maven-3/${MAVEN_VERSION}/binaries
# Install Java
RUN apt-get update \
    && apt-get install -y curl\
    && apt-get install -y openjdk-17-jdk ca-certificates-java\
    && apt-get clean \
    && update-ca-certificates -f
ENV JAVA_HOME /usr/lib/jvm/java-17-openjdk-amd64
RUN export JAVA_HOME
# Install Maven
RUN  curl -OLs ${BASE_URL}/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
 && tar -xzf apache-maven-${MAVEN_VERSION}-bin.tar.gz -C /usr/local/ \
 && ln -s /usr/local/apache-maven-${MAVEN_VERSION}/bin/mvn /usr/bin/mvn \
 && rm -f /tmp/apache-maven-${MAVEN_VERSION}-bin.tar.gz

COPY . .

RUN mvn clean package
COPY target/*.jar app.jar
CMD ["java","-jar","app.jar"]
```
We create a new image by running `docker build -t spring-first-web-app:1.0.0` and then we run our application by running `docker run  spring-first-web-app:1.0.0`.
So far so good. We see application starting up.
![docker1](/static/spring-first-web-app-docker1.png)

## First Improvement

While the above image works just fine, we can see some issues. First one is of size.
If we run `docker images spring-first-web-app:1.0.0` we see that the image is about 1 GB.
![docker1-size](/static/spring-first-web-app-docker1-size.png)
This is huge for a an application which has basically nothing.
First thing we can do to move to a smaller base image. 

```docker
FROM openjdk:17
ARG MAVEN_VERSION=3.8.5
ARG BASE_URL=https://downloads.apache.org/maven/maven-3/${MAVEN_VERSION}/binaries

RUN  curl -OLs ${BASE_URL}/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
 && tar -xzf apache-maven-${MAVEN_VERSION}-bin.tar.gz -C /usr/local/ \
 && ln -s /usr/local/apache-maven-${MAVEN_VERSION}/bin/mvn /usr/bin/mvn \
 && rm -f /tmp/apache-maven-${MAVEN_VERSION}-bin.tar.gz

COPY . .

RUN mvn clean package
COPY target/*.jar app.jar
CMD ["java","-jar","app.jar"]
```
We moved to a new base image. OpenJDK which is the open source branch of Java provides many base images. Using these images has its advantage that we do not need to manually install Java plus these images time to time get security patches and updates. So one task is off our list.

Now when we build our image we can see that the size is reduced to about 600MB

![docker2](/static/spring-first-web-app-docker2.png)
See the image tagged 2.0.0 .

```info
There is also another base image available which is based on alpine linux. Generally Alpine based images are even smaller but as per OpenJDK alpine based images are only supported in EA versions. More see https://hub.docker.com/_/openjdk .
```

## Second Improvement
Can we do better ? If we follow carefully we have two stage , in first stage we download maven and compile our java source code. In second stage we run our packaged application. Once we have our packaged application we do not need anymore mvn and JDK . All we need is a JRE. 
Here comes the idea of multistage docker build. In a multistage docker build we can pick and chose artifacts from previous stages and throw away anything from all previous stages.
Below is how we can do this.

```docker
FROM eclipse-temurin:17 as build
ARG MAVEN_VERSION=3.8.5
ARG BASE_URL=https://downloads.apache.org/maven/maven-3/${MAVEN_VERSION}/binaries

RUN  curl -OLs ${BASE_URL}/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
 && tar -xzf apache-maven-${MAVEN_VERSION}-bin.tar.gz -C /usr/local/ \
 && ln -s /usr/local/apache-maven-${MAVEN_VERSION}/bin/mvn /usr/bin/mvn \
 && rm -f /tmp/apache-maven-${MAVEN_VERSION}-bin.tar.gz

COPY . .

RUN mvn clean package
COPY target/*.jar app.jar

FROM eclipse-temurin:17-jre-alpine as production
COPY --from=build app.jar .
CMD ["java","-jar","app.jar"]
```
In the above docker file we have two stages. First stage `build` is build stage. In build stage we download maven and compile our java source code. In second stage `production` we run our packaged application. Learn more about multistage docker build from https://docs.docker.com/develop/develop-images/multistage-build/ .

Let's see how much is the docker size now - 

![multistage](/static/spring-first-web-app-multi-stage.png)

We see a huge reduction in size. Point to note here is I have moved to a different base image. Reason is , for production stage I want a JRE only image. However for newer version of Java the upsteam OpenJDK project didn't produce anymore JREs hence there is no JRE only image. There are a lot of discussion in github about this. For example see https://github.com/docker-library/openjdk/issues/468 . 

There is also another provider Adoptium (previously known as AdoptopenJDK) which under their version of Java called Temurin still provides a JRE image. You can find a nice article about their decision https://blog.adoptium.net/2021/12/eclipse-temurin-jres-are-back/ . Hence I moved to using temurin. 

## Are there still room for improvements ?

Indeed! though not much about size(Well you can try some distroless image).But I would like to focus on below two topic - 

* Spring Layered image 
* Spring Buildpack .

Spring buildpack provides support for creating image for spring application without needing to write Dockerfile I have alteady written a post about this https://dev.to/sabyasachi/buildpack-with-spring-boot-300o . This also touched a little about layering. 

In a separate posts I will explain how to create a spring layered image. 

So, that's it from this post. We now know how to create a bare minimum spring application and how to create an image. 