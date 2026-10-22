# 第 24 章：类 S3 对象存储（S3-like Object Storage）

## 引言

在本章中，我们将设计一个**对象存储（object storage）**服务，类似于 **Amazon S3**。

存储系统大致分为三大类：
- **块存储（Block storage）**
- **文件存储（File storage）**
- **对象存储（Object storage）**

**块存储**是一类设备，出现于 1960 年代。HDD 和 SSD 就是这样的例子。
这些设备通常物理连接到服务器，不过它们也可以通过高速网络协议以网络方式连接。
服务器可以格式化这些原始数据块，并将其用作文件系统，或者直接将控制权交给服务器。

**文件存储**构建在块存储之上。它提供了更高层次的抽象，使文件夹和文件更易于管理。

**对象存储**以牺牲性能为代价，换取高持久性、超大规模和低成本。
它面向「冷」数据，主要用于归档和备份。
它没有分层目录结构，所有数据都以对象的形式存储在扁平结构中。
与其他存储类型相比，它相对较慢。大多数云服务商都提供对象存储产品——Amazon S3、Google GCS 等。

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/storage-comparison.png" alt="存储方式比较" width="500" />
</div>

|                 | 块存储                           | 文件存储                                  | 对象存储                       |
|-----------------|----------------------------------|-----------------------------------------|--------------------------------|
| 内容可修改性    | Y                                | Y                                       | N（支持对象版本管理）          |
| 成本            | 高                               | 中到高                                   | 低                             |
| 性能            | 中到高，非常高                    | 中到高                                   | 低到中                         |
| 一致性          | 强一致性                         | 强一致性                                 | 强一致性 [5]                   |
| 数据访问        | SAS/iSCSI/FC                     | 标准文件访问，CIFS/SMB 和 NFS            | RESTful API                    |
| 可扩展性        | 中等可扩展性                      | 高可扩展性                               | 超大规模可扩展性                |
| 适用场景        | 虚拟机（VM）、数据库              | 通用文件系统访问                          | 二进制数据、非结构化数据        |

一些与对象存储相关的术语：
- **存储桶（Bucket）**——对象的逻辑容器。名称全局唯一。
- **对象（Object）**——存储在存储桶中的单个数据单元。包含对象数据和元数据（metadata）。
- **版本管理（Versioning）**——在同一存储桶中保留一个对象的多个版本的功能。
- **统一资源标识符（Uniform Resource Identifier，URI）**——每个资源都由一个 URI 唯一标识。
- **服务等级协议（Service-level Agreement，SLA）**——服务提供者与客户端之间的契约。

Amazon S3 低频访问（Standard-Infrequent Access）存储类的 SLA：
- 跨多个可用区（Availability Zone）持久性达 99.999999999%
- 在整个可用区被摧毁的情况下，数据仍具韧性
- 设计可用性为 99.9%

---

## 第 1 步：理解问题并确定设计范围

- 候选人（C）：应该包含哪些功能？
- 面试官（I）：存储桶创建、对象上传/下载、版本管理、列出存储桶中的对象
- 候选人（C）：典型的数据大小是多少？
- 面试官（I）：我们需要高效地存储超大对象和小对象
- 候选人（C）：我们一年存储多少数据？
- 面试官（I）：100 PB
- 候选人（C）：我们可以假设数据持久性达到六个九（99.9999%）、服务可用性达到四个九（99.99%）吗？
- 面试官（I）：可以，听起来合理

### **非功能需求**

- **100 PB 数据**
- **六个九的数据持久性**
- **四个九的服务可用性**
- 存储效率。在保持高可靠性和高性能的同时降低存储成本

### **粗略估算（Back-of-the-envelope estimation）**

对象存储的瓶颈很可能出现在磁盘容量或每秒输入输出操作数（IOPS）上。

