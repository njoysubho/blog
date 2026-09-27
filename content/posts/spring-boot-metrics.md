---
title: "Spring Boot Metrics"
date: 2022-05-22T10:12:53+02:00
draft: false
categories: ["Platform and reliability"]
---

## Introduction

Observability is one of the pillars of modern microservices architecture. Application metrics is one dimension of that observbility. When an application runs on production we may want to know various operational metrics like memory,cpu ,threadpool usage etc as well as buisness metrics for example how many request for a particular operations are made. Spring boot with the help of micrometer enables application developers to expose various metrices.

## Setup metrics

To add support for metrics , we need to add actuator dependency

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

and we also need to enable the metrics endpoint like following -

``` yml
management:
  endpoints:
    web:
      exposure:
        include: metrics
```

That’s it our application now set to expose metrics.

The below curl

    curl localhost:8080/actuator/metrics | jq .

will give response like -

``` json
{
  "names": [
    "application.ready.time",
    "application.started.time",
    "disk.free",
    "disk.total",
    "executor.active",
    "executor.completed",
    "executor.pool.core",
    "executor.pool.max",
    "executor.pool.size",
    "executor.queue.remaining",
    "executor.queued",
    "hikaricp.connections",
    "hikaricp.connections.acquire",
    "hikaricp.connections.active",
    "hikaricp.connections.creation",
    "hikaricp.connections.idle",
    "hikaricp.connections.max",
    "hikaricp.connections.min",
    "hikaricp.connections.pending",
    "hikaricp.connections.timeout",
    "hikaricp.connections.usage",
    "http.server.requests",
    "jdbc.connections.active",
    "jdbc.connections.idle",
    "jdbc.connections.max",
    "jdbc.connections.min",
    "jvm.buffer.count",
    "jvm.buffer.memory.used",
    "jvm.buffer.total.capacity",
    "jvm.classes.loaded",
    "jvm.classes.unloaded",
    "jvm.gc.live.data.size",
    "jvm.gc.max.data.size",
    "jvm.gc.memory.allocated",
    "jvm.gc.memory.promoted",
    "jvm.gc.overhead",
    "jvm.gc.pause",
    "jvm.memory.committed",
    "jvm.memory.max",
    "jvm.memory.usage.after.gc",
    "jvm.memory.used",
    "jvm.threads.daemon",
    "jvm.threads.live",
    "jvm.threads.peak",
    "jvm.threads.states",
    "logback.events",
    "process.cpu.usage",
    "process.files.max",
    "process.files.open",
    "process.start.time",
    "process.uptime",
    "system.cpu.count",
    "system.cpu.usage",
    "system.load.average.1m",
    "tomcat.sessions.active.current",
    "tomcat.sessions.active.max",
    "tomcat.sessions.alive.max",
    "tomcat.sessions.created",
    "tomcat.sessions.expired",
    "tomcat.sessions.rejected"
  ]
}
```

These are the metrics that spring boot provides out of the box. We can see it includes jvm memory, threads, cpu usage etc.

The below request will show used jvm memory

``` json
 curl 'http://localhost:8080/actuator/metrics/jvm.memory.max'| jq .


 {
  "name": "jvm.memory.max",
  "description": "The maximum amount of memory in bytes that can be used for memory management",
  "baseUnit": "bytes",
  "measurements": [
    {
      "statistic": "VALUE",
      "value": 5620367357
    }
  ],
  "availableTags": [
    {
      "tag": "area",
      "values": [
        "heap",
        "nonheap"
      ]
    },
    {
      "tag": "id",
      "values": [
        "CodeHeap 'profiled nmethods'",
        "G1 Old Gen",
        "CodeHeap 'non-profiled nmethods'",
        "G1 Survivor Space",
        "Compressed Class Space",
        "Metaspace",
        "G1 Eden Space",
        "CodeHeap 'non-nmethods'"
      ]
    }
  ]
}
```

A metrics can have multiple dimension. For example `jvm.memory.max` have `heap` and `nonheap` size. We can drill down to metrics using its like .

``` json
curl 'http://localhost:8080/actuator/metrics/jvm.memory.max?tag=area:heap' | jq .

{
  "name": "jvm.memory.max",
  "description": "The maximum amount of memory in bytes that can be used for memory management",
  "baseUnit": "bytes",
  "measurements": [
    {
      "statistic": "VALUE",
      "value": 4294967294
    }
  ],
  "availableTags": [
    {
      "tag": "id",
      "values": [
        "G1 Old Gen",
        "G1 Survivor Space",
        "G1 Eden Space"
      ]
    }
  ]
}
```

So far we know that spring boot exposes metrics and we can request metric endpoint to get those metrics and if required we can drill down to this metrics using available tags.

## Custom Metrics

What if we need more metrics?

Spring uses `Micrometer` underneath which takes care of generating and exposing metrics. `MeterRegistry` is the core concept of micrometer that holds multiple metrices.

We can simply inject an instance of `MeterRegistry` in our custom metrics provider like below -

``` java
@Component
public class InventoryMetrics{
    private Counter inventoryCounter;

    public InventoryMetrics(MeterRegistry meterRegistry){
        inventoryCounter = Counter.builder("products")
                .description("Product Count")
                .register(meterRegistry);
    }

    public void inventoryAdded(int count){
        inventoryCounter.increment(count);
    }
}
```

Here we have created a new metrics named `products` , every time we add a new product we will increment the value.

Now if we curl our metrics endpoint with we get product count

``` json
curl 'http://localhost:8080/actuator/metrics/products' | jq .

{
  "name": "products",
  "description": "Product Count",
  "baseUnit": null,
  "measurements": [
    {
      "statistic": "COUNT",
      "value": 2
    }
  ],
  "availableTags": []
}
```

## Streaming metrics

On production we would like to stream metrices to a data store like elasticsearch, influxdb etc. Spring Boot supports out of the box various data sinks <https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.metrics> .

For this post I ran a influxdb docker image locally . We can see our custom metrics pushed to influx -

![influx](/spring-metrics-influx.png)

On the application side configuration looks like -

``` yml
management:
  metrics:
    export:
      influx:
        uri: 'http://localhost:8086'
        token: '<your-token>'
        bucket: '<your bucket>'
        org: '<your org>'
        autoCreateDb: false
```

> ***NOTE:*** Config properties for influxdb 1.x and 2.x different. The above config is for influxdb 2.x. For 1.x we need to use username , password and db instead of token,bucket and org. Also autoCreateDb should be false as otherwise springboot will try to create a db named mydb .

## Micrometer metric Types

**Counter** Counters are monotonically increasing metrics. It can be incremented by a fixed amount which is always positive.

**Gauge** Gauges are instantaneous metrics this can either go up or down.

**Timer** Timers are short duration latencies and frequency of events.

Our custom metric is a type of counter to measure the added products .

This is at a very high level how to expose metrics.

In production though we need to be aware of the volume of metrics, because if volumes and frequency is high our dashboards can become really slow.

We should also think about a retention policy of data to not to store unecessary old data. This will help saving some storage.

In conclusion metrics are essential part of our services and spring boot make it easy to gather and expose it to various data sinks.
