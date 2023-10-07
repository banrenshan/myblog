---
title: reactor
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:47:03
---

- 响应式编程是观察者设计模式的扩展
- 对比迭代器模式，一个重要区别是，迭代器基于拉模式，reactive streams基于推模式



使用迭代器是一种命令式编程，即必须通过Iterable访问值，这就需要开发者手动next()访问值。reactive streams采用的是Publisher-Subscriber：发布者在新的可用值到来时通知订阅者，这种基于推的模式是响应式编程的核心。此外，应用于推送值的操作是以声明的方式而不是命令的方式表达的，程序员表达计算的逻辑而不是描述其确切的控制流程。



 发布者可以向其订阅者推送新值（通过调用 onNext），但也可以发出错误（通过调用 onError）或完成（通过调用 onComplete信号。 错误和完成信号都会终止序列。 这可以总结如下：

```shell
onNext x 0..N [onError | onComplete]
```

reactor实现了两种Publisher：

- Flux: 0..N个item的响应式序列
- Mono: 0..1个item的响应式序列

这两种类型更多的是语义上的区别。例如，一个http请求仅产生一个响应，这时候使用Mono<HttpResponse>比 Flux<HttpResponse>更好，毕竟前者表示了0或1的意思。两者之间也可以相互转化，例如count操作符，Flux执行count返回 Mono<Long>



下面是Flux的基本弹珠图：

![img](reactor/1624260292438-28a6795b-8eff-4f19-b70f-faa31d89e4b5.svg+xml)

所有的事件，甚至终止事件，都是可选的：

- 没有onNext事件，只有OnComplete事件，表示一个空序列。
- 没有任何事件，表示一个空的无限序列。
- 同样，无限序列不一定为空。 例如，Flux.interval（Duration）产生Flux <Long>，它是无限的并从时钟发出规则的滴答声。



Mono<T>是一个特殊的Publisher<T>，它通过onNext信号最多发出一项，然后以onComplete信号（成功的Mono）终止，或仅发出一个onError信号（失败的Mono）。





# 订阅

```java
subscribe();  //订阅并触发序列

subscribe(Consumer<? super T> consumer);  //对每个产生的价值做一些事情。

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer); //处理值的同时也要对错误做出反应。

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer); //处理值和错误，但也会在序列成功完成时运行一些代码。

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer,
          Consumer<? super Subscription> subscriptionConsumer); //处理值、错误和成功完成，但也要对由此 subscribe 调用产生的 Subscription 做一些事情。
```

这些方法返回Disposable，您可以使用它取消订阅。 调用`dispose()` 方法，发出停止生成元素的信号。 但是，它不能保证是立即的：某些源可能会非常快地生成元素，以至于它们甚至可以在收到取消指令之前完成。



Disposables.swap() 创建了一个 Disposable 包装器，让您可以原子地取消和替换成新的Disposable。 例如，在 UI 场景中，您希望在用户单击按钮时取消请求并将其替换为新请求。



Disposables.composite(… )允许您收集多个 Disposable ，并在以后一次性处理所有这些请求。



```java
Flux<Integer> ints = Flux.range(1, 4);
ints.subscribe(i -> System.out.println(i),
    error -> System.err.println("Error " + error),
    () -> System.out.println("Done"),
    sub -> sub.request(10));
```

当我们订阅时，我们会收到一个Subscription。request(10)表示我们希望获取源中最多 10 个元素（实际上会发出 4 个元素就发出完成信号）。



## `BaseSubscriber`

还有一个额外的 subscribe 方法更通用，它需要一个完整的 Subscriber 而不是从 lambdas 中组合一个。 为了帮助编写这样的订阅者，我们提供了一个名为 BaseSubscriber 的可扩展类。

BaseSubscriber 的实例（或其子类）是一次性的，这意味着如果 BaseSubscriber实例订阅了第二个发布者，它会取消对第一个发布者的订阅。 这是因为使用一个实例使用两次会违反 Reactive Streams 规则，即不能并行调用 Subscriber 的 onNext 方法。

```java
public class SampleSubscriber<T> extends BaseSubscriber<T> {

	public void hookOnSubscribe(Subscription subscription) {
		System.out.println("Subscribed");
		request(1);
	}

	public void hookOnNext(T value) {
		System.out.println(value);
		request(1);
	}
}

SampleSubscriber<Integer> ss = new SampleSubscriber<Integer>();
Flux<Integer> ints = Flux.range(1, 4);
ints.subscribe(ss);
```

它还有额外的钩子：hookOnComplete、hookOnError、hookOnCancel 和 hookFinally（当序列终止时总是调用它， SignalType作为参数传入）

## 背压

在 Reactor 中实现背压时，消费者压力传播回源的方式是通过向上游Oprator发送request。 当前request的总和(即调用 `request(n)` 函数)被称为 `当前需求` 。默认 ，需求上限为 Long.MAX_VALUE，代表一个无限的请求（意思是“尽可能快地生产” — 基本上禁用背压）。



```java
Flux.range(1, 10)
    .doOnRequest(r -> System.out.println("request of " + r))
    .subscribe(new BaseSubscriber<Integer>() {

      @Override
      public void hookOnSubscribe(Subscription subscription) {
        request(1);
      }

      @Override
      public void hookOnNext(Integer integer) {
        System.out.println("Cancelling after having received " + integer);
        cancel();
      }
    });
```

结果打印：

```shell
request of 1
Cancelling after having received 1
```



需要记住的是，上游链中的每个操作符都可以重塑订阅级别表达的需求。 例如 buffer(N) 运算符：如果它接收到 request(2)的请求，则将其解释为对两个完整缓冲区的需求。 因此，由于缓冲区需要 N 个元素才能被视为已满，因此缓冲区运算符将request重塑为 2 x N。



您可能还注意到，某些运算符的变体采用称为 prefetch 的 int 输入参数。 这是另一类修改下游请求的操作符。 这些通常是处理内部序列的运算符，从每个传入元素（如 flatMap）派生一个发布者。prefetch 是一种调整对这些内部序列发出的初始请求的方法。 如果未指定，这些运算符中的大多数都以 32 的需求开始。

这些算子通常也实施补货优化：一旦算子看到 75% 的预取请求得到满足，它就会从上游重新请求 75%。 这是一种启发式优化，以便这些Oprator主动预测即将到来的请求。



最后，有几个运算符可以让您直接调整请求：limitRate 和 limitRequest:

- limitRate(N) 拆分下游请求，以便它们以较小的批次向上游传播。 例如，对 limitRate(10) 发出的 100 个请求将导致最多 10 个 10 个请求传播到上游。 请注意，在这种形式中，limitRate 实际上实现了前面讨论的补充优化。有一个变体，它也可以让您调整补给量（在变体中称为低潮）：limitRate(highTide, lowTide)。 选择 0 的低潮会导致严格的高潮请求批次，而不是由补货策略进一步返工的批次。
- 另一方面，limitRequest(N) 将下游请求限制为最大总需求。 它将请求加起来最多为 N。如果单个请求没有使总需求溢出 N，则该特定请求将完全向上游传播。 在源发出该数量后，limitRequest 认为序列已完成，向下游发送 onComplete 信号，并取消源。



# 手动创建序列

创建Flux的时候手动定义关联的事件（OnNext,OnError和onComplete），所有这些方法都有一个共同点，即它们公开了一个 API 来触发我们称为Sink(接收器)的事件。



## 同步`generate`

```java
Flux<String> flux = Flux.generate(
    () -> 0,  //Supplier<S>
    (state, sink) -> { //BiFunction<S, SynchronousSink<T>, S>
      sink.next("3 x " + state + " = " + 3*state); 
      if (state == 10) sink.complete(); 
      return state + 1; 
    });
```

结果：

```shell
3 x 0 = 0
3 x 1 = 3
3 x 2 = 6
3 x 3 = 9
3 x 4 = 12
3 x 5 = 15
3 x 6 = 18
3 x 7 = 21
3 x 8 = 24
3 x 9 = 27
3 x 10 = 30
```

同步且一对一发射，这意味着sink是`SynchronousSink` ，每次回调的时候，只能调用一次next()方法，`error(Throwable)` 或`complete()`,是可选的.



你还可以使用可变值：

```java
Flux<String> flux = Flux.generate(
    AtomicLong::new, 
    (state, sink) -> {
      long i = state.getAndIncrement(); 
      sink.next("3 x " + i + " = " + 3*i);
      if (i == 10) sink.complete();
      return state; 
    });
```

## 异步多线程 create

create 是一种更高级的 Flux 编程创建形式，适用于每轮多次发射，甚至来自多个线程。它公开了一个 FluxSink，以及它的 next、error 和 complete 方法。 与generate相反，它没有基于state的变体。 另一方面，它可以在回调中触发多线程事件。

```java
interface MyEventListener<T> {
    void onDataChunk(List<T> chunk);
    void processComplete();
}

Flux<String> bridge = Flux.create(sink -> {
    myEventProcessor.register( 
      new MyEventListener<String>() { 

        public void onDataChunk(List<String> chunk) {
          for(String s : chunk) {
            sink.next(s);   //多次调用
              
          }
        }

        public void processComplete() {
            sink.complete(); 
        }
    });
});
```

## 异步单线程push

push 是 generate 和 create 之间的中间地带，适用于处理来自单个生产者的事件。 它与 create 类似，因为它也可以是异步的，并且可以使用 create 支持的任何溢出策略来管理背压。 但是，一次只有一个生产线程可以调用 next、complete 或 error。

```java
Flux<String> bridge = Flux.push(sink -> {
    myEventProcessor.register(
      new SingleThreadEventListener<String>() { 

        public void onDataChunk(List<String> chunk) {
          for(String s : chunk) {
            sink.next(s); 
          }
        }

        public void processComplete() {
            sink.complete(); 
        }

        public void processError(Throwable e) {
            sink.error(e); 
        }
    });
});
```

对于多数的操作符，例如create,是混合的push/pull模型，大多数情况下是push，一小部分是pull，例如request(n);

消费者从源中pull数据，因为它在第一次请求之前不会发出任何东西。 当数据可用时，源将数据推送给消费者，但在其请求的数量范围内。

## Handle

handle 方法有点不同：它是一个实例方法，这意味着它链接在现有源上（与常见运算符一样）。 它存在于 Mono 和 Flux 中。

它接近于generate，因为它使用 SynchronousSink 并且只允许一对一。 但是，handle 可用于从每个源元素中生成任意值，可能会跳过某些元素。 通过这种方式，它可以作为 map 和 filter 的组合。 句柄签名如下：

```java
Flux<R> handle(BiConsumer<T, SynchronousSink<R>>);
```

让我们考虑一个例子。 规范不允许序列中出现空值。 如果您想执行map但又想使用预先存在的方法作为map函数，并且该方法有时返回 null，该怎么办？

