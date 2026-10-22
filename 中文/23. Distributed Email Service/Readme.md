# 第 23 章：分布式电子邮件服务

## 引言

本章我们将设计一个**分布式电子邮件服务**，类似于 **Gmail**。

2020 年，**Gmail** 有 18 亿活跃用户，而 **Outlook** 在全球有 4 亿用户。

---

## 第 1 步：理解问题并确定设计范围

- C：有多少用户使用这个系统？
- I：10 亿用户。
- C：我认为以下功能很重要 —— 认证、发送/接收邮件、拉取邮件、过滤邮件、搜索邮件、反垃圾邮件保护。
- I：这份清单不错。认证的事现在先不用管。
- C：用户如何连接到邮件服务器？
- I：通常，邮件客户端通过 SMTP、POP、IMAP 连接，但这个问题我们用 HTTP。
- C：邮件可以带附件吗？
- I：可以。

### **非功能性需求**

- **可靠性（Reliability）** —— 我们不应该丢数据。
- **可用性（Availability）** —— 我们应该使用复制来防止单点故障。我们还应能容忍部分系统故障。
- **可扩展性（Scalability）** —— 随着用户群增长，我们的系统应能应付。
- **灵活性与可扩展性（Flexibility and extensibility）** —— 系统应灵活、易于用新功能扩展。这也是我们选择 HTTP 而非 SMTP/其他邮件协议的原因之一。

### **粗略估算（Back-of-the-envelope estimation）**

- **10 亿用户**。
- 假设每人每天发 10 封邮件 → **每秒 10 万封邮件**。
- 假设每人每天收 40 封邮件，每封邮件平均有 50kb 元数据 → **每年 730pb 存储**。
- 假设 20% 的邮件带存储型附件，平均大小 500kb → **每年 1460pb**。

---

## 第 2 步：提出高层级设计并获得认可

### **电子邮件基础知识 101**

收发邮件有多种协议：
- **SMTP** —— 把邮件从一台服务器发送到另一台服务器的标准协议。
- **POP** —— 把邮件从远程邮件服务器接收并下载到本地客户端的标准协议。一旦取回，邮件就从远程服务器删除。
- **IMAP** —— 与 POP 类似，用于从远程服务器接收和下载邮件，但它会把邮件保留在服务端。
- **HTTPS** —— 严格说不是邮件协议，但可用于基于 Web 的邮件客户端。

除了邮件协议之外，我们还需要为邮件服务器配置一些 DNS 记录 —— MX 记录：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/dns-lookup.png" alt="DNS 查询" width="500" />
</div>

邮件附件以 base64 编码发送，大多数邮件服务通常有 25mb 的大小上限。
这个值是可配置的，个人账户与企业账户之间各不相同。

### **传统邮件服务器**

当用户数量有限、都连接到单台服务器时，传统邮件服务器运转良好。

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/traditional-mail-server.png" alt="传统邮件服务器" width="500" />
</div>

- Alice 登录她的 Outlook 邮箱并点击「发送」。邮件被发送到 Outlook 邮件服务器。通信走 SMTP。
- Outlook 服务器查询 DNS 找到 gmail.com 的 MX 记录，并把邮件转交到他们的服务器。通信走 SMTP。
- Bob 通过 IMAP/POP 从他的 gmail 服务器拉取邮件。

在传统邮件服务器中，邮件存储在本地文件系统上。每封邮件都是一个单独的文件。

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/local-dir-storage.png" alt="本地目录存储" width="500" />
</div>

随着规模增长，磁盘 I/O 成了瓶颈。而且它也不满足我们对高可用性和可靠性的要求。
磁盘可能损坏，服务器可能宕机。

### **分布式邮件服务器**

分布式邮件服务器专为支持现代用例、解决现代可扩展性问题而设计。

这些服务器仍然可以为原生邮件客户端支持 IMAP/POP，并为跨服务器的邮件交换支持 SMTP。

