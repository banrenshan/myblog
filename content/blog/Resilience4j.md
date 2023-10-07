---
title: Resilience4j
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:54:32
---

# 简介

Resilience4j一个轻量级（只依赖Vavr第三方库）的，易于使用的容错框架，灵感来源于Netflix Hystrix，依托于java8的函数式编程。



Resilience4j提供了断路器、限速、重试、Bulkhead等功能。你可以任意选择和搭配这些功能。



Bulkhead(隔板模式)是一种容错的应用程序设计。 在隔板架构中，应用程序的元素被隔离到池中，因此，如果其中一个失败，则其他元素将继续运行。 它是根据船体的分段隔板（凸头）来命名的。 如果船体受损，则只有损坏的部分会充满水，从而防止船下沉。



以下示例显示了如何使用CircuitBreaker和Retry装饰lambda表达式，以便在发生异常时最多重试3次。

```java
// 创建默认的断路器
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendService");
// 创建默认的重试器，重拾3次，间隔为500ms
Retry retry = Retry.ofDefaults("backendService");
// 创建默认的隔离器
Bulkhead bulkhead = Bulkhead.ofDefaults("backendService");
//具体的业务调用
Supplier<String> supplier = () -> backendService.doSomething(param1, param2)

// 使用装扮器装扮函数
Supplier<String> decoratedSupplier = Decorators.ofSupplier(supplier)
  .withCircuitBreaker(circuitBreaker)
  .withBulkhead(bulkhead)
  .withRetry(retry)  
  .decorate();

// 执行函数，并设置后退函数
String result = Try.ofSupplier(decoratedSupplier)
  .recover(throwable -> "Hello from Recovery").get();

// 不使用装扮器，直接使用断路器调用函数
String result = circuitBreaker
  .executeSupplier(backendService::doSomething);

// 使用ThreadPoolBulkhead在另外的线程中异步运行
 ThreadPoolBulkhead threadPoolBulkhead = ThreadPoolBulkhead.ofDefaults("backendService");

// 设定超时机制，超时需要Scheduler来调度
ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(3);
TimeLimiter timeLimiter = TimeLimiter.of(Duration.ofSeconds(1));

CompletableFuture<String> future = Decorators.ofSupplier(supplier)
    .withThreadPoolBulkhead(threadPoolBulkhead)
    .withTimeLimiter(timeLimiter, scheduledExecutorService)
    .withCircuitBreaker(circuitBreaker)
    .withFallback(asList(TimeoutException.class, 
                         CallNotPermittedException.class, 
                         BulkheadFullException.class),  
                  throwable -> "Hello from Recovery")
    .get().toCompletableFuture();
```



您可以配置重试之间的等待间隔，还可以配置回退算法。

当所有重试均失败时，该示例使用Vavr的monad从异常中恢复并调用另一个lambda表达式作为后备。

# 模块

核心模块

- resilience4j-circuitbreaker: Circuit breaking
- resilience4j-ratelimiter: Rate limiting
- resilience4j-bulkhead: Bulkheading
- resilience4j-retry: Automatic retrying (sync and async)
- resilience4j-cache: Result caching
- resilience4j-timelimiter: Timeout handling

扩展模块

- resilience4j-retrofit: Retrofit adapter
- resilience4j-feign: Feign adapter
- resilience4j-consumer: Circular Buffer Event consumer
- resilience4j-kotlin: Kotlin coroutines support

框架模块

- resilience4j-spring-boot: Spring Boot Starter
- resilience4j-spring-boot2: Spring Boot 2 Starter
- resilience4j-ratpack: Ratpack Starter
- resilience4j-vertx: Vertx Future decorator

Reactive 模块

- resilience4j-rxjava2: Custom RxJava2 operators
- resilience4j-reactor: Custom Spring Reactor operators

Metrics modules

- resilience4j-micrometer: Micrometer Metrics exporter
- resilience4j-metrics: Dropwizard Metrics exporter
- resilience4j-prometheus: Prometheus Metrics exporter

# CircuitBreaker

断路器又三个正常状态：CLOSE，OPEN和HALF_OPEN ,两个特殊状态 DISABLED和FORCED_OPEN.

![img](Resilience4j/1621905009574-efb3e8d2-37b4-4c11-a31d-217d6ca40521.png)

断路器使用滑动窗口来存储和汇总调用结果，这个窗口可以基于计数或时间。基于计数的滑动窗口汇总了最近N个调用的结果。 基于时间的滑动窗口汇总了最近N秒的调用结果。

- 基于计数的滑动窗口由N个测量值的圆形数组实现。如果计数窗口大小为10，则圆形阵列始终具有10个测量值。滑动窗口以增量方式更新汇总结果。 当记录新的call结果时，将更新总汇总。 收回最旧的度量后，将从总聚合中减去该度量，然后重置存储桶。 （逐项扣除）
- 基于时间的滑动窗口是由N个部分汇总结果（存储桶）的圆形数组实现的。如果时间窗口大小为10秒，则圆形数组将始终具有10个部分汇总结果（存储桶）。 每个存储段都会汇总在某个纪元秒内发生的所有调用的结果。 圆形数组的头存储区存储当前纪元的call结果。 其他部分聚合存储前几秒的call结果。滑动窗口不会单独存储call结果（元组），而是以增量方式更新部分聚合（存储桶）和总聚合。当记录新的call结果时，总聚合将增量更新。 当最旧的存储桶被收回时，该存储桶的部分总聚合将从总聚合中减去，然后重置该存储桶（逐项扣除）。 `部分聚合` 由3个整数组成，以计算失败的call数，慢速call数和总call数。 一个长存储所有call的总持续时间。

## 故障率和慢速调用率阈值



当故障率等于或大于配置的阈值时，CircuitBreaker的状态将从“CLOSE”更改为“OPEN”。 例如，当超过50％的call失败时。默认情况下，所有异常均视为失败。 您可以定义应视为失败的异常列表。 除非忽略所有其他异常，否则所有其他异常均被视为成功。 也可以忽略异常，以使它们既不算作失败也不算成功。



当慢速呼叫的百分比等于或大于配置的阈值时，CircuitBreaker也会从CLOSED变为OPEN。 例如，当超过50％的call时间超过5秒。 这有助于减轻被调用系统的负担。



如果记录的call数量达到指定的要求，才会计算故障率和慢速呼叫率。 例如，如果所需call的最小数量为10，则必须至少记录10个call，然后才能计算出故障率。 如果仅评估了9个call，则即使所有9个call均失败，CircuitBreaker也不会跳闸。



当断路器打开时，返回CallNotPermittedException 异常。 经过一段等待时间后，CircuitBreaker状态从OPEN变为HALF_OPEN，并允许可配置数量的call以查看后端是否可用， 除非所有允许的call都已完成，否则将通过CallNotPermittedException拒绝进一步的call。



断路器支持另外两个特殊状态，即DISABLED（始终允许访问）和FORCED_OPEN（始终拒绝访问）。 在这两种状态下，不会生成任何断路器事件（状态转换除外），也不会记录任何度量。 退出这些状态的唯一方法是触发状态转换或重置断路器。