```java
public String alphabet(int letterNumber) {
	if (letterNumber < 1 || letterNumber > 26) {
		return null;
	}
	int letterIndexAscii = 'A' + letterNumber - 1;
	return "" + (char) letterIndexAscii;
}
Flux<String> alphabet = Flux.just(-1, 30, 13, 9, 20)
    .handle((i, sink) -> {
        String letter = alphabet(i); 
        if (letter != null) 
            sink.next(letter); 
    });

alphabet.subscribe(System.out::println);
```

# 线程和调度

获得 Flux 或 Mono 并不一定意味着它在专用线程中运行。 相反，大多数运算符继续在前一个运算符执行的线程中工作。 除非指定，否则最顶层的运算符（源）本身运行在进行 subscribe() 调用的线程上。 以下示例在新线程中运行 Mono：

```java
public static void main(String[] args) throws InterruptedException {
  final Mono<String> mono = Mono.just("hello ");  //在main线程中创建

  Thread t = new Thread(() -> mono
      .map(msg -> msg + "thread ")
      .subscribe(v ->  //在Thread-0中进行subscribe
          System.out.println(v + Thread.currentThread().getName()) 
      )
  )
  t.start();
  t.join();

}
```

输出：

```shell
hello thread Thread-0
```

在reactor中，执行模型和执行发生的位置由使用的Scheduler决定。Scheduler 具有类似于 ExecutorService 的调度职责，但拥有专用的抽象可以让它做更多的事情。



Schedulers 类具有静态方法，可以访问以下执行上下文：

- 没有执行上下文（Schedulers.immediate()）：在处理时，提交的 Runnable 将被直接执行，有效地在当前 Thread 上运行它们（可以看作是一个“空对象”或 no-op Scheduler）。
- 单个可重用线程 (Scheduler.single())。请注意，此方法为所有调用者重用相同的线程，直到调度程序被释放。 如果您想要每次调用的专用线程，请为每次调用使用 Schedulers.newSingle()。
- 一个无界的弹性线程池（Schedulers.elastic()）。 随着 Schedulers.boundedElastic() 的引入，这个不再受欢迎，因为它倾向于隐藏背压问题并导致线程过多（见下文）。
- 有界弹性线程池 (Scheduler.boundedElastic())。 与其前身 elastic() 一样，它根据需要创建新的工作池并重用空闲的工作池。 闲置时间过长（默认为 60 秒）的工作池也会被处理掉。 与它的 elastic() 前身不同，它对它可以创建的支持线程的数量有一个上限（默认为 CPU 内核数 x 10）。 达到上限后提交的多达 100 000 个任务被排队，并在线程可用时重新调度（当调度延迟时，延迟在线程可用时开始）。 这是 I/O 阻塞工作的更好选择。 Schedulers.boundedElastic() 是一种方便的方法，可以为阻塞进程提供自己的线程，这样它就不会占用其他资源。 请参阅如何包装同步阻塞调用？，但不会对系统施加太大的新线程压力。
- 针对并行工作（Schedulers.parallel()）进行调整的固定工作线程池。 它会创建与 CPU 内核一样多的工作线程。



此外，您可以使用 Schedulers.fromExecutorService(ExecutorService) 从任何预先存在的 ExecutorService 中创建调度程序。 （您也可以从 Executor 创建一个，但不鼓励这样做。）



您还可以使用 newXXX 方法创建各种调度程序类型的新实例。 例如，Schedulers.newParallel(yourScheduleName) 创建一个名为 yourScheduleName 的新并行调度程序。



默认情况下，某些操作符使用 Schedulers 中的特定调度程序（并且通常会为您提供提供不同调度程序的选项）。 例如，调用 Flux.interval(Duration.ofMillis(300)) 工厂方法会产生一个每 300 毫秒滴答一次的 Flux<Long>。 默认情况下，这是由 Schedulers.parallel() 启用的。 以下行将 Scheduler 更改为类似于 Schedulers.single() 的新实例：

```java
Flux.interval(Duration.ofMillis(300), Schedulers.newSingle("test"))
```

Reactor 提供了两种在反应链中切换执行上下文（或调度器）的方法：publishOn 和 subscribeOn。 两者都采用调度程序并让您将执行上下文切换到该调度程序。 但是publishOn 在链中的位置很重要，而subscribeOn 的位置则无关紧要。 要了解这种差异，您首先必须记住，在您订阅之前什么都不会发生。



在 Reactor 中，当您链接运算符时，您可以根据需要将尽可能多的 Flux 和 Mono 实现相互包装。 订阅后，将创建一个 Subscriber 对象链，向后（沿链向上）到第一个发布者。 这实际上是对你隐藏的。 你所能看到的只是 Flux（或 Mono）和 Subscription 的外层，但这些中间特定运算符的订阅者才是真正的工作发生的地方。



## `publishOn`

在订阅者链的中，publishOn 的应用方式与任何其他运算符相同。 它从上游获取信号并在下游重放它们，同时从关联的调度程序对工作线程执行回调。 因此，它会影响后续操作符的执行位置（直到链接到另一个 publishOn），如下所示：

- 将执行上下文更改为调度程序选择的一个线程
- 根据规范，onNext 调用按顺序发生，因此这会占用一个线程
- 除非它们在特定的调度程序上工作，否则 publishOn 之后的操作符会继续在同一线程上执行

```java
Scheduler s = Schedulers.newParallel("parallel-scheduler", 4);  //1

final Flux<String> flux = Flux
    .range(1, 2)
    .map(i -> 10 + i)  //2
    .publishOn(s)  //3
    .map(i -> "value " + i);  //4

new Thread(() -> flux.subscribe(System.out::println)); 
```

1. 创建`Scheduler` ，里面包含4个线程
2. `map` 运行在 5中 创建的线程上
3. publishOn 在从 1 选取的线程上切换整个序列。 
4. 第二个映射在 1 的线程上运行。
5. 这个匿名线程是订阅发生的地方。 打印发生在最新的执行上下文中，即来自 publishOn 的上下文。



## `subscribeOn`

subscribeOn 适用于订阅过程，当反向链被构建时。 因此，无论您将 subscribeOn 放在链中的哪个位置，它始终会影响源发射的上下文。 然而，这并不影响随后对publishOn 的调用的行为—— 它们仍然为之后的链部分切换执行上下文。

- 更改整个运算符链订阅的线程
- 从调度程序中选择一个线程

```java
Scheduler s = Schedulers.newParallel("parallel-scheduler", 4); //1

final Flux<String> flux = Flux
    .range(1, 2)
    .map(i -> 10 + i)  //2
    .subscribeOn(s)  //3
    .map(i -> "value " + i);  //4

new Thread(() -> flux.subscribe(System.out::println)); //5
```

1. 创建一个由四个线程支持的新调度程序。
2. map在这四个线程之一上运行......
3. 因为 subscribeOn 从订阅时间 (<5>) 开始切换整个序列。
4. `map` 运行在相同线程（第一个map的线程）
5. 这个匿名线程是最初发生订阅的线程，但 subscribeOn 立即将其转移到四个调度程序线程之一。



完全并行怎么达到？？？？ publishOn



# 处理异常

Reactive Streams规范中，异常是终止事件。一旦发生错误，它就会停止序列并沿着运算符链向下传播到最后一步，即您定义的 Subscriber 及其 onError 方法。



如果未定义，onError 将抛出 UnsupportedOperationException。 您可以使用 Exceptions.isErrorCallbackNotImplemented 方法进一步检测和分类它。



Reactor 还提供了错误处理运算符。 以下示例显示了如何执行此操作：

```java
        Flux.just(1, 2, 0, 8)
                .map(i -> "100 / " + i + " = " + (100 / i)) //this triggers an error with 0
                .onErrorReturn("Divided by zero :(")
                .subscribe(System.err::println);
```

输出：

```shell
100 / 1 = 100
100 / 2 = 50
Divided by zero :(
```

在您了解错误处理运算符之前，您必须记住，反应序列中的任何错误都是一个终止事件。 即使使用了错误处理运算符，它也不会让原始序列继续。 相反，它将 onError 信号转换为新序列的开始。 换句话说，它替换了它上游的终止序列。

## 错误处理操作符

您可能熟悉在 try-catch 块中处理异常的几种方法。这些包括以下内容：

- 捕获并返回静态值
- 捕获然后执行fallback方法
- 捕获并动态计算fallback值
- 捕获，然后包装异常成**BusinessException，重新抛出**
- **捕获，记录日志，重新抛出**
- 使用 finally 块来清理资源或 Java 7 “try-with-resource” 构造。



以上所有在ractor中都有对应的运算符。



订阅时，链末尾的 onError 回调类似于 catch 块。 在那里，如果抛出异常，执行会跳到捕获，如下例所示：

```java
Flux<String> s = Flux.range(1, 10)
    .map(v -> doSomethingDangerous(v)) 
    .map(v -> doSecondTransform(v)); 
s.subscribe(value -> System.out.println("RECEIVED " + value), 
            error -> System.err.println("CAUGHT " + error) 
);
```

上面的例子等同于：

```java
try {
    for (int i = 1; i < 11; i++) {
        String v1 = doSomethingDangerous(i); 
        String v2 = doSecondTransform(v1); 
        System.out.println("RECEIVED " + v2);
    }
} catch (Throwable t) {
    System.err.println("CAUGHT " + t); 
}
```

### 静态 fallback值

捕获并返回静态值：

```java
try {
  return doSomethingDangerous(10);
}
catch (Throwable error) {
  return "RECOVERED";
}
```

等同于：

```java
Flux.just(10)
    .map(this::doSomethingDangerous)
    .onErrorReturn("RECOVERED");
```

还有一个扩展方法：

```java
Flux.just(10)
    .map(this::doSomethingDangerous)
    .onErrorReturn(e -> e.getMessage().equals("boom10"), "recovered10"); 
```

### fallback 方法

捕获然后执行fallback方法

```java
String v1;
try {
  v1 = callExternalService("key1");
}
catch (Throwable error) {
  v1 = getFromCache("key1");
}

String v2;
try {
  v2 = callExternalService("key2");
}
catch (Throwable error) {
  v2 = getFromCache("key2");
}
```

等同于：

```java
Flux.just("key1", "key2")
    .flatMap(k -> callExternalService(k) 
        .onErrorResume(e -> getFromCache(k)) 
    );
```

### 动态fallback值

```java
try {
  Value v = erroringMethod();
  return MyWrapper.fromValue(v);
}
catch (Throwable error) {
  return MyWrapper.fromError(error);
}
```

等效：

```java
erroringFlux.onErrorResume(error -> Mono.just( 
        MyWrapper.fromError(error) 
));
```

### 捕获并重新抛出

```java
try {
  return callExternalService(k);
}
catch (Throwable error) {
  throw new BusinessException("oops, SLA exceeded", error);
}
Flux.just("timeout1")
    .flatMap(k -> callExternalService(k))
    .onErrorResume(original -> Flux.error(
            new BusinessException("oops, SLA exceeded", original))
    );
```

等同：

```java
Flux.just("timeout1")
    .flatMap(k -> callExternalService(k))
    .onErrorMap(original -> new BusinessException("oops, SLA exceeded", original));
```

### 记录日志后抛出

