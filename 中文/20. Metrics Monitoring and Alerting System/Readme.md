# 第 20 章：指标监控与告警系统

## 引言
本章聚焦于设计一个高度可扩展的**指标监控与告警系统**，它对于保障高可用性和可靠性至关重要。

---

## 第 1 步：理解问题并确定设计范围
指标监控系统可以指代很多不同的东西 —— 例如，当面试官只关心基础设施指标时，你就不该去设计一个日志聚合系统。

让我们先试着理解问题：
 - C：我们为谁构建这个系统？是一家大型科技公司的内部监控系统，还是像 DataDog 这样的 SaaS 服务？
 - I：我们仅为内部使用而构建。
 - C：我们想采集哪些指标？
 - I：操作系统指标 —— CPU 负载、内存、数据磁盘空间。但也包括像每秒请求数这样的高层级指标。业务指标不在范围内。
 - C：我们监控的基础设施规模有多大？
 - I：1 亿日活跃用户、1000 个服务器池、每个池 100 台机器。
 - C：数据应该保留多久？
 - I：假设保留 1 年。
 - C：对于长期存储，我们可以降低指标数据的分辨率吗？
 - I：新接收的指标保留 7 天。接下来的 30 天汇总（roll up）为 1 分钟分辨率。30 天后进一步汇总为 1 小时分辨率。
 - C：支持哪些告警渠道？
 - I：电子邮件、电话、PagerDuty 或 webhooks。
 - C：我们需要采集错误日志或访问日志之类的日志吗？
 - I：不需要。
 - C：我们需要支持分布式系统链路追踪吗？
 - I：不需要。

### **高层级需求与假设**
被监控的基础设施是大规模的：
 - 1 亿 DAU
 - 1000 个服务器池 × 100 台机器 × 每台机器约 100 个指标 → 约 1000 万个指标
 - 数据保留 1 年
 - 数据保留策略 —— 原始数据保留 7 天、1 分钟分辨率保留 30 天、1 小时分辨率保留 1 年

可以监控各种各样的指标：
 - CPU 负载
 - 请求计数
 - 内存使用量
 - 消息队列中的消息数

### **非功能性需求**
 - **可扩展性（Scalability）**：系统应可扩展，以容纳更多指标和告警。
 - **低延迟（Low latency）**：系统在仪表盘查询和告警查询上需要有低延迟。
 - **可靠性（Reliability）**：系统应高度可靠，避免遗漏关键告警。
 - **灵活性（Flexibility）**：系统未来应能轻松集成新技术。

哪些需求不在范围内？
 - **日志监控**：ELK 技术栈在这个用例下非常流行。
 - **分布式系统链路追踪**：指的是在一个请求流经系统内多个服务的过程中，采集其完整生命周期的数据。

---

## 第 2 步：提出高层级设计并获得认可

### **基础概念**
指标监控与告警系统涉及五个核心组件：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-monitoring-core-components.png" alt="指标监控核心组件" width="500" />
</div>

 - **数据采集**：从不同来源采集指标数据。
 - **数据传输**：把数据从来源传输到指标监控系统。
 - **数据存储**：组织并存储接收到的数据。
 - **告警**：分析接收到的数据、检测异常并生成告警。
 - **可视化**：用图形、图表等方式展示数据。

### **数据模型**
指标数据通常以时序（time-series）形式记录，包含一系列带时间戳的值。
该序列可以由名称和一组可选的标签（tags）来标识。

示例 1 —— 在 20:00 时，生产服务器实例 i631 上的 CPU 负载是多少？

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-example-1.png" alt="指标示例 1" width="500" />
</div>

该数据可由下表标识：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-example-1-data.png" alt="指标示例 1 的数据" width="500" />
</div>

该时序由指标名称、标签以及某个特定时间点上的单个数据点来标识。

示例 2 —— 过去 10 分钟内，us-west 区域所有 Web 服务器的平均 CPU 负载是多少？

```
CPU.load host=webserver01,region=us-west 1613707265 50

CPU.load host=webserver01,region=us-west 1613707265 62

CPU.load host=webserver02,region=us-west 1613707265 43

CPU.load host=webserver02,region=us-west 1613707265 53

...

CPU.load host=webserver01,region=us-west 1613707265 76

CPU.load host=webserver01,region=us-west 1613707265 83
```

这是我们为回答该问题而可能从存储中取出的示例数据。
平均 CPU 负载可以通过对各行最后一列的值求平均来计算。

上面展示的格式被称为行协议（line protocol），市场上许多流行的监控软件都在使用它 —— 例如 Prometheus、OpenTSDB。

每一条时序由什么组成：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/time-series-data-example.png" alt="时序数据示例" width="500" />
</div>

