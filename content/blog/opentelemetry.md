---
title: opentelemetry
tags:
  - 监控
  - opentelemetry
categories:
  - 中间件
date: 2022-11-27 12:41:11 
---



# collector基本组件

## Receiver
### Filelog Receiver
| **Field** | **Default** | **Description** |
| --- | --- | --- |
| include | required | 匹配的文件列表，支持glob模式 |
| exclude | [] | 排除的文件列表，支持glob模式 |
| start_at | end | 启动时，从哪里开始读取日志。选项是beginning 或 end |
| multiline |  | 定义日志行，默认文件中的每行作为一个日志行，注意和操作符recombine的区别 。multiline定义了什么是日志行。 recombine是将多个日志行合并在一起。 |
| force_flush_period | 500ms | 自上次从文件读取数据以来的时间，在此之后，当前缓冲的日志应发送到管道等待的时间。零意味着永远等待新数据 |
| encoding | utf-8 | 文件的编码 |
| include_file_name | true | 是否添加文件名称到属性 log.file.name. |
| include_file_path | false | 是否添加文件路径到属性 log.file.path. |
| include_file_name_resolved | false | 是否将符号链接解析后的文件名添加到属性log.file.name_resolved。 |
| include_file_path_resolved | false | 是否将符号链接解析后的文件路径添加到属性log.file.path_resolved。 |
| poll_interval | 200ms | 文件系统轮询之间的持续时间 |
| fingerprint_size | 1kb | 用于标识文件的字节数。 |
| max_log_size | 1MiB | 日志行的最大大小 |
| max_concurrent_files | 1024 | 并发读取日志的最大日志文件数。如果包含模式中匹配的文件数超过此数，则将批量处理文件。每个poll_interval 处理一批 |
| attributes | {} | 要添加到entry的 属性键：值对的映射 |
| resource | {} | 要添加到entry 资源的键：值对的映射 |
| operators | [] | 处理日志的操作 [operators](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.63.0/pkg/stanza/docs/operators/README.md#what-operators-are-available) |
|converter| {                                       <br/>max_flush_count: 100,                   <br/>flush_interval: 100ms,              <br/>worker_count:max(1,runtime.NumCPU()/4) <br/>} | 键：值对的映射，用于配置entry.Entry到plog.LogRecord |
| storage                    |                                                              | 存储扩展的ID。扩展名将用于存储文件检查点，这允许接收器在收集器重新启动的情况下从停止的位置开始。 |
|                            |                                                              |                                                              |
|                            |                                                              |                                                              |
|                            |                                                              |                                                              |
|                            |||
## Processors 
Processors 用于pipeline 的各个阶段。通常，Processors 在数据导出之前对其进行预处理（例如修改属性或修改采样率），或帮助确保数据成功通过管道（例如批处理/重试）。
### 推荐的 Processors
默认情况下，不启用 Processors 。根据使用的数据源，可能启用多个Processors 。
**并非所有Processors 都支持所有数据源**。
此外，处理器的顺序很重要。下面是各类型数据处理的最佳顺序。

Traces

1. memory_limiter
2. any sampling processors
3. Any processor relying on sending source from Context (e.g. k8sattributes)
4. batch
5. any other processors

Metrics
1. memory_limiter
2. Any processor relying on sending source from Context (e.g. k8sattributes)
3. batch
4. any other processors

#### Memory Limiter Processor
内存限制器处理器用于防止收集器出现内存不足的情况。鉴于采集器处理的数据量和类型是特定于环境的，采集器的资源利用率也取决于配置的处理器，因此必须对内存使用情况进行检查。
memory_limiter处理器允许定期检查内存使用情况，如果超过了定义的限制，则会开始丢弃数据并强制GC减少内存消耗。
memory_limiter使用软内存和硬内存限制。硬限制始终高于或等于软限制。
**当内存使用超过软限制时，处理器将开始丢弃数据，并将错误返回到管道中的前一个组件（通常应该是接收器）。**
当内存使用超过硬限制时，除了丢弃数据之外，处理器还会强制执行垃圾收集，以尝试释放内存。
当内存使用率降至软限制以下时，将恢复正常操作（不再丢弃数据，也不会执行强制垃圾收集）。
软限制和硬限制之间的差异通过`spike_limit_mib`配置选项定义。设定值时，应确保在内存检查间隔之间内存使用量的增加不会超过此值（否则内存使用量可能会超过硬限制，即使是暂时的）。spike_limit_mib的值建议是硬限制的20%。对于尖峰流量或更长的检查间隔，可能需要更大的spike_limit_mib值。
请注意，虽然处理器可以帮助缓解内存不足的情况，但它并不能取代收集器的正确大小和配置。请记住，如果超过软限制，收集器将向所有接收操作返回错误，直到释放足够的内存。这将导致数据丢失。
强烈建议在每个收集器上配置ballastextension和memory_limiter处理器。ballast 应配置为分配给收集器的内存的1/3至1/2。memory_limiter处理器应该是管道中定义的第一个处理器（紧跟在接收器之后）。这是为了确保可以将背压发送到适用的接收器，并在触发memory_limiter时最小化数据丢失的可能性。
> memory_limiter 需要 ballast 扩展 来做一些底层操作。ballast 是 go控制gc的一种方式。

配置选项：

- check_interval ：默认0s, 检测内存使用情况的间隔。建议值为1秒。如果收集器的预期流量非常高，则减少check_interval或增加spike_limit_mib，以避免内存使用超过硬限制。
- limit_mib ：默认0，进程堆分配的最大内存量（以MiB为单位）。请注意，通常进程的总内存使用量将比该值高约50MiB。这定义了硬限制。
- spike_limit_mib ： 默认20%，内存使用量测量之间的最大峰值。该值必须小于limit_mib。软限值将等于（limit_mib-spike_limit_mib）。spike_limit_mib的建议值约为20%的limit_mib。
- limit_percentage ：默认0，进程堆要分配的最大总内存量。这种配置在带有cgroups的Linux系统上受支持，并且打算在动态平台（如docker）中使用。此选项用于从总可用内存中计算memory_limit。例如，设置为75%，总内存为1GiB，将导致750MiB的限制。固定内存设置（limit_mib）优先于百分比配置。
- spike_limit_percentage ： 默认0，内存使用量测量之间的最大峰值。该值必须小于limit_percentage。此选项用于从总可用内存中计算spike_limit_mib。例如，在总内存为1GiB的情况下设置25%将导致250MiB的峰值限制。此选项仅用于limit_percentage。

配置的示例：

```
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 1500
    spike_limit_mib: 300
extensions:
  memory_ballast:
    size_mib: 500
```

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 50
    spike_limit_percentage: 30
```
#### Batch Processor
batch processor 接受spans, metrics, logs，并将它们放入批处理中。批处理有助于更好地压缩数据，并减少传输数据所需的传出连接数。此处理器支持基于大小和时间的批处理。
强烈建议在每个收集器上配置批处理器。批处理器应在memory_limiter 以及任何采样处理器 之后的管道中定义。这是因为批处理应该发生在任何数据丢失（如采样）之后。
配置选项：

- send_batch_size ：默认8192，
- timeout ：默认200ms
- send_batch_max_size ：默认0，批次大小的上限。0表示批次大小没有上限。此属性可确保将较大的批次拆分为较小的单元。它必须大于或等于send_batch_size。

### 数据所有权
管道中数据（pdata.Traces, pdata.Metrics 和 pdata.Logs）的所有权在数据通过管道时传递。数据由 receiver 创建，然后当调用ConsumeTraces/ConumeMetrics/ConsumeLogs函数时，所有权被传递给第一个processor 。
注意：receiver 可能连接到多条pipelines，在这种情况下，相同的数据将扇出连接器，传递到所有连接的pipelines。
从数据所有权的角度来看，管道可以在两种模式下工作：

- 独家数据所有权
- 共享数据所有权

模式在启动期间根据处理器报告的数据修改意图定义。每个处理器通过Capabilities函数返回的结构的MutatesData字段报告意图。如果管道中的任何处理器声明要修改数据，则该管道将以独占所有权模式工作。此外，具有独占所有权模式的接收器，其对应的管道也将以独占所有权方式运行。
#### 独占模式
在独占所有权模式下，数据在给定时刻由特定处理器独占，处理器可以自由修改其拥有的数据。
独占模式仅适用于从同一接收器接收数据的管道。如果管道被标记为处于独占模式，则从共享接收器接收的任何数据都将在扇出连接器处克隆，然后再传递到每个管道。这确保了每个管道都有自己的独占数据副本，并且可以在管道中安全地修改数据。
数据的独占所有权允许处理器在拥有数据时自由修改数据（例如，参见attributesprocessor）。处理器对数据的所有权持续时间是从ConsumeTraces/ConumeMetrics/ConsumeLogs调用开始，直到处理器调用下一个处理器的ConsumeTrace/Conumemetrics/ConsumeLogs函数，该函数将所有权传递给下一个处理者。此后，处理器不得再读取或写入数据，因为新所有者可能会同时修改数据。
#### 共享模式
在共享所有权模式下，没有特定的处理器拥有数据，也不允许任何处理器修改共享数据。
在此模式下，不在连接到多个管道的接收器的扇出连接器处执行克隆。在这种情况下，所有这样的管道将看到相同的数据共享副本。禁止以共享所有权模式运行的管道中的处理器修改其通过ConsumeTraces/ConumeMetrics/ConsumeLogs调用接收的原始数据。处理器只能读取数据，但不能修改数据。
如果处理器在执行处理时需要修改数据，但不想承担独占模式带来的数据克隆成本，则处理器可以声明不修改数据，并使用任何不同的技术来确保原始数据不被修改。例如，处理器可以为pdata.Traces/pdata.Metrics/pdata.Logs的各个子部分实现写时复制方法。
如果处理器使用这种技术，则应声明其不打算通过在其能力中设置MutatesData=false来修改原始数据，以避免将管道标记为独占所有权，并避免独占所有权部分中描述的数据克隆成本。

## extension
扩展提供了收集器主要功能之上的功能。通常，扩展用于实现可以添加到收集器的组件，但不需要直接访问遥测数据，也不属于管道的一部分（如接收器、处理器或导出器）。示例扩展包括：响应健康检查请求的健康检查扩展或允许获取收集器性能配置文件的PProf扩展。
为服务指定扩展的顺序很重要，因为这是每个扩展启动的顺序，也是它们关闭的相反顺序。例如：
```yaml
service:
  # Extensions specified below are going to be loaded by the service in the
  # order given below, and shutdown on reverse order.
  extensions: [memory_ballast, zpages]
```
### Memory Ballast
内存镇流器扩展使应用程序能够为进程配置内存镇流器。有关详细信息，请参见：

- [Go memory ballast blogpost](https://web.archive.org/web/20210929130001/https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap-26c2462549a2/)
- [Golang issue related to this](https://github.com/golang/go/issues/23044)

配置选项：

- size_mib (default = 0, disabled): 是内存镇流器大小，单位为MiB。如果同时指定了size_in_percentage和size_mib ，则优先级高于size_in_percentage 。
- size_in_percentage (default = 0, disabled): 基于总内存百分比设置内存镇流器，值范围为1-100。它在容器化（如docker、k8s）和物理主机环境中都受支持。
```yaml
extensions:
  memory_ballast:
    size_mib: 64
```

### zPages
zPages对于进程内诊断非常有用，无需依赖任何后端来检查trace或span。
配置选项：

- endpoint (default = localhost:55679)：指定为zPages提供服务的HTTP端点。使用`localhost:`使其仅在本地可用，或使用“:”使其在所有网络接口上可用。

配置示例：
```yaml
extensions:
  zpages:
```
暴漏的端点：

- [http://localhost:55679/debug/servicez](http://localhost:55679/debug/servicez)：ServiceZ概述了收集器服务以及对pipelinez、extensionz和featurez zPages的快速访问。该页面还提供构建和运行时信息。
- [http://localhost:55679/debug/pipelinez](http://localhost:55679/debug/pipelinez)：PipelineZ可以深入了解收集器中正在运行的管道。您可以找到关于类型的信息，如果数据发生了变异，以及用于每个管道的接收器、处理器和导出器。
-  [http://localhost:55679/debug/extensionz](http://localhost:55679/debug/extensionz)：ExtensionZ显示收集器中活动的扩展。
- [http://localhost:55679/debug/featurez](http://localhost:55679/debug/featurez)：FeatureZ列出了可用的要素及其当前状态和描述。
- [http://localhost:55679/debug/tracez](http://localhost:55679/debug/tracez)：例如，TraceZ路由可用于按延迟桶检查和分类跨度

（0us、10us、100us、1ms、10ms、100ms、1s、10s、1ms）它们还允许您快速检查错误样本

- [http://localhost:55679/debug/rpcz](http://localhost:55679/debug/rpcz)：Rpcz路由可用于帮助检测远程过程调用（RPC）的统计信息。
### File Storage
文件存储扩展可以将状态持久化到本地文件系统。
扩展需要对目录进行读写访问。可以使用默认目录，但它必须已经存在，扩展才能运行。
配置：

- directory是专用数据存储目录的相对或绝对路径。在Windows上，默认目录为%ProgramData%\Otelcol\FileStorage，否则为/var/lib/otelcol/file_storage。
- timeout是待文件锁定的最长时间。在大多数情况下，不需要修改该值。默认超时为1s。
- compaction 定义了压缩文件的方式和时间。有两种可用的压缩模式（这两种模式都可以同时设置）：
   - compaction.on_start (default: false):收集器启动时发生
   - compaction.on_rebound (default: false), 当满足某些标准时在线发生；下面将详细讨论
   - compaction.directory :  指定用于压缩的目录（作为中间步骤）。
   - compaction.max_transaction_size (default: 65536): 定义压缩事务的最大大小。值为零将忽略事务大小。
   - compaction.rebound_needed_threshold_mib (default: 100) : 当分配的数据超过此数量时，将启用“需要压缩”标志
   - compaction.rebound_trigger_threshold_mib (default: 10) : 如果设置了“需要压缩”标志，并且分配的数据低于该值，则将开始压缩，并清除“需要压缩的”标志
   - compaction.check_interval (default: 5s) : 指定检查y压缩条件的频率

在某些工作负载（例如，持久性队列）中，存储可能会显著增长（例如，当导出器由于网络问题而无法发送数据时），然后随着基础问题的消失（例如，网络连接恢复），存储将被清空。这留下了一个需要回收的重要空间。在线发生这种情况的最佳条件是在存储大量耗尽之后，这由bounder_trigger_threshold_mib控制。为了确保这一点不太敏感，还有一个rebound_neeed_threshold_mib，它指定了要考虑在线压缩必须满足的总声明空间大小。考虑下图中满足回弹（在线）压实条件的示例。
```yaml

  ▲
  │
  │             XX.............
m │            XXXX............
e ├───────────XXXXXXX..........────────────  rebound_needed_threshold_mib
m │         XXXXXXXXX..........
o │        XXXXXXXXXXX.........
r │       XXXXXXXXXXXXXXXXX....
y ├─────XXXXXXXXXXXXXXXXXXXXX..────────────  rebound_trigger_threshold_mib
  │   XXXXXXXXXXXXXXXXXXXXXXXXXX.........
  │ XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  └──────────────── time ─────────────────►
     │           |            |
     issue       draining     compaction happens
     starts      begins       and 回收空间

 X - 实际使用的空间
 . - 已申请但不再使用的空间

```
```
extensions:
  file_storage:
  file_storage/all_settings:
    directory: /var/lib/otelcol/mydir
    timeout: 1s
    compaction:
      on_start: true
      directory: /tmp/
      max_transaction_size: 65_536

service:
  extensions: [file_storage, file_storage/all_settings]
  pipelines:
    traces:
      receivers: [nop]
      processors: [nop]
      exporters: [nop]

# Data pipeline is required to load the config.
receivers:
  nop:
processors:
  nop:
exporters:
  nop:
```
## Exportor

### 基础配置
这是其他导出程序可以依赖的helper导出程序。它主要提供排队重试和资源属性到度量标签的转换。
**配置项目：**

- retry_on_failure
   - enabled (default = true)
   - initial_interval (default = 5s): 第一次失败后等待重试的时间；如果enabled为false，则忽略
   - max_interval (default = 30s): 是回退的上限；如果enabled为false，则忽略
   - max_elapsed_time (default = 300s): 尝试发送批处理所花费的最大时间；如果enabled为false，则忽略
- sending_queue
   - enabled (default = true)
   - num_consumers (default = 10): 消费者数量；如果enabled为false，则忽略
   - queue_size (default = 5000): 丢弃前内存中保留的最大批数；如果enabled为false，则忽略。用户应将其计算为num_seconds*request_per_second/request_per_batch，其中：
      - num_seconds是后端中断时要缓冲的秒数
      - requests_per_second是每秒的平均请求数
      - requests_per_batch是每个批次的平均请求数（如果使用批次处理器，则可以使用度量batch_send_size进行估计）
- timeout (default = 5s): 每次尝试向后端发送数据的等待时间
#### 持久化队列
要使用持久队列，需要设置以下设置：
sending_queue：storage (default = none): 设置后，启用持久性并使用指定为持久性队列的存储扩展的组件
可以使用sending_queue控制存储到磁盘的最大批数。queue_size参数（与内存缓冲类似，默认为5000）。
启用持久队列后，将使用提供的存储扩展对批进行缓冲-文件存储是一种流行且安全的选择。如果收集器实例在持久队列中有一些数据时被终止，则在重新启动时将拾取这些数据并继续导出。
```
                                                              ┌─Consumer #1─┐
                                                              │    ┌───┐    │
                              ──────Deleted──────        ┌───►│    │ 1 │    ├───► Success
        Waiting in channel    x           x     x        │    │    └───┘    │
        for consumer ───┐     x           x     x        │    │             │
                        │     x           x     x        │    └─────────────┘
                        ▼     x           x     x        │
┌─────────────────────────────────────────x─────x───┐    │    ┌─Consumer #2─┐
│                             x           x     x   │    │    │    ┌───┐    │
│     ┌───┐     ┌───┐ ┌───┐ ┌─x─┐ ┌───┐ ┌─x─┐ ┌─x─┐ │    │    │    │ 2 │    ├───► Permanent -> X
│ n+1 │ n │ ... │ 6 │ │ 5 │ │ 4 │ │ 3 │ │ 2 │ │ 1 │ ├────┼───►│    └───┘    │      failure
│     └───┘     └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ │    │    │             │
│                                                   │    │    └─────────────┘
└───────────────────────────────────────────────────┘    │
   ▲              ▲     ▲           ▲                    │    ┌─Consumer #3─┐
   │              │     │           │                    │    │    ┌───┐    │
   │              │     │           │                    │    │    │ 3 │    ├───► (in progress)
 write          read    └─────┬─────┘                    ├───►│    └───┘    │
 index          index         │                          │    │             │
   ▲                          │                          │    └─────────────┘
   │                          │                          │
   │                      currently                      │    ┌─Consumer #4─┐
   │                      dispatched                     │    │    ┌───┐    │     Temporary
   │                                                     └───►│    │ 4 │    ├───►  failure
   │                                                          │    └───┘    │         │
   │                                                          │             │         │
   │                                                          └─────────────┘         │
   │                                                                 ▲                │
   │                                                                 └── Retry ───────┤
   │                                                                                  │
   │                                                                                  │
   └────────────────────────────────────── Requeuing  ◄────── Retry limit exceeded ───┘
```
```yaml
receivers:
  otlp:
    protocols:
      grpc:
exporters:
  otlp:
    endpoint: <ENDPOINT>
    sending_queue:
      storage: file_storage/otc
extensions:
  file_storage/otc:
    directory: /var/lib/storage/otc
    timeout: 10s
service:
  extensions: [file_storage]
  pipelines:
    metrics:
      receivers: [otlp]
      exporters: [otlp]
    logs:
      receivers: [otlp]
      exporters: [otlp]
    traces:
      receivers: [otlp]
      exporters: [otlp]
```

# 日志处理
## 日志表示
Entry 是日志数据在管道中移动时的基本表示。所有 operators 都可以创建、修改或消费条目。
```json
{
  "resource": {
    "uuid": "11112222-3333-4444-5555-666677778888",
  },
  "attributes": {
    "env": "prod",
  },
  "body": {
    "message": "Something happened.",
    "details": {
      "count": 100,
      "reason": "event",
    },
  },
  "timestamp": "2020-01-31T00:00:00-00:00",
  "severity": 30,
  "severity_text": "INFO",
}
```

- resource:  用于描述日志源
- attributes：为日志提供附加上下文的键/值对。消费者通常使用此值来过滤日志。
- body:  日志的内容。该值通常在管道中被修改和重组。它可以是字符串、数字或对象。

### 表达式
表达式允许在静态配置中包含动态业务逻辑规则，从而提供了配置灵活性。最值得注意的是，表达式可以用于根据正在处理的日志条目的内容路由消息并添加新字段。
#### 支持的数据类型

- strings - 单引号或双引号 (e.g. "hello", 'hello')
- numbers - e.g. 103, 2.5, .5 ，10_000_000_000
- arrays - e.g. [1, 2, 3]
- maps - e.g. {foo: "bar"}
- booleans - true and false
- nil - nil
#### 数据访问

- 可以使用`.`或`[]`语法访问结构字段和映射元素。
```yaml
foo.Field
bar["some-key"]
```

- 函数访问： 使用括号调用函数foo.Method()

#### 运算符
数学运算符：
● + (addition)
● - (subtraction)
● * (multiplication)
● / (division)
● % (modulus)
● `**` (pow)

比较运算符：
● == (equal)
● != (not equal)
● < (less than)
● > (greater than)
● <= (less than or equal to)
● >= (greater than or equal to)

逻辑运算符：
● not or !
● and or &&
● or or ||

字符串运算符：
● + (concatenation)
● matches (regex match)
● contains (string contains)
● startsWith (has prefix)
● endsWith (has suffix)

```
"hello" matches "h.*"
```



成员运算符：
● in (contain)
● not in (does not contain)

```
user.Group in ["human_resources", "marketing"]
"foo" in {foo: 1, bar: 2}
```

序列运算符：

```
user.Age in 18..45
```

三元运算符

```
user.Age > 30 ? "mature" : "immature"
```



内置函数
● len ：数组， map 或字符串的长度
● all ：如果所有元素都满足谓词，则返回true
● none ：（如果所有元素都不满足谓词，则返回true）
● any ：（如果任何元素满足谓词，则返回true）
● one ：（如果只有一个元素满足谓词，则返回true）
● filter ：（按谓词筛选数组）
● map： （将所有项目与闭包进行映射）
● count ：（返回满足谓词的元素数）

```
all(Tweets, {.Size < 280})
one(Participants, {.Winner})
```

闭包
闭包是一个接受单个参数的表达式。要访问参数，请使用#符号。

```
map(0..9, {# / 2})
```

如果数组的元素是对象，则访问对象字段时，可以省略#符号（#.Value变为.Value）。

```
filter(Tweets, {len(.Value) > 280})
```

切片
```
array = [1,2,3,4,5].
array[1:5] == [2,3,4] 
array[3:] == [4,5] 
array[:4] == [1,2,3] 
array[:] == array
```

collector支持表达式中有几个特殊变量：

- body 
- attributes 
- resource 
- timestamp 
- env() 是一个允许您读取环境变量的函数
```yaml
- type: metadata
  attributes:
    stack: 'EXPR(env("STACK"))'
```

如果 field 不是以 resource, attributes, or body 开头, 则 默认是body . 例如, my_value 等于 body.my_value.
```yaml
- type: add
  field: body.key3
  value: val3
- type: remove
  field: body.key2.nested_key1
- type: add
  field: attributes.my_attribute
  value: my_attribute_value
```
```yaml
{
  "timestamp": "",
  "attributes": {},
  "body": {
    "key1": "value1",
    "key2": {
      "nested_key1": "nested_value1",
      "nested_key2": "nested_value2"
    }
  }
}
```
```yaml
{
  "timestamp": "",
  "attributes": {
    "my_attribute": "my_attribute_value"
  },
  "body": {
    "key1": "value1",
    "key2": {
      "nested_key2": "nested_value2"
    },
    "key3": "value3"
  }
}
```


## 日志文件采集的底层原理
### 指纹
使用指纹识别和跟踪文件。指纹是文件的前N个字节，默认N为1000。
当文件小于N字节时，指纹是文件的全部内容。小于N字节的指纹将使用前缀检查与其他指纹进行比较。随着文件的增长，其指纹将被更新，直到它达到N的完整大小。
处理具有相同指纹的多个文件时，就像它们是同一个文件一样。
最常见的情况是，在依赖于复制/截断策略的文件轮换过程中观察到这种情况。复制文件后，但在截断原始文件之前，两个具有相同内容的文件会短暂存在。如果file_input操作符碰巧同时观察到两个文件，它将检测到重复的指纹并只摄取其中一个文件。
如果将日志复制到多个文件，或者手动复制日志文件，则不认为摄取重复的日志有任何重要价值。因此，指纹无法区分这些文件，并且不支持自动对同一内容进行双重摄取。


Readers 是一个方便的结构，它的存在是为了管理文件及其相关元数据。其包括：

- 文件句柄（可能是打开的，也可能是关闭的）
- 文件指纹
- 文件读取的偏移量（也叫 checkpoint）
- 文件路径
- 解码器
### Reader的功能
正如名称所暗示的，当数据写入文件时，Readers 负责消费数据。
在Readers 开始使用之前，它将查找文件的最后已知偏移量。如果文件的偏移量未知，则读取器将根据`start_at`设置查找文件的开头或结尾。然后从那里开始阅读。
当一个文件比指纹的长度短时，它的读取器会不断地附加到指纹上，因为它会消耗新写入的数据。
Readers 使用bufio.Scanner 消费文件。Scanner 的缓冲区大小由max_log_size设置定义，Scanner 的拆分功能由multiline 设置定义。
当从文件中读取每个日志时，根据编码函数对其进行解码，然后从operator发出。
每当发出日志时，读取器的偏移量都会相应地更新。
### 持久化
Readers总是用打开的文件句柄实例化。最终，文件句柄被关闭，但Readers不会立即被丢弃。相反，它被维护固定数量的轮询周期，作为对文件元数据的引用，这对于检测已移动或复制的文件以及调用元数据可能很有用。
Readers 保持一段固定的时间，然后丢弃。
当file_input操作符使用持久性机制来保存和调用其状态时，它只是设置和获取一部分Readers。这些Readers包含了准确拾取operator 停止的位置所需的所有信息。
### 轮转
文件系统按poll_interval设置定义的定期间隔进行轮询。
每个轮询周期都经过一系列步骤，如下所示：

1. 出队：如果从上一个循环中有任何匹配项排队，则会有适当数量的匹配项出队列，并像处理新匹配的文件集一样进行处理。
2. 老化：如果上一个循环中没有剩余排队的文件，那么所有先前匹配的文件都已被消费，我们准备再次查询文件系统。在这样做之前，我们将增加所有历史Reader的“世代”。最终，这些Reader将根据他们的世代丢弃。
3. 匹配：
   1. 将在文件系统中搜索路径与`include`设置匹配的文件。
   2. 与`exclude `设置匹配的文件将被丢弃。
   3. 作为一种特殊情况，在第一个轮询周期中，如果没有匹配的文件，则会打印警告。不管如何，执行都会继续。
4. 入队：
   1. 如果匹配文件的数量小于或等于max_concurrent_files设置定义的最大并发度，则不会发生排队。
   2. 否则，将发生排队，这意味着：
      1. 匹配的文件被分成两组，第一组足够小，可以容纳max_concurrent_files，第二组包含剩余的文件（称为队列）。
      2. 当前轮询间隔将开始处理第一组文件。
      3. 后续轮询周期将从队列中取出匹配项，直到队列为空。
      4. 始终遵守max_concurrent_files设置。
5. 打开：将打开每个匹配的文件。注：
   1. 自文件匹配以来，已经过了一小段时间。
   2. 此时可能已将其移动或删除。
   3. 在文件匹配和打开之间只应进行最少的一组操作。
   4. 如果打开时发生错误，则会记录该错误。
6. 指纹： 读取每个文件的前N个字节。
7. 排除：
   1. 空文件将立即关闭并丢弃。（没有什么可读的。）
   2. 此批次中发现的指纹相互对照以检测重复。重复的文件将立即关闭并丢弃。在绝大多数情况下，这发生在使用复制/截断方法的文件轮换期间。
8. 创建Reader: 每个文件句柄与一些元数据一起打包到一个Reader中:
   1. 在创建读取器期间，文件的指纹与先前已知的指纹交叉引用。
   2. 如果文件的指纹与最近看到的指纹相匹配，那么元数据将从Reader的上一次迭代中复制过来。最重要的是，以这种方式精确地保持偏移。
   3. 如果文件的指纹与最近看到的任何文件都不匹配，则根据start_at设置初始化其偏移量。
9. 丢失文件的检测:
   1. 指纹用于将此轮询周期中的匹配文件与上一轮询周期的匹配文件进行交叉引用。在上一周期中匹配但在此周期中不匹配的文件称为“丢失文件”。
   2. 文件“丢失”有几个原因:
      1. 文件可能已被删除，通常是由于旋转限制或基于ttl的修剪。
      2. 文件可能已旋转到其他位置。如果文件被移动，则上一个轮询周期中的打开文件句柄可能有用。
10. 消费：
   1. 丢失的文件将被消耗。在某些情况下，如删除，此操作将失败。但是，如果一个文件被移动了，我们可能可以消费它的其余内容。
      1. 我们不希望再次匹配此文件，因此我们所能做的最好的事情就是完成对其当前内容的消费。
      2. 我们可以合理地预期，在大多数情况下，这些文件不再被写入。
   2. 匹配的文件（来自此轮询周期）将被消费。
      1. 这些文件句柄将保持打开状态，直到下一个轮询周期，届时它们将用于检测并潜在地消费丢失的文件。
      2. 通常，我们可以再次找到这些文件中的大部分。然而，这些文件被贪婪地消费，以防我们再也看不到它们。
   3. 同时使用所有打开的文件。这包括上一个周期中丢失的文件和此周期中匹配的文件。
11. 关闭: 上一个轮询周期中的所有文件都将关闭。
12. 归档：
   1. 在当前轮询周期中创建的Reader将添加到历史记录中。
   2. 相同的Reader采用单独的片保留，以便在下一个轮询周期中轻松访问。
13. 修剪：存在3代的Reader被清除。
14. 持久化：Reader的历史记录与提供给operator的任何持久性机制同步。
15. 结束 poll: 此时，operator 处于空闲状态，直到轮询计时器再次启动。

### 其他细节
#### 启动逻辑
每当operator 启动时：

- 请求Reader的历史记录，如轮询周期的步骤12-14所述。
- 启动轮询计时器。
#### 关闭逻辑
当Operator停机时，会发生以下情况：

- 如果当前没有轮询周期，操作员只需关闭所有打开的文件。
- 否则，当前轮询周期会被发出立即停止的信号，这反过来又会向所有Reader发出立即停止信号。
   - 如果Reader空闲或处于日志条目之间，它将立即返回。否则，它将在消耗最后一个日志条目后返回。
   - 一旦所有Reader停止，轮询周期的剩余部分将照常完成，包括标记为关闭、归档、修剪和持久性的步骤。
### 流程的缺陷
**当必须强制执行最大并发时，可能会丢失数据**
如果以下两种情况都成立，Operator可能会丢失少量日志：

1. 匹配的文件数超过了max_concurrent_files设置允许的最大并发度。
2. 文件正在“丢失”。也就是说，文件轮换将文件移出运算符的匹配模式，这样后续轮询周期将找不到这些文件。

当这两种情况都发生时，Oprator不可能同时：

- 遵守指定的并发限制。
- 如果文件从匹配模式中旋转出来，在关闭之前仍可能会被消费。

当这种情况发生时，必须进行设计权衡。可在以下两种情况中选择：

- 确保始终遵守max_concurrent_files。
- 可能会丢失一小部分日志条目。

当前的设计选择保证最大程度的并发，因为如果不这样做，可能会损害Operator的主机系统。虽然日志丢失并不理想，但它不太可能损害Operator的主机系统，因此被认为是两个选项中更容易接受的。
**通过复制/截断进行文件旋转时，可能会丢失数据**
如果以下两种情况都成立，操作员可能会丢失少量日志：

- 正在使用复制/截断策略旋转文件。
- 文件正在“丢失”。也就是说，文件轮换将文件移出运算符的匹配模式，这样后续轮询周期将找不到这些文件。

当这两种情况都发生时，可能会在操作员有机会消费新数据之前将文件写入（然后复制到其他位置），然后截断。
**在Windows上使用通过移动/创建的文件轮换时，可能无法使用文件**
在Windows上，使用“移动/创建”策略旋转文件可能会导致错误和数据丢失，因为Golang当前不支持用于FILE_SHARE_DELETE的Windows机制。