```java
try {
  return callExternalService(k);
}
catch (RuntimeException error) {
  //make a record of the error
  log("uh oh, falling back, service failed for key " + k);
  throw error;
}
LongAdder failureStat = new LongAdder();
Flux<String> flux =
Flux.just("unknown")
    .flatMap(k -> callExternalService(k) 
        .doOnError(e -> {
            failureStat.increment();
            log("uh oh, falling back, service failed for key " + k); 
        })
        
    );
```

### finally

```java
Stats stats = new Stats();
stats.startTimer();
try {
  doSomethingDangerous();
}
finally {
  stats.stopTimerAndRecordTiming();
}
```

等同：

```java
Stats stats = new Stats();
LongAdder statsCancel = new LongAdder();

Flux<String> flux =
Flux.just("foo", "bar")
    .doOnSubscribe(s -> stats.startTimer())
    .doFinally(type -> { 
        stats.stopTimerAndRecordTiming();
        if (type == SignalType.CANCEL) 
          statsCancel.increment();
    })
    .take(1); 
```

对于“try-with-resource”,还可以使用using

```java
Flux<String> flux =
Flux.using(
        () -> disposableInstance, 
        disposable -> Flux.just(disposable.toString()), 
        Disposable::dispose 
);
```

### 重试

要记住的是，它通过重新订阅上游 Flux 来工作。 这确实是一个不同的序列，原始的仍然终止。 为了验证这一点，我们可以重用前面的示例并附加一个 retry(1) 来重试一次，而不是使用 onErrorReturn。 

```java
Flux.interval(Duration.ofMillis(250))
    .map(input -> {
        if (input < 3) return "tick " + input;
        throw new RuntimeException("boom");
    })
    .retry(1)
    .elapsed() //elapsed 将每个值与自上一个值发出以来的持续时间相关联。
    .subscribe(System.out::println, System.err::println); 

Thread.sleep(2100);
```

结果：

```shell
259,tick 0
249,tick 1
251,tick 2
506,tick 0 
248,tick 1
253,tick 2
java.lang.RuntimeException: boom
```

从前面的示例中可以看出，retry(1) 只是重新订阅了原始间隔一次，从 0 重新开始计时。第二次，由于异常仍然发生，它放弃向下游传播错误。



有一个更高级的重试版本（称为 retryWhen），它使用“伴随” Flux 来判断特定的失败是否应该重试。 这个伴随 Flux 由操作符创建但由用户修饰，以自定义重试条件。



伴随的 Flux 是一个 Flux<RetrySignal>，它被传递给一个 Retry 策略/函数，作为 retryWhen 的唯一参数提供。 作为用户，您定义该函数并使其返回一个新的 Publisher<?>。 Retry 类是一个抽象类，但如果您想使用简单的 lambda (Retry.from(Function)) 来转换同伴，它提供了一个工厂方法。

```java
Flux<String> flux = Flux
    .<String>error(new IllegalArgumentException()) 
    .doOnError(System.out::println) //发生异常时，打印异常
    .retryWhen(Retry.from(companion -> 
        companion.take(3))); //在这里，我们将前三个错误视为可重试（take(3)）
```



- 每次发生错误（提供重试的可能性）时，都会向伴随的 Flux 发出一个 RetrySignal，该 Flux 已由您的函数装饰。 这里有一个 Flux 可以鸟瞰到目前为止的所有尝试。 RetrySignal 可以访问错误及其周围的元数据。
- 如果伴随 Flux 发出一个值，则会发生重试。
- 如果伴随 Flux 完成，则错误被吞并，重试循环停止，结果序列也完成。
- 如果伴随 Flux 产生错误 (e)，则重试循环停止并且由此产生的序列错误为 e。

## 处理运算符或函数中的异常



通常，所有操作符本身都可以包含可能触发异常的代码，或者调用用户定义的回调也可能失败，因此它们都包含某种形式的错误处理。



根据经验，未经检查的异常总是通过 onError 传播。 例如，在 map 函数中抛出 RuntimeException 会转换为 onError 事件，如下面的代码所示：

```java
Flux.just("foo")
    .map(s -> { throw new IllegalArgumentException(s); })
    .subscribe(v -> System.out.println("GOT VALUE"),
               e -> System.out.println("ERROR: " + e));
```

然而，Reactor 定义了一组总是被认为是致命的异常（例如 OutOfMemoryError）。 请参阅 Exceptions.throwIfFatal 方法。 这些错误意味着 Reactor 无法继续运行并被抛出而不是传播。



如果您需要调用一些声明抛出异常的方法，您仍然必须在 try-catch 块中处理这些异常。 不过，您有多种选择：

- 捕获异常并从中恢复。 该序列继续正常。
- 捕获异常，将其包装成未经检查的异常，然后抛出（中断序列）。 Exceptions 实用程序类可以帮助您解决这个问题（我们接下来会讲到）。
- 如果您需要返回 Flux（例如，您在 flatMap 中），请将异常包装在产生错误的 Flux 中，如下所示：return Flux.error(checkedException)。 （该序列也终止。）

Reactor 有一个 Exceptions 实用程序类，您可以使用它来确保仅当异常是已检查异常时才包装异常：

- 如有必要，使用 Exceptions.propagate 方法包装异常。 它还首先调用 throwIfFatal 并且不包装 RuntimeException。
- 使用 Exceptions.unwrap 方法获取原始未包装异常（回到特定于反应器的异常层次结构的根本原因）。

```java
public String convert(int i) throws IOException {
    if (i > 3) {
        throw new IOException("boom " + i);
    }
    return "OK " + i;
}
```

现在假设您想在map中使用该方法。 您现在必须显式捕获异常，并且您的 map 函数不能重新抛出它。 因此，您可以将其作为 RuntimeException 传播到地图的 onError 方法，如下所示：

```java
Flux<String> converted = Flux
    .range(1, 10)
    .map(i -> {
        try { return convert(i); }
        catch (IOException e) { throw Exceptions.propagate(e); }
    });
```

稍后，当订阅前面的 Flux 并对错误做出反应时，如果您想为 IOExceptions 做一些特殊的事情，您可以恢复到原始异常。 以下示例显示了如何执行此操作：



```java
converted.subscribe(
    v -> System.out.println("RECEIVED: " + v),
    e -> {
        if (Exceptions.unwrap(e) instanceof IOException) {
            System.out.println("Something bad happened with I/O");
        } else {
            System.out.println("Something bad happened");
        }
    }
);
```

# Processors和Sinks

Processors 既是 `Publisher` 又是**Subscriber，**它们最初旨在作为中间步骤的可能表示，然后可以在 Reactive Streams 实现之间共享。 然而，在 Reactor 中，这些步骤更像是由 Publisher 运算符来表示。



第一次遇到 Processor 时的一个常见错误是试图从 Subscriber 接口直接调用公开的 onNext、onComplete 和 onError 方法。应该小心进行此类手动调用，尤其是关于 Reactive Streams 规范的调用外部同步。 Processors 实际上可能用处不大，除非遇到基于响应式流的 API，该 API 需要传递订阅者，而不是公开发布者。



sink通常是更好的选择。 在 Reactor 中，sink是一个允许安全手动触发信号的类。 它可以与订阅相关联（来自操作符内部）或完全独立。



## 使用 Sinks.One 和 Sinks.Many 从多线程安全生产



reactor-core 公开的 Sink 的默认风格确保检测到多线程使用，并且不会从下游订阅者的角度导致规范违规或未定义的行为。 使用 tryEmit* API 时，并行调用会快速失败。 当使用emit* API 时，提供的EmissionFailureHandler 可能允许重试争用（例如，忙循环），否则sink将因错误而终止。



这是对 Processor.onNext 的改进，它必须在外部同步，否则从下游订阅者的角度来看会导致未定义的行为。



Sinks 构建器为主要支持的生产者类型提供了一个引导式 API。 您将认识到 Flux 中的一些行为，例如 onBackpressureBuffer。

```java
Sinks.Many<Integer> replaySink = Sinks.many().replay().all();
```

多个生产者线程可以通过执行以下操作在sink上并发生成数据：

```java
//thread1
sink.emitNext(1, FAIL_FAST);

//thread2, later
sink.emitNext(2, FAIL_FAST);

//thread3, concurrently with thread 2
EmitResult result = sink.tryEmitNext(3); //would return FAIL_NON_SERIALIZED
```

Sinks.Many 可以作为 Flux 呈现给下游消费者，如下例所示：

```java
Flux<Integer> fluxView = replaySink.asFlux();
fluxView
	.takeWhile(i -> i < 10)
	.log()
	.blockLast();
```

类似地，可以使用 asMono() 方法将 Sinks.Empty 和 Sinks.One 风格视为 Mono。



sink类别是：

- many().multicast()：它只向订阅者传输新推送的数据，尊重他们的背压（在“订阅者订阅后”开始推送）。
- many().unicast()：与上面相同，但在第一个订阅者注册之前推送的数据被缓冲。
- many().replay()：它将向新订阅者重播特定历史大小的推送数据，然后继续实时推送新数据。
- one()：将向其订阅者播放单个元素的接收器
- empty()：一个只向其订阅者播放终端信号的接收器（错误或完成），但仍然可以被视为 Mono<T>（注意泛型类型 <T>）。



## 可用的Sink

##### Sinks.many().unicast().onBackpressureBuffer(args?)

单播 Sinks.Many 可以通过使用内部缓冲区来处理背压。 权衡是它最多可以有一个订阅者。

基本的单播接收器是通过 Sinks.many().unicast().onBackpressureBuffer() 创建的。 但是在 Sinks.many().unicast() 中有一些额外的单播静态工厂方法可以进行更好的调整。



例如，默认情况下，它是无限的：如果您在其订阅者尚未请求数据时通过它推送任何数量的数据，它会缓冲所有数据。 您可以通过为 Sinks.many().unicast().onBackpressureBuffer(Queue) factory 方法 中的内部缓冲提供自定义 Queue 实现来改变这一点，如果该队列是有界的，则当缓冲区已满并且没有收到足够的下游请求时，接收器可以拒绝推送值。



##### Sinks.many().multicast().onBackpressureBuffer(args?)

多播 Sinks.Many 可以向多个订阅者发送，同时为每个订阅者提供背压。 订阅者在订阅后只接收通过接收器推送的信号。



基本的多播接收器是通过 Sinks.many().multicast().onBackpressureBuffer() 创建的。

默认情况下，如果它的所有订阅者都被取消（这基本上意味着他们都取消了订阅），它会清除其内部缓冲区并停止接受新订阅者。 您可以通过在 Sinks.many().multicast() 下的多播静态工厂方法中使用 autoCancel 参数来调整它。



##### Sinks.many().multicast().directAllOrNothing()

具有简单处理背压的多播 Sinks.Many：如果任何订阅者太慢（零需求），则所有订阅者的 onNext 都会被丢弃。



但是，慢速订阅者不会终止，一旦慢速订阅者再次开始请求，所有将继续接收从那里推送的元素。