但对于功能丰富的 Web 邮件客户端，通常在 HTTP 之上使用 RESTful API。

API 示例：
- `POST /v1/messages` - 向 To、Cc、Bcc 头部中的收件人发送一条消息。
- `GET /v1/folders` - 返回一个邮件账户的所有文件夹。

响应示例：

```
[{id: string        Unique folder identifier.
  name: string      Name of the folder.
                    According to RFC6154 [9], the default folders can be one of
                    the following: All, Archive, Drafts, Flagged, Junk, Sent,
                    and Trash.
  user_id: string   Reference to the account owner
}]
```

- `GET /v1/folders/{:folder_id}/messages` - 分页返回某文件夹下的所有消息。
- `GET /v1/messages/{:message_id}` - 获取某条消息的全部信息。

响应示例：

```
{
  user_id: string                      // Reference to the account owner.
  from: {name: string, email: string}  // <name, email> pair of the sender.
  to: [{name: string, email: string}]  // A list of <name, email> paris
  subject: string                      // Subject of an email
  body: string                         //  Message body
  is_read: boolean                     //  Indicate if a message is read or not.
}
```

下面是分布式邮件服务器的高层级设计：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/high-level-architecture.png" alt="高层级架构" width="500" />
</div>

- **Webmail** —— 用户用网页浏览器收发邮件。
- **Web 服务器（Web servers）**：面向公网的请求/响应服务，用于管理登录、注册、用户资料等。
- **实时服务器（Real-time servers）** —— 用于把新邮件更新实时推送给客户端。我们使用 websockets 做实时通信，但对不支持它的旧浏览器回退到长轮询（long-polling）。
- **元数据数据库（Metadata db）** —— 存储邮件元数据，如主题、正文、发件人、收件人等。
- **附件存储（Attachment store）** —— 对象存储（如 Amazon S3），适合存储大文件。
- **分布式缓存（Distributed cache）** —— 我们可以把最近的邮件缓存在 Redis 中，以改善用户体验。
- **搜索存储（Search store）** —— 分布式文档存储，用于支持全文搜索。

下面是邮件发送流程的样子：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/email-sending-flow.png" alt="邮件发送流程" width="500" />
</div>

- 用户写好邮件并点击「发送」。邮件被发送到负载均衡器。
- 负载均衡器对过量发信做限流，并把请求路由到某台 Web 服务器。
- Web 服务器做基本的邮件校验（如邮件大小），如果收件域名与发件人相同，则短路外发流程。但会先做垃圾邮件检查。
- 如果基本校验通过，邮件被发送到消息队列（附件从对象存储引用）。
- 如果基本校验失败，邮件被发送到错误队列。
- SMTP 外发 worker 从外发队列拉取消息，做垃圾邮件/病毒检查，并路由到目标邮件服务器。
- 邮件被存入「已发送邮件」文件夹。

我们还需要监控外发消息队列的大小。队列增长过大可能预示着问题：
- 收件人的邮件服务器不可用。我们可以稍后用指数退避（exponential backoff）重试发送。
- 消费者不足以处理负载，我们可能需要扩展消费者。

下面是邮件接收流程：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/email-receiving-flkow.png" alt="邮件接收流程" width="500" />
</div>

- 入站邮件到达 SMTP 负载均衡器。邮件被分发到各 SMTP 服务器，在那里执行邮件接收策略（例如无效邮件直接丢弃）。
- 如果邮件附件过大，我们可以把它放进对象存储（s3）。
- 邮件处理 worker 做初步检查，之后邮件被转发到存储、缓存、对象存储和实时服务器。
- 离线用户在重新上线后，通过 HTTP API 取回自己的新邮件。

---

## 第 3 步：深入设计

现在让我们更深入地探讨其中一些组件。

### **元数据数据库**

