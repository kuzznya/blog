---
title: Reactive programming basics
date: 2022-10-24T16:59:13+03:00
categories:
  - Programming
tags: 
  - programming
  - java
draft: false
---

## Reactive programming concept

### Main idea

Reactivity is a reaction on changes (as opposed to proactivity).

Reactive programming is a replacement of imperative code execution line by line.
Instead of executing the code in a thread line by line without an ability 
for another thread to interrupt this execution, in RP execution is divided into small atomic operations, 
short and non-blocking as much as possible. After one operation execution (Publisher) the result is passed to the next
(Subscriber).
Each of the operations can be executed on any thread. In fact, we split execution line by line by executing each operation
separately, and every operation is a Subscriber for the data that it requires and a Publisher of a result.
Ideally every operation should be non-blocking to allow RP work efficiently 
without making a thread that it runs on to wait for some event.

The chain of such operations is a reactive data stream. Reactive streams take data processing to the next level.
Instead of thinking about separate operations, we can operate with streams that can by splitted, filtered,
joined with other streams and flow to the other thread schedulers.

So, reactive programming is basically asynchronous execution + stream data processing + non-blocking operations.

### What is it for

1. Instant reaction on changes (this is used in UI, e.g. it is widely used in JS frameworks)
2. Stream data processing with high parallelism
3. Asynchronous execution without "callback hell" and other problems

### Components

- Publisher - a source of data that (probably) provides data to the subscriber at some point of time
- Subscriber - a component that waits data from a publisher
- Subscription - the way to tell publisher that subscriber is ready to get data (the way to communicate with a publisher)

### Custom implementation

Let's create our own implementation to understand how these 3 simple components
can be used to create more complex ones.

