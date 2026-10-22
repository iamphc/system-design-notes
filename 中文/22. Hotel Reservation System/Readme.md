# 第 22 章：酒店预订系统

## 引言
在本章中，我们要设计一个**酒店预订系统**，类似于万豪国际（Marriott International）。

它同样适用于其他类型的系统 —— Airbnb、机票预订、电影票预订。

---

## 第 1 步：理解问题并确定设计范围
在深入设计系统之前，我们应该先向面试官提问以澄清范围：
 - C：系统的规模有多大？
 - I：我们正在为一家拥有 5000 家酒店、100 万间客房的连锁酒店建一个网站。
 - C：客户是在预订时付款，还是入住酒店时付款？
 - I：他们在预订时全额付款。
 - C：客户只能通过网站预订客房吗？我们是否必须支持其他预订渠道，比如电话？
 - I：他们只能通过网站或 App 预订。
 - C：客户可以取消预订吗？
 - I：可以。
 - C：还有其他需要考虑的事情吗？
 - I：有，我们允许超订（overbooking）10%。酒店会卖出比实际数量更多的房间。酒店这么做是预料到客户会取消预订。
 - C：由于时间不多，我们将聚焦于 —— 展示酒店相关页面、酒店房间详情页、预订房间、管理后台面板、支持超订。
 - I：听起来不错。
 - I：还有一件事 —— 酒店价格一直在变动。假设一间客房的价格每天都会变。
 - C：好的。

### **非功能性需求**
 - 支持高并发 —— 在旺季，可能有很多客户试图预订同一家酒店。
 - 中等延迟 —— 理想情况下用户预订时延迟要低，但如果系统处理需要几秒钟也是可以接受的。

### **粗略估算（Back-of-the-envelope estimation）**
 - 共 5000 家酒店、100 万间客房。
 - 假设 70% 的客房被占用，平均入住时长为 3 天。
 - 预估每日预订量 —— 100 万 × 0.7 / 3 = 每天约 24 万次预订。
 - 每秒预订数 —— 24 万 / 一天中的 10^5 秒 = 约 3。平均预订 TPS 很低。

让我们估算一下 QPS。如果假设到达预订页面需要经过三个步骤，且每个页面的转化率为 10%，
我们可以估算出：如果有 3 笔预订，那么预订页面一定被浏览了 30 次，酒店房间详情页被浏览了 300 次。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/qps-estimation.png" alt="QPS 估算" width="500" />
</div>

---

## 第 2 步：提出高层级设计并获得认可
我们将探讨 —— API 设计、数据模型、高层级设计。

### **API 设计**
本 API 设计聚焦于为支撑酒店预订系统所必需的核心端点（采用 RESTful 风格）。

一个完整的系统需要更庞大的 API，支持基于大量条件搜索房间，但本节不聚焦于此。
原因是它们在技术上没有挑战性，因此不在范围内。

**酒店相关 API**
 - `GET /v1/hotels/{id}` - 获取某家酒店的详细信息
 - `POST /v1/hotels` - 新增一家酒店。仅运维可用
 - `PUT /v1/hotels/{id}` - 更新酒店信息。仅运维可用
 - `DELETE /v1/hotels/{id}` - 删除一家酒店。该 API 仅运维可用

**房间相关 API**
 - `GET /v1/hotels/{id}/rooms/{id}` - 获取某间客房的详细信息
 - `POST /v1/hotels/{id}/rooms` - 新增一间客房。仅运维可用
 - `PUT /v1/hotels/{id}/rooms/{id}` - 更新客房信息。仅运维可用
 - `DELETE /v1/hotels/{id}/rooms/{id}` - 删除一间客房。仅运维可用

**预订相关 API**
 - `GET /v1/reservations` - 获取当前用户的预订历史
 - `GET /v1/reservations/{id}` - 获取某笔预订的详细信息
 - `POST /v1/reservations` - 发起一笔新预订
 - `DELETE /v1/reservations/{id}` - 取消一笔预订