下面是邮件元数据的一些特征：
- 头部通常很小，且被频繁访问。
- 正文大小从小到大都有，但通常只读一次。
- 大多数邮件操作都隔离在单个用户范围内 —— 例如拉取邮件、标记为已读、搜索。
- 数据的新近度影响数据使用。用户通常只看最近的邮件。
- 数据有很高的可靠性要求。数据丢失是不可接受的。

在 gmail/outlook 这样的规模下，数据库通常是定制打造的，以降低每秒输入/输出操作数（IOPS）。

让我们看看有哪些数据库可选：
- **关系型数据库** —— 我们可以为头部和正文建索引，但这类 DB 通常是为小块数据优化的。
- **分布式对象存储** —— 它可以作为备份存储的不错选择，但无法高效支持搜索/标记已读等操作。
- **NoSQL** —— Google BigTable 被 gmail 使用，但它没有开源。

基于以上分析，现有方案中似乎很少有能完美满足我们需求的。
在面试场景下，设计一套新的分布式数据库方案是不现实的，但重要的是要提到这些特征：
- 单列可以是个位数 MB 量级。
- 强数据一致性。
- 为减少磁盘 I/O 而设计。
- 高可用、容错。
- 应易于做增量备份。

为了对数据分区，我们可以用 `user_id` 作为分区键，这样一个用户的数据就存储在单个分片上。
这会让我们无法把一封邮件分享给多个用户，但这不在本次面试的要求内。

让我们定义这些表：
- 主键由分区键（数据分布）和聚簇键（数据排序）组成。
- 我们需要支持的查询 —— 获取某用户的所有文件夹、展示某文件夹的所有邮件、创建/获取/删除一封邮件、拉取已读/未读邮件、获取会话线程（加分项）。

后面这些表的图例（legend）：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/legend.png" alt="图例" width="500" />
</div>

下面是 folders 表：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/folders-table.png" alt="folders 表" width="500" />
</div>

emails 表：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/emails-table.png" alt="emails 表" width="500" />
</div>

- email_id 是 timeuuid，允许按邮件创建时的时间戳排序。

附件存在单独的表中，以文件名标识：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/attachments.png" alt="附件" width="500" />
</div>

在传统关系型数据库中支持拉取已读/未读邮件很容易，但在 Cassandra 中不行，因为禁止对非分区/聚簇键做过滤。
一个变通办法是取出某文件夹中的所有邮件再在内存中过滤，但这对于足够大的应用并不奏效。

我们可以做的是把 emails 表反规范化（denormalize）成已读/未读邮件表：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/read-unread-emails.png" alt="已读未读邮件" width="500" />
</div>

为了支持会话线程（conversation threads），我们可以加入一些头部，邮件客户端会解析并用它们重建会话线程：

```
{
  "headers" {
     "Message-Id": "<7BA04B2A-430C-4D12-8B57-862103C34501@gmail.com>",
     "In-Reply-To": "<CAEWTXuPfN=LzECjDJtgY9Vu03kgFvJnJUSHTt6TW@gmail.com>",
     "References": ["<7BA04B2A-430C-4D12-8B57-862103C34501@gmail.com>"]
  }
}
```

最后，我们的分布式数据库会用可用性来换取一致性，因为一致性是这个问题的硬性要求。

因此，在发生故障转移（failover）或网络分区时，受影响用户的同步/更新操作会短暂不可用。

### **邮件可送达性（Email deliverability）**

搭建一台发邮件的服务器很容易，但要让邮件进入收件人的收件箱却很难，因为有反垃圾邮件算法。

如果我们只是搭一台新邮件服务器就开始通过它发信，我们的邮件很可能会落到垃圾邮件文件夹里。

