# 第 12 章：设计聊天系统

## 引言
**聊天系统（chat system）** 支持用户之间的实时消息收发。本章聚焦于设计一个聊天应用，它包含：
- **单聊（One-on-One Chat）**
- **群聊（Group Chat，最多 100 人）**
- **在线状态指示（Online Presence Indicators）**
- **多设备支持**
- **推送通知（Push Notifications）**

该系统目标为**5000 万日活跃用户（DAU）**，并永久存储聊天历史。

---

## 第 1 步：理解问题

### 需求
1. **功能：**
   - 单聊和群聊（最多 100 名成员）。
   - 基于文本的消息（最多 100,000 个字符）。
   - 在线/离线状态指示。
   - 支持多设备。
   - 推送通知。
2. **规模：** 按 5000 万 DAU 进行设计。
3. **存储：** 永久保存聊天历史。

---

## 第 2 步：高层设计

### 通信协议
1. **发送方侧：** 使用 HTTP 发送消息，并利用持久连接以提升效率。

      <div style="margin-left:2rem">
      <img src="../../12.%20Chat%20System/images/basic-design.png" alt="基础设计" width="500">    
      <div>

2. **接收方侧：**
   - **轮询（Polling）：**
      - 客户端周期性地询问服务器是否有可用消息。
      - 由于频繁、冗余的请求，效率较低。

         <img src="../../12.%20Chat%20System/images/polling.png" alt="轮询" width="400">    

   - **长轮询（Long Polling）：** 
      - 保持连接打开，直到消息到达。 
      - 对不活跃用户效率较低。

         <img src="../../12.%20Chat%20System/images/long-polling.png" alt="长轮询" width="400">

   - **WebSocket：** 
      - 一种双向、持久的连接，用于实时通信，发送和接收消息均选用它。
      - 使用 WebSocket（ws）协议发送和接收消息。

         <img src="../../12.%20Chat%20System/images/websocket.png" alt="WebSocket"  width="400">    
   
---

### 组件

<div style="margin-left:5rem">
   <img src="../../12.%20Chat%20System/images/high-level-stateless-arch.png" alt="高层架构（无状态）" height="350">    
   <img src="../../12.%20Chat%20System/images/high-level-statefull-arch.png" alt="高层架构（有状态）" height="350" width="550">
</div>

1. **无状态服务（Stateless Services）：**
   - 处理注册、登录和用户资料管理。
   - 与服务发现（service discovery）集成，以推荐最合适的聊天服务器。
2. **有状态服务（Stateful Services）：**
   - 聊天服务器维护持久的 WebSocket 连接。
   - 负责消息投递与同步。
3. **第三方集成：**
   - 推送通知服务就新消息通知用户。
   - 通知的实现请参考「通知系统」一章。


---
### 设计

客户端与某台聊天服务器维持一条持久的 WebSocket 连接，以进行实时消息收发。

<div style="margin-left:3rem">
      <img src="../../12.%20Chat%20System/images/high-level-design.png" alt="高层设计" width="450"> 
</div>

- 聊天服务器负责消息的发送/接收。
- 在线状态服务器（Presence servers）管理在线/离线状态。
- API 服务器处理一切事务，包括用户登录、注册、修改资料等。
- 通知服务器发送推送通知。
- 最后，键值存储（key-value store）用于存储聊天历史。聊天历史数据的数据库选用键值存储，原因如下：
   - 它易于水平扩展。
   - 键值存储提供极低的数据访问延迟。
   - 关系型数据库不能很好地处理数据的长尾分布。当索引变得很大时，随机访问代价高昂。
   - 键值存储已被其他经过验证、可靠的聊天应用采用，例如 Facebook Messenger 和 Discord。


