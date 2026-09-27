---
title: "Spring Cloud Feign"
date: 2022-08-06T11:22:42+02:00
draft: false
categories: ["Platform and reliability"]
---

Often a service needs to call Http endpoints , Feign comes from the OpenFeign project makes it easier to call http endpoints in a declarative fashion. Spring has openfeign integraton through its `Spring Cloud OpenFeign` integration.

## How To include feign clients

The actual project for feign is `OpenFeign` <https://github.com/OpenFeign/feign> . Spring comes with it’s own starter . Add below dependency to the project

``` xml
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
```

For an example let’s create a service named `feign-serviceA` which will call `feign-serviceB` . `feign-serviceB` exposes a `GET` endpoint `/hello` which returns a simple `hello` response body.

In order to call the endpoint using a feign client from `feign-serviceA` we need to do following -

- Create an interface annotated with `@FeignClient` like below

``` java
@FeignClient(url = "<host:port for serviceB>", name = "serviceB")
public interface ServiceBClient{

    @GetMapping("/hello")
    public ResponseEntity<String> sayHello();
}
```

- Add `@EnableFeignClients` annotation to main class .

and we are all set we can simply autowire `ServiceBClient` and call method `sayHello` like any normal method call .

> @EnableFeignClients takes a list of clients if we have many feign clients then it is better to mention the clients as argument otherwise spring will do a classpath scan to find out feign clients.

## Feign Configuration

Under the hood feign comes with some components which are used to make a call to remote endpoints and encode/decode request response .

- Client - To make HTTP call feign requires http client. By default openfeign comes with a Default Client. We can override it with `ApacheHttpClient`, `OkHttpClient` or `ApacheHC5FeignClient` . These feign clients are wrapper around a delegate client. For example ApacheHttpClient wraps a httpcomponents httpclient and converts the response to feign response.

- Decoder - Something needs to convert the feign `Response` to the actual type of the feign method’s return type. Decoders are that instruments. By default spring provides an `OptionalDecoder` which delegates to `ResponseEntityDecoder` which further delegates to `SpringDecoder`. We can override it by defining a bean of `Decoder` .

- Encoder - We call feign methods by passing objects to it something needs to convert it to http request body. `Encoder` does that job. Again By default spring provides `SpringEncoder`.

Along side above components there are also support for caching, metrics provided by spring feign starter .

We can create a configuration class and override the defaults for the above components. If we want to override default for single components `@Feign` accepts ` configuration` arguments which we can use to define custom override for default values.

## Retry

Feign has baked in support for retry mechanism. However by default it uses `Retry.NEVER_RETRY` . For example we can create a custom retryer which will retry any status code \> 400. Below is the code for our `CustomRetryer` .

``` java
public class CustomRetryer extends Retryer.Default{
    public CustomRetryer(long period, long maxPeriod, int maxAttempts){
        super(period, maxPeriod, maxAttempts);
    }

    @Override
    public void continueOrPropagate(RetryableException e){
        log.info("Going to retry for ", e);
        super.continueOrPropagate(e);
    }

    @Override
    public Retryer clone(){
        return new CustomRetryer(5,SECONDS.toMillis(1), 5);
    }
}
```

One important fact is feign Retry works either on `IOException` or `RetryableException` thrown from some `errorDecoder` . Below is how a custom decoder looks like -

``` java
    @Bean
    public ErrorDecoder errorDecoder(){
        return (methodKey, response) -> {
            byte[] body = {};
            try {
                if (response.body() != null) {
                    body = Util.toByteArray(response.body().asInputStream());
                }
            } catch (IOException ignored) { // NOPMD
            }
            FeignException exception = new FeignException.BadRequest(response.reason(), response.request(), body, response.headers());
            if (response.status() >= 400) {
                return new RetryableException(
                        response.status(),
                        response.reason(),
                        response.request().httpMethod(),
                        exception,
                        Date.from(Instant.now().plus(15, ChronoUnit.MILLIS)),
                        response.request());
            }
            return exception;
        };
    }
```

> Though documentation says feign retries on IOException internally when an IOException occurs it wraps it in a `RetryableException`.

## Support for resiliency

One form of resiliency through retries we saw in last section. Spring has `CircuitBreaker` support for feign . It achieves it through a separate Feign builder `FeignCircuitBreaker.Builder` . The actual implementation of circuitbreaker comes from `resilience4j` library.

## Interceptor

Sometimes we want to modify the request by adding some extra information. For example we may add some header for each request. We can achieve this by using `RequestInterceptor`. For experiment I added below interceptor which populates a header `userid`.

``` java
    @Bean
    public RequestInterceptor userIdRequestInterceptor(){
        return (template) -> {
            template.header("userid", "somerandomtext");
        };
    }
```

`feign-serviceB` reads this header and returns back as header. If we do a curl we get below response

    < HTTP/1.1 200
    < connection: keep-alive
    < date: Sat, 20 Aug 2022 15:27:47 GMT
    < keep-alive: timeout=60
    < userid: somerandomtext
    < Content-Type: text/plain;charset=UTF-8
    < Content-Length: 7
    <
    * transfer closed with 7 bytes remaining to read

We see userid is added to the response. A very useful application for interceptor is when feign has to send oauth2 access token . Out of the spring provides a `OAuth2FeignRequestInterceptor` which adds access token for each request.

## Client side loadbalancing support

From spring boot 2.4.0 feign has integration with spring-cloud-loadbalancer which can fetch client url info from various service discovery providers and make that info available to feign .

Usage of feign simplifies various aspect of making http request. In a typical production environment we may need to override several components like clients, decoder, errorDecoder etc . Also within Spring ecosystem feign is nicely integrated with resiliency, loadbalancing , metrics etc which makes it automatic choice when we are working in a microservices architecture .