一旦 Sinks.Many 终止（通常通过调用其 emitError(Throwable) 或 emitComplete() 方法），它会让更多订阅者订阅但立即向他们重播终止信号。

##### Sinks.many().multicast().directBestEffort()

多播接收器。许多尽最大努力处理背压：如果订阅者太慢（零需求），则仅针对此慢速订阅者丢弃 onNext。

但是，慢速订阅者不会终止，一旦他们再次开始请求，他们将继续接收新推送的元素。



一旦 Sinks.Many 终止（通常通过调用其 emitError(Throwable) 或 emitComplete() 方法），它会让更多订阅者订阅但立即向他们重播终止信号。

##### Sinks.many().replay()

重播 Sinks.Many 缓存发出的元素并将它们重播给迟到的订阅者。

它可以在多种配置中创建：

- 缓存有限历史（Sinks.many().replay().limit(int)）或无限历史（Sinks.many().replay().all()）。
- 缓存基于时间的重播窗口 (Sinks.many().replay().limit(Duration))。
- 缓存历史大小和时间窗口的组合（Sinks.many().replay().limit(int, Duration)）。

还可以在 Sinks.many().replay() 下找到用于微调上述内容的其他重载，以及允许缓存单个元素的变体（latest() 和 latestOrDefault(T)）。



##### Sinks.unsafe().many()

高级用户和操作符构建者可能需要考虑使用 Sinks.unsafe().many() ，它将提供相同的 Sinks.Many 工厂，而无需额外的生产者线程安全。 因此，每个接收器的开销将更少，因为线程安全接收器必须检测多线程访问。



库开发人员不应该公开不安全的接收器，而是可以在受控调用环境中在内部使用它们，在那里他们可以确保导致 onNext、onComplete 和 onError 信号的调用的外部同步，这与 Reactive Streams 规范有关。

##### Sinks.one()

该方法直接构造了一个简单的 Sinks.One<T> 实例。 这种 Sink 的风格可以作为 Mono 来查看（通过它的 asMono() 视图方法），并且有稍微不同的发射方法来更好地传达这种类似 Mono 的语义：

- emitValue(T value) 生成一个 onNext(value) 信号并且 - 在大多数实现中 - 也会触发一个隐式的 onComplete()
- emitEmpty() 生成一个孤立的 onComplete() 信号，旨在生成一个空 Mono 的等效信号
- emitError(Throwable t) 生成一个 onError(t) 信号

Sinks.one() 接受这些方法中的任何一个的调用，有效地生成一个 Mono ，它要么用一个值完成，要么完成为空或失败。



##### Sinks.empty()

该方法直接构造了一个简单的 Sinks.Empty<T> 实例。 这种类型的 Sinks 类似于 Sinks.One<T>，只是它不提供 emitValue 方法。

因此，它只能生成一个完成为空或失败的 Mono。



尽管无法触发 onNext，但接收器仍然使用通用 <T> 进行类型化，因为它允许轻松组合并包含在需要特定类型的运算符链中。



# 测试

三个核心类：

- 使用 StepVerifier 逐步测试序列是否遵循给定场景。
- 生成数据以使用 TestPublisher 测试下游运算符（包括您自己的运算符）的行为。
- 在可以通过多个替代发布者的序列中（例如，使用 switchIfEmpty 的链，探测这样的发布者以确保它被使用（即订阅）。



## 使用StepVerifier进行场景测试



```java
public <T> Flux<T> appendBoomError(Flux<T> source) {
  return source.concatWith(Mono.error(new IllegalArgumentException("boom")));
}
```

我希望这个 Flux 首先发出 thing1，然后发出 thing2，然后产生一个错误消息，boom。 订阅并验证这些期望。

```java
@Test
public void testAppendBoomError() {
  Flux<String> source = Flux.just("thing1", "thing2"); 

  StepVerifier.create( 
    appendBoomError(source)) 
    .expectNext("thing1") 
    .expectNext("thing2")
    .expectErrorMessage("boom") 
    .verify(); 
}
```

`StepVerifier` 从create方法开始，传入待测试的序列，然后提供一系列方法：



- 表达对接下来出现的信号的期望。 如果接收到任何其他信号（或信号的内容与预期不符），则整个测试失败并显示有意义的 AssertionError。 例如，您可以使用 expectNext(T… ) 和 expectNextCount(long)。
- 消费下一个信号。 当您想跳过序列的一部分或想对信号的内容应用自定义断言时（例如，检查是否存在 onNext 事件并断言发出的项目是大小列表 5）。 例如，您可以使用consumeNextWith(Consumer<T>)。
- 采取杂项操作，例如暂停或运行任意代码。 例如，如果您想操作特定于测试的状态或上下文。 为此，您可以使用 thenAwait(Duration) 和 then(Runnable)。



对于terminal 事件，相应的期望方法（expectComplete() 和 expectError() 及其所有变体）切换到您无法再表达期望的 API。 在最后一步中，您所能做的就是在 StepVerifier 上执行一些额外的配置，然后触发验证，通常使用 verify() 或其变体之一。



默认情况下，verify() 方法和派生的快捷方法（verifyThenAssertThat、verifyComplete() 等）没有超时。 他们可以无限期地阻塞。 您可以使用 StepVerifier.setDefaultTimeout(Duration) 为这些方法全局设置超时，或者使用 verify(Duration) 在每次调用的基础上指定一个超时。



### 更好地识别测试失败



StepVerifier 提供了两个选项来更好地准确识别导致测试失败的期望步骤：

- as(String)：在大多数 expect* 方法之后使用，用于对前面的期望进行描述。 如果期望失败，则其错误消息包含描述。 终端期望和验证不能这样描述。
- StepVerifierOptions.create().scenarioName(String)：通过使用 StepVerifierOptions 创建您的 StepVerifier，您可以使用 sceneName 方法为整个场景命名，该名称也用于断言错误消息中。

请注意，在这两种情况下，只有在 StepVerifier 方法产生自己的 AssertionError 时才保证在消息中使用描述或名称（例如，手动或通过 assertNext 中的断言库抛出异常不会将描述或名称添加到 错误消息）。



## 操纵时间

您可以将 StepVerifier 与基于时间的运算符一起使用，以避免相应测试的运行时间过长。 您可以通过 StepVerifier.withVirtualTime 构建器执行此操作。

```java
StepVerifier.withVirtualTime(() -> Mono.delay(Duration.ofDays(1)))
```

此虚拟时间功能插入 Reactor 的调度器工厂中的自定义调度器。 由于这些定时运算符通常使用默认的 Schedulers.parallel() 调度程序，因此将其替换为 VirtualTimeScheduler 即可。 然而，一个重要的先决条件是在激活虚拟时间调度程序后实例化操作符。



为了增加这种情况正确发生的机会，StepVerifier 不会将简单的 Flux 作为输入。 withVirtualTime 需要一个操作符，它会引导您在完成调度程序设置后懒惰地创建测试通量的实例。

有两种处理时间的期望方法，它们在有或没有虚拟时间的情况下都是有效的：

- **thenAwait(Duration)：**暂停对步骤的评估（允许出现一些信号或延迟用完）。
- **expectNoEvent(Duration)：**还让序列播放给定的持续时间，但如果在此期间出现任何信号，则测试失败。



expectNoEvent 也将订阅视为一个事件。 如果您将其用作第一步，它通常会失败，因为检测到订阅信号。 使用 expectSubscription().expectNoEvent(duration) 代替。



这两种方法在经典模式下暂停线程给定的持续时间，并在虚拟模式下推进虚拟时钟。为了快速评估我们上面 Mono.delay 的行为，我们可以按如下方式编写代码：



```java
StepVerifier.withVirtualTime(() -> Mono.delay(Duration.ofDays(1)))
    .expectSubscription() 
    .expectNoEvent(Duration.ofDays(1)) 
    .expectNext(0L) 
    .verifyComplete();
```

我们可以使用上面的 thenAwait(Duration.ofDays(1))，但是 expectNoEvent 的好处是可以保证没有比它应该发生的更早发生。

请注意，verify() 返回一个 Duration 值。 这是整个测试的实时持续时间。



## 使用 StepVerifier 执行执行后断言

在描述了您的场景的最终期望后，您可以切换到补充断言 API，而不是触发 verify()。 为此，请改用 verifyThenAssertThat()。

verifyThenAssertThat() 返回一个 StepVerifier.Assertions 对象，一旦整个场景成功播放，您就可以使用它来断言一些状态元素（因为它还调用了 verify()）。 典型的（尽管是高级的）用法是捕获被某些操作符删除的元素并断言它们（参见 Hooks 部分）。

## 测试上下文

StepVerifier 对 Context 的传播有一些期望：



- **expectAccessibleContext**：返回一个`ContextExpectations` 对象，您可以使用该对象在传播的上下文上设置期望。 一定要调用 then() 返回到序列期望的集合。
- **expectNoAccessibleContext：**设置一个期望，即 NO Context 可以在被测操作符链上传播。 当被测发布者不是 Reactor 或没有任何可以传播上下文的操作符（例如，生成器源）时，这很可能发生。



此外，您可以通过使用 StepVerifierOptions 创建验证器，将特定于测试的初始上下文关联到 StepVerifier。

```java
StepVerifier.create(Mono.just(1).map(i -> i + 10),
				StepVerifierOptions.create().withInitialContext(Context.of("thing1", "thing2"))) 
		            .expectAccessibleContext() 
		            .contains("thing1", "thing2") 
		            .then() 
		            .expectNext(11)
		            .verifyComplete();
```

## 使用 TestPublisher 手动发射





## 使用 PublisherProbe 检查执行路径





# 调试



## reactor 跟踪栈

```java
java.lang.IndexOutOfBoundsException: Source emitted more than one item
	at reactor.core.publisher.MonoSingle$SingleSubscriber.onNext(MonoSingle.java:129)
	at reactor.core.publisher.FluxFlatMap$FlatMapMain.tryEmitScalar(FluxFlatMap.java:445)
	at reactor.core.publisher.FluxFlatMap$FlatMapMain.onNext(FluxFlatMap.java:379)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:121)
	at reactor.core.publisher.FluxRange$RangeSubscription.slowPath(FluxRange.java:154)
	at reactor.core.publisher.FluxRange$RangeSubscription.request(FluxRange.java:109)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.request(FluxMapFuseable.java:162)
	at reactor.core.publisher.FluxFlatMap$FlatMapMain.onSubscribe(FluxFlatMap.java:332)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onSubscribe(FluxMapFuseable.java:90)
	at reactor.core.publisher.FluxRange.subscribe(FluxRange.java:68)
	at reactor.core.publisher.FluxMapFuseable.subscribe(FluxMapFuseable.java:63)
	at reactor.core.publisher.FluxFlatMap.subscribe(FluxFlatMap.java:97)
	at reactor.core.publisher.MonoSingle.subscribe(MonoSingle.java:58)
	at reactor.core.publisher.Mono.subscribe(Mono.java:3096)
	at reactor.core.publisher.Mono.subscribeWith(Mono.java:3204)
	at reactor.core.publisher.Mono.subscribe(Mono.java:3090)
	at reactor.core.publisher.Mono.subscribe(Mono.java:3057)
	at reactor.core.publisher.Mono.subscribe(Mono.java:3029)
	at reactor.guide.GuideTests.debuggingCommonStacktrace(GuideTests.java:995)
```

