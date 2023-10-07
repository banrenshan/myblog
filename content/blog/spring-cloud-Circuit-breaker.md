---
title: spring-cloud-Circuit-breaker
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:54:03
---

Spring Cloud Circuit breaker 提供了跨不同断路器实现的抽象。 它提供了在您的应用程序中使用的一致 API，让您（开发人员）可以选择最适合您的应用程序需求的断路器实现。



可用的实现如下：

- [Resilience4J](https://github.com/resilience4j/resilience4j)
- [Sentinel](https://github.com/alibaba/Sentinel)
- [Spring Retry](https://github.com/spring-projects/spring-retry)



# 快速入门



在API中使用`CircuitBreakerFactory` 类创建CircuitBreaker断路器。当你添加实现了断路器的依赖时，会自动创建`CircuitBreakerFactory` bean，下面代码是使用例子：

```java
@Service
public static class DemoControllerService {
    private RestTemplate rest;
    private CircuitBreakerFactory cbFactory;

    public DemoControllerService(RestTemplate rest, CircuitBreakerFactory cbFactory) {
        this.rest = rest;
        this.cbFactory = cbFactory;
    }

    public String slow() {
        return cbFactory.create("slow").run(  //断路器的名称
            () -> rest.getForObject("/slow", String.class),  //业务方法
            throwable -> "fallback"  //后备方法
        );
    }
}
```

如果reactor在你的classpath下，你可以使用`ReactiveCircuitBreakerFactory` 来处理响应式代码：

```java
@Service
public static class DemoControllerService {
    private ReactiveCircuitBreakerFactory cbFactory;
    private WebClient webClient;


    public DemoControllerService(WebClient webClient, ReactiveCircuitBreakerFactory cbFactory) {
        this.webClient = webClient;
        this.cbFactory = cbFactory;
    }

    public Mono<String> slow() {
        return webClient.get().uri("/slow").retrieve().bodyToMono(String.class).transform(
        it -> cbFactory.create("slow").run(it, throwable -> return Mono.just("fallback")));
    }
}
```

ReactiveCircuitBreakerFactory.create 创建 ReactiveCircuitBreaker ，run方法接受 Mono或Flux类型的参数。



# 自定义断路器

要为所有断路器提供默认配置，请创建一个自定义 bean，该 bean 传递 Resilience4JCircuitBreakerFactory 或 ReactiveResilience4JCircuitBreakerFactory。 configureDefault 方法可用于提供默认配置:

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
    return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(4)).build())
            .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
            .build());
}
```



```java
@Bean
public Customizer<ReactiveResilience4JCircuitBreakerFactory> defaultCustomizer() {
    return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
            .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(4)).build()).build());
}
```



与提供默认配置类似，您可以创建一个自定义 bean，它通过 Resilience4JCircuitBreakerFactory 或 ReactiveResilience4JCircuitBreakerFactory 传递。

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> slowCustomizer() {
    return factory -> factory.configure(builder -> builder.circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
            .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(2)).build()), "slow");
}
```



除了配置创建的断路器之外，您还可以在断路器创建之后但在返回给调用者之前自定义断路器。 为此，您可以使用 addCircuitBreakerCustomizer 方法。 这对于将事件处理程序添加到 Resilience4J 断路器非常有用。

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> slowCustomizer() {
    return factory -> factory.addCircuitBreakerCustomizer(circuitBreaker -> circuitBreaker.getEventPublisher()
    .onError(normalFluxErrorConsumer).onSuccess(normalFluxSuccessConsumer), "normalflux");
}
```



```java
@Bean
public Customizer<ReactiveResilience4JCircuitBreakerFactory> slowCustomizer() {
    return factory -> {
        factory.configure(builder -> builder
        .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(2)).build())
        .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults()), "slow", "slowflux");
        factory.addCircuitBreakerCustomizer(circuitBreaker -> circuitBreaker.getEventPublisher()
            .onError(normalFluxErrorConsumer).onSuccess(normalFluxSuccessConsumer), "normalflux");
     };
}
```





每次调用 CircuitBreaker#run 时，某些 CircuitBreaker 实现（例如 Resilience4JCircuitBreaker）都会调用`customize` 方法。 它可能效率低下。 在这种情况下，您可以使用 CircuitBreaker#once 方法。



下面的例子展示了每个 io.github.resilience4j.circuitbreaker.CircuitBreaker 消费事件的方式。

```java
Customizer.once(circuitBreaker -> {
  circuitBreaker.getEventPublisher()
    .onStateTransition(event -> log.info("{}: {}", event.getCircuitBreakerName(), event.getStateTransition()));
}, CircuitBreaker::getName)
```



您可以在应用程序的配置属性文件中配置 CircuitBreaker 和 TimeLimiter 实例。 属性配置比 Java 定制器配置具有更高的优先级。

```yaml
resilience4j.circuitbreaker:
 instances:
     backendA:
         registerHealthIndicator: true
         slidingWindowSize: 100
     backendB:
         registerHealthIndicator: true
         slidingWindowSize: 10
         permittedNumberOfCallsInHalfOpenState: 3
         slidingWindowType: TIME_BASED
         recordFailurePredicate: io.github.robwin.exception.RecordFailurePredicate

