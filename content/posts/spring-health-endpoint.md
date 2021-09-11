---
title: "Spring Boot Actuator Health Endpoint"
date: 2021-07-19T23:11:10+02:00
draft: false
---

Health endpoint for a service exposes status of it's state. This state information is useful to determine whether the application is up and ready to accept requests. Normally when a service is fronted by a Load Balancer, the loadbalancer uses the information derived from the health endpoint to decide whether to route traffic to that instance of service or not.
Spring Boot actuator provides Health Endpoint out of the box. If we add `spring-boot-actuator` dependency we can get the URI `/actuator/health`. 

The status of health endpoint is the aggregate of its components. Each Component is an instance of `HealthContributor`. Each 
`HealthContributor` will either be an `HealthIndicator` or `CompositeHealthContributor`. 

First, let's see what are the default HealthIndicator that Spring Boot Actuator configures. To do that we add the following property in application.yml -

`management.endpoint.health.show-details: always`

Now if we do `curl localhost:8080/actuator/health | jq .` we get back 

```
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
    }
    "ping": {
      "status": "UP"
    }
  }
}
```
We see here `diskSpace` and `ping`, by default all the health endpoint does is to check if there are enough diskspace and the context is up. 

The status of the health endpoint is the aggregate of its components. If any of the component is not in `UP` state, the health endpoint will be in `DOWN` state.

## Custom Health Indicator

To create a custom health indicator we need to implement `HealthIndicator` interface. Lets create two new Health Indicator classes.

- AlwaysUPHealthIndicator - will always return `UP`
- AlwaysDownHealthIndicator - will always return `DOWN`

```
@Component
public class AlwaysUpHealthIndicator implements HealthIndicator {
	@Override
	public Health health() {
		return Health.up().build();
	}
}
```
```
public class AlwaysDownHealthIndicator extends AbstractHealthIndicator {
	@Override
	protected void doHealthCheck(Health.Builder builder) throws Exception {
		 builder.down();
	}
}
```
I have deliberately used `AbstractHealthIndicator` to show that a custom health indicator can be a subclass of `AbstractHealthIndicator`. `AbstractHealthIndicator` implements `HealthIndicator` interface, and is useful if health check is going to throw an exception. It contains exception handling logic.

After adding this two classes we can see the health endpoint is down, because AlwaysDownHealthIndicator is always in down state.

```
{
  "status": "DOWN",
  "components": {
    "alwaysDown": {
      "status": "DOWN"
    },
    "alwaysUp": {
      "status": "UP"
    },
    "diskSpace": {
      "status": "UP"
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

Now because we have added our custom health indicators, we may wish to disable it . There's no out of the box property to diable a custom health indicator. However if we look how it has been achieved in case of default health indicators are by using `ConditionalOnEnabledHealthIndicator` annotation. So we can add this to our custom health indicators. I am adding it to `AlwaysDownHealthIndicator` class. The class looks like below

```
@ConditionalOnEnabledHealthIndicator("always-down")
@Component
public class AlwaysDownHealthIndicator extends AbstractHealthIndicator {
	@Override
	protected void doHealthCheck(Health.Builder builder) throws Exception {
		 builder.down();
	}
}

