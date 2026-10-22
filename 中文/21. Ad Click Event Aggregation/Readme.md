# 第 21 章：广告点击事件聚合

## 引言
随着 Facebook、YouTube、TikTok 等平台的崛起，**数字广告**已成为一个庞大的产业。

因此，跟踪广告点击事件十分重要。在本章中，我们探讨如何设计一个达到 Facebook/Google 规模的**广告点击事件聚合**系统。

数字广告中有一个称为**实时竞价（real-time bidding，RTB）**的流程，数字广告库存（inventory）通过它进行买卖：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/digital-advertising-example.png" alt="数字广告示例" width="500" />
</div>

RTB 的速度很重要，因为它通常需要在一秒内完成。
数据准确性也非常重要，因为它直接影响广告商支付的金额。

基于广告点击事件聚合结果，广告商可以做出诸如调整目标受众和关键词之类的决策。

---

## 第 1 步：理解问题并确定设计范围
 - 候选人（C）：输入数据的格式是什么？
 - 面试官（I）：每天 10 亿次广告点击，总计 200 万个广告。广告点击事件数量每年同比增长 30%。
 - 候选人（C）：我们的系统需要支持的最重要的查询有哪些？
 - 面试官（I）：需要重点考虑的查询：
   - 返回过去 Y 分钟内广告 X 的点击事件数量
   - 返回过去 1 分钟内点击量最高的前 100 个广告。两个参数都应可配置。聚合每分钟执行一次。
   - 支持按 `ip`、`user_id`、`country` 对上述查询进行数据过滤
 - 候选人（C）：我们需要考虑边界情况吗？我能想到的一些：
   - 可能会有事件晚于预期时间到达
   - 可能会有重复事件
   - 系统的不同部分可能宕机，因此我们需要考虑系统恢复
 - 面试官（I）：这个清单不错，把这些都考虑进去
 - 候选人（C）：延迟要求是多少？
 - 面试官（I）：广告点击聚合的端到端（e2e）延迟为几分钟。对于 RTB，则要求小于一秒。广告点击聚合有这样的延迟是可以接受的，因为它们通常用于计费和报表。

### **功能需求**
 - 聚合过去 Y 分钟内 `ad_id` 的点击次数
 - 每分钟返回点击量最高的前 100 个 `ad_id`
 - 支持按不同属性进行聚合过滤
 - 数据规模达到 Facebook 或 Google 级别

### **非功能需求**
 - 聚合结果的正确性很重要，因为它用于 RTB 和广告计费
 - 妥善处理延迟到达或重复的事件
 - 健壮性——系统应能抵御部分故障
 - 延迟——端到端延迟最多几分钟

### **粗略估算（Back-of-the-envelope estimation）**
 - 10 亿 DAU
 - 假设每个用户每天点击 1 个广告 → 每天 10 亿次广告点击
 - 广告点击 QPS = 10,000
 - 峰值 QPS 是该数值的 5 倍 = 50,000
 - 单次广告点击占用 0.1KB 存储。每日存储需求为 100GB
 - 每月存储 = 3TB

---

## 第 2 步：提出高层设计并获得认可
在本节中，我们讨论查询 API 设计、数据模型和高层设计。

### **查询 API 设计**
API 是客户端与服务器之间的契约。在我们的场景中，客户端是看板（dashboard）的使用者——数据科学家/分析师、广告商等。

以下是我们的功能需求：
 - 聚合过去 Y 分钟内 `ad_id` 的点击次数
 - 返回过去 M 分钟内点击量最高的前 N 个 `ad_id`
 - 支持按不同属性进行聚合过滤

我们需要两个端点来实现这些需求。其中一个端点可以通过查询参数来完成过滤。

**聚合过去 M 分钟内 ad_id 的点击次数**：

```
GET /v1/ads/{:ad_id}/aggregated_count
```