我们可能会假设source时一个FLux或Mono，正如下一行提到的 MonoSingle 所证实的那样。 参考 Mono#single 运算符的 Javadoc，我们看到 single 有一个约定：源必须只发出一个元素。 看来我们有一个发出不止一个的源，因此违反了该合同。

我们能否深入挖掘并确定该来源？ 以下几行不是很有帮助。通过浏览这些行，我们至少可以开始形成出错链的图片：它似乎涉及一个 MonoSingle、一个 FluxFlatMap 和一个 FluxRange（每个在跟踪中都有几行，但总的来说这三个 涉及类）。 那么 range().flatMap().single() 链可能吗？

但是如果我们在应用程序中大量使用这种模式呢？ 这个还是没有告诉我们太多，单纯搜索single是不会发现问题的。 然后最后一行引用了我们的一些代码。 终于，我们越来越近了。



不过，等一下。 当我们转到源文件时，我们看到的只是订阅了一个预先存在的 Flux，如下所示：

```java
toDebug.subscribe(System.out::println, Throwable::printStackTrace);
```

所有这一切都发生在订阅时，但 Flux 本身并没有在那里声明。 更糟糕的是，当我们转到声明变量的位置时，我们会看到以下内容：

```java
public Mono<String> toDebug; //please overlook the public class attribute
```

变量在声明的地方没有被实例化。 我们必须假设最坏的情况，我们发现可能有几个不同的代码路径在应用程序中设置它。 我们仍然不确定是哪一个导致了问题。



我们想要更容易地找出操作符添加到链中的位置 - 即 Flux 被声明的位置。 我们通常将其称为 Flux 的“组装”



## 激活调试模式 - 又名回溯



尽管堆栈跟踪仍然能够为有一定经验的人传达一些信息，但我们可以看到，在更高级的情况下，它本身并不理想。

幸运的是，Reactor 带有专为调试而设计的汇编时检测。

这是通过在应用程序启动时自定义 Hooks.onOperator 钩子来完成的（或至少在被指控的 Flux 或 Mono 可以被实例化之前），如下所示：

```java
Hooks.onOperatorDebug();
```

这开始通过包装运算符的构造并在那里捕获堆栈跟踪来检测对 Flux（和 Mono）运算符方法的调用（它们被组装到链中）。 由于这是在声明操作符链时完成的，因此应在此之前激活钩子，因此最安全的方法是在应用程序开始时激活它。



稍后，如果发生异常，失败的运算符能够引用该捕获并将其附加到堆栈跟踪中。 我们将此捕获的程序集信息称为回溯。

在下一节中，我们将看到堆栈跟踪有何不同以及如何解释这些新信息。



## 阅读堆栈信息



当我们重用初始示例但激活 operatorStacktrace 调试功能时，堆栈跟踪如下：

```java
java.lang.IndexOutOfBoundsException: Source emitted more than one item
	at reactor.core.publisher.MonoSingle$SingleSubscriber.onNext(MonoSingle.java:129)
	at reactor.core.publisher.FluxOnAssembly$OnAssemblySubscriber.onNext(FluxOnAssembly.java:375) <1>
...
<2>
...
	at reactor.core.publisher.Mono.subscribeWith(Mono.java:3204)
	at reactor.core.publisher.Mono.subscribe(Mono.java:3090)
	at reactor.core.publisher.Mono.subscribe(Mono.java:3057)
	at reactor.core.publisher.Mono.subscribe(Mono.java:3029)
	at reactor.guide.GuideTests.debuggingActivated(GuideTests.java:1000)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: <3>
Assembly trace from producer [reactor.core.publisher.MonoSingle] : <4>
	reactor.core.publisher.Flux.single(Flux.java:6676)
	reactor.guide.GuideTests.scatterAndGather(GuideTests.java:949)
	reactor.guide.GuideTests.populateDebug(GuideTests.java:962)
	org.junit.rules.TestWatcher$1.evaluate(TestWatcher.java:55)
	org.junit.rules.RunRules.evaluate(RunRules.java:20)
Error has been observed by the following operator(s): <5>
	|_	Flux.single ⇢ reactor.guide.GuideTests.scatterAndGather(GuideTests.java:949) <6>
```

1. 这是新的：我们看到了捕获堆栈的包装器运算符。
2. 除此之外，堆栈跟踪的第一部分在大多数情况下仍然相同，显示了一些运算符的内部结构（因此我们在这里删除了一些代码段）。
3. 这是回溯开始出现的地方。
4. 首先，我们获得了有关操作符在哪里组装的一些详细信息。
5. 我们还可以在错误通过操作符链传播时从第一个到最后一个（错误站点到订阅站点）获得错误的回溯。
6. 每个看到错误的操作符都与用户类和使用它的行一起被提及。



捕获的堆栈跟踪作为抑制的 OnAssemblyException 附加到原始错误。 它有两个部分，但第一部分是最有趣的。 它显示了触发异常的操作符的构造路径。 在这里，它表明导致我们问题的single 是在 scatterAndGather 方法中创建的，该方法本身是从通过 JUnit 执行的 populateDebug 方法调用的。



现在我们有了足够的信息来找到罪魁祸首，我们可以对 scatterAndGather 方法进行有意义的研究：

```java
private Mono<String> scatterAndGather(Flux<String> urls) {
    return urls.flatMap(url -> doRequest(url))
           .single(); 
}
```

现在我们可以看到错误的根本原因是一个 flatMap 对几个 URL 执行了多次 HTTP 调用，但是它用 single 链接，这太严格了。 在简短的 git blame 和与该行作者的快速讨论之后，我们发现他打算使用限制较少的 take(1) 来代替。



# 高级特性

## 组合运算符

从干净代码的角度来看，代码重用通常是一件好事。 Reactor 提供了一些模式，可以帮助您重用和共享代码，特别是对于您可能希望在代码库中定期应用的运算符或运算符组合。 如果您将一系列运算符视为一个配方，则可以创建一个运算符配方的“食谱”。



### `transform`

转换运算符让您可以将运算符链的一部分封装到一个函数中。 该函数在汇编时应用于原始运算符链，以使用封装的运算符对其进行扩充。 这样做对序列的所有订阅者应用相同的操作，基本上等同于直接链接操作符。 以下代码显示了一个示例：

```java
Function<Flux<String>, Flux<String>> filterAndMap =
f -> f.filter(color -> !color.equals("orange"))
      .map(String::toUpperCase);

Flux.fromIterable(Arrays.asList("blue", "green", "orange", "purple"))
	.doOnNext(System.out::println)
	.transform(filterAndMap)
	.subscribe(d -> System.out.println("Subscriber to Transformed MapAndFilter: "+d));
```

下图显示了变换运算符如何封装流：



![img](reactor/1624517098391-7f335e57-ef21-4d0f-969d-fa19d5b0b102.png)

结果：

```shell
blue
Subscriber to Transformed MapAndFilter: BLUE
green
Subscriber to Transformed MapAndFilter: GREEN
orange
purple
Subscriber to Transformed MapAndFilter: PURPLE
```

#### `transformDeferred`

transformDeferred 运算符与transform 类似，还允许您将运算符封装在函数中。 主要区别在于，此功能是基于每个订阅者应用于原始序列的。 这意味着该函数实际上可以为每个订阅生成不同的操作符链（通过维护一些状态）。 以下代码显示了一个示例：

```java
AtomicInteger ai = new AtomicInteger();
Function<Flux<String>, Flux<String>> filterAndMap = f -> {
	if (ai.incrementAndGet() == 1) {
return f.filter(color -> !color.equals("orange"))
        .map(String::toUpperCase);
	}
	return f.filter(color -> !color.equals("purple"))
	        .map(String::toUpperCase);
};

Flux<String> composedFlux =
Flux.fromIterable(Arrays.asList("blue", "green", "orange", "purple"))
    .doOnNext(System.out::println)
    .transformDeferred(filterAndMap);

composedFlux.subscribe(d -> System.out.println("Subscriber 1 to Composed MapAndFilter :"+d));
composedFlux.subscribe(d -> System.out.println("Subscriber 2 to Composed MapAndFilter: "+d));
```

输出结果：

```shell
blue
Subscriber 1 to Composed MapAndFilter :BLUE
green
Subscriber 1 to Composed MapAndFilter :GREEN
orange
purple
Subscriber 1 to Composed MapAndFilter :PURPLE
blue
Subscriber 2 to Composed MapAndFilter: BLUE
green
Subscriber 2 to Composed MapAndFilter: GREEN
orange
Subscriber 2 to Composed MapAndFilter: ORANGE
purple
```

## HOT vs Cold



到目前为止，我们认为所有的 Flux（和 Mono）都是一样的：它们都代表一个异步的数据序列，在你订阅之前什么都不会发生。

不过，确实有两大类publishers：热的和冷的。

前面的描述适用于cold的publishers。 他们为每个订阅重新生成数据。 如果未创建订阅，则永远不会生成数据。



想想一个 HTTP 请求：每个新订阅者都会触发一个 HTTP 调用，但如果没有人对结果感兴趣，则不会进行调用。



另一方面，hot publisher不依赖订阅者的数量。 他们可能会立即开始发布数据，并且会在新订阅者进入时继续这样做（在这种情况下，订阅者只会看到订阅后发出的新元素）。 对于hot出 publisher，在您订阅之前确实会发生一些事情。



Reactor 中为数不多的hot操作符例子**just**：它在组装时直接捕获值，然后将其重播给任何订阅它的人。 再次使用 HTTP 调用类比，如果捕获的数据是 HTTP 调用的结果，则仅进行一次网络调用，仅在实例化时just。



要想转变为cold publisher，你可以使用 defer。 它将我们示例中的 HTTP 请求推迟到订阅时间（并且会导致每个新订阅的单独网络调用）。



相反，share() 和 replay(… ) 可用于将cold发布者转变为hot发布者（至少在第一次订阅发生后）。 这两个在 Sinks 类中也有 Sinks.Many 等价物，允许以编程方式提供序列。



考虑两个示例，一个演示冷通量，另一个使用汇来模拟热通量。 以下代码显示了第一个示例：

```java
Flux<String> source = Flux.fromIterable(Arrays.asList("blue", "green", "orange", "purple"))
                          .map(String::toUpperCase);

source.subscribe(d -> System.out.println("Subscriber 1: "+d));
source.subscribe(d -> System.out.println("Subscriber 2: "+d));
```

输出结果：

```shell
Subscriber 1: BLUE
Subscriber 1: GREEN
Subscriber 1: ORANGE
Subscriber 1: PURPLE
Subscriber 2: BLUE
Subscriber 2: GREEN
Subscriber 2: ORANGE
Subscriber 2: PURPLE
```

