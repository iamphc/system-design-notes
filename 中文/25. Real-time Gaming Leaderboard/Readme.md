# 第 25 章：实时游戏排行榜

## 引言

我们要为一款在线手机游戏设计一个**排行榜（leaderboard）**：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/leaderboard.png" alt="排行榜" width="500" />
</div>

---

## 第 1 步：理解问题并确定设计范围

- 候选人（C）：排行榜的分数是如何计算的？
- 面试官（I）：用户每赢得一场比赛就得一分。
- 候选人（C）：所有玩家都包含在排行榜中吗？
- 面试官（I）：是的
- 候选人（C）：排行榜是否关联某个时间段？
- 面试官（I）：每个月都会开始新的赛季（tournament），从而开启一个新的排行榜。
- 候选人（C）：我们可以假设只关心前 10 名用户吗？
- 面试官（I）：我们想展示前 10 名用户，以及特定用户的名次。如果时间允许，我们可以讨论展示排行榜中某一特定用户周围的用户。
- 候选人（C）：一个赛季有多少用户？
- 面试官（I）：500 万 DAU 和 2500 万 MAU
- 候选人（C）：一个赛季期间平均进行多少场比赛？
- 面试官（I）：每个玩家平均每天进行 10 场比赛
- 候选人（C）：如果两个玩家分数相同，我们如何确定排名？
- 面试官（I）：在这种情况下，他们的排名相同。如果时间允许，我们可以讨论如何打破平局。
- 候选人（C）：排行榜需要实时吗？
- 面试官（I）：是的，我们希望呈现实时结果，或尽可能接近实时。呈现批量处理的历史结果是不可接受的。

### **功能需求**

- 在排行榜上展示前 10 名玩家
- 显示某用户的具体排名
- 展示给定用户前后各四名用户（加分项）

### **非功能需求**

- 分数的实时更新
- 分数更新实时反映到排行榜上
- 一般的可扩展性、可用性、可靠性

### **粗略估算（Back-of-the-envelope estimation）**

如果有 5000 万 DAU，且游戏在 24 小时内玩家分布均匀，那么平均每秒有 50 个用户。
然而，由于分布通常不均匀，我们可以估计峰值在线用户约为每秒 250 个。

用户得分的 QPS——按平均每天 10 场游戏计算，50 用户/秒 * 10 = 500 QPS。峰值 QPS = 2500。

获取排行榜前 10 名的 QPS——假设用户平均每天打开一次，QPS 为 50。

---

## 第 2 步：提出高层设计并获得认可

### **API 设计**

我们需要的第一个 API 是更新用户分数：

```
POST /v1/scores
```

这个 API 接收两个参数——`user_id` 和赢得一场游戏所得的 `points`。

这个 API 应该只允许游戏服务器访问，而不允许终端客户端访问。

下一个 API 用于获取排行榜前 10 名玩家：

```
GET /v1/scores
```

响应示例：

```
{
  "data": [
    {
      "user_id": "user_id1",
      "user_name": "alice",
      "rank": 1,
      "score": 12543
    },
    {
      "user_id": "user_id2",
      "user_name": "bob",
      "rank": 2,
      "score": 11500
    }
  ],
  ...
  "total": 10
}
```

你也可以获取特定用户的分数：

```
GET /v1/scores/{:user_id}
```

响应示例：

```
{
    "user_info": {
        "user_id": "user5",
        "score": 1000,
        "rank": 6,
    }
}
```

### **高层架构**

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/high-level-architecture.png" alt="高层架构" width="500" />
</div>

- 当玩家赢得一场游戏时，客户端向游戏服务发送请求
- 游戏服务验证该胜利是否有效，并调用排行榜服务来更新玩家的分数
- 排行榜服务在排行榜存储中更新用户的分数
- 玩家调用排行榜服务来获取排行榜数据，例如前 10 名玩家和给定玩家的排名

另一个被考虑过的设计是客户端直接在排行榜服务中更新自己的分数：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/alternative-design.png" alt="备选设计" width="500" />
</div>

这个选项不安全，因为它容易受到中间人攻击（man-in-the-middle attack）。玩家可以挂一个代理，随意修改自己的分数。

