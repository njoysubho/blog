---
title: "Spring Boot Liveness"
date: 2020-05-20T12:21:24+02:00
draft: false
---

Spring Boot Actuator Health endpoint since 2.3.0 comes with two health groups Liveness and Readiness. Actuator Health points are being used to reflect the status of the application. Spring Boot also provides a way to add multiple health indicators together and form a group. As a continuation to this concept Spring Boot provides two HealthIndicator
LivenessStateIndicator and ReadinessStateIndicator. These two indicators are added to liveness and readiness group respectively by AvailabilityProbesHealthEndpointGroups. 

It is worth mentioning Kubernetes also has the concept of Liveness and Readiness for pods and deployment level so the new health probes can be used as k8s readiness and liveness probe. 

But these probes are not tied to k8s in any way but can be used independent of deployment platform. 

## How to enable probes

- Add spring boot actuator as dependency .
- Add `management.health.probes.enabled: true` to application properties file.

A sample Kubernetes deployment spec with readiness and liveness probe looks like - 

```
.....
    spec:
      containers:
      - name: spring-feature
        image: <image>
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 1
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 1

```


Couple with GracefulShutdown and Health Probes Spring Boot eases container state maintanance.