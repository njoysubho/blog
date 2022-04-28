---
title: "Spring First Step"
date: 2022-04-10T10:16:33+02:00
draft: false
---

## Introduction

I have been using Spring framework ecosystem since 2012. Overtime many new modules has been added and the adoption of the framework has increased. As of today it is I can say most popular framework in the industry. But one problem remains for new users of Spring that because Spring does a lot of work under-the-hood, it is sometimes feels like magic and generally production systems differ a lot from your getting started guide it is even more important to not only rely on what is coming out of the box but also to understand how it works. I will be sharing a series of blog post aiming to start with very basics and then move to more advanced topics but focusing with underlying details. I hope this series will help beginners o tip their toes in this beautiful framework. 

In this blog we will see how to create your first Spring Boot application. Spring Boot is another offering in Spring framework universe which makes it easy to create a Spring application. It has been highly adopted in todays microservices world.

Spring Boot applications are simple java application with some spring specific dependencies. You can use your own dependency management like maven or gradle to create your application. In this blog I will be using Maven as dependency manager.

## First Step 

So how to start. There are a few options -
* You can use mvn archetype to create a new spring boot application. It can be a simple maven based project.

* Another option and which I use very often is to visit the site `start.spring.io` and create a project from there . Below is a screenshot of how it looks 

[![start.spring.io](/static/start-spring-io.png)](https://start.spring.io/)

As you see in the screenshot you can see there are various options . On the left you can use Maven or Gradle as build tool.

You can also chose Java or kotlin or Groovy as you application language. Spring Boot has currently two main version branch 2.x and upcoming  3.x version. You can choose the version you want to use. I will stick with 2.x version. 

After that you can provide your application name and other details . 

It offers to package your application as Jar or War. As of now we will chose Jar . More on how to package your application later.

And finally the Java version, you can chose upto Java 18. For this blog I will stick with Java 17 . 

Now lets look at the right side there is something called dependencies. Here is what the first magic happens with Spring Boot applications. As I mentioned earlier, Spring Boot applications are based on spring framework. So there are a few dependencies that are required to run your application. We can either manually provide those dependencies or we can use something called `spring-boot-startes` these are curated set of dependencies focusing to add some specific feature to your applications.

The first dependency that we will ue to create a web application is called 
`spring-boot-starter-web`. This will give you an embedded tomcat server and enable you to run your application as a web application. After adding it we click on Generate. This will download a zip file with a maven project.

## First Application 

The generated project structure looks like below 

![generated project structure](/static/spring-start-project-structure.png)

A typical maven project , the pom.xml looks like below 

![pom.xml](/static/spring-start-pom.png)]

First, we see we have two depedencies, `spring-boot-starter-web` and `spring-boot-starter-test`. The later one is used for unit testing and fetches junit5. 

Important thing to notice here apart from the dependencies is there is a parent pom which is inherited. The parent pom is `spring-boot-starter-parent`. This parent pom contains all build plugins and due to which when we do a simle mvn package command we get a runnable jar. It contains many configuration regarding configuration properties and other stuff. 

This parent starter in turn inherits from `spring-boot-dependencies` which contains compatible curated set of dependecies for spring boot. 

In your favourite editor you can go inside each of these parent poms and see what they actually fetch, but it is not absolutely necessary to understand each of them as most of the time you need not to configure them.

So now we have a spring boot project we saw what are the dependencies we have. Let's see how to run our very new application and what is the output. 

There are many ways to run your application - 

* Running the jar file. Spring boot applications produce a runnable uber jar with all dependencies inside it, which can be run directly. 

* Using IDE , every Spring boot application has a main method and we can just run the main method from IDE . 

* You can also dockerize your application and run the docker instant. More of it later. But spring has support for something called buildpack which will produce an image even without a Dockerfile. If you are interested you can read my another blog post about it here https://dev.to/sabyasachi/buildpack-with-spring-boot-300o .

So lets first see running the uber jar . First we package the application using `mvn package`. Then simply run `java -jar <your-jar-file>`. For my case the output looks like below 

![output](/static/spring-start-run.png)

From the output we can see it has started a tomcat server and it is listening on port 8080. This is the default settings. We can also use Jetty server instead of tomcat server. Also we can change the default port to some other port. 

## What's Inside Your Jar

We can extract the created jar file to see what is inside. To extract or unzip the jar file we can use command `jar -xvf <your-jar-file>`. The output looks like below 

![extracted jar](/static/start-spring-uber-jar.png). 

It is important to look inside the jar because a lot of worthy optimisation can be done which we will see in later that can contribute in faster builds and startup.

Now at this point we have a running application. We know how to run it locally. Let's see some quick configuration. 

## Changing the default port

The default port of 8080 is fine may be if we are running a single application, but what if we wanted to run another application ? In a typical production environment a single physical/virtual server may host multiple applications.
We can use `server.port` property in `application.properties` to change the port.

## Using Jetty instead of tomcat 

Tomcat comes out of the box however we can use Jetty as embedded server. To do that we need to use below pom config.

```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
			<exclusions>
				<exclusion>
					<groupId>org.springframework.boot</groupId>
					<artifactId>spring-boot-starter-tomcat</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jetty</artifactId>
		</dependency>
```
For details we can see https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.webserver .

So that's it from this post. We have just taken a tiny step to create a simple spring boot application. In the next post we will deep dive on how to package our application and how to deploy it.