下面是一个发起预订的请求示例：

```
{
  "startDate":"2021-04-28",
  "endDate":"2021-04-30",
  "hotelID":"245",
  "roomID":"U12354673389",
  "reservationID":"13422445"
}
```

注意，`reservationID` 是一个幂等键（idempotency key），用于避免重复预订。细节在[并发问题一节](#concurrency-issues)中说明。

### **数据模型**
在选择使用哪种数据库之前，让我们先考虑一下我们的访问模式。

我们需要支持以下查询：
 - 查看某家酒店的详细信息
 - 给定一个日期范围，找出可预订的房型
 - 记录一笔预订
 - 查询某笔预订或过去的预订历史

从我们的估算可知，系统规模并不大，但我们需要为流量激增做好准备。

基于这些认识，我们将选择关系型数据库，原因是：
 - 关系型数据库在「读多写少」的系统中表现良好。
 - NoSQL 数据库通常为写而优化，但我们知道写入不会很多，因为访问网站的用户中只有一小部分会下单预订。
 - 关系型数据库提供 ACID 保证。这对这类系统很重要，因为没有它们，我们将无法防止负余额、重复扣款等问题。
 - 关系型数据库可以很容易地对数据建模，因为结构非常清晰。

下面是我们的 schema 设计：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/schema-design.png" alt="Schema 设计" width="500" />
</div>

大部分字段都不言自明。唯一值得一提的是 `status` 字段，它表示某间客房的状态机：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/status-state-machine.png" alt="状态状态机" width="500" />
</div>

这个数据模型对 Airbnb 这样的系统很适用，但不适用于酒店 —— 在酒店里，用户预订的不是某一间特定客房，而是一个房型。
他们预订的是一种房型，具体房号是在预订时才选定的。

这个不足将在[改进的数据模型](#improved-data-model)一节中解决。

### **高层级设计**
本设计我们选择了微服务架构。近年来它非常流行：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/high-level-design.png" alt="高层级设计" width="500" />
</div>

 - **用户（Users）**：在手机或电脑上预订酒店客房。
 - **管理员（Admin）**：执行管理职能，如退款/取消支付等。
 - **CDN**：缓存静态资源，如 JS 包、图片、视频等。
 - **公共 API 网关（Public API Gateway）**：全托管服务，支持限流、认证等。
 - **内部 API（Internal APIs）**：仅对授权人员可见。通常由 VPN 保护。
 - **酒店服务（Hotel service）**：提供酒店和客房的详细信息。酒店和客房数据是静态的，可以激进地缓存。
 - **价格服务（Rate service）**：提供未来不同日期的房价。这个领域一个有趣的点是：价格取决于某一天酒店的满房程度。
 - **预订服务（Reservation service）**：接收预订请求并预订酒店客房。同时随着预订的下单/取消来跟踪客房库存。
 - **支付服务（Payment service）**：处理支付，并在成功后更新预订状态。
 - **酒店管理服务（Hotel management service）**：仅授权人员可用。允许某些管理职能，用于管理和查看预订、酒店等。

服务间通信可以通过 RPC 框架来完成，例如 gRPC。

---

## 第 3 步：深入设计
让我们更深入地探讨：
 - 改进的数据模型
 - 并发问题
 - 可扩展性
 - 解决微服务中的数据不一致

### **改进的数据模型**
如前一节所述，我们需要修改 API 和 schema，以支持「预订一个房型」而非「预订某一间特定客房」。

对于预订 API，我们不再预订 `roomID`，而是预订 `roomTypeID`：

```
POST /v1/reservations
{
  "startDate":"2021-04-28",
  "endDate":"2021-04-30",
  "hotelID":"245",
  "roomTypeID":"12354673389",
  "roomCount":"3",
  "reservationID":"13422445"
}
```

下面是更新后的 schema：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/updated-schema.png" alt="更新后的 schema" width="500" />
</div>

 - **room**：包含一间客房的信息。
 - **room_type_rate**：包含某房型价格的信息。
 - **reservation**：记录客人的预订数据。
 - **room_type_inventory**：存储酒店客房的库存数据。

让我们看看 `room_type_inventory` 的列，因为那张表更有意思：
 - **hotel_id**：酒店 id。
 - **room_type_id**：房型 id。
 - **date**：某一个日期。
 - **total_inventory**：客房总数减去那些被临时从库存中下架的房间。
 - **total_reserved**：在给定 (hotel_id, room_type_id, date) 下已被预订的房间总数。

这张表还有其他设计方式，但让每个 (hotel_id, room_type_id, date) 对应一行客房，可以让预订管理更轻松、查询更容易。

表中的行是通过一个每日运行的 CRON 任务预先填充的。

示例数据：
| hotel_id | room_type_id | date       | total_inventory | total_reserved |
|----------|--------------|------------|-----------------|----------------|
| 211      | 1001         | 2021-06-01 | 100             | 80             |
| 211      | 1001         | 2021-06-02 | 100             | 82             |
| 211      | 1001         | 2021-06-03 | 100             | 86             |
| 211      | 1001         | ...        | ...             |                |
| 211      | 1001         | 2023-05-31 | 100             | 0              |
| 211      | 1002         | 2021-06-01 | 200             | 16             |
| 2210     | 101          | 2021-06-01 | 30              | 23             |
| 2210     | 101          | 2021-06-02 | 30              | 25             |

下面是一个检查某房型是否可预订的示例 SQL 查询：

```
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}
```

如何用这些数据检查指定数量的房间是否可订（注意我们支持超订）：

```
if (total_reserved + ${numberOfRoomsToReserve}) <= 110% * total_inventory
```

现在让我们对存储量做一些估算。
 - 我们有 5000 家酒店。
 - 每家酒店有 20 种房型。
 - 5000 × 20 × 2（年）× 365（天）= 7300 万行。

7300 万行数据并不算多，单台数据库服务器就能处理。
不过，设置读复制（可能跨不同可用区）以实现高可用性是有意义的。

追问 —— 如果预订数据大到单台数据库都放不下，你会怎么做？
 - 只存储当前和未来的预订数据。预订历史可以迁移到冷存储。
 - 数据库分片（sharding）—— 我们可以按 `hash(hotel_id) % servers_cnt` 分片，因为我们的查询总是按 `hotel_id` 过滤。

### **并发问题**
另一个需要解决的重要问题是重复预订（double booking）。

有两个问题需要解决：
 - 同一个用户连续点了两次「预订」。
 - 多个用户同时试图预订同一间房。

下面是第一个问题的可视化：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/double-booking-single-user.png" alt="单用户重复预订" width="500" />
</div>

解决这个问题有两种方式：
 - 客户端处理 —— 前端可以在点击一次后禁用预订按钮。但如果用户禁用了 javascript，他们就看不到按钮变灰。
 - 幂等 API —— 给 API 加一个幂等键，使用户无论调用该端点多少次，某个动作都只会被执行一次：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/idempotency.png" alt="幂等性" width="500" />
</div>

这个流程是这样工作的：
 - 当你开始填写信息并下单时，会生成一个预订订单。该预订订单使用全局唯一标识符生成。
 - 使用上一步生成的 `reservation_id` 提交预订 1。
 - 如果第二次点击「完成预订」，发送的是同一个 `reservation_id`，后端会检测出这是一笔重复预订。
 - 通过给 `reservation_id` 列加上唯一约束来避免重复，防止在 DB 中存储多条同 id 的记录。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/unique-constraint-violation.png" alt="唯一约束冲突" width="500" />
</div>

那如果是多个用户在做同一笔预订呢？

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/double-booking-multiple-users.png" alt="多用户重复预订" width="500" />
</div>

 - 假设事务隔离级别不是可序列化（serializable）。
 - 用户 1 和用户 2 同时试图预订同一间房。
 - 事务 1 检查是否有足够房间 —— 有。
 - 事务 2 检查是否有足够房间 —— 有。
 - 事务 2 预订了该房间并更新库存。
 - 事务 1 也预订了该房间，因为它仍然看到 100 间中只订了 99 间 `total_reserved`。
 - 两个事务都成功提交了变更。

这个问题可以用某种形式的锁机制来解决：
 - 悲观锁（Pessimistic locking）
 - 乐观锁（Optimistic locking）
 - 数据库约束（Database constraints）

下面是我们用来预订房间的 SQL：

```sql
# step 1: check room inventory
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}

# For every entry returned from step 1
if((total_reserved + ${numberOfRoomsToReserve}) > 110% * total_inventory) {
  Rollback
}

# step 2: reserve rooms
UPDATE room_type_inventory
SET total_reserved = total_reserved + ${numberOfRoomsToReserve}
WHERE room_type_id = ${roomTypeId}
AND date between ${startDate} and ${endDate}

Commit
```

#### 方案 1：悲观锁
悲观锁通过在记录被更新期间对其加锁，来防止并发更新。

在 MySQL 中，可以使用 `SELECT... FOR UPDATE` 查询来做到这一点，它会把查询选中的行锁住，直到事务提交。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/pessimistic-locking.png" alt="悲观锁" width="500" />
</div>

优点：
 - 防止应用程序去更新正在被更改的数据。
 - 易于实现，通过把更新序列化来避免冲突。在数据竞争激烈时很有用。

缺点：
 - 当锁定多个资源时可能发生死锁。
 - 这种方式不可扩展 —— 如果一个事务被锁太久，会影响所有其他试图访问该资源的事务。
 - 当查询选中大量资源且事务是长事务时，影响会很严重。

由于可扩展性问题，作者不推荐这种方式。

#### 方案 2：乐观锁
乐观锁允许多个用户同时尝试更新同一条记录。

常见的实现方式有两种 —— 版本号和时间戳。推荐使用版本号，因为服务器时钟可能不准确。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/optimistic-locking.png" alt="乐观锁" width="500" />
</div>

 - 在数据库表中新增一个 `version` 列。
 - 用户修改某一行之前，先读取版本号。
 - 用户更新该行时，把版本号加 1 并写回数据库。
 - 如果新版本号没有大于旧版本号，数据库校验会阻止这次写入。

乐观锁通常比悲观锁更快，因为我们没有锁定数据库。
不过，当并发很高时它的性能会下降，因为那会导致大量回滚。

优点：
 - 它防止应用程序编辑陈旧数据。
 - 我们不需要在数据库中获取锁。
 - 当数据竞争较低（即很少发生更新冲突）时，是首选方案。

缺点：
 - 当数据竞争较高时性能较差。

乐观锁对我们的系统是个不错的选择，因为预订 QPS 并不是非常高。

#### 方案 3：数据库约束
这种方式与乐观锁非常相似，但防护栏是用数据库约束来实现的：

```
CONSTRAINT `check_room_count` CHECK((`total_inventory - total_reserved` >= 0))
```

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/database-constraint.png" alt="数据库约束" width="500" />
</div>

优点：
 - 易于实现。
 - 当数据竞争较小时效果良好。

缺点：
 - 与乐观锁类似，当数据竞争较高时性能较差。
 - 数据库约束不像应用代码那样容易做版本控制。
 - 并非所有数据库都支持约束。

由于易于实现，这也是酒店预订系统的一个不错选择。

### **可扩展性**
通常，酒店预订系统的负载并不高。

然而，面试官可能会问你：如果这个系统被一个更大、更热门的旅行网站（如 booking.com）采用，你会怎么应对？
那种情况下，QPS 可能是现在的 1000 倍。

遇到这种情况时，重要的是搞清楚我们的瓶颈在哪里。所有服务都是无状态的，因此可以通过复制轻松扩展。

而数据库是有状态的，如何扩展它并不那么显而易见。

一种扩展方式是实现数据库分片 —— 我们可以把数据拆分到多个数据库中，每个数据库各存一部分数据。

我们可以基于 `hotel_id` 分片，因为所有查询都是按它过滤的。
假设 QPS 为 30000，把数据库分成 16 个分片后，每个分片处理 1875 QPS，在单个 MySQL 集群的负载能力之内。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/database-sharding.png" alt="数据库分片" width="500" />
</div>

我们还可以通过 Redis 对客房库存和预订做缓存。我们可以设置 TTL，让过期日期的旧数据自然失效。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/inventory-cache.png" alt="库存缓存" width="500" />
</div>

我们存储库存的方式是基于 `hotel_id`、`room_type_id` 和 `date`：

```
key: hotelID_roomTypeID_{date}
value: the number of available rooms for the given hotel ID, room type ID and date.
```

数据一致性是异步发生的，通过 CDC 流式机制来管理 —— 读取数据库变更并应用到另一个独立系统。
Debezium 是把数据库变更同步到 Redis 的一个流行选择。

使用这种机制，缓存和数据库在一段时间内可能不一致。
在我们的场景下这没问题，因为数据库会阻止我们做出无效的预订。

这会在 UI 上造成一些问题：用户得刷新页面才能看到「房间已经订完了」，
但即便没有这个问题，这种情况也可能发生 —— 比如一个人在预订前犹豫了很久。

缓存的优点：
 - 降低数据库负载。
 - 高性能，因为 Redis 在内存中管理数据。

缓存的缺点：
 - 维持缓存与 DB 之间的数据一致性很难。我们需要考虑这种不一致会如何影响用户体验。

### **服务间的数据一致性**
单体应用让我们可以使用一个共享的关系型数据库来保证数据一致性。

在我们的微服务设计中，我们选择了一种混合方式：有些服务是独立的，
但预订和库存 API 由同一个服务处理。

这样做是因为我们想利用关系型数据库的 ACID 保证来确保一致性。

然而，面试官可能会质疑这种方式，因为它不是一个纯粹的微服务架构 —— 纯粹的架构里每个服务都有自己专用的数据库：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/microservices-vs-monolith.png" alt="微服务 vs 单体" width="500" />
</div>

这可能导致一致性问题。在单体服务器中，我们可以利用关系型 DB 的事务能力来实现原子操作：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/atomicity-monolith.png" alt="单体的原子性" width="500" />
</div>

然而，当操作跨多个服务时，要保证这种原子性就更具挑战性了：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/microservice-non-atomic-operation.png" alt="微服务中的非原子操作" width="500" />
</div>

有一些著名的技术可以处理这类数据不一致：
 - **两阶段提交（Two-phase commit）**：一种数据库协议，保证跨多个节点的事务原子提交。
   但它性能不佳，因为只要有一个节点延迟，就会导致所有节点阻塞该操作。
 - **Saga**：一串本地事务，如果工作流中的任何一步失败，就触发补偿事务。这是一种最终一致性的方式。

值得注意的是，解决跨微服务的数据不一致是一个富有挑战性的问题，它会抬高系统复杂度。
考虑到我们更务实的做法 —— 把相互依赖的操作封装在同一个关系型数据库内 —— 值得权衡这样做的成本是否划算。

---

## 第 4 步：收尾
我们展示了一个酒店预订系统的设计。

这些是我们走过的步骤：
 - 收集需求并做粗略估算，以理解系统规模。
 - 在高层级设计中展示了 API 设计、数据模型和系统架构。
 - 在深入设计中，我们随着需求变化探讨了备选的数据库 schema 设计。
 - 我们讨论了竞态条件并提出了解决方案 —— 悲观锁/乐观锁、数据库约束。
 - 通过数据库分片和缓存来扩展系统的方式。
 - 最后，我们讨论了如何处理多个微服务之间的数据一致性问题。