查询参数：
 - from——起始分钟。默认为现在 - 1 分钟
 - to——结束分钟。默认为现在
 - filter——不同过滤策略的标识符。例如 001 表示「非美国点击」。

响应：
 - ad_id——广告标识符
 - count——起始与结束分钟之间的聚合计数

**返回过去 M 分钟内点击量最高的前 N 个 ad_id**

```
GET /v1/ads/popular_ads
```

查询参数：
 - count——点击量最高的前 N 个广告
 - window——聚合窗口大小，单位为分钟
 - filter——不同过滤策略的标识符

响应：
 - ad_id 列表

### **数据模型**
在我们的系统中，既有原始数据，也有聚合数据。

原始数据如下所示：

```
[AdClickEvent] ad001, 2021-01-01 00:00:01, user 1, 207.148.22.22, USA
```

下面是结构化格式的示例：
| ad_id | click_timestamp     | user  | ip            | country |
|-------|---------------------|-------|---------------|---------|
| ad001 | 2021-01-01 00:00:01 | user1 | 207.148.22.22 | USA     |
| ad001 | 2021-01-01 00:00:02 | user1 | 207.148.22.22 | USA     |
| ad002 | 2021-01-01 00:00:02 | user2 | 209.153.56.11 | USA     |

下面是聚合后的版本：
| ad_id | click_minute | filter_id | count |
|-------|--------------|-----------|-------|
| ad001 | 202101010000 | 0012      | 2     |
| ad001 | 202101010000 | 0023      | 3     |
| ad001 | 202101010001 | 0012      | 1     |
| ad001 | 202101010001 | 0023      | 6     |

`filter_id` 帮助我们实现过滤需求。
| filter_id | region | IP        | user_id |
|-----------|--------|-----------|---------|
| 0012      | US     | *         | *       |
| 0013      | *      | 123.1.2.3 | *       |

为了支持快速返回过去 M 分钟内点击量最高的前 N 个广告，我们还会维护如下结构：
| most_clicked_ads   |           |                                                  |
|--------------------|-----------|--------------------------------------------------|
| window_size        | integer   | 聚合窗口大小（M），单位为分钟                     |
| update_time_minute | timestamp | 最后更新时间戳（以 1 分钟为粒度）                 |
| most_clicked_ads   | array     | 广告 ID 列表，JSON 格式。                        |

存储原始数据与存储聚合数据各有哪些优缺点？
 - 原始数据能够使用完整数据集，并支持数据过滤和重新计算
 - 另一方面，聚合数据让我们拥有更小的数据集和更快的查询
 - 原始数据意味着更大的数据存储和更慢的查询
 - 然而，聚合数据是派生数据，因此会有一定的数据损失。

在我们的设计中，将结合使用这两种方式：
 - 保留原始数据用于调试是个好主意。如果聚合过程中出现 bug，我们可以发现该 bug 并进行回填（backfill）。
 - 聚合数据也应存储，以获得更快的查询性能。
 - 原始数据可以存储在冷存储（cold storage）中，以避免额外的存储成本。

在数据库选择方面，有几个因素需要考虑：
 - 数据是什么样的？是关系型、文档型还是 blob？
 - 工作负载是读多、写多还是读写都多？
 - 是否需要事务？
 - 查询是否依赖 OLAP 函数，如 SUM 和 COUNT？

对于原始数据，我们可以看到平均 QPS 为 10k，峰值 QPS 为 50k，因此系统是写密集型（write-heavy）的。
另一方面，读流量很低，因为原始数据主要在出现问题时用作备份。

关系型数据库可以胜任这项工作，但扩展写入能力可能具有挑战性。
或者，我们可以使用 Cassandra 或 InfluxDB，它们对重写入负载有更好的原生支持。

另一种选择是使用 Amazon S3，并采用列式数据格式，如 ORC、Parquet 或 AVRO。由于这种方案我们不太熟悉，我们将继续使用 Cassandra。

