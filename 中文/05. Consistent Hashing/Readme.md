# 第 5 章：设计一致性哈希

## 引言
本章探讨一致性哈希（consistent hashing），这是一种通过在服务器之间高效分发请求与数据来实现水平扩展的关键技术。它能在增加或移除服务器时最大限度地减少数据重新分布，并确保数据均匀分布，从而缓解服务器热点等问题。

## 重新哈希问题
### 解释
在传统的哈希方法中，例如 `serverIndex = hash(key) % N`，当服务器数量发生变化时，数据重新分布就会成为问题。例如：
- 移除一台服务器会导致大部分键被重新分配，从而引发缓存未命中（cache miss）。
- 增加一台服务器会导致不必要的键重新分布。

  <img src="../../05.%20Consistent%20Hashing/images/server-hashing.png"  alt="服务器哈希" width="450">

- 当服务器池规模固定时，这种方法效果很好。然而，当新增服务器或移除现有服务器时，问题就出现了。

  <img src="../../05.%20Consistent%20Hashing/images/server-hashing-miss.png"  alt="服务器哈希未命中" width="450">

### 核心问题
当服务器数量变化时，大部分键都要重新分布，这会导致效率低下和过载。

## 一致性哈希
### 定义
一致性哈希保证在增加或移除服务器时，只有一小部分键被重新映射。这最大限度地减少了扰动，并增强了可扩展性。

### 关键概念
1. **哈希空间与环：** 哈希空间构成一个连续的环，哈希值从 `0` 分布到 `2^160-1`（例如使用 SHA-1 这样的哈希函数）。把两端连接起来就得到一个环。
    <p align="center">
    <img src="../../05.%20Consistent%20Hashing/images/hash-ring.png"  alt="哈希环" width="450">
    </p>

- 使用同一个哈希函数 f，我们根据服务器的 IP 或名称把服务器映射到环上。  

    <p align="center">
    <img src="../../05.%20Consistent%20Hashing/images/server-ring.png"  alt="服务器环" width="450">
    </p>

1. **服务器查找**
- 一个键对应的服务器，是通过在环上顺时针遍历直到找到一台服务器来确定的。

  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/server-lookup.png"  alt="服务器查找" width="450">
  </p>

2. **添加与移除服务器**
- 添加一台服务器只会重新分布邻近的键。只有一小部分键被重新分配到新服务器上。
  
  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/adding-server.png"  alt="添加服务器" width="450">
  </p>

- 移除一台服务器只会影响其范围内的键。只有被移除服务器上的键会被重新分配到顺时针方向的下一台服务器。

  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/removing-server.png"  alt="移除服务器" width="450">
  </p>

## 挑战与解决方案
### 基本方案中的两个问题
1. **分区大小不均：** 各服务器上的数据分区可能并不相等。
2. **键分布不均匀：** 某些服务器可能比其他服务器接收到多得多的键。

### 解决方案：虚拟节点
- 每台服务器由环上多个均匀分布的虚拟节点（virtual node）来表示。
- 虚拟节点改善了键的分布并均衡了负载。随着虚拟节点数量增加，键的分布变得更加均衡。这是因为虚拟节点越多，标准差越小，从而带来均衡的数据分布。
   
  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/virtual-nodes.png"   alt="虚拟节点" width="450">
  </p>

## 受影响的键
当添加或移除服务器时：
- **新增服务器：** 受影响的键是位于新服务器与其前驱之间的那些键。
  在下面的例子中，服务器 4 被添加到环上。受影响的范围从 s4（新加入的节点）开始，沿环逆时针移动，直到找到一台服务器（s3）。因此，位于 s3 与 s4 之间的键需要被重新分布到 s4。

  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/server-addition.png"   alt="服务器添加" width="450">
  </p>

- **移除服务器：** 受影响的键是位于被移除服务器与其前驱之间的那些键。在下面的例子中，当一台服务器（s1）被移除时，受影响的范围从 s1（被移除的节点）开始，沿环逆时针移动，直到找到一台服务器（s0）。因此，位于 s0 与 s1 之间的键必须被重新分布到 s2。
   
  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/server-removed.png"   alt="服务器移除" width="450">
  </p>

## 一致性哈希的好处
- **重新分布最小化：** 只有一小部分键被重新分配。
- **可扩展性：** 支持水平扩展。
- **缓解热点：** 均衡数据分布，避免服务器过载。

## 实际应用
- Amazon Dynamo DB
- Apache Cassandra
- Discord
- Akamai CDN
- Maglev 负载均衡器
