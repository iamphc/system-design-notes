# 第 16 章：邻近服务（Proximity Service）

## 引言
**邻近服务（proximity service）** 用于查找附近的地点，例如餐厅、酒店、加油站和其他商户。这项功能被用于 **Google Maps** 和 **Yelp** 等应用中，帮助用户发现指定半径范围内的地点。


## 第 1 步：理解问题并确定范围

### **功能需求**
1. 根据用户位置（纬度、经度）和搜索半径**搜索商户**。
2. **允许商户所有者**添加、更新或删除商户（非实时）。
3. 在被请求时**提供详细的商户信息**。

### **非功能需求**
- **低延迟**：用户应能获得快速响应。
- **数据隐私**：符合 GDPR 和 CCPA 法规。
- **高可用性**：应对繁忙地段高峰时段的流量尖峰。

### **粗略估算（Back-of-the-Envelope Estimation）**
- **1 亿日活跃用户**。
- 系统中有 **2 亿个商户**。
- **搜索 QPS 计算**：
  - 用户每天进行 **5 次搜索**。
  - **搜索 QPS** = (100M × 5) / 86,400 ≈ **5,000 QPS**。

---

## 第 2 步：高层设计

### **API 设计**
#### **搜索附近的商户**
GET /v1/search/nearby

- **请求参数**：
  - `latitude`：用户位置纬度。
  - `longitude`：用户位置经度。
  - `radius`：搜索半径（默认值：5000m）。

#### **商户 API**
| API 端点                     | 说明                                      |
|-----------------------------------|--------------------------------------------------|
| `GET /v1/businesses/{id}`         | 获取商户详细信息                    |
| `POST /v1/businesses`             | 添加一个新商户                              |
| `PUT /v1/businesses/{id}`         | 更新商户详情                         |
| `DELETE /v1/businesses/{id}`      | 从系统中移除一个商户               |


### **数据模型**
- 由于有两个功能被非常频繁地使用，读量很大，因此像 MySQL 这样的关系型数据库是一个不错的选择。
  - 搜索附近的商户
  - 查看某个商户的详细信息

### **数据 Schema**
- 关键的数据库表是商户表（business table）和地理空间索引表（geospatial index table）。
- 商户表包含关于某个商户的详细信息。

### **高层系统架构**
该系统由两部分组成：基于位置的服务（LBS，Location Based Service）和商户相关服务。

<div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/high-level-design.png" alt="高层设计" width="400" />
</div>

- **基于位置的服务（LBS）**： 
  - 处理基于位置的搜索查询。
  - 一个读多写少（read-heavy）的服务，没有写请求。
  - QPS 很高，尤其是在密集地段的高峰时段，并且该系统是无状态的。
- **商户服务（Business Service）**：处理两类请求。
  - 商户所有者创建、更新或删除商户。
  - 顾客查看某个商户的详细信息。
- **负载均衡器：** 将流量路由到 LBS 和商户服务。
- **数据库集群：** 
  - 对读多写少的工作负载使用**主-副本架构（primary-replica architecture）**。
  - LBS 读取的数据与主数据库写入的数据之间可能存在一些差异。
  - 这种不一致不是问题，因为商户信息并不是实时更新的。


---

## 第 3 步：获取附近商户的算法

### **方案 1：二维搜索（朴素方法）**

<div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/2d-search.png" alt="二维搜索" width="250" />
</div>

最直观的方法是画一个具有预定义半径的圆，找出圆内的所有商户。

**SQL 查询：**
```
SELECT business_id, latitude, longitude
FROM business
WHERE (latitude BETWEEN :lat - radius AND :lat + radius)
AND (longitude BETWEEN :long - radius AND :long + radius);
```
**问题：**
- **效率低**：需要扫描整个数据库。
- **受限于一维索引**（纬度/经度）。

一个可能的改进是在经度和纬度列上建立索引，尽管这稍微好一点，但仍然非常慢。