下图显示了重放行为：

![img](reactor/1624529728953-2dfb2a6f-ecce-44d4-87c3-b0ae1d3a7abe.png)

两个订阅者都捕获所有四种颜色，因为每个订阅者都会导致 Flux 上的运算符定义的进程运行。



将第一个示例与第二个示例进行比较，如下面的代码所示：

```java
Sinks.Many<String> hotSource = Sinks.unsafe().many().multicast().directBestEffort();

Flux<String> hotFlux = hotSource.asFlux().map(String::toUpperCase);

hotFlux.subscribe(d -> System.out.println("Subscriber 1 to Hot Source: "+d));

hotSource.emitNext("blue", FAIL_FAST); 
hotSource.tryEmitNext("green").orThrow(); 

hotFlux.subscribe(d -> System.out.println("Subscriber 2 to Hot Source: "+d));

hotSource.emitNext("orange", FAIL_FAST);
hotSource.emitNext("purple", FAIL_FAST);
hotSource.emitComplete(FAIL_FAST);
```

结果：

```shell
Subscriber 1 to Hot Source: BLUE
Subscriber 1 to Hot Source: GREEN
Subscriber 1 to Hot Source: ORANGE
Subscriber 2 to Hot Source: ORANGE
Subscriber 1 to Hot Source: PURPLE
Subscriber 2 to Hot Source: PURPLE
```

![img](reactor/1624529879460-a856ca0c-2e24-4827-b313-22d2ab827d46.png)



## 使用 ConnectableFlux 向多个订阅者广播

有时，您可能不希望仅将某些处理推迟到一个订阅者的订阅时间，但您实际上可能希望其中几个人会合，然后触发订阅和数据生成。



这就是 ConnectableFlux 的用途。 Flux API 中涵盖了返回 ConnectableFlux 的两种主要模式：publish和replay。

- publish动态地尝试尊重其各种订阅者的需求,在背压方面，通过将这些请求转发到源。 最值得注意的是，如果任何订阅者的未决需求为 0，则发布将暂停其对源的请求。
- `replay` : 缓冲通过第一次订阅看到的数据，最多可配置限制（时间和缓冲区大小）。 它将数据重播给后续订阅者。



ConnectableFlux 提供了额外的方法来管理下游订阅与原始源订阅。 这些额外的方法包括：

- 一旦您获得足够的 Flux 订阅，就可以手动调用 connect()。 这会触发对上游源的订阅。
- 一旦完成了 n 个订阅，autoConnect(n) 就可以自动完成相同的工作。
- refCount(n) 不仅会自动跟踪传入的订阅，还会检测这些订阅何时被取消。 如果没有跟踪到足够多的订阅者，则源将“断开连接”，如果出现其他订阅者，则稍后会导致对源进行新订阅。
- refCount(int, Duration) 添加了一个“宽限期”。 一旦被跟踪的订阅者数量变得太少，它会在断开源之前等待 Duration，这可能允许足够多的新订阅者进入并再次跨越连接阈值。

```java
Flux<Integer> source = Flux.range(1, 3)
                           .doOnSubscribe(s -> System.out.println("subscribed to source"));

ConnectableFlux<Integer> co = source.publish();

co.subscribe(System.out::println, e -> {}, () -> {});
co.subscribe(System.out::println, e -> {}, () -> {});

System.out.println("done subscribing");
Thread.sleep(500);
System.out.println("will now connect");

co.connect();
```

输出：

```shell
one subscribing
will now connect
subscribed to source
1
1
2
2
3
3
```

下面使用**autoConnect**

```java
Flux<Integer> source = Flux.range(1, 3)
                           .doOnSubscribe(s -> System.out.println("subscribed to source"));

Flux<Integer> autoCo = source.publish().autoConnect(2);

autoCo.subscribe(System.out::println, e -> {}, () -> {});
System.out.println("subscribed first");
Thread.sleep(500);
System.out.println("subscribing second");
autoCo.subscribe(System.out::println, e -> {}, () -> {});
```

结果：

```shell
subscribed first
subscribing second
subscribed to source
1
1
2
2
3
3
```

##  三种批处理

当您有很多元素并且想要将它们分成批次时，您可以在 Reactor 中提供三种广泛的解决方案：分组、窗口化和缓冲。 这三个在概念上很接近，因为它们将 Flux<T> 重新分配到聚合中。 分组和窗口创建一个 Flux<Flux<T>>，缓冲聚合到一个 Collection<T> 中。



### `Flux<GroupedFlux<T>>分组`

分组是将源 Flux<T> 分成多个批次的行为，每个批次都匹配一个键。关联的运算符是 groupBy。

每个组都表示为 GroupedFlux<T>，它允许您通过调用其 key() 方法来检索key。



组的内容没有必要的连续性。 一旦一个源元素产生一个新的键，这个键的组就会被打开，与键匹配的元素最终会出现在组中（可以同时打开几个组）。这意味着组：

- 总是不相交的（一个源元素属于一个且只属于一个组）。
- 可以包含来自原始序列中不同位置的元素。
- 永远不会为空

以下示例按值是偶数还是奇数对值进行分组：

```java
StepVerifier.create(
	Flux.just(1, 3, 5, 2, 4, 6, 11, 12, 13)
		.groupBy(i -> i % 2 == 0 ? "even" : "odd")
		.concatMap(g -> g.defaultIfEmpty(-1) //if empty groups, show them
				.map(String::valueOf) //map to string
				.startWith(g.key())) //start with the group's key
	)
	.expectNext("odd", "1", "3", "5", "11", "13")
	.expectNext("even", "2", "4", "6", "12")
	.verifyComplete();
```

#### `Flux<Flux<T>> 窗口`

窗口化是根据大小、时间、边界定义谓词或边界定义发布者的标准将源 Flux<T> 拆分为窗口的行为。

相关的操作符是 window、windowTimeout、windowUntil、windowWhile 和 windowWhen。



与 groupBy 不同，groupBy 根据传入的键随机重叠，窗口（大部分时间）是按顺序打开的。



但是，某些变体仍然可以重叠。 例如，在 window(int maxSize, int skip) 中， maxSize 参数是窗口关闭后的元素数，而 skip 参数是源中新窗口打开后的元素数。 因此，如果 maxSize > skip，则会在前一个窗口关闭之前打开一个新窗口，并且两个窗口重叠。

```java
StepVerifier.create(
	Flux.range(1, 10)
		.window(5, 3) //overlapping windows
		.concatMap(g -> g.defaultIfEmpty(-1)) //show empty windows as -1
	)
		.expectNext(1, 2, 3, 4, 5)
		.expectNext(4, 5, 6, 7, 8)
		.expectNext(7, 8, 9, 10)
		.expectNext(10)
		.verifyComplete();
```

在通过 windowUntil 和 windowWhile 进行基于谓词的窗口化的情况下，具有与谓词不匹配的后续源元素也可能导致空窗口，如以下示例所示：

```java
StepVerifier.create(
	Flux.just(1, 3, 5, 2, 4, 6, 11, 12, 13)
		.windowWhile(i -> i % 2 == 0)
		.concatMap(g -> g.defaultIfEmpty(-1))
	)
		.expectNext(-1, -1, -1) //respectively triggered by odd 1 3 5
		.expectNext(2, 4, 6) // triggered by 11
		.expectNext(12) // triggered by 13
		// however, no empty completion window is emitted (would contain extra matching elements)
		.verifyComplete();
```

#### `Flux<List<T>> 缓冲`

缓冲类似于窗口化，但有以下区别：它不发射窗口（每个窗口都是 Flux<T>），而是发射缓冲区（它们是 Collection<T> — 默认情况下是 List<T>）。

用于缓冲的运算符反映了用于窗口的运算符：buffer、bufferTimeout、bufferUntil、bufferWhile 和 bufferWhen。



当相应的窗口操作符打开一个窗口时，缓冲操作符会创建一个新集合并开始向其中添加元素。 当窗口关闭时，缓冲操作符发出集合。

缓冲还可能导致删除源元素或具有重叠缓冲区，如以下示例所示：

```java
StepVerifier.create(
	Flux.range(1, 10)
		.buffer(5, 3) //overlapping buffers
	)
		.expectNext(Arrays.asList(1, 2, 3, 4, 5))
		.expectNext(Arrays.asList(4, 5, 6, 7, 8))
		.expectNext(Arrays.asList(7, 8, 9, 10))
		.expectNext(Collections.singletonList(10))
		.verifyComplete();
```

与窗口化不同，bufferUntil 和 bufferWhile 不会发出空缓冲区，如下例所示：

```java
StepVerifier.create(
	Flux.just(1, 3, 5, 2, 4, 6, 11, 12, 13)
		.bufferWhile(i -> i % 2 == 0)
	)
	.expectNext(Arrays.asList(2, 4, 6)) // triggered by 11
	.expectNext(Collections.singletonList(12)) // triggered by 13
	.verifyComplete();
```



## 使用 ParallelFlux 并行化工作

随着多核架构成为当今的一种商品，能够轻松地并行化工作非常重要。 Reactor 通过提供一种特殊类型 ParallelFlux 来帮助解决这个问题，该类型公开了针对并行化工作进行了优化的运算符。

要获得 ParallelFlux，您可以在任何 Flux 上使用 parallel() 运算符。 就其本身而言，此方法不会并行化工作。 相反，它将工作负载划分为“轨道”（默认情况下，轨道与 CPU 内核的数量一样多）。

为了告诉生成的 ParallelFlux 在哪里运行每个轨道（并且，通过扩展，并行运行轨道），您必须使用 runOn(Scheduler)。 请注意，有一个推荐的并行工作专用调度程序：Schedulers.parallel()。

```java
Flux.range(1, 10)
    .parallel(2) 
    .subscribe(i -> System.out.println(Thread.currentThread().getName() + " -> " + i));
```

输出：

```shell
main -> 1
main -> 2
main -> 3
main -> 4
main -> 5
main -> 6
main -> 7
main -> 8
main -> 9
main -> 10
Flux.range(1, 10)
    .parallel(2)
    .runOn(Schedulers.parallel())
    .subscribe(i -> System.out.println(Thread.currentThread().getName() + " -> " + i));
```

输出：

```shell
parallel-1 -> 1
parallel-2 -> 2
parallel-1 -> 3
parallel-2 -> 4
parallel-1 -> 5
parallel-2 -> 6
parallel-1 -> 7
parallel-1 -> 9
parallel-2 -> 8
parallel-2 -> 10
```

如果在并行处理序列后，您想恢复到“正常” Flux 并以顺序方式应用运算符链的其余部分，则可以使用 ParallelFlux 上的 Sequential() 方法。



请注意，如果您使用 Subscriber 订阅 ParallelFlux 而不是使用基于 lambda 的 subscribe 变体，则 Sequential() 将被隐式应用。



另请注意， subscribe(Subscriber<T>) 合并所有 rails，而 subscribe(Consumer<T>) 运行所有 rails。 如果 subscribe() 方法有一个 lambda，则每个 lambda 都会执行与 Rails 一样多的次数。