1. `Publisher`, `Subscriber`, `Subscription`
2. `JustPublisher`
3. `CollectingSubscriber` (+ `.collect()` в `Publisher`'е)
4. `MapPublisher` (+ `.map(Function)`, `.peek(Consumer)`)
5. `DeferredPublisher` (+ `.defer(Supplier<Publisher>)`, `.from(Supplier<V>)`)
6. `ConsumingSubscriber` (+ `.consume(Consumer)`)
7. `ParallelPublisher` (`.parallel(n)`)

# TODO 24.10 describe implementation


I should mention an important thing. Adding a new element does not change existing ones, so almost everything is immutable here.

### Cons of reactive programming

1. Problems with debugging
2. Increased testing complexity
3. High level of abstraction can bring hardly detected errors without full understanding of how RP works under the hood
4. Increased complexity of separating the code into methods, of responsibility segregation

### Existing implementations

Initially Reactive Streams Initiative was established. 
Its' goal was to standardize the reactive programming. They created the specification Publisher/Subscriber/Subscription (и Processor, that is just a Publisher and a Subscriber),
that later became a part of a JDK - `java.util.concurrent.Flow` (`Flow.Publisher`, `Flow.Subscriber`, etc.).

The two mainly used implementations of reactive streams are RxJava and Project Reactor.
RxJava is mostly used in Android apps while Project Reactor is mainly used in backend.
RxJava calls Pub/Sub Observable/Observer, but it is generally similar to Project Reactor.

## Project Reactor

[Docs](https://projectreactor.io/docs/core/release/reference/#getting-started)

### Adding to the project

Maven:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- ... -->

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>io.projectreactor</groupId>
                <artifactId>reactor-bom</artifactId>
                <version>2020.0.6</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-core</artifactId>
        </dependency>

        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <!-- ... -->

</project>
```

Gradle:

```groovy
dependencyManagement {
    imports {
        mavenBom "io.projectreactor:reactor-bom:2020.0.6"
    }
}

dependencies {
   implementation 'io.projectreactor:reactor-core'
   testImplementation 'io.projectreactor:reactor-test'
}
```

### Mono and Flux

Two main parts of Project Reactor are Mono and Flux. 
Mono is a source of one value and Flux is a source of multiple values (a stream of data).

### Hot and cold sources

There is an important difference between these two types of data sources.

A subscriber of a hot source after subscriptions gets the data that comes since the moment of subscription
(so this source is already warmed up and transmits the data). 
For example, if the source emits integers from 1 to 10 (non-inclusive), 
then subscriber who subscribed after the number 6 receives only the last 3 integers.

The subscriber of a cold source receives the data since the very beggining.
For instance, subscriber for a previous example receives all 10 integers.

### Creating Mono / Flux

`.just`, `.from`, `.fromIterable`

### Data transformation, processing

- `.map` - одно значение в другое
- `.transform` - преобразовать один Publisher в другой

### Event processing

- `.doOnNext`
- `.doOnError`
- `.doOnComplete`
- `.doOnCancel`
- `doOnSubscribe`
- and others

### Back to the synchronous world

Flux:
- `.blockFirst`
- `.blockLast`
- `.toIterable`
- `.toStream`

Mono: `.block`

### Combining multiple sources

`Flux.merge` / `.mergeWith`, `Flux.zip` / `.zipWith`, `Flux.combine` и прочие.

### Parallelism

`.parallel()`, `ParallelFlux`

NB: if you don't provide the scheduler that the flux should use, the execution will still be single-threaded.

### Schedulers

- single - single-threaded
- parallel - non-blocking fast operations
- elastic - long operations like I/O
- boundedElastic - restricted possible number of threads to avoid thread starvation
- Custom schedulers (newSingle, newParallel, newBoundedElastic, others) - to avoid the situation when two different operations affect each other due to shared scheduler

### Backpressure

Due to the model with Publisher that pushes the data to the subscriber, there can be a situation
when a subscriber is not ready yet for the new data 
(most often when the source of data is very fast and a processor is slow). This situation is called backpressure.
There are multiple strategies to handle that:

`.onBackpressureLatest`, `.onBackpressureDrop`, `.onBackpressureBuffer`

### Error handling strategies

There are multiple ways to handle errors:

- `.onErrorContinue` - continue execution by handling the exception
- `.onErrorResume` - swith to the other reactive flow - "fallback"
- `.onErrorReturn` - return a fallback value instead of an errored value and stop the flow
- `.onErrorStop` - stop the execution

### Testing

Main class for testing purposes is `StepVerifier`:

```java
StepVerifier.create(source)
    .expectNext("test1")
    .expectNoEvent(Duration.ofSeconds(1000))
    .expectNext("test2")
    .expectNextCount(10)
    .expectComplete()
    .verify();
```

### Debugging

# TODO 25.10 find and add a link to the video about Reactor debugging

Mode `Hooks.onOperatorDebug()` should be enabled in IDEA settings.
It maked the stacktrace more informative.

There also is a mehtod `.log()` that simplifies error searching.

### Sinks

The immutability of streams does not always suit you, sometimes you have to add values to the Flux dynamically.
There is a tool for that - it is called Sink.

There are multiple types of Sinks. They are created with `Sinks.one()` / `Sinks.many()`
and transform to Mono/Flux via `.asMono` / `.asFlux`

## Spring WebFlux

[Docs](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html)
 
 Spring created a reactive web framework on top of Project Reactor.

![MVC vs WebFlux](spring-mvc-and-webflux-venn.png)

### Adding to the project

Let's assume that you're using Spring Boot, so you do not need to set up everything manually.

Maven:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

Gradle:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    testImplementation 'io.projectreactor:reactor-test'
}
```

### Web controllers

Spring WebFlux supports controllers with the same annotations as usual Web MVC controllers do:

```java
@RestContoller
public class MonoController {

    @GetMapping("/hello")
    public Mono<String> hello(@RequestParam String name) {
        return Mono.just("Hello, " + name);
    }
}
```

### Functional endpoints

In addition to usual controllers, WebFlux allows to register endpoints using functional style:

```java
@Configuration
public class CalculationRouter {

    @Bean
    public RouterFunction<ServerResponse> calculationRoutes(CalculationHandler handler) {
        return RouterFunctions
                .route()
                .POST("/api/v1/calculations/unordered", handler::calculateUnordered)
                .POST("/api/v1/calculations/ordered", handler::calculateOrdered)
                .build();
    }
}
```

```java
@Component
public class CalculationHandler {

    private final CalculationSeriesService seriesService;

    public CalculationHandler(CalculationSeriesService seriesService) {
        this.seriesService = seriesService;
    }

    public Mono<ServerResponse> calculateUnordered(ServerRequest request) {
        Mono<CalculationRequest> bodyMono = request.bodyToMono(CalculationRequest.class);
        return bodyMono.flatMap(calculationRequest ->
                ServerResponse.ok()
                .contentType(MediaType.TEXT_EVENT_STREAM)
                .body(
                        seriesService.calculateUnordered(
                                calculationRequest.getFunction1(),
                                calculationRequest.getFunction2(),
                                calculationRequest.getCount())
                                .map(UnorderedCalculationResult::getDataAsString)
                                .onErrorResume(ExecutionTimeoutException.class,
                                        ex -> Mono.just("EXECUTION TIMEOUT"))
                                .onErrorResume(FunctionExecutionException.class,
                                        ex -> Mono.just("EXECUTION " + ex.getExecutionId() + " ERROR"))
                                .onErrorReturn(Predicate.not(e -> e instanceof FunctionEvaluationException),
                                        "EXECUTION ERROR"),
                        String.class
                )
        );
    }

    public Mono<ServerResponse> calculateOrdered(ServerRequest request) {
        Mono<CalculationRequest> bodyMono = request.bodyToMono(CalculationRequest.class);
        return bodyMono.flatMap(calculationRequest ->
                ServerResponse.ok()
                        .contentType(MediaType.TEXT_EVENT_STREAM)
                        .body(
                                seriesService.calculateOrdered(
                                        calculationRequest.getFunction1(),
                                        calculationRequest.getFunction2(),
                                        calculationRequest.getCount())
                                        .map(OrderedCalculationResult::getDataAsString)
                                        .onErrorResume(ExecutionTimeoutException.class,
                                                ex -> Mono.just("EXECUTION TIMEOUT"))
                                        .onErrorResume(FunctionExecutionException.class,
                                                ex -> Mono.just("EXECUTION " + ex.getExecutionId() + " ERROR"))
                                        .onErrorReturn(Predicate.not(e -> e instanceof FunctionEvaluationException),
                                                "EXECUTION ERROR"),
                                String.class
                        )
        );
    }
}
```

### SSE

In case Flux is returned from a cntroller method, Spring transforms the response to the Server Side Events (SSE) format.
SSE are useful when one-way data streaming is required (e.g. notifying the frontend about some events - one way, in contrast to WebSockets).
Response content type can be either `text/event-stream` (официальный тип для SSE), либо `application/stream+json` ([deprecated for streaming JSON by Spring](https://github.com/spring-projects/spring-framework/issues/21283)).

According to the spec, SSE can only be used for POST requests.