另一个注意事项是，对于游戏逻辑由服务器管理的游戏，客户端不需要显式调用服务器来记录自己的胜利。
服务器会根据游戏逻辑自动为他们记录。

另一个需要考虑的问题是，我们是否应该在游戏服务器和排行榜服务之间放一个消息队列。如果其他服务对游戏结果感兴趣，这会很有用，但面试到目前为止这并不是一个明确的需求，因此没有包含在设计中：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/message-queue-based-comm.png" alt="基于消息队列的通信" width="500" />
</div>

### **数据模型**

让我们讨论存储排行榜数据的可选方案——关系型数据库、Redis、NoSQL。

NoSQL 方案在深入设计一节中讨论。

#### 关系型数据库方案

如果规模不重要、用户也不多，关系型数据库就能很好地满足我们的需求。

我们可以从一个简单的排行榜表开始，每月一张表（作者注——这没有道理。你完全可以只加一个 `month` 列，避免每月维护新表的麻烦）：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/leaderboard-table.png" alt="排行榜表" width="500" />
</div>

还有一些额外的数据需要包含在内，但这与我们要运行的查询无关，因此省略了。

当用户赢得一分时会发生什么？

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/user-wins-point.png" alt="用户得分" width="500" />
</div>

如果用户在表中尚不存在，我们需要先插入他们：

```
INSERT INTO leaderboard (user_id, score) VALUES ('mary1934', 1);
```

在后续调用中，我们只需更新他们的分数：

```
UPDATE leaderboard set score=score + 1 where user_id='mary1934';
```

我们如何找到排行榜的顶尖玩家？

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/find-leaderboard-position.png" alt="查找排行榜位置" width="500" />
</div>

我们可以运行以下查询：

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC;
```

但这性能不佳，因为它会进行一次全表扫描来对数据库表中的所有记录排序。

我们可以通过在 `score` 上添加索引并使用 `LIMIT` 操作来优化它，以避免扫描所有内容：

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC
LIMIT 10;
```

然而，如果用户不在排行榜顶部，而你又想定位他们的排名，这种方法的可扩展性就不好。

#### Redis 方案

我们希望找到一种即使在面对数百万玩家时也能良好工作的解决方案，而不必求助于复杂的数据库查询。

Redis 是一种内存数据存储，它速度快是因为在内存中工作，并且有一种合适的数据结构来满足我们的需求——有序集合（sorted set）。

有序集合是一种类似于编程语言中集合的数据结构，它允许你按照给定条件对数据结构进行排序。
在内部，它使用哈希表（hash-map）来维护键（user_id）与值（score）之间的映射，并使用跳表（skip list）按排序顺序将分数映射到用户：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/sorted-set.png" alt="有序集合" width="500" />
</div>

跳表是如何工作的？
- 它是一种支持快速查找的链表
- 它由一个有序链表和多级索引组成

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/skip-list.png" alt="跳表" width="500" />
</div>

当数据集足够大时，这种结构使我们能够快速搜索特定值。
在下面的示例中（64 个节点），在基础链表中查找给定值需要遍历 62 个节点，而在跳表中只需遍历 11 个节点：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/skip-list-performance.png" alt="跳表性能" width="500" />
</div>

有序集合比关系型数据库性能更高，因为数据始终保持排序，代价是 O(logN) 的添加和查找操作。

相比之下，下面是在关系型数据库中查找给定用户排名所需运行的嵌套查询示例：

```
SELECT *,(SELECT COUNT(*) FROM leaderboard lb2
WHERE lb2.score >= lb1.score) RANK
FROM leaderboard lb1
WHERE lb1.user_id = {:user_id};
```

我们需要哪些操作来在 Redis 中操作排行榜？
- **ZADD**——如果用户不在集合中，则将其插入。否则，更新分数。O(logN) 时间复杂度。
- **ZINCRBY**——将用户的分数增加给定数值。如果用户不存在，分数从零开始。O(logN) 时间复杂度。
- **ZRANGE/ZREVRANGE**——获取一段用户，按其分数排序。我们可以指定顺序（ASC/DESC）、偏移量和结果数量。O(logN+M) 时间复杂度，其中 M 是结果数量。
- **ZRANK/ZREVRANK**——按 ASC/DESC 顺序获取给定用户的位置（排名）。O(logN) 时间复杂度。