一个很好的数据可视化方式：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/time-series-data-viz.png" alt="时序数据可视化" width="500" />
</div>

 - x 轴是时间。
 - y 轴是你正在查询的维度 —— 例如指标名称、标签等。

数据访问模式是写多读少，且读呈突发状：我们采集了大量指标，但它们并不经常被访问，不过在出现正在处理的事故等情况下会突发式访问。

数据存储系统是本设计的核心。
 - 不建议为这个问题使用通用数据库，尽管凭借专家级调优你也能达到不错的规模。
 - 使用 NoSQL 数据库在理论上可行，但很难设计出可扩展的 schema，来高效存储和查询时序数据。

有许多数据库是专门为存储时序数据而定制的。其中许多都支持自定义查询接口，可以高效地查询时序数据。
 - OpenTSDB 是一个分布式时序数据库，但它基于 Hadoop 和 HBase。如果你没有预先准备好那套基础设施，就很难使用这项技术。
 - Twitter 使用 MetricsDB，而 Amazon 提供 Timestream。
 - 最流行的两个时序数据库是 InfluxDB 和 Prometheus。
 - 它们都是为存储大规模时序数据而设计的。两者都基于「内存缓存 + 磁盘存储」。

InfluxDB 的一个规模示例 —— 在配备 8 核 CPU 和 32GB 内存的配置下，每秒可写入超过 25 万次：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/influxdb-scale.png" alt="InfluxDB 规模" width="500" />
</div>

并不要求你理解指标数据库的内部原理，因为那属于冷门知识。只有当你在简历上提到过它时，才可能被问到。

就面试而言，理解「指标就是时序数据」并了解像 InfluxDB 这样流行的时序数据库就足够了。

时序数据库的一个不错特性是能够按标签对大量时序数据进行高效聚合与分析。
例如，InfluxDB 会为每个标签建立索引。

然而，关键在于保持标签的低基数（cardinality）—— 即不要使用太多唯一标签。

### **高层级设计**

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/high-level-design.png" alt="高层级设计" width="500" />
</div>

 - **指标来源（Metrics source）**：可以是应用服务器、SQL 数据库、消息队列等。
 - **指标采集器（Metrics collector）**：采集指标数据并写入时序数据库。
 - **时序数据库（Time-series database）**：把指标存储为时序。提供自定义查询接口，用于分析大量指标。
 - **查询服务（Query service）**：让从时序 DB 中查询和检索数据变得容易。如果 DB 自身的接口足够强大，查询服务可以完全被其取代。
 - **告警系统（Alerting system）**：向各种告警目的地发送告警通知。
 - **可视化系统（Visualization system）**：以图形/图表的形式展示指标。

---

## 第 3 步：深入设计
让我们深入探讨系统中几个更有意思的部分。

### **指标采集**
对于指标采集，偶尔的数据丢失并不关键。客户端采用「发后即忘」（fire and forget）是可以接受的。

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-collection.png" alt="指标采集" width="500" />
</div>

指标采集有两种实现方式 —— 拉取（pull）或推送（push）。

下面是拉取模型可能的样子：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/pull-model-example.png" alt="拉取模型示例" width="500" />
</div>

对于这种方案，指标采集器需要维护一份最新的服务列表和指标端点列表。
为此我们可以使用 ZooKeeper 或 etcd —— 即服务发现（service discovery）。

服务发现包含关于「何时、从何处采集指标」的配置规则：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/service-discovery-example.png" alt="服务发现示例" width="500" />
</div>

下面是指标采集流程的详细说明：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-collection-flow.png" alt="指标采集流程" width="500" />
</div>

 - 指标采集器从服务发现拉取配置元数据。这包括拉取间隔、IP 地址、超时与重试参数。
 - 指标采集器通过预定义的 http 端点（例如 `/metrics`）拉取指标数据。这通常由客户端库完成。
 - 另一种方式是，指标采集器向服务发现注册变更事件通知，以便在服务端点发生变化时收到通知。
 - 还有一种方式是，指标采集器定期轮询指标端点的配置变更。

在我们的规模下，单个指标采集器是不够的。必须有多个实例。
然而，它们之间也必须有某种同步机制，以避免两个采集器把同一批指标采集两遍。

一个解决方案是把采集器和服务器放在一致性哈希环（consistent hash ring）上，并让一组服务器只归属给一个采集器：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/consistent-hash-ring.png" alt="一致性哈希环" width="500" />
</div>

而在推送模型下，服务会主动把自己的指标推送给指标采集器：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/push-model-example.png" alt="推送模型示例" width="500" />
</div>

在这种方式中，通常会在服务实例旁边安装一个采集代理（collection agent）。
该代理从服务器采集指标，并把它们推送给指标采集器。

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-collector-agent.png" alt="指标采集器代理" width="500" />
</div>

在这个模型下，我们有可能在把指标发给采集器之前先做聚合，从而减少采集器处理的数据量。