```
We can disable `AlwaysDownHealthIndicator` by adding `management.health.always-down.enabled: false` property in application.yml.

If we curl the health endpoint now, we will get back

```
{
  "status": "UP",
  "components": {
    "alwaysUp": {
      "status": "UP"
    },
    "diskSpace": {
      "status": "UP"
    },
    "ping": {
      "status": "UP"
    }
  }
}
```
Notice not only the component `alwaysDown` is missing, but also now the health check returns `UP`, because none of the components are `DOWN` right now.

## Health Group
Individual health components can be grouped together , each of this group then can be accessed by `/actuator/health/<group-name>` endpoint.

Let's create two new groups

- default - it will have diskspace and ping components.
- custom - it will have our custom alwaysUp and alwaysDown endpoints. 

This time health endpoint one additonal property in reply is `group`


```
{
  "status": "DOWN",
  "components": {
    "alwaysDown": {
      "status": "DOWN" ---> I have enabled it again for this part
    },
    "alwaysUp": {
      "status": "UP"
    },
    "diskSpace": {
      "status": "UP"
    },
    "ping": {
      "status": "UP"
    }
  },
  "groups": [
    "custom",
    "default"
  ]
}
```

Upon accessing `/actuator/health/default` we get back 

```
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP"
    },
    "ping": {
      "status": "UP"
    }
  }
}
```
and upon accessing `/actuator/health/custom` we get back

```
{
  "status": "DOWN",
  "components": {
    "alwaysDown": {
      "status": "DOWN"
    },
    "alwaysUp": {
      "status": "UP"
    }
  }
}
```


## Liveness and Readiness

Spring boot comes with two special group called `liveness` and `readiness` . But before looking into details what is the use of this group lets understand what is the meaning in general about liveness and readiness.

liveness - indicates that the internal state is valid for the instance. In simpler terms the service instance is up and running.

readiness - indicates if the service is functional or it can serve request. A service may have some dependency and that dependency need to be in place for the service to serve any request, so although the service is up and running it may happend that the dependency is not available.

We see a similar concept of liveness and readiness in kubernetes. We can define liveness probe and readiness probe for a pod. If the liveness probe fails, kubernetes will try to restart , however if the readiness probe fails kubernetes will not route any request to the pod.

Spring boot actuator provide this two group out of the box. To enable it we need to add

`management.endpoint.health.probes.enabled: true`

A request to health endpoint will return 

```
{
  "status": "DOWN",
  "components": {
    "alwaysDown": {
      "status": "DOWN"
    },
    "alwaysUp": {
      "status": "UP"
    },
    "diskSpace": {
      "status": "UP"
    },
    "livenessState": {
      "status": "UP"
    },
    "ping": {
      "status": "UP"
    },
    "readinessState": {
      "status": "UP"
    }
  },
  "groups": [
    "custom",
    "default",
    "liveness",
    "readiness"
  ]
}
```
Along side previous groups that we created , we see two new groups `liveness` and `readiness`

We can request this individual group similarly as we did for our custom groups 

A request to `/actuator/health/liveness` will return 

```
{
  "status": "UP"
}
```
## A Little deep dive into liveness and readiness health indicator

We may wonder how liveness and readiness know when application is up and when it is ready to accept request. In the heart of it lies a spring event based architecture. Spring comes with `LivenessHealthIndicator` and `ReadinessHealthIndicator` class.
Both of this class gets the information of the service availability for a dependency named `ApplicationAvailability` .
`ApplicationAvailability` is an interface and `ApplicationAvailabilityBean` is an implementation. `ApplicationAvailabilityBean` is also a listener for `AvailabilityChangeEvent`. Any event it receieves is stored in a map named `events`.

Now there are other components in Spring which publishes Liveness and Readiness events. For example `EventPublishingRunListener` publishes a `LivenessState.CORRECT` event when the context is refreshed or application is started. Similarly when all the Applicationrunner is already run it publishes `ReadinessState.ACCEPTING_TRAFFIC`. Another example is in `ServletWebServerApplicationContext#doClose` method that publishes ` ReadinessState.REFUSING_TRAFFIC`. 

`ApplicationAvailabilityBean` recieves these messages and store it into the in memory map. Upon querying liveness and readiness endpoint we get the momentan status for liveness and readiness events.

Lastly, ofcourse we can add our own custom HealthIndicator to any of the liveness and readiness group. However we have to be alert about what indicator we are adding. It is advisable to not include any external dependency related check in liveness indicator, because a momentan hiccup in the dependency can cause the service to get restarted, or kicked out from Loadbalancers list of services. 

Finally, spring boot also enables various other health indicators based on jars on classpath for example redis, datasource, elasticsearch and many more.

That's it from this blog, we see how spring boot configures health endpoints out of the box, and how to group them and we also know now what is the magic between these features.
















