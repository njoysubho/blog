---
title: "Consul Feign"
date: 2020-09-12T16:04:01+02:00
draft: true
---

Service discover is one of the central dogma in a Microservices architecture. Consul is a popular service registry and has good integration with Spring. In this post we will see how to use consul and openfeign to call another service.

- First of all we need to deploy consul ,we can achieve this by deploying consul in a container . 

- Add `spring-boot-actuator-starter` as dependency in service pom. The service will register in consul service registry with this name.

- Add `spring.cloud.consul.host` and `spring.cloud.consul.port` this are consul host and port respectively.

That's all now the service is registered in consul and also it will discover other services.

Let's see how Feign and consul integration works in Spring.

Spring cloud open feign comes with `FeignLoadBalancerAutoConfiguration` . If we see closely it imports three more LoadBalancerAutoConfigurations
   - `HttpClientFeignLoadBalancerConfiguration`, `OkHttpFeignLoadBalancerConfiguration`,
      `DefaultFeignLoadBalancerConfiguration`
   -  If your path has Apache http client then it uses     HttpClientFeignLoadBalancerConfiguration , if Ok http client then it uses OkHttpFeignLoadBalancerConfiguration otherwise uses it DefaultFeignLoadBalancerConfiguration which in turn uses Apache Http client.

   -  The HttpClientFeignLoadBalancerConfiguration creates a `FeignBlockingLoadBalancerClient`  which uses an instance of `LoadBalancerClient`. 
   - `FeignBlockingLoadBalancerClient` is a wrapper around ApacheHttpClient vs OkHttpClient.
   - If we look into the class of `LoadBalancerClient` we see it is an interface and it represents a client side load balancer.
   - spring cloud consul starter brings spring cloud loadbalancer which provides one of the implementations `BlockingLoadBalancerClient` .

   - BlockingLoadBalancerClient further uses by defaut RoundRobinLoadBalancer .
   - RoundRobinLoadbalancer make use of ServerInstanceListSupplier which provides the list of server.
   - By default we get `CachingServiceInstanceListSupplier` from LoadBalancerClientConfig. Which in turn uses ConsulDiscoveryClient.

