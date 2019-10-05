---
title: "Log TraceId Micronaut"
date: 2019-10-05T09:46:54+02:00
draft: true
---

Distributed tracing is must have when we create our microservices . Frameworks like Spring Boot,Micronaut comes with the support of creating TraceId,SpanId and propagate to visualization tools like Zipkin,Jaegar . 
This post shortly describe what we need to add in order to log the TraceId in application or access log . Logging traceId helps us in correlating the logs . 
For logging I have used Slf4j with logback . In micronaut support for generating TraceId comes quite easily just matter of adding proper dependencies which can be found here https://guides.micronaut.io/micronaut-microservices-distributed-tracing-zipkin/guide/index.html . However to log the same traceId we need to have a Tracing Bean like following 
```
@Bean
    public Tracing tracing() {
        val tracing = Tracing.newBuilder()
                .localServiceName("hello")
                .spanReporter(Reporter.NOOP)
                .currentTraceContext(MDCCurrentTraceContext.create());
        return tracing.build();
    }

This will put the traceId in MDC and this can be used in logback.xml as follows 
```
```
 <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <withJansi>true</withJansi>
        <!-- encoders are assigned the type
             ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
        <encoder>
            <pattern>[%X{traceId}]- %msg%n</pattern>
        </encoder>
</appender>
```

MDCCurrentContext is deprecated , instead of that 
```
ThreadLocalCurrentTraceContext.newBuilder().addScopeDecorator(new Slf4jScopeDecorator()).build()
```