CircuitBreaker是线程安全的，如下所示：

- CircuitBreaker的状态存储在AtomicReference中
- CircuitBreaker使用原子操作来更新状态。
- 从“滑动窗口”记录call和读取快照是同步操作

如果有20个并发线程请求执行函数的许可，并且CircuitBreaker的状态关闭，则允许所有线程调用该功能。 即使滑动窗口的大小为15。滑动窗口也不意味着仅允许15个调用并发运行。 如果要限制并发线程的数量，请使用Bulkhead。 您可以组合使用Bulkhead和CircuitBreaker。



**单个线程的示意图**

![img](Resilience4j/1621905043342-a94cf788-6465-4725-8411-c9d235694a40.png)

**三个线程的示意图**

![img](Resilience4j/1621907196798-f3dc2a24-6363-48f5-a47b-cd4ca0728914.png)

## 如何使用

基于ConcurrentHashMap的CircuitBreakerRegistry，可提供线程安全性和原子性保证。 您可以使用CircuitBreakerRegistry来管理（创建和检索）CircuitBreaker实例。 您可以为所有CircuitBreaker实例创建一个具有全局默认CircuitBreakerConfig的CircuitBreakerRegistry，如下所示：

```java
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
```

### 断路器配置

您可以提供自定义的全局CircuitBreakerConfig。 为了创建定制的全局CircuitBreakerConfig，可以使用CircuitBreakerConfig构建器。 您可以使用构建器来配置以下属性。

| 配置的属性                                    | 默认值                     | 描述                                                         |
| --------------------------------------------- | -------------------------- | ------------------------------------------------------------ |
| failureRateThreshold                          | 50                         | 以百分比配置故障率阈值。当故障率等于或大于阈值时，CircuitBreaker转换为打开状态。 |
| slowCallRateThreshold                         | 100                        | 当call持续时间slowCallDurationThreshold时，CircuitBreaker会将call视为慢速call 当慢速call的百分比等于或大于阈值时，CircuitBreaker转换为打开状态 |
| slowCallDurationThreshold                     | 60000 [ms]                 | 当call的调用时间超过改值时，认为时慢速的                     |
| permittedNumberOfCalls InHalfOpenState        | 10                         | 配置CircuitBreaker半打开时允许的call数。                     |
| maxWaitDurationInHalfOpenState                | 0                          | 该时间用于控制CircuitBreaker在切换到打开之前可以保持在Half Open状态的最长时间。值0表示断路器将在HalfOpen状态下无限期等待，直到所有允许的call完成。 |
| slidingWindowType                             | COUNT_BASED                | 配置滑动窗口的类型，如果是 COUNT_BASED, 最后的 slidingWindowSize个call被记录和汇总；如果是 TIME_BASED, 最后slidingWindowSize秒数内的请求被记录和汇总 |
| slidingWindowSize                             | 100                        | 配置滑动窗口的大小，该窗口用于在CircuitBreaker关闭时记录call结果。 |
| minimumNumberOfCalls                          | 100                        | 配置CircuitBreaker可以计算错误率或慢速呼叫率之前所需的最小呼叫数（每个滑动窗口时段）。例如，如果minimumNumberOfCalls为10，则必须至少记录10个呼叫，然后才能计算失败率。如果仅记录了9个呼叫，则即使所有9个呼叫均失败，CircuitBreaker也不会转换为打开。 |
| waitDurationInOpenState                       | 60000 [ms]                 | 从断开到半开之前CircuitBreaker应该等待的时间。               |
| automaticTransition FromOpenToHalfOpenEnabled | false                      | 如果设置为true，则意味着CircuitBreaker将自动从打开状态转换为半打开状态，不需要调用即可触发该转换。 一旦waitDurationInOpenState通过，就会创建一个线程来监视CircuitBreakers的所有实例，以将其转换为HALF_OPEN。 而如果将其设置为false，则即使在传递了waitDurationInOpenState之后，也只有在进行调用时才发生向HALF_OPEN的转换。 这里的优点是没有线程监视所有CircuitBreakers的状态。 |
| recordExceptions                              | empty                      | 记录为故障的异常列表。 除非通过ignoreExceptions表明确忽略，否则任何与列表之一匹配或继承的异常都将视为失败。 如果指定ignoreExceptions例外列表，则所有其他例外都将视为成功，除非它们被显式忽略 |
| ignoreExceptions                              | empty                      | 忽略的异常列表，既不算作失败也不算成功。 从列表之一匹配或继承的任何异常都不会被视为失败或成功，即使该异常是recordExceptions其中的一部分 |
| recordException                               | throwable -> true 所有异常 | 一个自定义谓词，用于评估是否应将异常记录为失败。 如果异常应计为失败，则谓词必须返回true。  除非成功将ignoreExceptions异常显式忽略，否则应该算是成功 |
| ignoreException                               | throwable -> false 没有yi  | 一个自定义谓词，用于评估是否应忽略异常，并且该异常既不算作失败也不算成功。 如果应忽略异常，则谓词必须返回true。 如果异常应计为失败，则谓词必须返回false。 |

```java
// 自定义CircuitBreaker的配置
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
  .failureRateThreshold(50)
  .slowCallRateThreshold(50)
  .waitDurationInOpenState(Duration.ofMillis(1000))
  .slowCallDurationThreshold(Duration.ofSeconds(2))
  .permittedNumberOfCallsInHalfOpenState(3)
  .minimumNumberOfCalls(10)
  .slidingWindowType(SlidingWindowType.TIME_BASED)
  .slidingWindowSize(5)
  .recordException(e -> INTERNAL_SERVER_ERROR
                 .equals(getResponse().getStatus()))
  .recordExceptions(IOException.class, TimeoutException.class)
  .ignoreExceptions(BusinessException.class, OtherBusinessException.class)
  .build();

// 使用自定义配置创建CircuitBreakerRegistry
CircuitBreakerRegistry circuitBreakerRegistry = 
  CircuitBreakerRegistry.of(circuitBreakerConfig);

// 创建默认的断路器
CircuitBreaker circuitBreakerWithDefaultConfig = 
  circuitBreakerRegistry.circuitBreaker("name1");

// 自定义断路器
CircuitBreaker circuitBreakerWithCustomConfig = circuitBreakerRegistry
  .circuitBreaker("name2", circuitBreakerConfig);
```

创建由多个实例共享的配置:

```java
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
  .failureRateThreshold(70)
  .build();

circuitBreakerRegistry.addConfiguration("someSharedConfig", config);

CircuitBreaker circuitBreaker = circuitBreakerRegistry
  .circuitBreaker("name", "someSharedConfig");
```

覆盖默认配置:

```java
CircuitBreakerConfig defaultConfig = circuitBreakerRegistry
   .getDefaultConfig();

CircuitBreakerConfig overwrittenConfig = CircuitBreakerConfig
  .from(defaultConfig)
  .waitDurationInOpenState(Duration.ofSeconds(20))
  .build();
```

如果您不想使用CircuitBreakerRegistry管理CircuitBreaker实例，也可以直接创建实例:

```java
// Create a custom configuration for a CircuitBreaker
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
  .recordExceptions(IOException.class, TimeoutException.class)
  .ignoreExceptions(BusinessException.class, OtherBusinessException.class)
  .build();

CircuitBreaker customCircuitBreaker = CircuitBreaker
  .of("testName", circuitBreakerConfig);
```

您还可以使用其生成器方法创建CircuitBreakerRegistry：

```java
Map <String, String> circuitBreakerTags = Map.of("key1", "value1", "key2", "value2");

CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.custom()
    .withCircuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
    .addRegistryEventConsumer(new RegistryEventConsumer() {
        @Override
        public void onEntryAddedEvent(EntryAddedEvent entryAddedEvent) {
            // implementation
        }
        @Override
        public void onEntryRemovedEvent(EntryRemovedEvent entryRemoveEvent) {
            // implementation
        }
        @Override
        public void onEntryReplacedEvent(EntryReplacedEvent entryReplacedEvent) {
            // implementation
        }
    })
    .withTags(circuitBreakerTags)
    .build();

CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
```

如果要插入自己的Registry实现，则可以提供Interface RegistryStore的自定义实现，并使用builder方法插入:

```java
CircuitBreakerRegistry registry = CircuitBreakerRegistry.custom()
    .withRegistryStore(new YourRegistryStoreImplementation())
    .withCircuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
    .build():
```

### 使用断路器装扮函数并调用

您可以使用CircuitBreaker装饰任何Callable，Supplier，Runnable，Consumer，CheckedRunnable，CheckedSupplier，CheckedConsumer或CompletionStage。



您可以使用Vavr中的Try.of（...）或Try.run（...）来调用修饰的函数。 这允许使用map，flatMap，filter，recover或andThen链接其他功能。 仅当CircuitBreaker为CLOSED或HALF_OPEN时，才调用链接函数。



在下面的示例中，如果函数调用成功，则Try.of（...）返回Success <String> Monad。 如果函数引发异常，则返回Failure <Throwable> Monad，并且不调用map。

```java
// Given
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");

// When I decorate my function
CheckedFunction0<String> decoratedSupplier = CircuitBreaker
        .decorateCheckedSupplier(circuitBreaker, () -> "This can be any method which returns: 'Hello");

// and chain an other function with map
Try<String> result = Try.of(decoratedSupplier)
                .map(value -> value + " world'");

// Then the Try Monad returns a Success<String>, if all functions ran successfully.
assertThat(result.isSuccess()).isTrue();
assertThat(result.get()).isEqualTo("This can be any method which returns: 'Hello world'");
```

您可以在CircuitBreakerRegistry上注册事件使用者，并在每次创建，替换或删除CircuitBreaker时采取措施。

```java
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
circuitBreakerRegistry.getEventPublisher()
  .onEntryAdded(entryAddedEvent -> {
    CircuitBreaker addedCircuitBreaker = entryAddedEvent.getAddedEntry();
    LOG.info("CircuitBreaker {} added", addedCircuitBreaker.getName());
  })
  .onEntryRemoved(entryRemovedEvent -> {
    CircuitBreaker removedCircuitBreaker = entryRemovedEvent.getRemovedEntry();
    LOG.info("CircuitBreaker {} removed", removedCircuitBreaker.getName());
  });
```

CircuitBreakerEvent可以是状态转换，断路器重置，成功调用，记录的错误或忽略的错误。 所有事件都包含其他信息，例如事件创建时间和呼叫的处理持续时间。 如果要使用事件，则必须注册事件使用者。

```java
circuitBreaker.getEventPublisher()
    .onSuccess(event -> logger.info(...))
    .onError(event -> logger.info(...))
    .onIgnoredError(event -> logger.info(...))
    .onReset(event -> logger.info(...))
    .onStateTransition(event -> logger.info(...));
// Or if you want to register a consumer listening
// to all events, you can do:
circuitBreaker.getEventPublisher()
    .onEvent(event -> logger.info(...));
```

您可以使用CircularEventConsumer将事件存储在具有固定容量的循环缓冲区中。

```java
CircularEventConsumer<CircuitBreakerEvent> ringBuffer = 
  new CircularEventConsumer<>(10);
circuitBreaker.getEventPublisher().onEvent(ringBuffer);
List<CircuitBreakerEvent> bufferedEvents = ringBuffer.getBufferedEvents()
```

您可以使用RxJava或RxJava2适配器将EventPublisher转换为响应流。

您可以通过自定义实现覆盖内存RegistryStore。 例如，如果要使用在一定时间后删除未使用实例的缓存。

```java
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.custom()
  .withRegistryStore(new CacheCircuitBreakerRegistryStore())
  .build();
```

如果您想在CircuitBreaker将异常记录为失败之后从异常中恢复，则可以链接Try.recover（）方法。 仅当Try.of（）返回Failure <Throwable> Monad时，才调用恢复方法。

```java
// Given
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");

// When I decorate my function and invoke the decorated function
CheckedFunction0<String> checkedSupplier =
  CircuitBreaker.decorateCheckedSupplier(circuitBreaker, () -> {
    throw new RuntimeException("BAM!");
});
Try<String> result = Try.of(checkedSupplier)
        .recover(throwable -> "Hello Recovery");

// Then the function should be a success, 
// because the exception could be recovered
assertThat(result.isSuccess()).isTrue();
// and the result must match the result of the recovery function.
assertThat(result.get()).isEqualTo("Hello Recovery");
```

如果要在CircuitBreaker将异常记录为失败之前从异常中恢复，可以执行以下操作：

```java
Supplier<String> supplier = () -> {
            throw new RuntimeException("BAM!");
        };

Supplier<String> supplierWithRecovery = SupplierUtils
  .recover(supplier, (exception) -> "Hello Recovery");

String result = circuitBreaker.executeSupplier(supplierWithRecovery);

assertThat(result).isEqualTo("Hello Recovery");
```

SupplierUtils和CallableUtils包含其他方法，例如andThen，可以采用这些方法来链接函数。 例如，检查HTTP响应的状态代码，以便可以引发异常。

```java
Supplier<String> supplierWithResultAndExceptionHandler = SupplierUtils
  .andThen(supplier, (result, exception) -> "Hello Recovery");

Supplier<HttpResponse> supplier = () -> httpClient.doRemoteCall();
Supplier<HttpResponse> supplierWithResultHandling = SupplierUtils.andThen(supplier, result -> {
    if (result.getStatusCode() == 400) {
       throw new ClientException();
    } else if (result.getStatusCode() == 500) {
       throw new ServerException();
    }
    return result;
});
HttpResponse httpResponse = circuitBreaker
  .executeSupplier(supplierWithResultHandling);
```

断路器支持重置为原始状态，丢失所有指标并有效重置其滑动窗口。

```java
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
circuitBreaker.reset();
```

手动转换为状态:

```java
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
circuitBreaker.transitionToDisabledState();
// circuitBreaker.onFailure(...) won't trigger a state change
circuitBreaker.transitionToClosedState(); // will transition to CLOSED state and re-enable normal behaviour, keeping metrics
circuitBreaker.transitionToForcedOpenState();
// circuitBreaker.onSuccess(...) won't trigger a state change
circuitBreaker.reset(); //  will transition to CLOSED state and re-enable normal behaviour, losing metrics
```