您还可以通过 group() 方法以 Flux<GroupedFlux<T>> 的形式访问单个轨道或“组”，并通过 composeGroup() 方法将其他运算符应用于它们。

## 替换默认的`Schedulers`

正如我们在线程和调度器部分所描述的，Reactor Core 带有几个调度器实现。 虽然您始终可以通过 new* 工厂方法创建新实例，但每个 Scheduler 风格还有一个默认的单例实例，可通过直接工厂方法访问（例如 Schedulers.boundedElastic() 与 Schedulers.newBoundedElastic(… )）。



这些默认实例是在您未明确指定时需要调度程序才能工作的运算符使用的实例。 例如， Flux#delayElements(Duration) 使用 Schedulers.parallel() 实例。



但是，在某些情况下，您可能需要以横切方式使用其他内容更改这些默认实例，而不必确保您调用的每个操作员都将您的特定调度程序作为参数。 一个例子是通过包装真实的调度程序来测量每个计划任务所花费的时间，以用于检测目的。 换句话说，您可能想要更改默认的调度程序。



可以通过 Schedulers.Factory 类更改默认调度程序。 默认情况下，工厂通过类似命名的方法创建所有标准调度程序。 您可以使用自定义实现覆盖其中的每一个。



此外，工厂公开了一种额外的自定义方法：decorateExecutorService。 它在每个由 ScheduledExecutorService 支持的 Reactor Core Scheduler 的创建过程中被调用（甚至是非默认实例，例如通过调用 Schedulers.newParallel() 创建的实例）。



这使您可以调整要使用的 ScheduledExecutorService：默认的作为供应商公开，并且根据配置的调度程序的类型，您可以选择完全绕过该供应商并返回您自己的实例，或者您可以 get() 默认 实例并包装它。



最后，调度程序中有最后一个可自定义的钩子：onHandleError。 每当提交给调度程序的 Runnable 任务抛出异常时，都会调用此挂钩（请注意，如果为运行该任务的线程设置了 UncaughtExceptionHandler，则调用处理程序和挂钩）。



## 使用全局Hooks

Reactor 有另一类可配置回调，由 Reactor 操作符在各种情况下调用。 它们都设置在 Hooks 类中，它们分为三类：

-  掉钩（Dropping ）
- 内部错误挂钩（Internal Error Hook）
- 装配钩（Assembly Hooks）



### Dropping Hooks

当操作符的源不符合 Reactive Streams 规范时，会调用丢弃钩子。 这些类型的错误在正常执行路径之外（也就是说，它们不能通过 onError 传播）。



通常，发布者会在操作符上调用 onNext，尽管之前已经调用了 onCompleted。 在这种情况下， onNext 值将被删除。 对于无关的 onError 信号也是如此。



相应的钩子，onNextDropped 和 onErrorDropped，让你为这些 drop 提供一个全局的 Consumer。 例如，如果需要，您可以使用它来记录丢弃并清理与值关联的资源（因为它永远不会进入反应链的其余部分）。



连续两次设置挂钩是附加的：您提供的每个使用者都会被调用。 可以使用 Hooks.resetOn*Dropped() 方法将钩子完全重置为其默认值。

### Internal Error Hook

当在执行 onNext、onError 和 onComplete 方法期间引发意外异常时，操作符会调用一个钩子 onOperatorError。



与前一类不同，这仍然在正常执行路径内。 一个典型的例子是带有抛出异常（例如除以零）的 map 函数的 map 操作符。 此时仍然可以通过通常的 onError 通道，这就是操作符所做的。



首先，它通过 onOperatorError 传递异常。 该挂钩可让您检查错误（以及相关值，如果相关）并更改异常。 当然，你也可以在旁边做一些事情，比如记录并返回原始的Exception。



请注意，您可以多次设置 onOperatorError 钩子。 您可以为特定的 BiFunction 提供字符串标识符，随后使用不同键的调用连接所有执行的函数。 另一方面，重复使用相同的键两次可以让您替换之前设置的功能。



因此，默认挂钩行为可以完全重置（通过使用 Hooks.resetOnOperatorError()）或仅针对特定键部分重置（通过使用 Hooks.resetOnOperatorError(String)）。

### Assembly Hooks

这些钩子与操作符的生命周期有关。 当一连串的运算符被组装（即实例化）时，它们会被调用。 onEachOperator 允许您通过返回不同的发布者，在链中组装时动态更改每个运算符。 onLastOperator 类似，不同之处在于它仅在 subscribe 调用之前链中的最后一个操作符上调用。



如果你想用一个横切的 Subscriber 实现来装饰所有的操作符，你可以查看 Operators#lift* 方法来帮助你处理各种类型的 Reactor Publishers（Flux、Mono、ParallelFlux、GroupedFlux 和 ConnectableFlux） ，以及它们的 Fuseable 版本。



像 onOperatorError 一样，这些钩子是累积的，可以用一个键来识别。 它们也可以部分或全部重置。



### 挂钩预设

Hooks 实用程序类提供了两个预设的钩子。 这些是默认行为的替代方案，您可以通过调用相应的方法来使用它们，而不是自己提出钩子：

- onNextDroppedFail(): onNextDropped 用于抛出 Exceptions.failWithCancel() 异常。 它现在默认在 DEBUG 级别记录删除的值。 要返回到抛出的旧默认行为，请使用 onNextDroppedFail()。
- onOperatorDebug()：此方法激活调试模式。 它与 onOperatorError 挂钩，因此调用 resetOnOperatorError() 也会重置它。 您可以使用 resetOnOperatorDebug() 独立重置它，因为它在内部使用特定的键。



## 向反应式序列添加上下文

从命令式编程角度切换到反应式编程思维方式时遇到的重大技术挑战之一在于您如何处理线程。



与您可能习惯的相反，在反应式编程中，您可以使用 Thread 来处理几个大致同时运行的异步序列（实际上，以非阻塞锁步）。 执行也可以轻松且经常地从一个线程跳转到另一个线程。



对于依赖使用更“稳定”的线程模型（例如 ThreadLocal）的开发人员来说，这种安排尤其困难。 由于它允许您将数据与线程相关联，因此在响应式上下文中使用变得棘手。 因此，依赖 ThreadLocal 的库在与 Reactor 一起使用时至少会带来新的挑战。 在最坏的情况下，它们工作得不好甚至失败。 使用 Logback 的 MDC 来存储和记录相关 ID 是这种情况的一个主要例子。



如果要在ractor中使用ThreadLocal,通常的解决方案，组装成Tuple2<T, C>（T是业务数据，C是上下文数据）序列 ，这看起来不太好，并且会将正交问题（上下文数据）泄漏到您的方法和 Flux 签名中。



从 3.1.0 版本开始，Reactor 附带了一个与 ThreadLocal 有点类似的高级功能，但可以应用于 Flux 或 Mono 而不是 Thread。 此功能称为上下文。



以下示例同时读取和写入 Context：

```java
String key = "message";
Mono<String> r = Mono.just("Hello")
    .flatMap(s -> Mono.deferContextual(ctx ->
         Mono.just(s + " " + ctx.get(key))))
    .contextWrite(ctx -> ctx.put(key, "World"));

StepVerifier.create(r)
            .expectNext("Hello World")
            .verifyComplete();
```

在以下部分中，我们将介绍 Context 及其使用方法，以便您最终理解前面的示例。



### `Context` API



Context 是一个让人联想到 Map 的接口。它存储键值对，并允许您获取通过其键存储的值。 它有一个仅公开读取方法的简化版本，即 ContextView。 进一步来说：



- 键和值都是 Object 类型，因此 Context（和 ContextView）实例可以包含来自不同库和源的任意数量的高度不同的值。
- 上下文是不可变的。 它公开了像 put 和 putAll 这样的写方法，但它们会产生一个新的实例。
- 对于一个甚至不公开此类写入方法的只读 API，自 3.4.0 起就有了 ContextView 超接口
- 您可以使用 hasKey(Object key) 检查密钥是否存在。
- 使用 getOrDefault(Object key, T defaultValue) 检索值（转换为 T）或如果 Context 实例没有该键则回退到默认值。
- 使用 getOrEmpty(Object key) 获取 Optional<T>（Context 实例尝试将存储的值转换为 T）。
- 使用 put(Object key, Object value) 存储一个键值对，返回一个新的 Context 实例。 您还可以使用 putAll(ContextView) 将两个上下文合并为一个新的上下文。
- 使用 delete(Object key) 删除与键关联的值，返回一个新的上下文。



创建 Context 时，您可以使用静态 Context.of 方法创建具有最多五个键值对的预值 Context 实例。 它们采用 2、4、6、8 或 10 个对象实例，每对对象实例都是要添加到上下文的键值对。

或者，您也可以使用 Context.empty() 创建一个空的 Context。



### 将上下文与 Flux 联系起来并写入

为了使 Context 有用，它必须绑定到一个特定的序列并且可以被链中的每个操作符访问。 请注意，运算符必须是 Reactor 本地运算符，因为 Context 特定于 Reactor。

实际上，上下文与链中的每个订阅者相关联。 它使用订阅传播机制使每个操作符都可以使用它，从最终订阅开始并向上移动。



为了填充只能在订阅时完成的 Context，您需要使用 contextWrite 运算符。

contextWrite(ContextView) 合并您提供的 ContextView 和来自下游的 Context（请记住，Context 从链的底部向顶部传播）。 这是通过调用 putAll 来完成的，从而为上游产生一个新的上下文。



您还可以使用更高级的 contextWrite(Function<Context, Context>)。 它从下游接收 Context 的副本，允许您根据需要放置或删除值，并返回新的 Context 以供使用。 您甚至可以决定返回一个完全不同的实例，尽管确实不建议这样做（这样做可能会影响依赖于 Context 的第三方库）。



### 通过 ContextView 读取上下文

填充 Context 后，您可能希望在运行时查看它。 大多数情况下，将信息放入 Context 的责任在最终用户一侧，而利用该信息则在第三方库一侧，因为此类库通常位于客户端代码的上游。

面向读取的操作符允许通过暴露其 ContextView 从操作符链中的 Context 获取数据：

- 要从类似源的运算符访问上下文，请使用 deferContextual 工厂方法
- 要从运算符链的中间访问上下文，请使用 transformDeferredContextual(BiFunction)
- 或者，在处理内部序列时（例如在 flatMap 内部），可以使用 Mono.deferContextual(Mono::just) 来具体化 ContextView。 通常，您可能希望直接在 defer 的 lambda 中执行有意义的工作，例如。 Mono.deferContextual(ctx → doSomethingAsyncWithContextData(v, ctx.get(key))) 其中 v 是被平面映射的值。



为了在不误导用户的情况下从 Context 中读取数据，让用户认为可以在数据通过管道运行时对其进行写入，上述操作符仅公开了 ContextView。



### 例子

本节中的示例旨在更好地理解使用上下文的一些注意事项。

我们首先更详细地回顾一下介绍中的简单示例，如下例所示：