下面是我们可以采取的预防措施：
- **专用 IP（Dedicated IPs）** —— 使用专用 IP 发邮件，否则收件方服务器不会信任你。
- **给邮件分类（Classify emails）** —— 避免从同一批服务器发送营销邮件，以免更重要的邮件被归类为垃圾邮件。
- **慢慢预热你的 IP 地址（Warm up your IP address）**，以在大型邮件提供商那里建立良好声誉。预热一个新 IP 需要 2 到 6 周。
- **迅速封禁垃圾邮件发送者**，以免损害你的声誉。
- **反馈处理（Feedback processing）** —— 与 ISP 建立反馈闭环，跟踪投诉率并迅速封禁垃圾账户。
- **邮件认证（Email authentication）** —— 使用常见技术对抗钓鱼，如发件人策略框架（SPF）、域名密钥识别邮件（DKIM）等。

这些你都不必记住。只需知道：搭建一台好的邮件服务器需要大量领域知识。

### **搜索（Search）**

搜索包括基于邮件内容做全文搜索，或基于发件人、收件人、主题、未读等过滤条件做更高级的查询。

邮件搜索的一个特点是：它是用户本地的，而且写多于读，因为每次操作后我们都要重建索引，但用户很少使用搜索标签页。

让我们把谷歌搜索与邮件搜索做个比较：

|               | 范围（Scope）                | 排序（Sorting）                               | 准确性（Accuracy）                                          |
|---------------|----------------------|---------------------------------------|---------------------------------------------------|
| 谷歌搜索 | 整个互联网   | 按相关性排序                     | 建索引需要一些时间，因此不是即时结果。 |
| 邮件搜索  | 用户自己的邮箱 | 按属性排序，如时间、日期等 | 建索引应该快，结果要准确。    |

要实现这个搜索功能，一个选择是使用 Elasticsearch 集群。我们可以用 `user_id` 作为分区键，把数据归到同一节点下：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/elasticsearch.png" alt="Elasticsearch" width="500" />
</div>

变更操作通过 Kafka 异步进行，以便把服务与重建索引流程解耦。
真正的数据搜索是同步发生的。

Elasticsearch 是最流行的搜索引擎数据库之一，对邮件的全文搜索支持得很好。

另一条路是，我们可以尝试开发自己的定制搜索方案，以满足特定需求。

设计这样一个系统不在范围内。自建时的核心挑战之一，就是针对写多读少的工作负载做优化。

要做到这一点，我们可以使用 LSM 树（Log-Structured Merge-Trees）在磁盘上组织索引数据。写入路径只针对顺序写入做优化。
这项技术被用于 Cassandra、BigTable 和 RocksDB。

它的核心思想是：先在内存中存储数据，直到达到预定义阈值，然后把它合并到下一层（磁盘）：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/lsm-tree.png" alt="LSM 树" width="500" />
</div>

两种方式之间的主要权衡：
- Elasticsearch 能扩展到一定程度，而定制搜索引擎可以针对邮件用例做微调，从而扩展得更远。
- Elasticsearch 是一个需要与元数据存储一起维护的独立服务。定制方案则可以就是数据存储本身。
- Elasticsearch 是现成方案，而定制搜索引擎需要大量工程投入来构建。

### **可扩展性与可用性**

由于单个用户的操作不会与其他用户冲突，大多数组件都可以独立扩展。

为了确保高可用性，我们还可以采用多数据中心（multi-DC）设置，在故障时用主从（leader-follower）故障转移：

<div style="margin-left:3rem">
    <img src="../../23.%20Distributed%20Email%20Service/images/multi-dc-example.png" alt="多数据中心示例" width="500" />
</div>

---

## 第 4 步：收尾

其他可谈的点：
- **容错（Fault tolerance）** —— 系统的许多部分都可能故障。值得讨论我们会如何处理节点故障。
- **合规（Compliance）** —— 考虑到欧洲的 GDPR 法律，个人身份信息（PII）需要以合理方式存储。
- **安全（Security）** —— 邮件加密、反钓鱼、安全浏览等。
- **优化（Optimizations）** —— 例如防止同一附件被不同用户重复发送时重复存储。