# Bulkhead

Resilience4j 提供了两种方式实现舱壁模式来隔离线程的执行：

- SemaphoreBulkhead使用信号量
- FixedThreadPoolBulkhead 使用有界队列和固定大小的线程池

SemaphoreBulkhead在各种线程和I/O模型上都能很好地工作。 它基于信号量，与Hystrix不同，它不提供“影子”线程池选项。 由客户端来确保正确的线程池大小（与配置一致）。



跟CircuitBreaker 模块一样，Bulkhead也提供了基于内存的BulkheadRegistry 和ThreadPoolBulkheadRegistry ，你可以使用他们来管理（创建和检索）Bulkhead 实例。

```java
BulkheadRegistry bulkheadRegistry = BulkheadRegistry.ofDefaults();

ThreadPoolBulkheadRegistry threadPoolBulkheadRegistry = 
  ThreadPoolBulkheadRegistry.ofDefaults();
```

你可以自定义全局配置，使用BulkheadConfig 构建，相关的可配置属性如下：

| 属性               | 默认值 | 描述                                       |
| ------------------ | ------ | ------------------------------------------ |
| maxConcurrentCalls | 25     | 最大的并行执行数                           |
| maxWaitDuration    | 0      | 尝试进入饱和舱壁时，应阻塞线程的最长时间。 |

```java
// Create a custom configuration for a Bulkhead
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(150)
    .maxWaitDuration(Duration.ofMillis(500))
    .build();

// Create a BulkheadRegistry with a custom global configuration
BulkheadRegistry registry = BulkheadRegistry.of(config);

// Get or create a Bulkhead from the registry - 
// bulkhead will be backed by the default config
Bulkhead bulkheadWithDefaultConfig = registry.bulkhead("name1");

// Get or create a Bulkhead from the registry, 
// use a custom configuration when creating the bulkhead
Bulkhead bulkheadWithCustomConfig = registry.bulkhead("name2", custom);
```

ThreadPoolBulkhead配置的属性有所不同，使用ThreadPoolBulkheadConfig 构建配置项，属性列表如下：

| 配置属性           | 默认值                                          | 描述                                                 |
| ------------------ | ----------------------------------------------- | ---------------------------------------------------- |
| maxThreadPoolSize  | Runtime.getRuntime() .availableProcessors()     | 线程池的最大大小                                     |
| coreThreadPoolSize | Runtime.getRuntime() .availableProcessors() - 1 | 线程池的核心大小                                     |
| queueCapacity      | 100                                             | 等待队列的容量                                       |
| keepAliveDuration  | 20 [ms]                                         | 当线程数大于内核数时，这是多余的空闲线程最大空闲时间 |

```java
ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
  .maxThreadPoolSize(10)
  .coreThreadPoolSize(2)
  .queueCapacity(20)
  .build();
        
// Create a BulkheadRegistry with a custom global configuration
ThreadPoolBulkheadRegistry registry = ThreadPoolBulkheadRegistry.of(config);

// Get or create a ThreadPoolBulkhead from the registry - 
// bulkhead will be backed by the default config
ThreadPoolBulkhead bulkheadWithDefaultConfig = registry.bulkhead("name1");

// Get or create a Bulkhead from the registry, 
// use a custom configuration when creating the bulkhead
ThreadPoolBulkheadConfig custom = BulkheadConfig.custom()
  .maxThreadPoolSize(5)
  .build();

ThreadPoolBulkhead bulkheadWithCustomConfig = registry.bulkhead("name2", custom);
```

可以猜到，Bulkhead具有各种类似于CircuitBreaker的高阶装饰器功能。 您可以用隔板装饰任何Callable，Supplier，Runnable，Consumer，CheckedRunnable，CheckedSupplier，CheckedConsumer或CompletionStage。

```java
// Given
Bulkhead bulkhead = Bulkhead.of("name", config);

// When I decorate my function
CheckedFunction0<String> decoratedSupplier = Bulkhead
  .decorateCheckedSupplier(bulkhead, () -> "This can be any method which returns: 'Hello");

// and chain an other function with map
Try<String> result = Try.of(decoratedSupplier)
  .map(value -> value + " world'");

// Then the Try Monad returns a Success<String>, if all functions ran successfully.
assertThat(result.isSuccess()).isTrue();
assertThat(result.get()).isEqualTo("This can be any method which returns: 'Hello world'");
assertThat(bulkhead.getMetrics().getAvailableConcurrentCalls()).isEqualTo(1);
ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(10)
    .coreThreadPoolSize(2)
    .queueCapacity(20)
    .build();

ThreadPoolBulkhead bulkhead = ThreadPoolBulkhead.of("name", config);

CompletionStage<String> supplier = ThreadPoolBulkhead
    .executeSupplier(bulkhead, backendService::doSomething);
```

您可以在BulkheadRegistry上注册事件使用者，并在创建，替换或删除Bulkhead时执行操作。

```java
BulkheadRegistry registry = BulkheadRegistry.ofDefaults();
registry.getEventPublisher()
  .onEntryAdded(entryAddedEvent -> {
    Bulkhead addedBulkhead = entryAddedEvent.getAddedEntry();
    LOG.info("Bulkhead {} added", addedBulkhead.getName());
  })
  .onEntryRemoved(entryRemovedEvent -> {
    Bulkhead removedBulkhead = entryRemovedEvent.getRemovedEntry();
    LOG.info("Bulkhead {} removed", removedBulkhead.getName());
  });
```

BulkHead发出BulkHeadEvents流。 发出两种类型的事件：允许执行，拒绝执行和完成执行。 如果要使用这些事件，则必须注册一个事件使用者。

```java
bulkhead.getEventPublisher()
    .onCallPermitted(event -> logger.info(...))
    .onCallRejected(event -> logger.info(...))
    .onCallFinished(event -> logger.info(...));
```

# TimeLimiter

超时器，跟断路器一样，提供了基于内存的TimeLimiterRegistry ，可以使用该实例管理TimeLimiter 

 

TimeLimiterRegistry timeLimiterRegistry = TimeLimiterRegistry.ofDefaults();

使用TimeLimiterConfig自定义超时器的配置，可以配置的内容有两种：

\* 超时时间

\* 是否调用cancle

 

TimeLimiterConfig config = TimeLimiterConfig.custom()

  .cancelRunningFuture(true)

  .timeoutDuration(Duration.ofMillis(500))

  .build();

// Create a TimeLimiterRegistry with a custom global configuration

TimeLimiterRegistry timeLimiterRegistry = TimeLimiterRegistry.of(config);

// Get or create a TimeLimiter from the registry - 

// TimeLimiter will be backed by the default config

TimeLimiter timeLimiterWithDefaultConfig = registry.timeLimiter("name1");

// Get or create a TimeLimiter from the registry, 

// use a custom configuration when creating the TimeLimiter

TimeLimiterConfig config = TimeLimiterConfig.custom()

  .cancelRunningFuture(false)

  .timeoutDuration(Duration.ofMillis(1000))

  .build();

