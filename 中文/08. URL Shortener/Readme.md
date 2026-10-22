# 第 8 章：设计 URL 缩短服务

## 引言
本章讨论如何设计一个像 TinyURL 这样的 URL 缩短服务。系统的主要目标包括 **URL 缩短**、**重定向**以及**高可扩展性**，以承载巨大的流量。

### 需求
- 缩短后的 URL 必须**唯一**，并且尽可能**短**。
- 每天处理 **1 亿次 URL 生成**，支持容量按 10 年规划。
- 支持**高效的读操作**，读写比为 10:1。
- 存储 3650 亿条记录，10 年下来约需要 **365 TB** 存储。

---

## 第 1 步：高层设计

### API 端点
1. **URL 缩短：**  
   - 端点：`POST api/v1/data/shorten`  
   - 参数：`{longUrl: longURLString}`  
   - 返回：`shortURL`

2. **URL 重定向：**  
   - 端点：`GET api/v1/shortUrl`  
   - 返回：用于重定向的 `longURL`。

    <p align="center">
    <img src="../../08.%20URL%20Shortener/images/url-redirection.png" alt="URL 重定向" width="600">
    </p>

### URL 重定向
- **301 重定向：** 301 重定向表示所请求的 URL 被「永久」移动到了长 URL。浏览器会缓存该响应，后续对同一 URL 的请求将不再发送到 URL 缩短服务。
- **302 重定向：** 临时的；便于做点击统计等分析。

### URL 缩短
<p align="center">
    <img src="../../08.%20URL%20Shortener/images/url-shortening.png" alt="URL 缩短" width="400">
</p>

- 使用**哈希函数**生成短 URL，把长 URL 映射为唯一的缩短版本。
- 哈希函数必须满足以下要求：
    - 每个 longURL 必须被哈希为一个 hashValue。
    - 每个 hashValue 都能映射回对应的 longURL。
    

---

## 第 2 步：深入设计

### 数据模型
把 `<shortURL, longURL>` 映射存储在关系型数据库中，以优化内存使用。表结构包含：
- `id`（主键 primary key），
- `shortURL`，
- `longURL`。

    <img src="../../08.%20URL%20Shortener/images/table-schema.png" alt="表结构" width="300">

### 哈希函数
#### 1. Base 62 转换：
- 使用字符 `[0-9, a-z, A-Z]` 对数字编码，共提供 **62 个可用字符**。
- 进制转换是 URL 缩短服务常用的另一种方法。
- 可以为短 URL 分配一个唯一 ID，再把该 ID 做 Base 62 转换，得到短 URL。
- 7 个字符的哈希最多支持 **3.5 万亿个唯一 URL**，足够应对 3650 亿个 URL。

**示例：**  
把 ID `2009215674938` 转换为 Base 62：
- `2009215674938` → `zn9edcu`。

#### 2. 哈希 + 冲突解决：
- 使用 CRC32、MD5 或 SHA-1 等哈希函数。

    <img src="../../08.%20URL%20Shortener/images/hash-function.png" alt="哈希函数" width="500">

- 一种做法是取哈希值的前 7 个字符；然而这种方法可能导致哈希冲突。
- 为解决冲突，可以递归地追加一个新的预定义字符串，直到不再冲突，但这样做代价可能很高。
- 使用**布隆过滤器（Bloom Filter）**解决冲突，以实现高效查找。

    <p align="center">
    <img src="../../08.%20URL%20Shortener/images/url-lookup.png" alt="URL 查找" width="500">
    </p>

### 两种方案对比

- **哈希 + 冲突解决：**
    - 短 URL 长度固定
    - 不需要唯一 ID 生成器
    - 可能发生冲突，需要解决
    - 无法找到下一个可用的短 URL，因为它不依赖 ID

- **Base 62 转换**
    - 长度不固定，会随 ID 增大而变长
    - 需要唯一 ID 生成器
    - 不会发生冲突
    - 如果 ID 按 1 递增，很容易推算出下一个短 URL（这可能是一个安全隐患）


---

### URL 缩短流程

<p align="center">
    <img src="../../08.%20URL%20Shortener/images/url-shortening-flow.png" alt="URL 缩短流程" width="500">
</p>

1. 检查 `longURL` 是否已存在于数据库中。
2. 如果已存在，返回已有的 `shortURL`。
3. 否则：
   - 使用**分布式 ID 生成器**生成唯一 ID。
   - 用 Base 62 把 ID 转换为 `shortURL`。
   - 把 `<id, shortURL, longURL>` 映射存入数据库。



---

### URL 重定向流程
<p align="center">
    <img src="../../08.%20URL%20Shortener/images/url-redirecting-flow.png" alt="URL 重定向流程" width="600">
</p>

1. 用户点击某个 `shortURL`。
2. 查询 `<shortURL, longURL>` 映射：
   - 先查**缓存**，以获得更快的访问。
   - 如果缓存未命中，再查询数据库。
3. 把用户重定向到 `longURL`。


---

## 其他考量
### 限流器
- 通过限制每个 IP 的请求数来防止滥用。

### 可扩展性
1. **Web 层：** 无状态，可通过增减 Web 服务器来扩展。
2. **数据库层：** 使用复制与分片。

### 数据分析
- 收集点击率、来源、时间戳等数据，用于业务洞察。

### 高可用性与可靠性
- 通过数据库复制与容错设计，确保服务一致、可靠。