对于聚合数据，工作负载既是读密集型也是写密集型，因为聚合数据会被持续查询以用于看板和告警。
它也是写密集型的，因为聚合服务每分钟都会对数据进行聚合并写入。
因此，这里我们也使用相同的数据存储（Cassandra）。

### **高层设计**
我们的系统如下所示：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/high-level-design-1.png" alt="高层设计 1" width="500" />
</div>

数据在输入和输出两端都以无界数据流（unbounded data stream）的形式流动。

为了避免采用同步的数据落点（sink）——即某个消费者崩溃可能导致整个系统停滞——
我们将利用消息队列（Kafka）进行异步处理，以解耦消费者和生产者。

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/high-level-design-2.png" alt="高层设计 2" width="500" />
</div>

第一个消息队列存储广告点击事件数据：
| ad_id | click_timestamp | user_id | ip | country |
|-------|-----------------|---------|----|---------|

第二个消息队列包含按分钟聚合的广告点击计数：
| ad_id | click_minute | count |
|-------|--------------|-------|

以及按分钟聚合的点击量最高的前 N 个广告：
| update_time_minute | most_clicked_ads |
|--------------------|------------------|

第二个消息队列的存在是为了实现端到端的恰好一次（exactly-once）原子提交语义：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/atomic-commit.png" alt="原子提交" width="500" />
</div>

对于聚合服务，使用 MapReduce 框架是一个不错的选择：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/ad-count-map-reduce.png" alt="广告计数 MapReduce" width="500" />
</div>

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/top-100-map-reduce.png" alt="前 100 名 MapReduce" width="500" />
</div>

每个节点负责单个任务，并将处理结果发送给下游节点。

map 节点负责从数据源读取数据，然后对数据进行过滤和转换。

例如，map 节点可以根据 `ad_id` 将数据分配到不同的聚合节点：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/map-node.png" alt="map 节点" width="500" />
</div>

或者，我们可以将广告分布到 Kafka 分区中，让聚合节点在一个消费者组（consumer group）内直接订阅。
然而，映射（mapping）节点使我们能够在后续处理之前对数据进行清洗（sanitize）或转换。

另一个原因可能是我们无法控制数据的产生方式，
因此与同一个 `ad_id` 相关的事件可能会进入不同的分区。

聚合（aggregate）节点每分钟在内存中按 `ad_id` 统计广告点击事件数量。

reduce 节点从聚合节点收集聚合结果，并生成最终结果：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/reduce-node.png" alt="reduce 节点" width="500" />
</div>

这个 DAG 模型采用 MapReduce 范式。它接收大数据，并利用并行分布式计算将其转化为常规规模的数据。

在 DAG 模型中，中间数据存储在内存中，不同节点之间通过 TCP 或共享内存进行通信。

让我们看看这个模型现在如何帮助我们实现各种用例。

**用例 1——聚合点击次数**：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/use-case-1.png" alt="用例 1" width="500" />
</div>

 - 广告使用 `ad_id % 3` 进行分区

**用例 2——返回点击量最高的前 N 个广告**：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/use-case-2.png" alt="用例 2" width="500" />
</div>

 - 在本例中，我们聚合的是前 3 个广告，但这可以很容易地扩展到前 N 个广告
 - 每个节点维护一个堆（heap）数据结构，以便快速检索前 N 个广告

**用例 3——数据过滤**：
为了支持快速数据过滤，我们可以预定义过滤条件，并基于此进行预聚合：
| ad_id | click_minute | country | count |
|-------|--------------|---------|-------|
| ad001 | 202101010001 | USA     | 100   |
| ad001 | 202101010001 | GPB     | 200   |
| ad001 | 202101010001 | others  | 3000  |
| ad002 | 202101010001 | USA     | 10    |
| ad002 | 202101010001 | GPB     | 25    |
| ad002 | 202101010001 | others  | 12    |

这种技术称为**星型模式（star schema）**，广泛应用于数据仓库中。
这些过滤字段称为**维度（dimensions）**。