假设：
- 我们有 20% 的小对象（小于 1MB）、60% 的中等对象（1-64MB）和 20% 的大对象（大于 64MB），
- 一块硬盘（SATA，7200rpm）每秒能够进行 100-150 次随机寻道（100-150 IOPS）

根据这些假设，我们可以估算系统能够持久化的对象总数。
- 让我们使用每种对象类型的中位数大小来简化计算——小对象 0.5MB，中对象 32MB，大对象 200MB。
- 给定 100PB 存储（10^11 MB），按 40% 的存储使用率计算，得到 6.8 亿个对象
- 如果假设元数据为 1KB，那么我们需要 0.68TB 的空间来存储元数据信息

---

## 第 2 步：提出高层设计并获得认可

在深入设计之前，让我们先探讨对象存储的一些有意思的特性：
- **对象不可变性（Object immutability）**——对象存储中的对象是不可变的（其他存储系统并非如此）。我们可以删除或替换它们，但不能更新。
- **键值存储（Key-value store）**——对象 URI 就是它的键，我们可以通过发起一次 HTTP 调用来获取其内容
- **一次写入、多次读取（Write once, read many times）**——数据访问模式是写入一次、读取多次。根据 LinkedIn 的一些研究，95% 的操作是读取
- 同时支持小对象和大对象

对象存储的设计哲学与 UNIX 类似——当我们保存文件时，它会在一个称为 inode 的数据结构中创建文件名，而文件数据则存储在不同的磁盘位置。
inode 包含一个文件块指针列表，这些指针指向磁盘上的不同位置。

访问文件时，我们首先从 inode 获取其元数据，然后再获取文件内容。

对象存储的工作方式类似——元数据存储用于保存文件信息，而内容则存储在磁盘上：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/object-store-vs-unix.png" alt="对象存储与 UNIX 对比" width="500" />
</div>

通过将元数据与文件内容分离，我们可以独立地扩展不同的存储：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/bucket-and-object.png" alt="存储桶与对象" width="500" />
</div>

### **高层设计**

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/high-level-design.png" alt="高层设计" width="500" />
</div>

- **负载均衡器（Load balancer）**——将 API 请求分发到各个服务副本
- **API 服务（API service）**——无状态服务器，协调对元数据存储、对象存储以及 IAM 服务的调用。
- **身份与访问管理（Identity and access management，IAM）**——认证（auth）、授权（authz）、访问控制的中心位置。
- **数据存储（Data store）**——存储和检索实际数据。操作基于对象 ID（UUID）。
- **元数据存储（Metadata store）**——存储对象元数据

### **上传对象（Uploading an object）**

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/uploading-object.png" alt="上传对象" width="500" />
</div>

- 通过 HTTP PUT 请求创建一个名为「bucket-to-share」的存储桶
- API 服务调用 IAM 以确认用户已授权并具有写权限
- API 服务调用元数据存储创建一条存储桶记录。创建成功后，返回成功响应。
- 存储桶创建后，发送 HTTP PUT 以创建一个名为「script.txt」的对象
- API 服务验证用户身份，并确认用户具有写权限
- 验证通过后，对象负载（payload）通过 HTTP PUT 发送到数据存储。数据存储将其持久化，并返回一个 UUID。
- API 服务调用元数据存储，创建一条包含 object_id、bucket_name 以及其他元数据的新记录。

对象上传请求示例：

```
PUT /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 17:51:00 GMT
Authorization: authorization string
Content-Type: text/plain
Content-Length: 4567
x-amz-meta-author: Alex

[4567 bytes of object data]
```

### **下载对象（Downloading an object）**

存储桶没有目录层级，但我们可以通过拼接存储桶名和对象名来创建逻辑层级，以模拟文件夹结构。

获取对象的 GET 请求示例：

```
GET /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 18:30:01 GMT
Authorization: authorization string
```

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/download-object.png" alt="下载对象" width="500" />
</div>