TimeLimiter timeLimiterWithCustomConfig = registry.timeLimiter("name2", config);

TimeLimiter具有较高阶的装饰器函数来装饰CompletionStage或Future，以限制执行时间。

 

// Given I have a helloWorldService.sayHelloWorld() method which takes too long

HelloWorldService helloWorldService = mock(HelloWorldService.class);

// Create a TimeLimiter

TimeLimiter timeLimiter = TimeLimiter.of(Duration.ofSeconds(1));

// The Scheduler is needed to schedule a timeout on a non-blocking CompletableFuture

ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(3);

// The non-blocking variant with a CompletableFuture

CompletableFuture<String> result = timeLimiter.executeCompletionStage(

  scheduler, () -> CompletableFuture.supplyAsync(helloWorldService::sayHelloWorld)).toCompletableFuture();

// The blocking variant which is basically future.get(timeoutDuration, MILLISECONDS)

String result = timeLimiter.executeFutureSupplier(

  () -> CompletableFuture.supplyAsync(() -> helloWorldService::sayHelloWorld));

# 缓存

以下示例显示如何使用Cache抽象装饰lambda表达式。 缓存抽象将lambda表达式的结果放入缓存实例（JCache）中，并在调用lambda表达式之前尝试从缓存中检索以前的缓存结果。 如果从分布式缓存中检索缓存失败，则会处理该异常并调用lambda表达式。

 

// Create a CacheContext by wrapping a JCache instance.

javax.cache.Cache<String, String> cacheInstance = Caching

  .getCache("cacheName", String.class, String.class);

Cache<String, String> cacheContext = Cache.of(cacheInstance);

// Decorate your call to BackendService.doSomething()

CheckedFunction1<String, String> cachedFunction = Decorators

   .ofCheckedSupplier(() -> backendService.doSomething())

   .withCache(cacheContext)

   .decorate();

String value = Try.of(() -> cachedFunction.apply("cacheKey")).get();

缓存发出CacheEvents流。 事件可以是高速缓存命中，高速缓存未命中或错误。

 

cacheContext.getEventPublisher()

   .onCacheHit(event -> logger.info(...))

   .onCacheMiss(event -> logger.info(...))

   .onError(event -> logger.info(...));

Ehcache 使用实例：

compile 'org.ehcache:ehcache:3.7.1'

 

// Configure a cache (once)

this.cacheManager = Caching.getCachingProvider().getCacheManager();

this.cache = Cache.of(cacheManager

   .createCache("booksCache", new MutableConfiguration<>()));

// Get books using a cache

List<Book> books = Cache.decorateSupplier(cache, library::getBooks)

   .apply(BOOKS_CACHE_KEY);

不建议在生产环境中使用JCache参考实现，因为它会导致一些并发问题。 使用Ehcache，Caffeine，Redisson，Hazelcast，Ignite或其他JCache实现。

# 重试

像断路器一样，提供了基于内存的RetryRegistry ：

```java
RetryRegistry retryRegistry = RetryRegistry.ofDefaults();
```



| 属性                      | 默认值                                                    | 描述                                                         |
| ------------------------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| maxAttempts               | 3                                                         | 最大尝试次数（首次调用也被算作一次）                         |
| waitDuration              | 500 [ms]                                                  | 重试之间的固定等待时间                                       |
| intervalFunction          | numOfAttempts -> waitDuration                             | 发生故障后修改等待间隔的功能。 默认情况下，等待时间保持不变。 |
| intervalBiFunction        | (numOfAttempts, Either<throwable, result) -> waitDuration | 根据尝试次数和结果或异常修改失败后的等待间隔的功能。 与intervalFunction一起使用时，将抛出IllegalStateException。 |
| retryOnResultPredicate    | result -> false                                           | 配置一个谓词，该谓词评估是否应重试结果。 如果要重试结果，则谓词必须返回true，否则必须返回false。 |
| retryOnExceptionPredicate | throwable -> true                                         | 配置一个谓词，该谓词评估是否应重试异常。 如果应重试异常，则谓词必须返回true，否则必须返回false。 |
| retryExceptions           | empty                                                     | 配置Throwable类的列表，这些类被记录为失败并因此被重试。 此参数支持子类型。  注意：如果使用的是Checked Exception，则必须使用CheckedSupplier |
| ignoreExceptions          | empty                                                     | 配置被忽略因而不会重试的Throwable类的列表。 此参数支持子类型。 |
| failAfterMaxRetries       | false                                                     | 当重试已达到配置的maxAttempts且结果仍未通过retryOnResultPredicate时，启用或禁用抛出MaxRetriesExceededException的布尔值 |

```java
RetryConfig config = RetryConfig.custom()
  .maxAttempts(2)
  .waitDuration(Duration.ofMillis(1000))
  .retryOnResult(response -> response.getStatus() == 500)
  .retryOnException(e -> e instanceof WebServiceException)
  .retryExceptions(IOException.class, TimeoutException.class)
  .ignoreExceptions(BusinessException.class, OtherBusinessException.class)
  .failAfterMaxAttempts(true)
  .build();
// Create a RetryRegistry with a custom global configuration
RetryRegistry registry = RetryRegistry.of(config);
// Get or create a Retry from the registry - 
// Retry will be backed by the default config
Retry retryWithDefaultConfig = registry.retry("name1");
// Get or create a Retry from the registry, 
// use a custom configuration when creating the retry
RetryConfig custom = RetryConfig.custom()
    .waitDuration(Duration.ofMillis(100))
    .build();
Retry retryWithCustomConfig = registry.retry("name2", custom);
```



可以猜到，重试具有各种高级装饰器功能，就像CircuitBreaker一样。 您可以使用重试装饰任何Callable，Supplier，Runnable，Consumer，CheckedRunnable，CheckedSupplier，CheckedConsumer或CompletionStage。

```java
// Given I have a HelloWorldService which throws an exception
HelloWorldService  helloWorldService = mock(HelloWorldService.class);
given(helloWorldService.sayHelloWorld())
  .willThrow(new WebServiceException("BAM!"));
// Create a Retry with default configuration
Retry retry = Retry.ofDefaults("id");
// Decorate the invocation of the HelloWorldService
CheckedFunction0<String> retryableSupplier = Retry
  .decorateCheckedSupplier(retry, helloWorldService::sayHelloWorld);
// When I invoke the function
Try<String> result = Try.of(retryableSupplier)
  .recover((throwable) -> "Hello world from recovery function");
// Then the helloWorldService should be invoked 3 times
BDDMockito.then(helloWorldService).should(times(3)).sayHelloWorld();
// and the exception should be handled by the recovery function
assertThat(result.get()).isEqualTo("Hello world from recovery function");
```



您可以在RetryRegistry上注册事件使用者，并在创建，替换或删除重试时执行操作。

```java
RetryRegistry registry = RetryRegistry.ofDefaults();
registry.getEventPublisher()
  .onEntryAdded(entryAddedEvent -> {
    Retry addedRetry = entryAddedEvent.getAddedEntry();
    LOG.info("Retry {} added", addedRetry.getName());
  })
  .onEntryRemoved(entryRemovedEvent -> {
    Retry removedRetry = entryRemovedEvent.getRemovedEntry();
    LOG.info("Retry {} removed", removedRetry.getName());
  });
```