resilience4j.timelimiter:
 instances:
     backendA:
         timeoutDuration: 2s
         cancelRunningFuture: true
     backendB:
         timeoutDuration: 1s
         cancelRunningFuture: false
```



如果resilience4j-bulkhead 在类路径上，Spring Cloud CircuitBreaker 将使用Resilience4j Bulkhead 包装所有方法。 您可以通过将 spring.cloud.circuitbreaker.bulkhead.resilience4j.enabled 设置为 false 来禁用 Resilience4j Bulkhead。



Spring Cloud CircuitBreaker Resilience4j 提供了两种舱壁模式的实现：

- 使用信号量的 SemaphoreBulkhead
- 使用有界队列和固定线程池的 FixedThreadPoolBulkhead。



默认情况下，Spring Cloud CircuitBreaker Resilience4j 使用 FixedThreadPoolBulkhead。 

Customizer<Resilience4jBulkheadProvider> 可用于提供默认的 Bulkhead 和 ThreadPoolBulkhead 配置。

```java
@Bean
public Customizer<Resilience4jBulkheadProvider> defaultBulkheadCustomizer() {
    return provider -> provider.configureDefault(id -> new Resilience4jBulkheadConfigurationBuilder()
        .bulkheadConfig(BulkheadConfig.custom().maxConcurrentCalls(4).build())
        .threadPoolBulkheadConfig(ThreadPoolBulkheadConfig.custom().coreThreadPoolSize(1).maxThreadPoolSize(1).build())
        .build()
);
}
```

与默认的“Bulkhead”或“ThreadPoolBulkhead”配置类似，您可以创建一个自定义 bean，该 bean 传递给 Resilience4jBulkheadProvider。

```java
@Bean
public Customizer<Resilience4jBulkheadProvider> slowBulkheadProviderCustomizer() {
    return provider -> provider.configure(builder -> builder
        .bulkheadConfig(BulkheadConfig.custom().maxConcurrentCalls(1).build())
        .threadPoolBulkheadConfig(ThreadPoolBulkheadConfig.ofDefaults()), "slowBulkhead");
}
```

除了配置创建的 Bulkhead 之外，您还可以在它们被创建之后但在返回给调用者之前自定义它们。 为此，您可以使用 addBulkheadCustomizer 和 addThreadPoolBulkheadCustomizer 方法。



```java
@Bean
public Customizer<Resilience4jBulkheadProvider> customizer() {
    return provider -> provider.addBulkheadCustomizer(bulkhead -> bulkhead.getEventPublisher()
        .onCallRejected(slowRejectedConsumer)
        .onCallFinished(slowFinishedConsumer), "slowBulkhead");
}
```



```java
@Bean
public Customizer<Resilience4jBulkheadProvider> slowThreadPoolBulkheadCustomizer() {
    return provider -> provider.addThreadPoolBulkheadCustomizer(threadPoolBulkhead -> threadPoolBulkhead.getEventPublisher()
        .onCallRejected(slowThreadPoolRejectedConsumer)
        .onCallFinished(slowThreadPoolFinishedConsumer), "slowThreadPoolBulkhead");
}
```

您可以在应用程序的配置属性文件中配置 ThreadPoolBulkhead 和 SemaphoreBulkhead 实例。 属性配置比 Java 定制器配置具有更高的优先级。

```java
resilience4j.thread-pool-bulkhead:
    instances:
        backendA:
            maxThreadPoolSize: 1
            coreThreadPoolSize: 1
resilience4j.bulkhead:
    instances:
        backendB:
            maxConcurrentCalls: 10
```

Spring Cloud Circuit Breaker Resilience4j 包括自动配置以设置指标收集，只要正确的依赖项位于类路径上。 要启用指标收集，您必须包含 org.springframework.boot:spring-boot-starter-actuator 和 io.github.resilience4j:resilience4j-micrometer。 有关存在这些依赖项时生成的指标的更多信息，请参阅 Resilience4j 文档。