这种方法有以下优点：
 - 易于理解和构建
 - 可以复用现有的聚合服务，在星型模式中创建更多维度。
 - 基于过滤条件访问数据速度很快，因为结果是预先计算好的

这种方法的一个局限是它会产生多得多的存储桶（bucket）和记录，尤其是在我们有大量过滤条件时。

---

## 第 3 步：深入设计
让我们更深入地探讨一些更有意思的主题。

### **流处理与批处理（Streaming vs. Batching）**
我们提出的高层架构是一类流处理系统。
下面是三种系统之间的比较：
|                         | 服务（在线系统）             | 批处理系统（离线系统）                                  | 流处理系统（近实时系统）                       |
|-------------------------|-------------------------------|--------------------------------------------------------|----------------------------------------------|
| 响应性                  | 快速响应客户端                | 无需响应客户端                                          | 无需响应客户端                                |
| 输入                    | 用户请求                      | 有界输入，规模有限。数据量大                            | 输入无边界（无限流）                          |
| 输出                    | 对客户端的响应                | 物化视图、聚合指标等                                    | 物化视图、聚合指标等                          |
| 性能度量                | 可用性、延迟                  | 吞吐量                                                  | 吞吐量、延迟                                  |
| 示例                    | 在线购物                      | MapReduce                                              | Flink [13]                                    |

在我们的设计中，混合使用了批处理和流处理。

我们使用流处理来处理到达的数据，并近实时地生成聚合结果。
另一方面，我们使用批处理进行历史数据备份。

一个系统同时包含两条处理路径——批处理和流处理——这种架构称为 Lambda。
它的一个缺点是你有两条处理路径，需要维护两套不同的代码库。

Kappa 是另一种架构，它将批处理和流处理合并到同一条处理路径中。
其核心思想是使用单一的流处理引擎。

Lambda 架构：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/lambda-architecture.png" alt="Lambda 架构" width="500" />
</div>

Kappa 架构：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/kappa-architecture.png" alt="Kappa 架构" width="500" />
</div>

我们的高层设计采用 Kappa 架构，因为历史数据的重新处理也会经过聚合服务。

每当我们需要重新计算聚合数据时（例如由于聚合逻辑中的重大 bug），我们可以从存储的原始数据重新计算聚合。
 - 重新计算服务从原始存储中检索数据。这是一个批处理作业。
 - 检索到的数据被发送到一个专用的聚合服务，这样实时处理的聚合服务就不会受到影响。
 - 聚合结果被发送到第二个消息队列，之后我们更新聚合数据库中的结果。

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/recalculation-example.png" alt="重新计算示例" width="500" />
</div>

### **时间（Time）**
我们需要一个时间戳来执行聚合。它可以在两个地方生成：
 - 事件时间（event time）——广告点击发生时
 - 处理时间（processing time）——服务器处理事件时的系统时间

由于使用了异步处理（消息队列）和网络延迟，事件时间与处理时间之间可能存在显著差异。
 - 如果我们使用处理时间，聚合结果可能不准确
 - 如果我们使用事件时间，我们就必须处理延迟到达的事件

没有完美的解决方案，我们需要权衡取舍：
|                 | 优点                                  | 缺点                                                                                 |
|-----------------|---------------------------------------|--------------------------------------------------------------------------------------|
| 事件时间        | 聚合结果更准确                        | 客户端可能时间不正确，或时间戳可能由恶意用户生成                                      |
| 处理时间        | 服务器时间戳更可靠                    | 如果事件迟到，时间戳就不准确                                                          |

由于数据准确性很重要，我们将使用事件时间进行聚合。

为了缓解延迟事件的问题，可以利用一种称为「水位线（watermark）」的技术。

在下面的示例中，事件 2 错过了它本应被聚合的窗口：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/watermark-technique.png" alt="水位线技术" width="500" />
</div>