- 客户端向负载均衡器发送 HTTP GET 请求，即 `GET /bucket-to-share/script.txt`
- API 服务查询 IAM，以验证用户具有读取该存储桶的正确权限
- 验证通过后，从元数据存储中检索对象的 UUID
- 根据 UUID 从数据存储中检索对象负载，并返回给客户端

---

// sprint 1

## 第 3 步：深入设计

### **数据存储（Data store）**

下面是 API 服务与数据存储交互的方式：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/data-store-interactions.png" alt="数据存储交互" width="500" />
</div>

数据存储的主要组件：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/data-store-main-components.png" alt="数据存储主要组件" width="500" />
</div>

数据路由服务（data routing service）提供 RESTful 或 gRPC API，用于访问数据节点集群。
它是一个无状态服务，通过添加更多服务器来扩展。

它的主要职责是：
- 查询放置服务（placement service），以获得存储数据的最佳数据节点
- 从数据节点读取数据并返回给 API 服务
- 将数据写入数据节点

放置服务决定哪些数据节点应该存储某个对象。
它维护一张虚拟集群映射（virtual cluster map），该映射决定了集群的物理拓扑。

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/virtual-cluster-map.png" alt="虚拟集群映射" width="500" />
</div>

该服务还向所有数据节点发送心跳（heartbeat），以判断它们是否应从虚拟集群中移除。

由于这是一个关键服务，建议维护一个由 5 或 7 个副本组成的集群，通过 Paxos 或 Raft 共识算法进行同步。
例如，一个 7 节点集群可以容忍 3 个节点故障。

数据节点存储实际的对象数据。
通过将数据复制到多个数据节点来确保可靠性和持久性。

每个数据节点都运行一个守护进程（daemon），它向放置服务发送心跳。

心跳包含：
- 该数据节点管理多少个磁盘驱动器（HDD 或 SSD）？
- 每个驱动器上存储了多少数据？

#### 数据持久化流程

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/data-persistence-flow.png" alt="数据持久化流程" width="500" />
</div>

- API 服务将对象数据转发给数据存储
- 数据路由服务将数据发送到主数据节点（primary data node）
- 主数据节点在本地保存数据，并将其复制到两个从数据节点（secondary data nodes）。复制成功后发送响应。
- 对象的 UUID 返回给 API 服务。

注意事项：
- 给定一个对象 UUID，它的复制组是通过一致性哈希（consistent hashing）确定性地选择的
- 在第 4 步中，主数据节点在返回响应之前复制对象数据。这优先保证了强一致性，而不是更低的延迟。

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/consistency-vs-latency.png" alt="一致性与延迟权衡" width="500" />
</div>

#### 数据如何组织

管理数据的一种简单方法是将每个对象存储在单独的文件中。

这可行，但在文件系统中有大量小文件时性能不佳：
- HDD 上的数据块被浪费了，因为每个文件都占用整个块大小。典型的块大小是 4KB。
- 文件多意味着 inode 多。操作系统对过多 inode 的处理能力不佳，而且还有 inode 数量上限。

这些问题可以通过预写日志（write-ahead log，WAL）将许多小文件合并成大文件来解决。一旦文件达到其容量（通常为几 GB），就创建一个新文件：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/wal-optimization.png" alt="WAL 优化" width="500" />
</div>

这种方法的缺点是对文件的写访问必须串行化（serialized）。多个核心访问同一个文件时必须相互等待。
为了解决这个问题，我们可以将文件限定到特定的核心，以避免锁竞争（lock contention）。

#### 对象查找

为了支持在同一文件中存储多个对象，我们需要维护一张表，它告诉数据节点：
- `object_id`
- 对象存储的 `filename`
- 对象起始位置的 `file_offset`
- `object_size`

我们可以把这张表部署在基于文件的数据库中，如 RocksDB，或传统的关系型数据库中。
由于访问模式是低写入 + 高读取，关系型数据库更合适。

