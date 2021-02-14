---
title: "The Weekend Journal- Week-06"
date: 2021-02-14T07:54:22+01:00
draft: false
---

**Resilience4j**

This work I deep dive into resilience4j which is an alternative to Hystrix.
Resilience4j is a lightweight fault tolerance library and a possible alternative to Hystrix.
Resilience4j comes with following core modules - 

- resilience4j-circuitbreaker: Circuit breaking

- resilience4j-ratelimiter

- resilience4j-bulkhead

- resilience4j-retry

- resilience4j-timelimiter

- resilience4j-cache

one can also stacked together combinations of different resiliency patterns. It also comes with spring-boot starter.

Features

- Pick and choose resiliency pattern.
- Annotation driven integration with FeignClients.
- spring boot starter, with integration for application properties.
- Integration with Feign by module `resilience4j-feign`

**Spring Cloud CircuitBreaker**

Spring circuitbreaker provides circuit breaker pattern backed by below libraries

- Netflix Hystrix
- Resilience4j
- Sentinel
- Spring Retry

At the heart of it lies an interface called `CircuitBreaker` that as a mehtod `run` we have various implementation of this like `HystrixCircuitBreaker` `Resilience4jCircuitbreaker` etc.

Spring cloud circuitbreaker has integration with Feign clients via `Spring Cloud Open Feign`. However it till now does not have any properties driven support unlike resilience4j spring boot starter.

Resilience4j is a collection of implementation of resilience pattern however Spring Cloud Circuitbreaker has only CircuitBreaker implemenation with resilience4j. 
In my opinion at this moment it is better to go with resilience4j spring boot starter as one will be able to use all the other resilience4j pattern.

**Elements of Statistical Learning**

Started reading this book, it deals with the core concepts of ML and IMO a must read. Sometime the books become hard to follow with deep mathematical notations but otherwise it is pretty lucid. 