然而，如果我们有意识地延长聚合窗口，就可以降低事件被错过的可能性。
窗口延长的部分称为「水位线（watermark）」：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/watermark-2.png" alt="水位线 2" width="500" />
</div>

 - 短水位线会增加事件被错过的可能性，但会降低延迟
 - 长水位线会降低事件被错过的可能性，但会增加延迟

无论水位线多大，事件被错过的可能性始终存在。但为这种低概率事件做优化并没有意义。

我们可以改为通过日终对账（end-of-day reconciliation）来解决这类不一致问题。

### **聚合窗口（Aggregation window）**
窗口函数有四种类型：
 - 滚动窗口（tumbling，固定窗口）
 - 跳跃窗口（hopping window）
 - 滑动窗口（sliding window）
 - 会话窗口（session window）

在我们的设计中，我们对广告点击聚合使用滚动窗口：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/tumbling-window.png" alt="滚动窗口" width="500" />
</div>

以及对 M 分钟内点击量最高的前 N 个广告聚合使用滑动窗口：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/sliding-window.png" alt="滑动窗口" width="500" />
</div>

### **投递保证（Delivery guarantees）**
由于我们聚合的数据将用于计费，数据准确性是首要任务。

因此，我们需要讨论：
 - 如何避免处理重复事件
 - 如何确保所有事件都被处理

我们可以使用三种投递保证——至多一次（at-most-once）、至少一次（at-least-once）和恰好一次（exactly once）。

在大多数情况下，当可以接受少量重复时，至少一次就足够了。
但我们的系统并非如此，因为即使是很小百分比的差异也可能导致数百万美元的出入。
因此，我们需要使用恰好一次（exactly-once）投递语义。

### **数据去重（Data deduplication）**
最常见的数据质量问题之一就是重复数据。

它可能来自多种来源：
 - 客户端——客户端可能会多次重发同一事件。带有恶意意图发送的重复事件，最好由风险引擎（risk engine）处理。
 - 服务器宕机——聚合服务节点在聚合过程中宕机，上游服务尚未收到确认（acknowledgment），因此事件被重发。

下面是一个由于最后一跳（last hop）未确认事件而导致数据重复的示例：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/data-duplication-example.png" alt="数据重复示例" width="500" />
</div>

在本例中，偏移量（offset）100 会被处理并多次发送到下游。

一种缓解方法是将最后处理过的偏移量存储在 HDFS/S3 中，但这会导致结果可能永远无法到达下游：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/data-duplication-example-2.png" alt="数据重复示例 2" width="500" />
</div>

最后，我们可以在与下游交互时原子地存储偏移量。要实现这一点，我们需要实现一个分布式事务：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/data-duplication-example-3.png" alt="数据重复示例 3" width="500" />
</div>

**作者侧注**：或者，如果下游系统以幂等（idempotent）方式处理聚合结果，就不需要分布式事务了。

### **系统扩展（Scale the system）**
让我们讨论如何随着系统的增长对其进行扩展。

我们有三个独立组件——消息队列、聚合服务和数据库。
由于它们是解耦的，我们可以独立地扩展它们。

如何扩展消息队列：
 - 我们不对生产者数量设限，因此生产者可以很容易地扩展
 - 消费者可以通过将它们分配到消费者组并增加消费者数量来扩展。
 - 为此，我们还需要确保预先创建了足够的分区
 - 此外，当有成千上万个消费者时，消费者再平衡（rebalancing）可能需要一段时间，因此建议在非高峰时段进行
 - 我们还可以考虑按地理位置对 topic 进行分区，例如 `topic_na`、`topic_eu` 等

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/scale-consumers.png" alt="扩展消费者" width="500" />
</div>

如何扩展聚合服务：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/aggregation-service-scaling.png" alt="聚合服务扩展" width="500" />
</div>

 - map-reduce 节点可以通过添加更多节点轻松扩展
 - 聚合服务的吞吐量可以通过利用多线程（multi-threading）来扩展
 - 或者，我们可以利用 Apache YARN 等资源提供者来利用多进程（multi-processing）
 - 方案 1 更简单，但方案 2 在实践中应用更广泛，因为它的可扩展性更好
 - 下面是多线程的示例：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/multi-threading-example.png" alt="多线程示例" width="500" />