当用户得分时会发生什么？

```
ZINCRBY leaderboard_feb_2021 1 'mary1934'
```

每个月都会创建一个新的排行榜，而旧的排行榜会被移到历史存储中。

当用户获取前 10 名玩家时会发生什么？

```
ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES
```

结果示例：

```
[(user2,score2),(user1,score1),(user5,score5)...]
```

那用户获取自己在排行榜中的位置呢？

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/leaderboard-position-of-user.png" alt="用户在排行榜中的位置" width="500" />
</div>

假设我们已经知道用户在排行榜中的位置，这可以很容易地通过以下查询实现：

```
ZREVRANGE leaderboard_feb_2021 357 365
```

用户的位置可以使用 `ZREVRANK <user-id>` 获取。

让我们看看存储需求是什么：
- 假设最坏情况：2500 万 MAU 全部在给定月份参与游戏
- ID 是 24 个字符的字符串，分数是 16 位整数，我们需要 26 字节 * 2500 万 = 约 650MB 的存储
- 即使由于跳表的开销而使存储成本翻倍，这也能轻松放进一个现代的 Redis 集群

另一个需要考虑的非功能需求是支持每秒 2500 次更新。这完全在单台 Redis 服务器的能力范围内。

其他注意事项：
- 我们可以启动一个 Redis 副本，以避免在 Redis 服务器崩溃时丢失数据
- 我们仍然可以利用 Redis 持久化，以在崩溃时不丢失数据
- 我们需要在 MySQL 中使用两张辅助表来获取用户详情（如用户名、显示名等），以及记录用户何时赢得游戏
- MySQL 中的第二张表可用于在基础设施故障时重建排行榜
- 作为一个小的性能优化，我们可以缓存前 10 名玩家的用户详情，因为它们会被频繁访问

---

## 第 3 步：深入设计

### **使用云服务商还是自己部署**

我们可以选择自行部署和管理服务，或者使用云服务商来替我们管理。

如果选择自己管理服务，我们将使用 Redis 存储排行榜数据，使用 MySQL 存储用户资料，如果想扩展数据库，还可能为用户资料加一层缓存：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/manage-services-ourselves.png" alt="自行管理服务" width="500" />
</div>

或者，我们可以使用云产品来替我们管理许多服务。例如，我们可以使用 AWS API Gateway 将 API 调用路由到 AWS Lambda 函数：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/api-gateway-mapping.png" alt="API 网关映射" width="500" />
</div>

AWS Lambda 使我们无需自行管理或配置服务器即可运行代码。它仅在需要时运行，并自动扩展。

用户得分示例：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/user-scoring-point-lambda.png" alt="用户在 Lambda 中得分" width="500" />
</div>

用户获取排行榜示例：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/user-retrieve-leaderboard.png" alt="用户获取排行榜" width="500" />
</div>

Lambda 是无服务器（serverless）架构的一种实现。我们不需要管理扩展和环境配置。

作者建议，如果我们是从零开始构建游戏，就采用这种方式。

### **扩展 Redis**

对于 500 万 DAU，无论是从存储还是 QPS 角度看，单台 Redis 实例都足以应付。

然而，如果设想用户群增长 10 倍到 5 亿 DAU，那么我们将需要 65GB 存储，QPS 也会达到 25 万。

这种规模将需要分片。

实现它的一种方法是对数据进行范围分区（range-partitioning）：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/range-partition.png" alt="范围分区" width="500" />
</div>

在本例中，我们将根据用户的分数进行分片。我们会在应用代码中维护 user_id 与分片之间的映射。
我们可以通过 MySQL 或另一个缓存来保存这个映射本身。

要获取前 10 名玩家，我们会查询分数最高的分片（`[900-1000]`）。

