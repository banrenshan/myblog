---
title: loki
tags:
  - loki
  - 监控
categories:
  - 中间件
date: 2022-11-27 19:52:20
---

# 架构

![img](loki/1668838005010-12be54ec-8ee0-4f08-83ba-64ce1caf5da9.svg+xml)

## Distributor

分发服务器服务负责处理客户端的传入流。这是日志数据写入路径的第一站。一旦分发服务器接收到一组流，就验证每个流的正确性，并确保其在配置的租户（或全局）限制内。然后将有效的块分割成批，并并行发送给多个[ingesters](https://grafana.com/docs/loki/latest/fundamentals/architecture/components/#ingester)。

### Validation

distributor 采取的第一步是确保所有传入数据符合规范。这包括检查标签是否是有效的Prometheus标签，以及确保时间戳不是太旧或太新，或者日志行不是太长。

### Preprocessing

目前，distributor改变传入数据的唯一方法是规范标签。这意味着使{foo=“bar”，bazz=“buzz”}等同于{bazz=”buzz“，foo=”bar“}，或者换句话说，对标签进行排序。这使得Loki可以确定地缓存和散列它们。

### Rate limiting

distributor还可以根据每个租户对传入日志进行速率限制。它通过检查每个租户的限额并将其除以当前的distributor来实现这一点。这允许在集群级别为每个租户指定速率限制，并使我们能够向上或向下扩展distributor，并相应地调整每个distributor的限额。例如，假设我们有10个distributor，租户A有10MB的费率限制。在限制之前，每个分配器最多允许1MB/秒。现在，假设另一个大租户加入集群，我们需要再组建10个distributor。现在的20家distributor将调整其租户A的费率限制为（10MB/20家distributor）=500KB/s！这就是为什么全局限制允许Loki集群更简单、更安全的操作。

### Forwarding

一旦distributor完成了所有的验证任务，它就会将数据转发给最终负责确认写入的ingester 组件。

#### Replication factor

为了减少在任何单个ingester上丢失数据的可能性，distributor将在转发写操作时添加复制因子。通常情况下，这是3。复制允许在写入失败的情况下重新启动和rollouts ingester，并在某些情况下增加了防止数据丢失的额外保护。

对于推送到distributor的每个标签集（称为流），它将对标签进行哈希，根据hash值查找环中的 ingesters （需要多个，根据replication_factor决定）。然后，它将尝试将相同的数据写入到多个ingesters。如果成功写入的次数少于法定人数（*quorum* ），则会出错。

*quorum* 被定义为floor（replication_factor/2）+1。因此，对于replication_factor为3，我们需要两次写入成功。如果成功写入的次数少于两次，则distributor返回错误，可以重试写入。



不过，复制因素并不是防止数据丢失的唯一方式。ingester 组件现在包括一个预写日志，该日志保存对磁盘的传入写入，以确保在磁盘未损坏的情况下不会丢失这些写入。复制因子和WAL的互补性确保了数据不会丢失，除非这两种机制都出现重大故障（即多个摄取者死亡并丢失/损坏其磁盘）。

### Hashing

分发服务器使用一致的哈希和可配置的复制因子来确定ingester 服务的哪些实例应该接收给定的流。

流是与租户和唯一标签集相关联的一组日志。使用租户ID和标签集对流进行哈希，然后使用哈希查找要将流发送到的ingester 。

为了进行哈希查找，分发者会找到值大于流哈希值的最小适当令牌。当复制因子大于1时，属于不同ingester 的下一个后续令牌（环中的顺时针方向）也将包含在结果中。

这种哈希设置的效果是，ingester 拥有的每个令牌都负责一系列哈希。如果存在值为0、25和50的三个令牌，ingester 拥有令牌25负责1-25的哈希范围。

## Ingester

**ingester**服务负责将日志数据写入写入路径上的长期存储后端（DynamoDB、S3、Cassandra等），并在读取路径上返回内存查询的日志数据。



**ingester**接收的每个日志流都在内存中构建成一组多个“块”，并以可配置的间隔刷新到备份存储后端。在以下情况下，块将被压缩并标记为只读：

- 当前区块已达到容量（可配置值）。
- 当前区块已过了太多时间没有更新
- 发生flush。

每当一个块被压缩并标记为只读时，一个可写块就会取代它。

如果**ingester**进程突然崩溃或退出，所有尚未刷新的数据都将丢失。Loki通常被配置为复制每个日志的多个副本（通常为3个），以减轻此风险。

当持久存储提供程序发生刷新时，将根据其租户、标签和内容对块进行哈希。这意味着具有相同数据副本的多个**ingester**不会将相同数据写入备份存储两次，但如果对其中一个副本的任何写入失败，将在备份存储中创建多个不同的块对象。有关如何消除重复数据的信息，请参阅查询器。

### Timestamp Ordering

当未配置为接受无序写入时，摄取器将验证摄取的日志行是否正常。当摄取者接收到不符合预期顺序的日志行时，该行将被拒绝，并向用户返回错误。
 摄取器验证日志行是否按时间戳升序接收。每个日志都有一个时间戳，该时间戳发生的时间晚于之前的日志。当摄取器接收到不遵循此顺序的日志时，将拒绝日志行并返回错误。

如果ingester 进程突然崩溃或退出，所有尚未刷新的数据都可能丢失。Loki通常配置有预写日志，该日志可以在ingester 重新启动时重播，并且每个日志的复制因子（通常为3）可以减轻此风险。

当未配置为接受无序写入时，针对给定流（标签的唯一组合）推送到Loki的所有行必须具有比之前接收到的行更新的时间戳。然而，有两种情况可用于处理具有相同纳秒时间戳的同一流的日志：

- 如果传入的行与先前接收的行完全匹配（同时匹配先前的时间戳和日志文本），则传入的行将被视为完全重复的行并被忽略。
- 如果传入行具有与前一行相同的时间戳，但内容不同，则接受日志行。这意味着可以对同一时间戳使用两个不同的日志行。



虽然ingesters 确实支持通过BoltDB写入文件系统，但这只在单进程模式下工作，因为查询者需要访问同一后端存储，而BoltDB只允许一个进程在给定时间锁定数据库。

## Query frontend

查询前端将较大的查询拆分为多个较小的查询，在下游查询器上并行执行这些查询，并再次将结果拼接在一起。这可以防止大型（多天等）查询在单个查询器中导致内存不足问题，并有助于更快地执行它们。



查询前端在内部执行一些查询调整，并将查询保存在内部队列中。在此设置中，查询器充当从队列中提取作业、执行作业并将其返回到查询前端进行聚合的工作人员。查询器需要配置查询前端地址（通过-queries.frontend-address CLI标志），以允许它们连接到查询前端。

查询前端排队机制用于：

- 确保在失败时重试可能导致查询器内存不足（OOM）错误的大型查询。这允许管理员为查询提供不足的内存，或者乐观地并行运行更多的小查询，这有助于降低TCO。
- 通过使用先进先出队列（FIFO）将多个大型请求分发到所有查询器，防止在单个查询器上护送这些请求。
- 通过公平调度租户之间的查询，防止单个租户一直占用而拒绝服务（DOSing）其他租户。



查询前端支持缓存度量查询结果，并在后续查询中重用它们。如果缓存的结果不完整，查询前端将计算所需的子查询，并在下游查询器上并行执行它们。查询前端可以选择将查询与其步骤参数对齐，以提高查询结果的可缓存性。结果缓存与任何loki缓存后端兼容（当前为memcached、redis和内存缓存）。

缓存日志（过滤器、正则表达式）查询正在积极开发中。

## Querier

querier服务使用LogQL查询语言处理查询，从ingesters 和长期存储中获取日志。

查询器先查询所有ingesters中的内存数据，查询不到则去后端存储运行相同的查询。由于复制因素，查询器可能会收到重复的数据。为了解决这个问题，查询器在内部对具有相同纳秒时间戳、标签集和日志消息的数据进行重复数据消除。



# 原理

![img](loki/1668841189426-666d5442-8a15-40e7-b356-3ebe1762a176.png)

# Rule

Grafana Loki 包含一个名为 Ruler 的组件，负责持续**评估**一组可配置的**查询**并根据结果**执行操作,例如触发告警**。支持两种rules：

- 告警：根据查询的结果，判断是否触发告警
- 记录(recording)：将查询的结果作为指标数据发送的存储后端，用于可视化。

## 告警规则

我们支持与 Prometheus 兼容的警报规则。 警报规则允许您根据 Prometheus 表达式语言表达式定义警报条件，并将有关触发警报的通知发送到外部服务。示例如下：

```yaml
groups:
  - name: should_fire
    rules:
      - alert: HighPercentageError
        expr: |
          sum(rate({app="foo", env="production"} |= "error" [5m])) by (job)
            /
          sum(rate({app="foo", env="production"}[5m])) by (job)
            > 0.05
        for: 10m
        labels:
            severity: page
        annotations:
            summary: High request latency
  - name: credentials_leak
    rules: 
      - alert: http-credentials-leaked
        annotations: 
          message: "{{ $labels.job }} is leaking http basic auth credentials."
        expr: 'sum by (cluster, job, pod) (count_over_time({namespace="prod"} |~ "http(s?)://(\\w+):(\\w+)@" [5m]) > 0)'
        for: 10m
        labels: 
          severity: critical
```

## 记录规则

支持 与 Prometheus 兼容的记录规则。 记录规则允许您预先计算经常需要或计算量大的表达式，并将其结果保存为一组新的时间序列。

Loki允许您在日志上运行度量查询，这意味着您可以从日志中导出数字聚合，例如从NGINX访问日志中计算一段时间内的请求数。

```yaml
name: NginxRules
interval: 1m
rules:
  - record: nginx:requests:rate1m
    expr: |
      sum(
        rate({container="nginx"}[1m])
      )
    labels:
      cluster: "us-central1"
```

此查询（expr）将每1分钟（间隔）执行一次，其结果将存储在我们定义的度量名称（record）中。这个名为nginx:requests:rate1m的度量现在可以发送到普罗米修斯存储。

### 配置远程写入

```yaml
ruler:
  ... other settings ...
  
  remote_write:
    enabled: true
    client:
      url: http://localhost:9090/api/v1/write
```











## 规则的存储

Ruler支持五种存储：azure、gcs、s3、swift和local。大多数类型的存储共享rule配置，即将所有rule配置为使用同一后端。

local实现从本地文件系统中读取规则文件。这是一个只读后端，不支持通过Ruler API创建和删除规则。尽管它读取本地文件系统，但如果操作员注意将相同的规则加载到每个Ruler，则该方法仍然可以在分片Ruler配置中使用。例如，这可以通过在每个Ruler pod上安装Kubernetes ConfigMap来实现。

典型的local配置可能类似于：

```yaml
  -ruler.storage.type=local
  -ruler.storage.local.directory=/tmp/loki/rules
```

使用上述配置，rule将预期以下布局：

```yaml
/tmp/loki/rules/<tenant id>/rules1.yaml
                           /rules2.yaml
```





# 存储

Grafana Loki需要存储两种不同类型的数据：块和索引。

Loki在单独的流中接收日志，其中每个流由其租户ID和标签集唯一标识。当来自流的日志条目到达时，它们被压缩为“块”并保存在块存储中。

索引存储每个流的标签集，并将它们链接到各个块。