如果不想在重试尝试之间使用固定的等待时间，则可以配置一个IntervalFunction，该函数用于计算每次尝试的等待时间。 Resilience4j提供了几种工厂方法来简化IntervalFunction的创建。

```java
IntervalFunction defaultWaitInterval = IntervalFunction
  .ofDefaults();
// This interval function is used internally 
// when you only configure waitDuration
IntervalFunction fixedWaitInterval = IntervalFunction
  .of(Duration.ofSeconds(5));
IntervalFunction intervalWithExponentialBackoff = IntervalFunction
  .ofExponentialBackoff();
IntervalFunction intervalWithCustomExponentialBackoff = IntervalFunction
  .ofExponentialBackoff(IntervalFunction.DEFAULT_INITIAL_INTERVAL, 2d);
IntervalFunction randomWaitInterval = IntervalFunction
  .ofRandomized();
// Overwrite the default intervalFunction with your custom one
RetryConfig retryConfig = RetryConfig.custom()
  .intervalFunction(intervalWithExponentialBackoff)
  .build();
```



# RateLimiter

速率限制是一项必不可少的技术，它可以为扩展您的API做好准备并建立服务的高可用性和可靠性。 而且，此技术还提供了很多不同的选择，例如如何处理检测到的限制超额，或您要限制哪种类型的请求。 您可以简单地拒绝此超限请求，或建立队列以稍后执行它们，或以某种方式组合这两种方法。



Resilience4j提供了一个RateLimiter，它将从纪元开始将所有纳秒级分为多个周期。 每个周期都有一个由RateLimiterConfig.limitRefreshPeriod配置的持续时间。 在每个周期的开始，RateLimiter会将活动许可的数量设置为RateLimiterConfig.limitForPeriod。



RateLimiter的默认实现是AtomicRateLimiter，它通过AtomicReference管理其状态。 AtomicRateLimiter.State是完全不可变的，并且具有以下字段：

\* activeCycle : 上次调用使用的cycle号

\* activePermissions :上次call后的可用权限计数。如果保留某些权限，则可以为负

\* nanosToWait :等待上一次call的等待时间（以纳秒为单位）

还有一个使用信号量的SemaphoreBasedRateLimiter和一个计划程序，该计划程序将在每个RateLimiterConfig＃limitRefreshPeriod之后刷新权限。

创建默认的速率器：

```java
RateLimiterRegistry rateLimiterRegistry = RateLimiterRegistry.ofDefaults();
```

RateLimiterConfig配置速率器：

| 属性               | 默认值   | 描述                                                         |
| ------------------ | -------- | ------------------------------------------------------------ |
| timeoutDuration    | 5 [s]    | 线程等待权限的默认等待时间                                   |
| limitRefreshPeriod | 500 [ns] | 限制刷新的时间段。 在每个时间段之后，速率限制器将其权限计数重新设置为limitForPeriod值 |
| limitForPeriod     | 50       | 一个限制刷新期间可用的权限数                                 |



```java
RateLimiterConfig config = RateLimiterConfig.custom()
  .limitRefreshPeriod(Duration.ofMillis(1))
  .limitForPeriod(10)
  .timeoutDuration(Duration.ofMillis(25))
  .build();
// Create registry
RateLimiterRegistry rateLimiterRegistry = RateLimiterRegistry.of(config);
// Use registry
RateLimiter rateLimiterWithDefaultConfig = rateLimiterRegistry
  .rateLimiter("name1");
RateLimiter rateLimiterWithCustomConfig = rateLimiterRegistry
  .rateLimiter("name2", config);
```

如您所料，RateLimiter具有各种类似于CircuitBreaker的高阶装饰器功能。 您可以使用RateLimiter装饰任何Callable，Supplier，Runnable，Consumer，CheckedRunnable，CheckedSupplier，CheckedConsumer或CompletionStage。

```java
// Decorate your call to BackendService.doSomething()
CheckedRunnable restrictedCall = RateLimiter
    .decorateCheckedRunnable(rateLimiter, backendService::doSomething);
Try.run(restrictedCall)
    .andThenTry(restrictedCall)
    .onFailure((RequestNotPermitted throwable) -> LOG.info("Wait before call it again :)"));
```



您可以使用changeTimeoutDuration和changeLimitForPeriod方法在运行时更改速率限制器参数。

新的超时时间不会影响当前正在等待许可的线程。

新的限制不会影响当前期限的权限，并且仅从下一个限制开始适用。

```java
// Decorate your call to BackendService.doSomething()
CheckedRunnable restrictedCall = RateLimiter
    .decorateCheckedRunnable(rateLimiter, backendService::doSomething);
// during second refresh cycle limiter will get 100 permissions
rateLimiter.changeLimitForPeriod(100);
```



您可以在RateLimiterRegistry上注册事件使用者，并在创建，替换或删除RateLimiter时执行操作。

```java
RateLimiterRegistry registry = RateLimiterRegistry.ofDefaults();
registry.getEventPublisher()
  .onEntryAdded(entryAddedEvent -> {
    RateLimiter addedRateLimiter = entryAddedEvent.getAddedEntry();
    LOG.info("RateLimiter {} added", addedRateLimiter.getName());
  })
  .onEntryRemoved(entryRemovedEvent -> {
    RateLimiter removedRateLimiter = entryRemovedEvent.getRemovedEntry();
    LOG.info("RateLimiter {} removed", removedRateLimiter.getName());
  });
```



RateLimiter发出RateLimiterEvents流。 事件可以是成功的许可获取或获取失败。

所有事件都包含其他信息，例如事件创建时间和速率限制器名称。

如果要使用事件，则必须注册事件使用者。

```java
rateLimiter.getEventPublisher()
    .onSuccess(event -> logger.info(...))
    .onFailure(event -> logger.info(...));
```



您可以通过自定义实现覆盖内存RegistryStore。 例如，如果要使用在一定时间后删除未使用实例的缓存。

```java
RateLimiterRegistry rateLimiterRegistry = RateLimiterRegistry.custom()
  .withRegistryStore(new CacheRateLimiterRegistryStore())
  .build();
```

# 和spring boot结合

## 依赖

```groovy
repositories {
    jCenter()
}

dependencies {
  compile "io.github.resilience4j:resilience4j-spring-boot2:${resilience4jVersion}"
  compile('org.springframework.boot:spring-boot-starter-actuator')
  compile('org.springframework.boot:spring-boot-starter-aop')
}
```

## 相关配置

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
            minimumNumberOfCalls: 20
            waitDurationInOpenState: 50s
            failureRateThreshold: 50
            eventConsumerBufferSize: 10
            recordFailurePredicate: io.github.robwin.exception.RecordFailurePredicate
            