另一方面，指标采集器在扛不住负载时可以拒绝推送请求。
因此，把采集器放到负载均衡器后面的自动扩缩容组（auto-scaling group）中非常重要。

那么哪一种更好呢？两种方案各有权衡，不同的系统采用不同的方式：
 - Prometheus 采用拉取架构。
 - Amazon CloudWatch 和 Graphite 采用推送架构。

下面是推送与拉取之间的一些主要差异：
|                                        | 拉取（Pull）                                                                                                                                                                                                    | 推送（Push）                                                                                                                                                                                                                                    |
|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 易于调试                         | 应用服务器上用于拉取指标的 `/metrics` 端点可随时用来查看指标。你甚至可以在自己的笔记本上做这件事。拉取胜出。                                          | 如果指标采集器没收到指标，问题可能出在网络上。                                                                                                                                        |
| 健康检查                           | 如果某台应用服务器不响应拉取，你可以快速判断出它是否宕机。拉取胜出。                                                                           | 如果指标采集器没收到指标，问题可能出在网络上。                                                                                                                                        |
| 短生命周期任务                       |                                                                                                                                                                                                         | 有些批处理任务生命周期很短，存活时间短到来不及被拉取。推送胜出。对拉取模型而言，可以通过引入推送网关（push gateway）来解决这个问题 [22]。                                                                 |
| 防火墙或复杂网络设置 | 让服务器拉取指标要求所有指标端点都可达。这在多数据中心设置下可能有问题。可能需要更复杂的网络基础设施。 | 如果指标采集器搭配负载均衡器和自动扩缩容组搭建，则可以从任何地方接收数据。推送胜出。                                                                                             |
| 性能                            | 拉取方式通常使用 TCP。                                                                                                                                                                         | 推送方式通常使用 UDP。这意味着推送方式能以更低延迟传输指标。反方观点是：与发送指标负载相比，建立 TCP 连接的开销很小。 |
| 数据真实性                      | 从哪些应用服务器采集指标是在配置文件中预先定义好的。从这些服务器采集到的指标保证是真实可信的。                                                 | 任何客户端都可以向指标采集器推送指标。可以通过设置白名单（只接受白名单服务器的指标）或要求认证来解决。                                                                   |

没有明确的赢家。一个大型组织很可能需要同时支持两种。而且一开始可能根本就没法安装推送代理。

### **扩展指标传输管道**

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-transmission-pipeline.png" alt="指标传输管道" width="500" />
</div>

无论使用推送还是拉取模型，指标采集器都部署在一个自动扩缩容组中。

然而，如果时序 DB 宕机，仍有数据丢失的可能。为缓解这一点，我们会部署一个排队机制：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/queuing-mechanism.png" alt="排队机制" width="500" />
</div>

 - 指标采集器把指标数据推送到 kafka。
 - 消费者或流处理服务（如 Apache Storm、Flink 或 Spark）处理数据，并把它推送到时序 DB。

这种方式有几个优点：
 - Kafka 被用作高可靠、可扩展的分布式消息平台。
 - 它把数据采集与数据处理解耦。
 - 它可以通过在 Kafka 中保留数据来防止数据丢失。

Kafka 可以配置为每个指标名一个分区，这样消费者就可以按指标名聚合数据。
要进一步扩展，我们可以再按标签/标签集（tags/labels）做分区，并对指标分类、排定优先级，优先采集最重要的指标。

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-collection-kafka.png" alt="指标采集 Kafka" width="500" />
</div>

在这个问题上使用 Kafka 的主要缺点是维护/运营开销。
一个替代方案是使用像 [Gorilla](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf) 这样的大规模摄入系统。
可以论证，使用它所能达到的可扩展性，与使用 Kafka 做排队是相当的。

### **聚合可以发生在哪里**
指标可以在几个地方聚合。不同选择之间各有权衡：
 - **采集代理（Collection agent）**：客户端采集代理只支持简单的聚合逻辑。例如对一个计数器采集 1 分钟，再发给指标采集器。
 - **摄入管道（Ingestion pipeline）**：要在写入 DB 之前聚合数据，我们需要像 Flink 这样的流处理引擎。这减少了写入量，但由于不存储原始数据，我们会损失数据精度。
 - **查询侧（Query side）**：我们可以在通过可视化系统运行查询时聚合数据。不会丢失数据，但由于要处理大量数据，查询可能较慢。

### **查询服务**
让查询服务与时序 DB 相互独立，可以把可视化与告警系统从数据库解耦出来，从而让我们可以把 DB 与客户端解耦、并随意更换它。

我们可以在这里加一层缓存，以减轻时序数据库的负载：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/cache-layer-query-service.png" alt="查询服务的缓存层" width="500" />
</div>