我们该如何部署它？
我们可以部署这个数据库，并在一个集群中单独扩展它，供所有数据节点访问。

缺点：
- 我们需要积极地扩展集群来服务所有请求
- 数据节点与数据库集群之间存在额外的网络延迟

另一种方法是利用这样一个事实：数据节点只关心与自己相关的数据，
因此我们可以将关系型数据库部署在数据节点内部。

SQLite 是一个不错的选择，因为它是一个轻量级的基于文件的关系型数据库。

#### 更新后的数据持久化流程

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/updated-data-persistence-flow.png" alt="更新后的数据持久化流程" width="500" />
</div>

- API 服务发送保存新对象的请求
- 数据节点服务将新对象追加到一个名为「/data/c」的文件末尾
- 为该对象在对象映射表中插入一条新记录

#### 持久性（Durability）

数据持久性是我们设计中的一项重要需求。为了实现六个九的持久性，每一种故障情况都需要妥善考察。

首先要解决的问题是硬件故障。我们可以通过复制数据节点来最大限度地降低故障概率。
但除此之外，我们还应该跨不同的故障域（failure domain）进行复制（跨机架、跨数据中心、独立网络等）。
一个关键事件可能导致同一域内多个硬件同时故障：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/failure-domain-isolation.png" alt="故障域隔离" width="500" />
</div>

假设一块典型 HDD 的年故障率为 0.81%，制作三个副本就能让我们获得六个九的持久性。

这样复制数据节点为我们提供了所需的持久性，但我们也可以利用纠删码（erasure coding）来降低存储成本。

纠删码使我们能够使用奇偶校验位（parity bits），在发生故障时重建丢失的数据位：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/erasure-coding.png" alt="纠删码" width="500" />
</div>

想象这些位就是数据节点。如果其中两个宕机，就可以用剩下的四个来恢复它们。

纠删码有不同的方案。在我们的场景中，可以使用 8+4 纠删码，分布到不同的故障域中，以最大化可靠性：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/erasure-coding-across-failure-domains.png" alt="跨故障域的纠删码" width="500" />
</div>

纠删码使我们能够以低得多的存储成本（提升 50%）实现目标，代价是访问速度变慢，因为数据路由服务必须从多个位置收集数据：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/erasure-coding-vs-replication.png" alt="纠删码与复制对比" width="500" />
</div>