resilience4j.retry:
    instances:
        backendA:
            maxAttempts: 3
            waitDuration: 10s
            enableExponentialBackoff: true
            exponentialBackoffMultiplier: 2
            retryExceptions:
                - org.springframework.web.client.HttpServerErrorException
                - java.io.IOException
            ignoreExceptions:
                - io.github.robwin.exception.BusinessException
        backendB:
            maxAttempts: 3
            waitDuration: 10s
            retryExceptions:
                - org.springframework.web.client.HttpServerErrorException
                - java.io.IOException
            ignoreExceptions:
                - io.github.robwin.exception.BusinessException
                
resilience4j.bulkhead:
    instances:
        backendA:
            maxConcurrentCalls: 10
        backendB:
            maxWaitDuration: 10ms
            maxConcurrentCalls: 20
            
resilience4j.thread-pool-bulkhead:
  instances:
    backendC:
      maxThreadPoolSize: 1
      coreThreadPoolSize: 1
      queueCapacity: 1
        
resilience4j.ratelimiter:
    instances:
        backendA:
            limitForPeriod: 10
            limitRefreshPeriod: 1s
            timeoutDuration: 0
            registerHealthIndicator: true
            eventConsumerBufferSize: 100
        backendB:
            limitForPeriod: 6
            limitRefreshPeriod: 500ms
            timeoutDuration: 3s
            
resilience4j.timelimiter:
    instances:
        backendA:
            timeoutDuration: 2s
            cancelRunningFuture: true
        backendB:
            timeoutDuration: 1s
            cancelRunningFuture: false
```

你可以覆盖默认配置、定义共享配置：

```yaml
resilience4j.circuitbreaker:
    configs:
        default:
            slidingWindowSize: 100
            permittedNumberOfCallsInHalfOpenState: 10
            waitDurationInOpenState: 10000
            failureRateThreshold: 60
            eventConsumerBufferSize: 10
            registerHealthIndicator: true
        someShared:
            slidingWindowSize: 50
            permittedNumberOfCallsInHalfOpenState: 10
    instances:
        backendA:
            baseConfig: default
            waitDurationInOpenState: 5000
        backendB:
            baseConfig: someShared
```

你还可以通过代码覆盖yaml中的配置：

```java
@Bean
public CircuitBreakerConfigCustomizer testCustomizer() {
    return CircuitBreakerConfigCustomizer
        .of("backendA", builder -> builder.slidingWindowSize(100));
}
```

Resilience4j 提供了自定义配置的类：

| Resilienc4j Type   | Instance Customizer class          |
| ------------------ | ---------------------------------- |
| Circuit breaker    | CircuitBreakerConfigCustomizer     |
| Retry              | RetryConfigCustomizer              |
| Rate limiter       | RateLimiterConfigCustomizer        |
| Bulkhead           | BulkheadConfigCustomizer           |
| ThreadPoolBulkhead | ThreadPoolBulkheadConfigCustomizer |
| Time Limiter       | TimeLimiterConfigCustomizer        |



## 注解

spring boot提供了注解支持，这些注解标注的方法的返回值可以是异步或同步的

Bulkhead注解的type属性可以指定采用何种类型的隔离器，默认是semaphore ：

```java
@Bulkhead(name = BACKEND, type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<String> doSomethingAsync() throws InterruptedException {
        Thread.sleep(500);
        return CompletableFuture.completedFuture("Test");
 }
@CircuitBreaker(name = BACKEND, fallbackMethod = "fallback")
@RateLimiter(name = BACKEND)
@Bulkhead(name = BACKEND)
@Retry(name = BACKEND, fallbackMethod = "fallback")
@TimeLimiter(name = BACKEND)
public Mono<String> method(String param1) {
    return Mono.error(new NumberFormatException());
}

private Mono<String> fallback(String param1, IllegalArgumentException e) {
    return Mono.just("test");
}

private Mono<String> fallback(String param1, RuntimeException e) {
    return Mono.just("test");
}
```

需要注意的是，fallback必须放在同个类中，并且有相同的方法签名并追加一个异常参数。



如果有多个fallback方法，则最近的方法会执行，例如上面的方法抛出 NumberFormatException，IllegalArgumentException方法会执行。



仅当多个方法具有相同的返回类型，并且您要一劳永逸地为它们定义相同的后备方法时，才可以使用异常参数定义一个全局后备方法。



### 注解的顺序

```
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( Function ) ) ) ) )
```



如果需要其他顺序，则必须使用功能链样式而不是Spring批注样式，或使用以下属性来显式设置外观顺序：

```yaml
- resilience4j.retry.retryAspectOrder
- resilience4j.circuitbreaker.circuitBreakerAspectOrder
- resilience4j.ratelimiter.rateLimiterAspectOrder
- resilience4j.timelimiter.timeLimiterAspectOrder
```

例如，让短路器在重试器后执行，你必须设置`retryAspectOrder` 的值大于`circuitBreakerAspectOrder` ：

```yaml
resilience4j:
  circuitbreaker:
    circuitBreakerAspectOrder: 1
  retry:
    retryAspectOrder: 2
```

## 指标端点

```
/actuator/metrics
{
    "names": [
        "resilience4j.circuitbreaker.calls",
        "resilience4j.circuitbreaker.buffered.calls",
        "resilience4j.circuitbreaker.state",
        "resilience4j.circuitbreaker.failure.rate"
        ]
}
```

获取具体的指标信息，`GET  /actuator/metrics/{metric.name}`,例如：`/actuator/metrics/resilience4j.circuitbreaker.calls`：

```json
{
    "name": "resilience4j.circuitbreaker.calls",
    "measurements": [
        {
            "statistic": "VALUE",
            "value": 3
        }
    ],
    "availableTags": [
        {
            "tag": "kind",
            "values": [
                "not_permitted",
                "successful",
                "failed"
            ]
        },
        {
            "tag": "name",
            "values": [
                "backendB",
                "backendA"
            ]
        }
    ]
}
```

如果想要使用 prometheus，需要添加依赖 io.micrometer:micrometer-registry-prometheus，端点是/actuator/prometheus



### health 端点

默认，CircuitBreaker 和 RateLimiter 的health端点是禁用的。当CircuitBreaker打开时，由于应用程序状态为DOWN，因此会禁用运行状况指示器。 这可能不是您想要实现的。

```yaml
management.health.circuitbreakers.enabled: true
management.health.ratelimiters.enabled: true

resilience4j.circuitbreaker:
  configs:
    default:
      registerHealthIndicator: true


resilience4j.ratelimiter:
  configs:
    default:
      registerHealthIndicator: true
