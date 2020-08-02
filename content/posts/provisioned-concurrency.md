---
title: "Provisioned Concurrency"
date: 2020-08-02T11:09:40+02:00
draft: false
---

AWS Lambda is the serverless offering from amazon. In serverless the provider is responsible for settung up a runtime as and when needed for the deployed application/function to run. Typically these are small functions which are quick to start up and finish their task. Lambdas when invoked, has a small startup overhead. This startup overhead involves fetching the function artifact from S3 , setting up networking, starting up the application. Java applications often suffer from this cold start up problem , and depends on application this can go up quite a bit. So the very first call to lambda will suffer from latency. AWS lambda keep the context of lambda from last run typically apprix 10 min to serve any new request, so any subsequent requestmay not experience this latency. However caching the context is not a feature that the application should depend upon. Also in case of scaling up lambda to serve concurrent request, the same cold start issue can be seen if the request is served by the just spawned lambda. This is an issue which causes application developers to look for approaches to keep their lambda warm and take advantage of various approaches such as use a cloud watch event to trigger the lambda every 5-10 min , or going to a different language which provides relatively less cold start overhead than Java. 

Another approach is configuring provisioned concurrency option in AWS Lambda. It provison, initialize and make container runtime hyper ready to serve requests. One can define how many lambda container runtime , one want to be ready. Even it can be configured to provision container runtime for a time interval so that application can be able to handle burst request in a time of day.

Provisoned concurrency can only be configured on versioned lambdas. So lambda needs to published and either the version number or the alias name can be provided . It is not possible to use $LATEST alias to configure provisoned concurrency.

It also has an extra cost associated to keep those lambda runtime ready. For a details pricing please see [here](https://aws.amazon.com/lambda/pricing/) .

Provisoned concurrency is worth to try, this week I try it out in on of my application, although it does speed up cold start up but should not be looked upon by a silver bullet. Also as the number of concurrent request grows it's performance suffer. So in short one can expect a little improvement but it will definitely not elimanating complete cold start issue.