### 更好的方法
- 上一种方法的问题在于，数据库索引只能提高一个维度上的搜索速度。
- 一种最优的方法是使用地理空间索引，将二维数据表示为一维。
  - 哈希（Hash）：均匀网格（Even grid）、地理哈希（Geo Hash）
  - 树（Tree）：四叉树（Quadtree）、Google S2、R 树（RTree）

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/geospatial-index-types.png" alt="地理空间索引类型" width="500" />
  </div>


### **方案 2：均匀划分的网格**

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/even-grid.png" alt="均匀网格" width="400" />
  </div>

- **将世界划分为固定大小的网格**。
- **问题**：商户分布不均匀（城市密度高，农村地区稀疏）。

### **方案 3：地理哈希（Geohash）**
- 沿本初子午线和赤道将地球划分为四个象限。然后将每个网格再划分为四个更小的网格。 
- 每个网格可以通过在经度位和纬度位之间交替来表示。
- 重复这种细分。

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/geohash.png" alt="地理哈希" width="300" />
    <img src="../../16.%20Proximity%20Service/images/geohash-1.png" alt="地理哈希" width="285" />
  </div>


- **将纬度和经度编码为单个字母数字字符串**。它有 12 个精度（层级）。
- **分层的网格结构**允许高效搜索。
- 根据对照表，使用最小的地理哈希长度来选择合适的精度。
  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/geohash-radius-mapping.png" alt="地理哈希半径对照" width="400" />
  </div>
- 地理哈希保证：两个地理哈希之间共享的前缀越长，它们在地理位置上就越近。

- **挑战**：
  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/boundary-issue.png" alt="边界问题" width="300" />
  </div>

  - **边界问题**（靠近网格边缘的商户可能被排除在外）。
    - 两个地点可能非常近，但完全没有共享前缀（可能在赤道的另一侧）。
    - 两个地点可能有很长的共享前缀，却属于不同的地理哈希。
  - 解决方案：需要搜索相邻的网格。


### **方案 4：四叉树（Quadtree）**

  四叉树是一种树状数据结构，它递归地将二维空间划分为四个象限，每个内部节点恰好有四个子节点，表示该空间的四个子区域。
  - 四叉树是一种内存中的数据结构，它运行在每台 LBS 服务器上，并在服务器启动时构建。

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/quadtree.png" alt="四叉树" width="500" />
  </div>

  - 根节点被递归地分解为 4 个象限，直到没有哪个节点包含超过 x 个商户为止（本例中为 100 个）。

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/building-quadtree.png" alt="构建四叉树" width="500" />
  </div>

- 四叉树索引占用的内存不多（通常以 GB 计），可以轻松放进一台服务器。
- 由于构建该树的时间复杂度为 nlogn，构建这棵树可能需要几分钟。
- **对 k 近邻搜索查询很高效**（例如，找到最近的加油站）。

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/realworld-quadtree.png" alt="真实世界的四叉树" width="400" />
  </div>

#### 运维考量
 - 对于大约 2 亿个商户，在服务器启动时构建一棵四叉树可能需要几分钟。
 - 在四叉树构建期间，它无法对外提供流量，因此新版本应该增量地发布到一部分服务器上。
 - 更新一个商户或新增一个商户时，最简单的方法是增量地重建四叉树。（这会导致大量缓存失效。）
 - 也可以动态地更新四叉树，但实现起来更复杂。（需要锁机制。）

### **方案 5：Google S2**
它基于希尔伯特曲线（Hilbert curve）将球面映射为一维索引。在希尔伯特曲线上相互靠近的两个点，在一维空间中也是靠近的。


  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/hilbert-curve.png" alt="希尔伯特曲线" width="300" />
    <img src="../../16.%20Proximity%20Service/images/geofence.png" alt="地理围栏" width="355" />
  </div>

- **使用希尔伯特曲线将地球划分为小单元格（cell）**。
- 非常适合地理围栏（geofencing），因为它可以用不同层级覆盖任意区域。
- 地理围栏还允许围绕感兴趣的区域定义参数。
- 另一个优势是，在 S2 中我们可以指定最小、最大层级和最大单元格数，而不必使用固定的精度层级。


## 权衡对比 