```java
String key = "message";
Mono<String> r = Mono.just("Hello")
    .flatMap(s -> Mono.deferContextual(ctx ->
         Mono.just(s + " " + ctx.get(key))))  //2
    .contextWrite(ctx -> ctx.put(key, "World"));  //1

StepVerifier.create(r)
            .expectNext("Hello World") //3
            .verifyComplete();
```

1. 运算符链以对 contextWrite(Function) 的调用结束，该调用将“World”放入“message”键下的上下文中。
2. 我们在源元素上进行 flatMap，使用 Mono.deferContextual() 实现 ContextView 并直接提取与“message”关联的数据并将其与原始单词连接。



上面的编号与实际的行顺序没有错误。 它代表执行顺序。 即使 contextWrite 是链的最后一部分，它也是最先执行的部分（由于其订阅时间性质以及订阅信号从下到上流动的事实）。



在您的操作员链中，您写入 Context 的位置以及您从中读取的位置的相对位置很重要。 Context 是不可变的，它的内容只能被它上面的操作符看到，如下例所示：

```java
String key = "message";
Mono<String> r = Mono.just("Hello")
    .contextWrite(ctx -> ctx.put(key, "World")) //1
    .flatMap( s -> Mono.deferContextual(ctx ->
        Mono.just(s + " " + ctx.getOrDefault(key, "Stranger")))); //2

StepVerifier.create(r)
            .expectNext("Hello Stranger") //3
            .verifyComplete();
```

1. 上下文在链中写得太高了。结果，在flatMap中，没有任何价值



同样，在多次尝试将相同的键写入 Context 的情况下，写入的相对顺序也很重要。 读取 Context 的运算符会看到最接近其下方设置的值，如以下示例所示：



```java
String key = "message";
Mono<String> r = Mono
    .deferContextual(ctx -> Mono.just("Hello " + ctx.get(key)))
    .contextWrite(ctx -> ctx.put(key, "Reactor")) 
    .contextWrite(ctx -> ctx.put(key, "World")); 

StepVerifier.create(r)
            .expectNext("Hello Reactor") 
            .verifyComplete();
```

在前面的示例中，上下文在订阅期间填充了“World”。 然后订阅信号向上游移动并发生另一次写入。 这会产生一个值为“Reactor”的第二个不可变上下文。 之后，数据开始流动。 deferContextual 看到最接近它的 Context，这是我们的第二个具有“Reactor”值的 Context（作为 ContextView 暴露给用户）。



您可能想知道 Context 是否与数据信号一起传播。 如果是这种情况，在这两次写入之间放置另一个 flatMap 将使用来自顶部 Context 的值。 但事实并非如此，如以下示例所示：

```java
String key = "message";
Mono<String> r = Mono
    .deferContextual(ctx -> Mono.just("Hello " + ctx.get(key))) 
    .contextWrite(ctx -> ctx.put(key, "Reactor")) 
    .flatMap( s -> Mono.deferContextual(ctx ->
        Mono.just(s + " " + ctx.get(key)))) 
    .contextWrite(ctx -> ctx.put(key, "World")); 

StepVerifier.create(r)
            .expectNext("Hello Reactor World") 
            .verifyComplete();
```

原因是 Context 与 Subscriber 相关联，每个 operator 通过从其下游 Subscriber 请求来访问 Context。

最后一个有趣的传播案例是 Context 也被写入到 flatMap 中，如下例所示：



```java
String key = "message";
Mono<String> r = Mono.just("Hello")
    .flatMap( s -> Mono
        .deferContextual(ctxView -> Mono.just(s + " " + ctxView.get(key)))
    )
    .flatMap( s -> Mono
        .deferContextual(ctxView -> Mono.just(s + " " + ctxView.get(key)))
        .contextWrite(ctx -> ctx.put(key, "Reactor")) 
    )
    .contextWrite(ctx -> ctx.put(key, "World")); 

StepVerifier.create(r)
            .expectNext("Hello World Reactor")
            .verifyComplete();
```

在前面的示例中，最终发出的值是“Hello World Reactor”而不是“Hello Reactor World”，因为写入“Reactor”的 contextWrite 是作为第二个 flatMap 的内部序列的一部分这样做的。 因此，它在主序列中不可见或无法传播，第一个 flatMap 也看不到它。 传播和不变性将 Context 隔离在创建中间内部序列（例如 flatMap）的运算符中。



### 完整示例

现在我们可以考虑一个更真实的例子，从上下文中读取信息的库：一个响应式 HTTP 客户端，它接受 Mono<String> 作为 PUT 的数据源，但也寻找特定的 Context 键来添加相关 ID 到请求的标头。

从用户的角度来看，它的调用方式如下：

```java
doPut("www.example.com", Mono.just("Walter"))
```

为了传播相关 ID，它将按如下方式调用：

```java
doPut("www.example.com", Mono.just("Walter"))
	.contextWrite(Context.of(HTTP_CORRELATION_ID, "2-j3r9afaf92j-afkaf"))
```

正如前面的片段所示，用户代码使用 contextWrite 来填充一个带有 HTTP_CORRELATION_ID 键值对的上下文。 运算符的上游是由 HTTP 客户端库返回的 Mono<Tuple2<Integer, String>>（HTTP 响应的简单表示）。 因此它有效地将信息从用户代码传递到库代码。

以下示例从库的角度显示了模拟代码，该代码读取上下文并在可以找到相关 ID 时“增加请求”：

```java
static final String HTTP_CORRELATION_ID = "reactive.http.library.correlationId";

Mono<Tuple2<Integer, String>> doPut(String url, Mono<String> data) {
  Mono<Tuple2<String, Optional<Object>>> dataAndContext =
      data.zipWith(Mono.deferContextual(c -> 
          Mono.just(c.getOrEmpty(HTTP_CORRELATION_ID))) 
      );

  return dataAndContext.<String>handle((dac, sink) -> {
      if (dac.getT2().isPresent()) { 
        sink.next("PUT <" + dac.getT1() + "> sent to " + url +
            " with header X-Correlation-ID = " + dac.getT2().get());
      }
      else {
        sink.next("PUT <" + dac.getT1() + "> sent to " + url);
      }
        sink.complete();
      })
      .map(msg -> Tuples.of(200, msg));
```

库片段使用 Mono.deferContextual(Mono::just) 压缩数据 Mono。 这为库提供了一个 Tuple2<String, ContextView>，并且该上下文包含来自下游的 HTTP_CORRELATION_ID 条目（因为它位于到订阅者的直接路径上）。

然后，库代码使用 map 为该键提取一个 Optional<String> ，并且如果该条目存在，它将使用传递的相关 ID 作为 X-Correlation-ID 标头。 最后一部分由手柄模拟。

验证使用相关ID的库代码的整个测试可以写成如下:

```java
@Test
public void contextForLibraryReactivePut() {
  Mono<String> put = doPut("www.example.com", Mono.just("Walter"))
      .contextWrite(Context.of(HTTP_CORRELATION_ID, "2-j3r9afaf92j-afkaf"))
      .filter(t -> t.getT1() < 300)
      .map(Tuple2::getT2);

  StepVerifier.create(put)
              .expectNext("PUT <Walter> sent to www.example.com" +
                  " with header X-Correlation-ID = 2-j3r9afaf92j-afkaf")
              .verifyComplete();
}
```



## 处理需要清理的对象

在非常特殊的情况下，您的应用程序可能会处理不再使用后需要某种形式的清理的类型。 这是一个高级场景 — 例如，当您有引用计数的对象或处理堆外对象时。 Netty 的 ByteBuf 是两者的典型例子。

为了确保对此类对象进行适当的清理，您需要在 Flux-by-Flux 的基础上以及在几个全局钩子中对其进行说明（请参阅使用全局钩子）：

- doOnDiscard Flux/Mono 运算符
- onOperatorError 钩子
- onNextDropped 钩子
- 特定于操作符的处理程序







# 核心函数

## defer

```java
        Flux<LocalDateTime> flux1 = Flux.defer(() -> Flux.just(LocalDateTime.now()));
        Flux<LocalDateTime> flux2 = Flux.just(LocalDateTime.now());
        flux1.subscribe(System.out::println); //2021-06-21T10:59:19.436406100
        flux2.subscribe(System.out::println); //2021-06-21T10:59:19.432403600

        Thread.sleep(5000);

        flux1.subscribe(System.out::println); //2021-06-21T10:59:24.439308500  和第一次不相等，说明每次订阅都会创建一次
        flux2.subscribe(System.out::println); //2021-06-21T10:59:19.432403600  和第一次打印相等，说明只创建了一次
```

defer函数是在订阅的时候才会触发序列中元素的创建，即每次订阅都会执行supply函数。just等函数在声明阶段就已经创建好序列中的元素了。



## using

```java
        Flux.using(() -> dataSource.getConnection(), connection -> {
          List result=  connection.query(" select * from user");
          return Flux.fromIterable(result);
        },connection -> connection.close()).subscribe(System.out::println);
```



## map

![img](reactor/1624246301947-d24d7ecd-d340-4dce-a28d-7e2806ec8816.svg+xml)

```java
        Flux.just("1", "2")
                .map(item -> "zhao" + item)
                .subscribe(System.out::println);
zhao1
zhao2
```

## cast

对序列中的元素进行向上转型

```java
        Flux.just(new Cat("小米", "mmm~"), new Cat("小哈", "aaa~"))
                .cast(Animal.class)
                .subscribe(item-> System.err.println(item.getClass().getName()));
```

## index

给序列中的元素添加索引号，转化成<index,value>形式

![img](reactor/1624247362955-a05071b0-43b4-4fdb-a738-28084f1337c2.svg+xml)

```java
        Flux.just("zhao", "qian", "sun")
                .index()
                .subscribe(item -> System.err.println(item.getT1() + ":" + item.getT2()));
0:zhao
1:qian
2:sun
```

## flatMap

将此 Flux 发出的元素异步转换为 Publishers，然后通过合并将这些内部Publisher扁平化为单个 Flux，这允许它们交错。

![img](reactor/1624253186893-300a9e8f-41c2-4cf8-88fe-a97fa80637e4.svg+xml)

```java
        Flux.just("a", "b")
                .flatMap(item -> {
                    if ("a".equals(item)) {
                        return Flux.just(1, 2, 3, 4);
                    } else {
                        return Flux.just(5, 6, 7);
                    }
                }).subscribe(System.out::println);

    }
```

该方法适用于http请求，例如我们要同时调用3个http请求，可以使用flatMap来并发操作。

## flatMapSequential

将此 Flux 发出的元素异步转换为发布者，然后将这些内部发布者扁平化为单个 Flux，但按源元素的顺序合并它们。

![img](reactor/1624253158346-52d85c9d-e5b7-468e-b8d4-3706a672f39e.svg+xml)

## 

## 

## handle

处理序列中的元素，手动控制元素向下游传播

```java
        Flux.just("a", "b")
                .handle((item, sink) -> {
                    if ("a".equals(item)) {
                        sink.next(1);
                    } 
                }).subscribe(System.out::println);
```