其他注意事项：
- 复制需要 200% 的存储开销（3 副本的情况下），而纠删码只需 50%
- 纠删码[为我们提供 11 个九的持久性](https://github.com/Backblaze/erasure-coding-durability)，而复制只有 6 个九
- 纠删码需要更多计算来计算和存储奇偶校验

总之，复制更适合对延迟敏感的应用，而纠删码在存储成本效率和持久性方面更具吸引力。
纠删码的实现也要困难得多。

#### 正确性校验

如果磁盘完全故障，那么故障很容易检测。但如果磁盘内存的一部分损坏，就没那么直接了。

为了检测这种情况，我们可以使用校验和（checksum）——文件内容的哈希，可用于验证文件的完整性。

在我们的场景中，我们会为每个文件和每个对象存储校验和：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/checksums-for-correctness.png" alt="用于正确性校验的校验和" width="500" />
</div>

在使用纠删码（8+4）的情况下，我们需要分别获取 8 份数据，并验证每份的校验和。

// sprint 2

### **元数据数据模型**

表结构：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/metadata-data-model.png" alt="元数据数据模型" width="500" />
</div>

我们需要支持的查询：
- 按名称查找对象 ID
- 根据名称插入/删除对象
- 列出存储桶中具有相同前缀的对象

用户可创建的存储桶数量通常有限制，因此 buckets 表的规模很小，可以放进单台数据库服务器。
但我们仍然需要为读吞吐量扩展这台服务器。

不过，object 表可能无法放进单台数据库服务器。因此，我们可以通过分片（sharding）来扩展这张表：
- 按 bucket_id 分片会导致热点问题，因为一个存储桶可能有数十亿个对象
- 按 bucket_id 分片能让负载更均匀地分布，但我们的查询会变慢
- 我们选择按 `hash(bucket_name, object_name)` 分片，因为大多数查询都是基于对象/存储桶名称的。

不过，即使采用这种分片方案，列出存储桶中的对象仍然会很慢。

### **列出存储桶中的对象**

在单个数据库中，基于前缀（看起来像目录）列出对象的工作方式如下：

```
SELECT * FROM object WHERE bucket_id = "123" AND object_name LIKE `abc/%`
```

当数据库被分片时，实现这一点具有挑战性。要实现它，我们可以在每个分片上运行该查询，并在内存中聚合结果。
但这会让分页变得具有挑战性，因为不同分片包含的结果大小不同，我们需要为每个分片维护独立的 limit/offset。

我们可以利用这样一个事实：对象存储通常并未针对列出对象做优化，因此我们可以牺牲列出性能。
我们还可以创建一张反规范化（denormalized）表来列出对象，按存储桶 ID 分片。
这会让我们的列出查询足够快，因为它被隔离在单个数据库实例中。

### **对象版本管理（Object versioning）**

版本管理通过增加一个 `object_version` 列来实现，该列类型为 TIMEUUID，使我们能够据此对记录排序。

每个新版本都会产生一个新的 `object_id`：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/object-versioning.png" alt="对象版本管理" width="500" />
</div>

删除对象会创建一个新版本，该版本带有一个特殊的 `object_id`，表明对象已被删除。查询它会返回 404：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/deleting-versioned-object.png" alt="删除带版本的对象" width="500" />
</div>

### **优化大文件上传**

上传大文件可以通过使用分片上传（multipart upload）来优化——将一个大文件拆分成多个数据块（chunk），独立上传：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/multipart-upload.png" alt="分片上传" width="500" />
</div>

- 客户端调用服务以发起一次分片上传
- 数据存储返回一个上传 ID，唯一标识此次上传
- 客户端将大文件拆分成多个数据块，使用上传 ID 独立上传
- 上传完一个数据块后，数据存储返回一个 etag，它是一个 md5 校验和，标识该上传数据块
- 所有部分上传完成后，客户端发送一个完成分片上传请求，其中包含 upload_id、各部分编号和所有 etag
- 数据存储从各部分重组对象。这个过程可能需要几分钟。之后，成功响应返回给客户端。

此时可以移除不再有用的旧部分。我们可以引入一个垃圾回收器（garbage collector）来处理。

### **垃圾回收（Garbage collection）**

垃圾回收是回收不再使用的存储空间的过程。数据变成垃圾有几种方式：
- **惰性对象删除（lazy object deletion）**——对象被标记为已删除，但并未真正删除
- **孤儿数据（orphan data）**——例如上传中途失败，旧的部分需要删除
- **损坏数据（corrupted data）**——未通过校验和验证的数据

垃圾回收器还负责回收副本中未使用的空间。使用复制时，数据会从主节点和副本中删除。使用纠删码（8+4）时，数据会从全部 12 个节点中删除。

为了便于删除，我们将使用一个称为压缩（compaction）的过程：
- 垃圾回收器将未删除的对象从「data/b」复制到「data/d」
- 复制完成后，使用一个数据库事务更新 `object_mapping` 表
- 为避免产生过多小文件，只对增长超过某个阈值的文件进行压缩

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/compaction.png" alt="压缩" width="500" />
</div>

---

## 第 4 步：总结

我们涵盖的内容：
- 设计一个类 S3 的对象存储
- 比较对象存储、块存储和文件存储之间的差异
- 涵盖存储桶中对象的上传、下载、列出、版本管理
- 深入设计——数据存储与元数据存储、复制与纠删码、分片上传、分片