我们也可以完全不单独引入查询服务，因为大多数可视化和告警系统都有强大的插件，可以与大多数时序数据库集成。
如果时序 DB 选得好，我们甚至可能不需要引入自己的缓存层。

大多数时序 DB 不支持 SQL，原因很简单：SQL 查询时序数据效率低下。下面是一个计算指数移动平均（exponential moving average）的 SQL 查询示例：

```
select id,
       temp,
       avg(temp) over (partition by group_nr order by time_read) as rolling_avg
from (
  select id,
         temp,
         time_read,
         interval_group,
         id - row_number() over (partition by interval_group order by time_read) as group_nr
  from (
    select id,
    time_read,
    "epoch"::timestamp + "900 seconds"::interval * (extract(epoch from time_read)::int4 / 900) as interval_group,
    temp
    from readings
  ) t1
) t2
order by time_read;
```

下面是同一个查询用 Flux（InfluxDB 使用的查询语言）表达的样子：

```
from(db:"telegraf")
  |> range(start:-1h)
  |> filter(fn: (r) => r._measurement == "foo")
  |> exponentialMovingAverage(size:-10s)
```

### **存储层**
谨慎选择时序数据库非常重要。

根据 Facebook 发表的研究，对运维存储的查询中，约 85% 都是查询过去 26 小时内的数据。

如果我们选择一个能利用这一特性的数据库，可能会对系统性能产生重大影响。InfluxDB 就是这样一个选择。

无论选择哪个数据库，我们都可以采用一些优化手段。

数据编码和压缩可以显著减小数据体积。这些特性通常内置在优秀的时序数据库中。

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/double-delta-encoding.png" alt="双差分编码" width="500" />
</div>

在上面的例子中，我们不必存储完整时间戳，而可以存储时间戳的差值（delta）。

另一个可以采用的技术是降采样（down-sampling）—— 把高分辨率数据转换为低分辨率，以减少磁盘占用。

我们可以对旧数据使用降采样，并让规则由数据科学家配置，例如：
 - 7 天 —— 不降采样
 - 30 天 —— 降采样到 1 分钟
 - 1 年 —— 降采样到 1 小时

例如，下面是一张 10 秒分辨率的指标表：
| metric | timestamp            | hostname | Metric_value |
|--------|----------------------|----------|--------------|
| cpu    | 2021-10-24T19:00:00Z | host-a   | 10           |
| cpu    | 2021-10-24T19:00:10Z | host-a   | 16           |
| cpu    | 2021-10-24T19:00:20Z | host-a   | 20           |
| cpu    | 2021-10-24T19:00:30Z | host-a   | 30           |
| cpu    | 2021-10-24T19:00:40Z | host-a   | 20           |
| cpu    | 2021-10-24T19:00:50Z | host-a   | 30           |

降采样到 30 秒分辨率：
| metric | timestamp            | hostname | Metric_value (avg) |
|--------|----------------------|----------|--------------------|
| cpu    | 2021-10-24T19:00:00Z | host-a   | 19                 |
| cpu    | 2021-10-24T19:00:30Z | host-a   | 25                 |

最后，我们还可以使用冷存储（cold storage）来存放不再使用的旧数据。冷存储的财务成本要低得多。

### **告警系统**

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/alerting-system.png" alt="告警系统" width="500" />
</div>

配置被加载到缓存服务器中。规则通常以 YAML 格式定义。下面是一个示例：

```
- name: instance_down
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: instance_down
    expr: up == 0
    for: 5m
    labels:
      severity: page
```

告警管理器（alert manager）从缓存中拉取告警配置。根据配置规则，它还会以预定义的间隔调用查询服务。
如果某条规则被满足，就会创建一个告警事件。

告警管理器的其他职责还有：
 - 过滤、合并和去重告警。例如，如果单个实例的告警被多次触发，只生成一个告警事件。
 - 访问控制 —— 把告警管理操作限制给特定人员非常重要。
 - 重试 —— 管理器保证告警至少被传递一次。

告警存储是一个键值数据库（如 Cassandra），它保存所有告警的状态。它保证通知至少被发送一次。
一旦告警被触发，就会被发布到 Kafka。

最后，告警消费者从 Kafka 拉取告警数据，并通过不同渠道发送通知 —— 电子邮件、短信、PagerDuty、webhooks。

在现实世界中，告警系统有许多现成的解决方案。很难论证为什么要自己从零搭建一个内部系统。

### **可视化系统**
可视化系统展示一段时间内的指标和告警。下面是一个用 Grafana 搭建的仪表盘：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/grafana-dashboard.png" alt="Grafana 仪表盘" width="500" />
</div>

一个高质量的可视化系统非常难搭建。很难论证为什么不直接用像 Grafana 这样的现成方案。

---

## 第 4 步：收尾
下面是我们的最终设计：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/final-design.png" alt="最终设计" width="500" />
</div>