```

闭合的CircuitBreaker状态映射为UP，打开状态映射为DOWN，半打开状态映射为UNKNOWN。

```json
{
  "status": "UP",
  "details": {
    "circuitBreakers": {
      "status": "UP",
      "details": {
        "backendB": {
          "status": "UP",
          "details": {
            "failureRate": "-1.0%",
            "failureRateThreshold": "50.0%",
            "slowCallRate": "-1.0%",
            "slowCallRateThreshold": "100.0%",
            "bufferedCalls": 0,
            "slowCalls": 0,
            "slowFailedCalls": 0,
            "failedCalls": 0,
            "notPermittedCalls": 0,
            "state": "CLOSED"
          }
        },
        "backendA": {
          "status": "UP",
          "details": {
            "failureRate": "-1.0%",
            "failureRateThreshold": "50.0%",
            "slowCallRate": "-1.0%",
            "slowCallRateThreshold": "100.0%",
            "bufferedCalls": 0,
            "slowCalls": 0,
            "slowFailedCalls": 0,
            "failedCalls": 0,
            "notPermittedCalls": 0,
            "state": "CLOSED"
          }
        }
      }
    }
  }
}
```



### events端点

发出的CircuitBreaker，Retry，RateLimiter，Bulkhead和TimeLimiter事件存储在单独的循环事件消费者缓冲区中。 可以在application.yml文件（eventConsumerBufferSize）中配置事件消费者缓冲区的大小。

端点 `/ctuator/circuitbreakers` 列出了所有CircuitBreaker实例的名称。 该端点也可用于Retry，RateLimiter，Bulkhead和TimeLimiter。

```json
{
    "circuitBreakers": [
      "backendA",
      "backendB"
    ]
}
```

端点/ actuator / circuitbreakerevents默认情况下列出所有CircuitBreaker实例的最近100个发出的事件。 该端点也可用于Retry，RateLimiter，Bulkhead和TimeLimiter。

```json
{
    "circuitBreakerEvents": [
        {
            "circuitBreakerName": "backendA",
            "type": "ERROR",
            "creationTime": "2017-01-10T15:39:17.117+01:00[Europe/Berlin]",
            "errorMessage": "org.springframework.web.client.HttpServerErrorException: 500 This is a remote exception",
            "durationInMs": 0
        },
        {
            "circuitBreakerName": "backendA",
            "type": "SUCCESS",
            "creationTime": "2017-01-10T15:39:20.518+01:00[Europe/Berlin]",
            "durationInMs": 0
        },
        {
            "circuitBreakerName": "backendB",
            "type": "ERROR",
            "creationTime": "2017-01-10T15:41:31.159+01:00[Europe/Berlin]",
            "errorMessage": "org.springframework.web.client.HttpServerErrorException: 500 This is a remote exception",
            "durationInMs": 0
        },
        {
            "circuitBreakerName": "backendB",
            "type": "SUCCESS",
            "creationTime": "2017-01-10T15:41:33.526+01:00[Europe/Berlin]",
            "durationInMs": 0
        }
    ]
}
```



# spring cloud 结合

将Spring Cloud 2 Starter of Resilience4j添加到您的编译依赖项中。

Spring Cloud 2 Starter允许您将Spring Cloud Config用作在运行时管理和刷新属性。

```java
repositories {
    jCenter()
}

dependencies {
    compile "io.github.resilience4j:resilience4j-spring-cloud2:${resilience4jVersion}"
    compile('org.springframework.boot:spring-boot-starter-actuator')
    compile('org.springframework.boot:spring-boot-starter-aop')
    compile('org.springframework.cloud:spring-cloud-starter-config')  
}
```

# 和feign结合

当前仅支持断路器、限流器和fallback.

## 装扮接口

Resilience4jFeign.builder是用于创建feign的容错实例的主要类。

它扩展了Feign.builder，添加自定义InvocationHandlerFactory。 Resilience4jFeign使用其自己的InvocationHandlerFactory来应用装饰器。 可以使用FeignDecorators类来构建装饰器。 可以组合多个装饰器。

以下示例显示如何使用RateLimiter和CircuitBreaker装饰feign接口：

```java
public interface MyService {
            @RequestLine("GET /greeting")
            String getGreeting();
            
            @RequestLine("POST /greeting")
            String createGreeting();
        }

CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendName");
RateLimiter rateLimiter = RateLimiter.ofDefaults("backendName");
FeignDecorators decorators = FeignDecorators.builder()
                                 .withRateLimiter(rateLimiter)
                                 .withCircuitBreaker(circuitBreaker)
                                 .build();
MyService myService = Resilience4jFeign.builder(decorators).target(MyService.class, "http://localhost:8080/");
```

调用MyService实例的任何方法将先调用CircuitBreaker，然后再调用RateLimiter。

如果这些机制之一生效，则将引发相应的RuntimeException，例如CircuitBreakerOpenException或RequestNotPermitted（提示：这些不会扩展FeignException类）。![img](Resilience4j/1621912949221-8935c2a6-f611-4ac4-8e98-f5666bbaa829.png)

## 装扮器的顺序

装饰器的应用顺序与声明它们的顺序相对应。

在构建FeignDecorators时，请注意这一点，因为顺序会影响最终的行为，这一点很重要。

```java
FeignDecorators decoratorsA = FeignDecorators.builder()
                                         .withCircuitBreaker(circuitBreaker)
                                         .withRateLimiter(rateLimiter)
                                         .build();
                                         
        FeignDecorators decoratorsB = FeignDecorators.builder()
                                         .withRateLimiter(rateLimiter)
                                         .withCircuitBreaker(circuitBreaker)
                                         .build();
```

使用decoratorsA时，将在CircuitBreaker之前调用RateLimiter。 这意味着即使CircuitBreaker打开，RateLimiter仍将限制呼叫速率。 decoratorsB应用相反的顺序。 这意味着一旦CircuitBreaker打开，RateLimiter将不再起作用。

## Fallback



可以定义在抛出异常时调用的fallback。 当HTTP请求失败时，也可能在FeignDecorators中的一个激活时，例如CircuitBreaker，就会发生异常。



```java
public interface MyService {
            @RequestLine("GET /greeting")
            String greeting();
        }

MyService requestFailedFallback = () -> "fallback greeting";
MyService circuitBreakerFallback = () -> "CircuitBreaker is open!";
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendName");
FeignDecorators decorators = FeignDecorators.builder()
                         .withFallback(requestFailedFallback, FeignException.class)
                         .withFallback(circuitBreakerFallback, CircuitBreakerOpenException.class)
                         .build();
MyService myService = Resilience4jFeign.builder(decorators).target(MyService.class, "http://localhost:8080/", fallback);
```

在此示例中，当抛出FeignException时（通常在HTTP请求失败时）将调用requestFailedFallback，而仅在CircuitBreakerOpenException情况下才调用circuitBreakerFallback。 检查FeignDecorators类，以了解更多过滤后备的方法。



所有回退必须实现在“目标”（Resilience4jFeign.Builder＃target）方法中声明的相同接口，否则将抛出IllegalArgumentException。

可以分配多个回退来处理相同的Exception，并在前一个失败时调用下一个回退。



如果需要，回退可以消耗抛出的Exception。 如果回退取决于异常可能具有不同的行为，或者仅记录异常，这将很有用。

请注意，将为抛出的每个异常实例化这种回退。



```java
public interface MyService {
            @RequestLine("GET /greeting")
            String greeting();
        }

        public class MyFallback implements MyService {
            private Exception cause;

            public MyFallback(Exception cause) {
                this.cause = cause;
            }

            public String greeting() {
                if (cause instanceOf FeignException) {
                    return "Feign Exception";
                } else {
                    return "Other exception";
                }
            }
        }

        FeignDecorators decorators = FeignDecorators.builder()
                .withFallbackFactory(MyFallback::new)
                .build();
```
