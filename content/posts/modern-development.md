---
title: "Start Journey for a modern application development"
date: 2019-05-01T18:56:02+02:00
draft: false
---
The technology landscape for developers is changing rapidly. From monolith applications to microservices, from an application deployed on-premise
datacenter to cloud platform, also applications deployed on virtualized machines to the containerized environment, the shift is huge.
As a developer, it is important to keep up to date with this changing landscape both upgrading oneself as well as  looking for a new job.
As there are a plethora of technologies and there are always much hype and evangelism around a new technology it is always good to start
with a much stable language, framework, and tool.Below are some frameworks, platform, tools to start with.This is based on what
I have used in my day job ,because I have worked mostly in JAVA based frameworks, the article is more skewed towards JVM ecosystem .
This write up for some one who is new and try to find out what things that one need to learn to get started with new era of 
application development which will be deployed on cloud .
 
 Languages & Frameworks-

* JAVA - Though new languages are gaining popularity JAVA is still going strong. Start with JAVA 8  although JDK 12 has released. Understand and use language features like lambda, streams . With the new cadence of JAVA release we will see more features coming to language much faster so it is important to keep an eye on language level features.Java 8 is a good starting point as many frameworks listed below has Java8 as a baseline.     
* Spring - Learn about Spring framework mostly Dependency Injection, how to declare Beans and Autowire beans.This serves as a base.
* Spring Boot - Start with learning how to make a simple REST application in Spring Boot. Its auto-configuration and annotations make it easy to build applications . Industry adoption is very high for Spring boot so it also helps with your resume.
* Spring data  - Highly likely the application you will make will have some kind of persistent storage. Spring data is an umbrella project that provides libraries to talk with various data sources both RDBMS and NoSQL.
* Spring Cloud - Spring cloud is an umbrella project which has many libraries that provide many functionalities of today's distributed applications. It provides support for distributed configurations, messaging, fault tolerance, tracing to mention a few.
  With the above technologies, we can cover the base of an application that can then leverage the architecture pattern like microservice and can be run on a cloud platform both in VMs and in a containerized environment.




 Architecture-


* Microservices - There's increasing adoption of microservices also sometimes much-abused term. 
  It is not merely having multiple services talk to each other but a lot more, so it is good to learn and understand the pattern at its philosophy. 
  To do so I highly recommend going through the book Sam Newman's "Building Microservices"

 Tools-
 
* Git - Learn how to use git. Git is highly adopted in a development environment.Git official documentation is a good point to start.
        Initially, some find it confusing with some usage it will be clear how Git works.
* Github - Github is a provider of git repositories. Use Github upload your projects there , this not only helps you playing with 
           Git but also over time will create a portfolio of the projects you have created. 
		   This can earn you some brownie points in your next job hunt.
* Jenkins - Don't need to go very deep just understand how a build works there how can a job be configured to build artifacts.
* Maven - Maven is a dependency management and build tool, learn to use Maven in your projects. 
          There are other tools gaining popularity like Gradle but maven is a good point to start.


With the above tools try to understand how some code files from your version management system finally get build and deployed 


 Packaging - 
 
 * Docker - Modern applications are adopting a containerized deployment approach. 
           Learn about Docker and how to package an application in docker and run in a local docker environment. 
		   I recommend docker also to try out new technology, for example, you want to try out some database lets say Postgres
		   you can simply run a docker image of Postgres rather than installing a Postgres to your local machine. 
		   I highly recommend whenever you create an application think about how you package it in docker and run as a docker image,
		   to understand not only writing code but also how you will deploy that.




 Platform-
 
 * AWS - It is a leading cloud platform. It provides a vast range of services.Learning EC2, RDS is a good point to start. 
        Using these two a simple application with database calls can be deployed and then can learn about its other services from there.
		AWS provides a free tier for 12 months which has enough resources to use from learning purpose.


 Resources to keep up with technology news -
 
 Below some resources to keep you updated about latest news in technologies -

 *  Safari books online - it has huge collection of books .Some organizations provide subscription of safari to their employees.
                         If your company does not provide one it is not a bad investment to subscribe for one yourself.      
 * Infoq -  https://www.infoq.com/ .
 * Dzone -  https://dzone.com/ - Check out section as Guides and Refzcard .
 * Java Magazine Oracle - Bimonthly magazine with latest development in Java World
 * Twitter - is a good source all projects now a days has their twitter handle and tweet about their latest news.
 * LinkedIn 
 * Podcast - Very recently I have started to listen to podcasts.Java Pub house, Google Kubernetes Podcast are two that I like  .


It is not possible to use every new technolgy in your day job , you have a business value to deliver there ,
so it is important to set aside some dedicated time for your own tinkering with technology at home and pitch for same if any opportunity you found to use it in your workplace.


Hope this write up will help to get started .Happy coding !!!

