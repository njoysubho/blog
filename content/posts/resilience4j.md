---
title: "Resilience4j, the path beyond Hystrix"
date: 2021-03-08T08:39:44+01:00
draft: true
---


**Introduction**

In discussions of cloud-native applications or microservices, the theme that takes a center stage is Resiliency. Hystrix from Netflix OSS has been the go-to library for developers to implement resiliency patterns, in many cases, it's integration with other application frameworks like Spring, Micronaut has made the developer life easier. However, Hystrix has been put into maintenance mode and Spring cloud 2020.0.0 (aka Ilford) has stopped its support for Hystrix. This is a huge change, what's the migration path ahead, what is the alternative? A quick look into the Hystrix GitHub repo readme gives us a hint, I paste an excerpt from that.

```
Hystrix is no longer in active development and is currently
 in maintenance mode.

  Hystrix (at version 1.5.18) is stable enough to meet the needs
  of Netflix for our existing applications. Meanwhile, our focus
  has shifted towards more adaptive implementations that react to 
  an application’s real-time performance rather than pre-configured
  settings (for example, through adaptive concurrency limits). 
  For the cases where something like Hystrix makes sense,
  we intend to continue using Hystrix for existing applications and 
  to leverage open and active projects 
  like resilience4j for new internal projects. We are beginning to recommend others do the same.
```
I will talk about Resilience4j in the post. However this is of course not the only alternative, other alternatives include `Spring Cloud Circuitbreaker`, `Sentinel`, `Spring Retry` or if you want to move the whole phenomenon of resiliency from application's responsibility to infrastructure then we can look at service meshes (though meshes are a different story altogether and they are much bigger than only resiliency).

**What is Resilience4j?**

Resilience4j is a Java library that implements various resiliency patterns. Below are the resiliency patterns that it supports

- Circuit Breaker
- TimeLimiter
- Threadpool Bulkhead
- Semaphore Bulkhead
- Retry
- RateLimiter

Resilience4j is designed as modular, each of the above patterns resides as a different library so as a developer we can pick and chose only the libraries that we need. 

Not just implementing resiliency pattern but Resilience4j also provide below capabilities

-  Spring Boot integration via a starter.
-  Micronaut integration
-  Kotlin integration
-  Feign integration via resilience4j-feign module 
-  Micrometer and Grafana metrics integration
-  Programmatic as well as Annotation driven way of defining resiliency patterns

Again due to its modular nature we can pick and choose the capabilities that we require.

To show how it works I have created a demo application that consists of two services, below is the setup used 

**Setup**

- r4j-serviceA is the caller service, it is a spring boot service, with the `resilience4j-spring-boot2` starter present as a dependency. It uses Feign client to call other services and uses annotations to define what resilience pattern to apply. This service also has micrometer in the classpath to generate metrics.

- r4j-serviceB exposes some endpoints which will be called by r4j-serviceA, to observe the behaviour of different resiliency pattern.