要获取用户的排名，我们需要计算该用户在其分片内的排名，再加上其他分片中分数更高的所有用户。
后者是一个 O(1) 操作，因为每个分片的总记录数可以通过 info keyspace 命令快速获取。

或者，我们可以通过 Redis Cluster 使用哈希分区（hash partitioning）。它是一个代理，基于类似于一致性哈希的分区方式将数据分布到 Redis 节点上，但又不完全相同：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/hash-partition.png" alt="哈希分区" width="500" />
</div>

在这种设置下，计算前 10 名玩家具有挑战性。我们需要获取每个分片的前 10 名玩家，并在应用中合并结果：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/top-10-players-calculation.png" alt="前 10 名玩家的计算" width="500" />
</div>

哈希分区有一些局限：
- 如果我们需要获取前 K 名用户，而 K 很大，延迟会增加，因为我们需要从所有分片中获取大量数据
- 延迟会随着分区数量的增长而增加
- 没有直接的方法来确定用户的排名

综合考虑，作者倾向于在这个问题上使用固定分区。

其他注意事项：
- 一个最佳实践是为写密集型 Redis 节点分配两倍于所需的内存，以便在需要时容纳快照
- 我们可以使用一个名为 Redis-benchmark 的工具来跟踪 Redis 设置的性能，并做出数据驱动的决策

### **备选方案：NoSQL**

另一个值得考虑的方案是使用合适的 NoSQL 数据库，它针对以下方面做了优化：
- 重写入
- 高效地按分数对同一分区内的条目排序

DynamoDB、Cassandra 或 MongoDB 都是不错的选择。

在本章中，作者决定使用 DynamoDB。它是一个全托管的 NoSQL 数据库，提供可靠的性能和出色的可扩展性。
当我们需要查询不属于主键的字段时，它还支持使用全局二级索引（global secondary index）。

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/dynamo-db.png" alt="DynamoDB" width="500" />
</div>

让我们从一张为国际象棋游戏存储排行榜的表开始：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/chess-game-leaderboard-table-1.png" alt="国际象棋游戏排行榜表 1" width="500" />
</div>

这很好用，但如果需要按分数查询任何内容，它的可扩展性就不好。因此，我们可以把分数作为排序键（sort key）：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/chess-game-leaderboard-table-2.png" alt="国际象棋游戏排行榜表 2" width="500" />
</div>

这个设计的另一个问题是我们按月进行分区。这会导致热点分区（hotspot partition），因为最新月份相比其他月份会被不均匀地访问。

我们可以使用一种称为写分片（write sharding）的技术，为每个键附加一个分区编号，通过 `user_id % num_partitions` 计算：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/chess-game-leaderboard-table-3.png" alt="国际象棋游戏排行榜表 3" width="500" />
</div>

一个需要考虑的重要权衡是应该使用多少个分区：
- 分区越多，写入可扩展性越高
- 然而，读取可扩展性会受损，因为我们需要查询更多分区来收集聚合结果

使用这种方法需要用到我们之前看到的「分散-收集」（scatter-gather）技术，随着分区数量增加，其时间复杂度也会增长：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/scatter-gather-2.png" alt="分散-收集 2" width="500" />
</div>

要对分区数量做出合理的评估，我们需要做一些基准测试。

这种 NoSQL 方法还有一个主要缺点——很难计算用户的具体排名。

如果我们的规模大到需要分片，那么也许可以告诉用户他们的分数处于哪个「百分位（percentile）」。

可以定期运行一个定时任务（cron job）来分析分数分布，据此确定用户的百分位，例如：

```
10th percentile = score < 100
20th percentile = score < 500
...
90th percentile = score < 6500
```

---

## 第 4 步：总结

如果时间允许，还可以讨论的其他内容：
- **更快的检索**——我们可以通过 Redis 哈希缓存用户对象，映射关系为 `user_id -> user object`。相比查询数据库，这样检索更快。
- **打破平局**——当两个玩家分数相同时，我们可以根据他们最近一场游戏的时间来排序，从而打破平局。
- **系统故障恢复**——在大规模 Redis 中断的情况下，我们可以通过遍历 MySQL 的 WAL 条目，用一个临时脚本（ad-hoc script）重建排行榜
