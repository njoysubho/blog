---
title: "Spring Under the hood-Inside Feign and client side discovery with consul"
date: 2020-09-12T16:04:01+02:00
draft: false
---

Feign using consul as discovery service is easy to setup thanks to Spring Autoconfigurations, but let's see how this magic is actually done - 

Spring cloud open feign comes with `FeignLoadBalancerAutoConfiguration` . If we see closely it imports three more LoadBalancerAutoConfigurations
   - `HttpClientFeignLoadBalancerConfiguration`, `OkHttpFeignLoadBalancerConfiguration`,
      `DefaultFeignLoadBalancerConfiguration`
   -  If classpath has Apache http client then it uses     `HttpClientFeignLoadBalancerConfiguration` , if path has Ok http client then it uses `OkHttpFeignLoadBalancerConfiguration` otherwise uses it `DefaultFeignLoadBalancerConfiguration` which in turn uses Apache Http client.

   -  The `HttpClientFeignLoadBalancerConfiguration` creates a `FeignBlockingLoadBalancerClient`  which uses an instance of `LoadBalancerClient`. 
   - `FeignBlockingLoadBalancerClient` is a wrapper around ApacheHttpClient vs OkHttpClient.
   - If we look into the class of `LoadBalancerClient` we see it is an interface and it represents a client side load balancer.
   - spring cloud consul starter brings spring cloud loadbalancer which provides one of the implementations `BlockingLoadBalancerClient` .

   - BlockingLoadBalancerClient further uses by defaut RoundRobinLoadBalancer .
   - RoundRobinLoadbalancer make use of `ServerInstanceListSupplier` which provides the list of server.
   - By default we get `CachingServiceInstanceListSupplier` from LoadBalancerClientConfig. Which in turn uses ConsulDiscoveryClient.
   - `ConsulDiscoveryClient` is an implementation of `DiscoveryClient` which reads from service registry .

Thus from the above flow of various configurations Feign gets integrated with Consul and discovers the actual service url to call.