- To generate concurrent calls and repetitive calls I have used ab (apache benchmark tool https://httpd.apache.org/docs/2.4/programs/ab.html)

- A Prometheus has been set up to scrape metrics endpoint and two plot graphs for metrics emitted by resilience4j.


**Threadpool Bulkhead**

Let's start with the thread pool bulkhead. In r4j-serviceB an endpoint `sayHello` is setup which will be called. In r4j-serviceA side calling method is decorated like below 

```
@Bulkhead(name = "serviceB#getReply", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<String> getReply(){
    return CompletableFuture.completedFuture(serviceBClient.getReply()
    .getBody());
}
```
Also, we need to define how many threads are allowed in the pool to call the endpoint, due to `resilience4j-spring-boot2` we can achieve this via properties in application yaml( like the hystrix. command... properties we used to define in Hystrix). I define the below property for this endpoint -

```
resilience4j:
  thread-pool-bulkhead:
    instances:
      serviceB#getReply:
        maxThreadPoolSize: 3
        coreThreadPoolSize: 1
        queueCapacity: 1
```
The configuration above defines, the endpoint can be called by concurrently up to a maximum of 3 threads, it will try to reuse the threads in corepool but whenever we have several threads waiting for more than queueCapacity a new thread will be spawned maximum of up to maxThreadPoolSize. 

To create a scenario where threads are busy and we have more than 3 threads try to call the endpoint, I introduce a 2-sec delay in the endpoint and call endpoint with 

```
ab -c 4  -n 100  localhost:8080/v1/reply

``` 

 I get the below result from ab - 

```
Concurrency Level:      4
Time taken for tests:   56.266 seconds
Complete requests:      100
Failed requests:        19  <--- Some Failed Requests
```
Also in logs, I can see several exceptions - 

```
io.github.resilience4j.bulkhead.BulkheadFullException: 
Bulkhead 'serviceB#getReply' is full and does not permit further calls
```
We see below graph for `resilience4j_bulkhead_thread_pool_size` metrics in Prometheus

![ThreadPool-Bulkhead](/threadpool-bulkhead.png)

We see at some point number of threads used reaches maximum thread configured, and whenever more threads try to call endpoint we see exception in the logs.

**Semaphore Bulkhead**

Another type of bulkheading is the Semaphore bulkhead, here instead of a separate thread pool per downstream call, we define maximum permissible calls per downstream.

To demonstrate this pattern, I created an endpoint in r4j-serviceB `/v1/semaphore-bulkhead` which will be called by r4j-serviceA.
The caller method is decorated as below -

```
  @Bulkhead(name = "serviceB#semaphore",type = Bulkhead.Type.SEMAPHORE)
  public String semaphoreBulkheadResponse(){
    return serviceBClient.semaphoreBulkheadResponse().getBody();
  }
```

The configuration properties are below - 

```
  bulkhead:
    instances:
      "[serviceB#semaphore]":
        maxConcurrentCalls: 2
```
Trying to call the endpoint like below - 

```
ab -c 3 -n 10  localhost:8080/v1/semaphore-bulkhead
```

```
Complete requests:      10
Failed requests:        3
   (Connect: 0, Receive: 0, Length: 3, Exceptions: 0)
Non-2xx responses:      3
```
Also in the log, I see below exceptions- 

```
Bulkhead 'serviceB#semaphore' is full and does not permit further calls
```

**Timelimiter**

Often we set a time limit to wait for the call to complete, To use time limiter I am going to use the same endpoint as the Threadpool endpoint, the reason behind this is Timeout can happen when the call is made in a separate thread. 

I decorate the method as below - 

```
@TimeLimiter(name = "service#getReply")
@Bulkhead(name = "serviceB#getReply", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<String> getReply(){
    return CompletableFuture.completedFuture(serviceBClient.getReply()
    .getBody());
}
```
Our endpoint has waiting for 2s so I set a lower timeout in the config 

```
resilience4j.timelimiter:
  instances:
    "[serviceB#getReply]":
      timeoutDuration: 1s
```
So that thread pool bulkheading does not ruin the party so this time I avoid the concurrent call and only call as below 

```
ab -n 10  localhost:8080/v1/reply
```
Immediately I see an exception in logs - 

```
java.util.concurrent.TimeoutException: 
TimeLimiter 'service#getReply' recorded a timeout exception.
```
Prometheus shows below graph with only failed as 10 and other metrics as 0 

![Timelimiter](/timelimiter.png)

**Some notes about Timelimiter**

```
- A nice read for combining Semaphore bulkhead and Timelimiter 
  https://stackoverflow.com/a/59988262
- There's also an option to define the fallback method with Timeout,
  turns out the return type has to be CompletableFuture 
  if we are using annotations driven approach.
```


**CircuitBreaker**

Next, we see the circuit-breaking pattern. This time a create an endpoint in r4j-serviceB which will always cause an exception. I decorate the caller method as below - 

```
    @CircuitBreaker(name = "serviceB#circuitbreaker")
    public String circuitBreakingResponse(){
        return serviceBClient.circuitBreakingResponse().getBody();
    }
```

The respective configuration looks like 

```
  circuitbreaker:
    instances:
      "[serviceB#circuitbreaker]":
        slidingWindowSize: 2
        failureRateThreshold: 50
```
I am configuring a window size of 2 calls and if 50% of those calls are failing then the circuit will be open. Note there are other useful properties like how long the circuit stays open or withing a sliding window minimum how many calls should take place to count the failure threshold etc. But for now, I keep it simple 

With call like below
```
ab -n 10  localhost:8080/v1/circuit-breaker
```

I can see below logs 
```
io.github.resilience4j.circuitbreaker.CallNotPermittedException: 
CircuitBreaker 'serviceB#circuitbreaker' is OPEN and does not permit 
further calls
```
It generates metrics for `not permitted calls` and Prometheus reports below the graph 

![Circuit Breaker](/circuitbreaker.png)

The circuit breaker also accepts a fallback method. This is a method with a `Throwable` argument. When the circuit is opened it uses the fallback.



**Ratelimiter**

Many APIs deploys rate-limiting capabilities so that it does not become overwhelmed and in general it answers with a 429 TooManyRequests status code. When I first read about Ratelimiter I was a bit confused because most of the patterns above deploy on the caller side and protects it from calling service and get stuck, however when we are talking about Ratelimiter how can a caller decide how many requests is too much? Should not it the service being called is the one who decides how many call it can receive? While I search for an answer I found a GitHub discussion and the comment here https://github.com/resilience4j/resilience4j/issues/350#issuecomment-475868062 which suggests using resilience4j as a client-side rate limiter. In case one needs to implement a server-side rate limiter an API Gateway is a better alternative. I still have my doubts about how much a client-side rate limiter worth and who should dictate how much request is too much let's see how it works. Just like all the above pattern here too I create an endpoint, and I decorate the method as below - 

```
  @RateLimiter(name = "serviceB#ratelimiter")
  public String rateLimiter(){
    return serviceBClient.rateLimiter().getBody();
  }
```
And the configuration as below 
```
ratelimiter.instances:
    "[serviceB#ratelimiter]":
        limitForPeriod: 3   <-- number of call allowed in a period
        limitRefreshPeriod: 1s  <-- 1 second is the length of a period
        timeoutDuration: 500ms  <-- how long a thread wait for
                                     permision
```

A rate limiter divides the running time of an application in some period and allows a configured number of calls to happen in that period, a period is refreshed after `limitRefreshPeriod` time.

To demonstrate the effect I make below call 
```
ab -c 3 -n 100  localhost:8080/v1/rate-limiter
```
and I see some calls are blocked as 

```
io.github.resilience4j.ratelimiter.RequestNotPermitted: 
RateLimiter 'serviceB#ratelimiter' does not permit further calls
```
Let's see what Prometheus reveals here 

![Ratelimiter available permission](/ratelimiter-available.png)

The graph above shows at some point available permission drops to 0 and that's where we got a RequestNotPermitted exception.

At the same time, we see waiting thread graph has spiked as the threads wait for permission

![Ratelimiter Waiting Threads](/waiting-threads.png)



**Retry**

That's the last pattern I am going to show here. To demonstrate retry I add an endpoint that will randomly throw an exception. As usual, the caller method is decorated as below - 

```
@Retry(name = "serviceB#retry")
public String retry(){
    return serviceBClient.retry().getBody();
}
```
and respective configuration property looks like 

```
  retry:
    instances:
      "[serviceB#retry]":
        waitDuration: 500ms <-- wait between two retries
        maxAttempt: 3 <-- max retries
```
and then I call like below 

```
ab -c 3 -n 100  localhost:8080/v1/retry
```
Not much useful entry in the log this time, but Prometheus shows a nice graph with calls that successful without retry, calls which are successful with retry, calls which are failed with retry and calls which are failed without retry.

![Retry](/retry.png)

At first, what perplexed me is `failed without retry = 0` while we have many `successful with retry` what does it mean to fail without retry. It turns out in resilience4j we can define a predicate that can decide whether to retry or not so in that case it is possible no further retry is attempted and it failed without retry. In this demo, I do not have such a predicate.

Some more useful properties for Retry are -

- Exponential backoff and backoff multiplier, so that instead of waiting for a fixed amount we can wait for a variable amount.
- Retry on exception, we can define on which exception we want to retry.
- Retry on the predicate, this is what I mentioned earlier, a predicate can be added.

We saw how we can configure various resilience pattern. In the demo of mine I have chosen the property driven and annotation path like what we used to do in Hystrix, however everything of the above is possible in a programmatic way. Often we use multiple patterns together for eg: Threadpool Bulkhead with a time limiter as shown above. Each of the annotation shown above has its order of how it is applied. The default order is as follows

```
Retry(CircuitBreaker(RateLimiter(TimeLimiter(Bulkhead(Function)))) 
```
So bulkhead is at first and a retry at the end. Note that this order can be configurable. 

I must also add here there's a nice `Decorators` class in resilience4j which allows us to programmatically configure resilience patterns. 

**Things that I haven't tried in this demo**  

- Reactive Support
- Alternative to HystrixConcurrencyStrategy where we could have additional support for thread-local propagation. In the resilience4j world, it is known as ContextPropagator.
- Support for Traceability can the trace context be propagated in the caller threads.

Maybe I will need a separate post to try out the above. Overall resilience4j looks like a matured project that encompasses all the resiliency patterns. It provides many integrations and programming styles. Community is also vibrant and I see the author of the project is active. Only the test of production will prove it's worth but from a first look, it shows promise.  

**Resources**

- https://resilience4j.readme.io/docs/getting-started
- https://github.com/resilience4j/resilience4j
- https://github.com/njoysubho/resilience4j-playground