以下是单聊和群聊的数据模型。
   - 主键是消息 ID，它有助于确定消息的顺序。
   - 对于群聊，复合主键是 `(channel_id, message_id)`。 
      - ID 可以使用全局 64 位序列号生成器生成，例如雪花算法（Snowflake）。
      - 更好的做法是使用本地序列号生成器。「本地」意味着 ID 仅在一个群组内唯一。
      - 本地 ID 之所以可行，是因为在单聊频道或群聊频道内维护消息顺序就足够了。 
      
      <img src="../../12.%20Chat%20System/images/one-to-one-chat.png" alt="单聊设计" width="300">   
      <img src="../../12.%20Chat%20System/images/group-chat.png" alt="群聊设计" width="300">   


## 第 3 步：深入设计

### 服务发现

<div style="margin-left:3rem">
   <img src="../../12.%20Chat%20System/images/zookeeper.png" alt="ZooKeeper" width="400">   
</div>

- 服务发现的主要作用是，根据地理位置、服务器容量等标准，为客户端推荐最合适的聊天服务器。 
- 使用 **Apache ZooKeeper**，根据地理位置和服务器容量等标准分配聊天服务器。
- 确保高效的负载分配，并将延迟降至最低。


### 消息流程
#### 单聊


1. 用户 A 向聊天服务器 1 发送一条消息。
2. 聊天服务器 1 分配一个唯一的消息 ID，并将该消息存入键值存储。
3. 如果用户 B 在线，则将消息转发到聊天服务器 2（维持一条持久的 WebSocket 连接）。
4. 如果用户 B 离线，则发送一条推送通知。



#### 群聊

<div style="margin-left:3rem">
   <img src="../../12.%20Chat%20System/images/group-chat-flow.png" alt="群聊流程" width="400">  
</div>

- 消息被复制到群组中每个接收者各自的收件箱（inbox）。
- 这简化了同步，但对于更大的群组来说代价高昂。
- 在接收方一侧，一个接收者可能从多个用户那里接收消息。每个接收者都有一个收件箱（消息同步队列），其中包含来自不同发送者的消息。

---

#### 消息同步

许多用户拥有多台设备。我们需要在这些设备之间同步消息。
每台设备维护一个名为 `cur_max_message_id` 的变量，用于跟踪该设备上最新的消息 ID。满足以下两个条件的消息被视为新消息：

<div style="margin-left:3rem">
   <img src="../../12.%20Chat%20System/images/message-synchronization.png" alt="消息同步"  width="400">  
</div>

- 接收者 ID 等于当前登录用户的 ID。
- 键值存储中的消息 ID 大于 `cur_max_message_id`

---

### 在线状态
1. **心跳机制（Heartbeat Mechanism）：** 
   <div style="margin-left:3rem">
      <img src="../../12.%20Chat%20System/images/heartbeat-mechanism.png" alt="心跳机制" width="400"> 
   </div>
   
   - 客户端定期向在线状态服务器发送心跳，以表明自己在线。 
   - 如果在一个阈值内（例如 x = 30）未收到心跳，则将该用户标记为离线。

     

2. **扇出模型（Fanout Model）：** 

   <div style="margin-left:3rem">
      <img src="../../12.%20Chat%20System/images/fanout-presence.png" alt="在线状态扇出" width="400"> 
   </div>

   - 在线状态更新通过发布-订阅模型推送给好友，其中每一对好友都维护一个频道（channel）。
   - 当用户 A 的在线状态发生变化时，它将该事件发布到三个频道：频道 A-B、A-C 和 A-D。 
   - 这三个频道分别由用户 B、C 和 D 订阅，从而获得在线状态更新。
   - 上述设计对于小型用户群体是有效的。


---

## 附加考虑
### 可扩展性
- **水平扩展：** 随着用户数量增长增加服务器。
- **负载均衡：** 将流量均匀地分发到各服务器。
- **缓存：** 降低数据库负载并改善延迟。

### 错误处理
- **重试机制：** 通过重试和排队来处理消息投递失败。
- **服务器故障：** 发生故障时，使用服务发现分配新的服务器。

### 未来扩展
1. **媒体支持：** 增加对照片和视频的处理，包括压缩和云存储。
2. **端到端加密：** 确保消息隐私。
3. **客户端缓存：** 减少数据传输以获得更好的性能。
4. **改善加载时间：** 使用地理上分布式的缓存网络。