#### 地理哈希（Geohash）
- 易于使用和实现——无需构建/重建树。
- 支持固定半径的结果。
- 更新索引很容易。
- 无法根据人口密度动态调整网格大小。

#### 四叉树（Quadtree）
- 实现起来稍难。
- 支持获取 k 近邻商户。
- 可以根据人口密度动态调整网格大小。
- 更新索引更复杂，因为可能需要重建整棵树。

---

## 第 4 步：扩展数据库与缓存策略

### **扩展商户表**
- **按商户 ID 分片**确保数据均匀分布。
- 表中每个商户都有单独的行。

| 地理哈希 | 商户 ID |
|---------|------------|
| 9q9hvu  | 343        |
| 9q9hvu  | 347        |
| 9q9hvu  | 112        |

### **扩展地理空间索引**
- 地理哈希表可能不太适合分片。在本例中，所有数据都能放进单台服务器，因此没有技术上的分片理由。
- 更好的做法是使用读副本（read-replica）来分担读负载。



---

### **缓存策略**
最显而易见的缓存键选择是位置坐标，然而它有几个问题：
 - GPS 给出的位置坐标并不准确。
 - 用户可能移动，导致位置坐标发生变化。
 - 更好的键是地理哈希。

| 缓存键  | 缓存值 |
|------------|------------|
| `geohash`  | 该网格内的商户 ID 列表 |
| `business_id` | 商户详情（名称、地址、评价等） |

---

## 第 5 步：部署策略与最终架构

### **区域与可用区**
- **跨多个区域**部署 LBS 和商户服务。

### **处理实时更新**
- **商户更新按天批量处理**。

### **最终系统架构**


  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/final-design.png" alt="最终设计" width="500" />
  </div>


这个最终算法的工作过程如下：

## 检索附近商户的步骤
1. **用户请求：**  
   - 用户搜索 **500 米**范围内的餐厅。  
   - 客户端将**纬度（37.776720）、经度（-122.416730）和半径（500m）**发送给**负载均衡器**。

2. **请求转发：**  
   - **负载均衡器（LB）** 将请求转发给**基于位置的服务（LBS）**。

3. **地理哈希计算：**  
   - LBS 确定与半径匹配的**地理哈希长度**。  
   - 通过一张对照表，**500m 对应地理哈希长度 = 6**。

4. **获取相邻地理哈希：**  
   - LBS 计算**相邻的地理哈希**，以涵盖周边区域。  
   - 结果是一个列表：  
     ```
     [my_geohash, neighbor1_geohash, neighbor2_geohash, ..., neighbor8_geohash]
     ```

5. **从 Redis 获取商户 ID：**  
   - 对于列表中的每个地理哈希，LBS 查询**地理哈希 Redis 服务器**以获取**商户 ID**。  
   - 使用并行查询以最小化延迟。

6. **检索与排序商户：**  
   - LBS 从**商户信息 Redis 服务器**获取**完整的商户详情**。  
   - 商户按与用户位置的**距离排序**。  
   - **排序后的结果**被发回给客户端。

## 关键优化
- **并行 Redis 调用**：减少响应时间。  
- **地理哈希索引**：确保高效的空间查询。  
- **缓存**：加快商户数据的查找和检索。

这种方法确保了对用户位置附近商户的**低延迟、可扩展**检索。

---

### **选择最佳的索引方法**
| 索引方法 | 优点 | 缺点 |
|----------------|------|------|
| **地理哈希（Geohash）** | 易于实现，邻近搜索高效 | 边界问题，网格大小固定 |
| **四叉树（Quadtree）** | 可根据密度动态调整，支持 k 近邻查询 | 更复杂，需要重新平衡树 |
| **Google S2** | 高级地理围栏，用于 Google Maps | 实现难度更高 |

---

## 参考资料
1. [地理哈希算法（Geohash Algorithm）](https://www.movable-type.co.uk/scripts/geohash.html)
2. [四叉树索引（Quadtree Indexing）](https://en.wikipedia.org/wiki/Quadtree)
3. [Google S2 几何库（Google S2 Geometry）](https://s2geometry.io/)