</div>

如何扩展数据库：
 - 如果我们使用 Cassandra，它利用一致性哈希（consistent hashing）原生支持水平扩展
 - 如果向集群添加新节点，数据会自动在所有（虚拟）节点之间重新平衡
 - 通过这种方式，无需手动（重新）分片

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/cassandra-scalability.png" alt="Cassandra 可扩展性" width="500" />
</div>

另一个需要考虑的可扩展性问题是热点（hotspot）问题——如果某个广告比其他广告更受欢迎、获得更多关注怎么办？

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/hotspot-issue.png" alt="热点问题" width="500" />
</div>

 - 在上面的示例中，聚合服务节点可以通过资源管理器申请额外资源
 - 资源管理器分配更多资源，这样原节点就不会过载
 - 原节点将事件分成 3 组，每个聚合节点处理 100 个事件
 - 结果写回原聚合节点

处理热点问题的其他更复杂的方法：
 - 全局-本地聚合（Global-Local Aggregation）
 - 拆分去重聚合（Split Distinct Aggregation）

### **容错（Fault Tolerance）**
在聚合节点内部，我们在内存中处理数据。如果节点宕机，已处理的数据就会丢失。

一旦另一个节点接替工作，我们可以利用 Kafka 中的消费者偏移量从中断处继续处理。
然而，我们还需要维护额外的中间状态，因为我们在聚合 M 分钟内的前 N 个广告。

我们可以在特定分钟点对正在进行的聚合做快照（snapshot）：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/fault-tolerance-example.png" alt="容错示例" width="500" />
</div>

如果节点宕机，新节点可以读取最新已提交的消费者偏移量以及最新的快照来继续作业：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/fault-tolerance-recovery-example.png" alt="容错恢复示例" width="500" />
</div>

### **数据监控与正确性**
由于我们聚合的数据至关重要（用于计费），因此建立严格的监控以确保正确性非常重要。

我们可能需要监控的一些指标：
 - **延迟（Latency）**：可以跟踪不同事件的时间戳，以了解系统的端到端延迟
 - **消息队列大小**：如果队列大小突然增加，我们需要添加更多聚合节点。由于 Kafka 是通过分布式提交日志（commit log）实现的，我们需要改为跟踪 records-lag 指标。
 - **聚合节点上的系统资源**：CPU、磁盘、JVM 等。

我们还需要实现一个对账流程（reconciliation flow），它是一个批处理作业，在日终运行。
它从原始数据计算聚合结果，并将其与聚合数据库中存储的实际数据进行比较：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/reconciliation-flow.png" alt="对账流程" width="500" />
</div>

### **备选设计（Alternative design）**
在通用的系统设计面试中，并不要求你了解大数据处理所用专用软件的内部实现。

解释思考过程并讨论权衡取舍，比了解具体工具更重要，这也是本章介绍通用解决方案的原因。

另一种利用现成工具的设计是，将广告点击数据存储在 Hive 中，并在其上构建一层 ElasticSearch，以实现更快的查询。

聚合通常在 OLAP 数据库中完成，如 ClickHouse 或 Druid。

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/alternative-design.png" alt="备选设计" width="500" />
</div>

---

## 第 4 步：总结
我们涵盖的内容：
 - 数据模型与 API 设计
 - 使用 MapReduce 聚合广告点击事件
 - 扩展消息队列、聚合服务和数据库
 - 缓解热点问题
 - 持续监控系统
 - 使用对账确保正确性
 - 容错

广告点击事件聚合是一个典型的大数据处理系统。

如果你具备相关技术的先验知识，会更容易理解和设计它：
 - Apache Kafka
 - Apache Spark
 - Apache Flink
