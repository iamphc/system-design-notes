# 第 18 章：设计谷歌地图（Google Maps）

## 引言

我们将设计一个简单版的**谷歌地图（Google Maps）**。

关于谷歌地图的一些事实：
 * 始于 2005 年
 * 提供各种服务 —— 卫星图像、街道地图、实时路况、路径规划
 * 到 2021 年，拥有 10 亿日活跃用户，覆盖全球 99% 区域，每天 2500 万次实时位置信息更新

---

## 第 1 步：理解问题并确立设计范围

候选人与面试官之间的问答示例：
 * C：我们要应对多少日活跃用户？
 * I：10 亿 DAU
 * C：我们应该聚焦哪些功能？
 * I：位置更新、导航、ETA、地图渲染
 * C：道路数据有多大？我们能拿到吗？
 * I：我们从各种来源获得道路数据，是 TB 级的原始数据
 * C：我们应该考虑路况吗？
 * I：是的，为了准确的时间估算应该考虑
 * C：不同出行方式呢 —— 步行、骑自行车、开车？
 * I：我们应该支持这些
 * C：多站点导航呢？
 * I：为控制面试范围，先不聚焦于此
 * C：商户地点和照片呢？
 * I：好问题，但不需要考虑这些

我们将聚焦三个关键功能 —— 用户位置更新、包含预计到达时间（ETA）的导航服务、地图渲染。

### **非功能需求**

- **准确性**：用户不应该得到错误的路线指引
- **流畅的导航**：用户应体验到流畅的地图渲染
- **数据和电量消耗**：客户端应尽可能少地使用数据和电量。这对移动设备很重要
- 一般的可用性和可扩展性需求

### **地图入门（Map 101）**

在进入设计之前，我们应该了解一些与地图相关的概念。

#### 定位系统

地球是一个球体，绕轴自转。位置由纬度（你在南北方向上的位置）和经度（你在东西方向上的位置）定义：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/partitioning-system.png" alt="定位系统" width="500" />
</div>

#### 从 3D 到 2D

将点从 3D 平面转换到 2D 平面的过程称为"地图投影（map projection）"。

有多种实现方式，每种都有优缺点。几乎所有投影都会扭曲真实的几何形状。

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/map-projections.png" alt="地图投影" width="500" />
</div>

谷歌地图选择了墨卡托投影的修改版本，称为"Web 墨卡托（Web Mercator）"。

#### 地理编码

地理编码（geocoding）是将地址转换为地理坐标的过程。

反向过程称为"逆地理编码（reverse geocoding）"。

实现方法之一是使用插值 —— 利用来自不同来源（如 GIS）的数据，其中街道网络被映射到地理坐标空间。

#### 地理哈希

地理哈希（geohashing）是一种编码系统，将地理区域编码为一串字母和数字。

它将世界描绘成一个平面，并递归地将其细分为四个象限：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/geohashing.png" alt="地理哈希" width="500" />
</div>

#### 地图渲染

地图渲染通过切片（tiling）进行。不是把整张地图渲染成一张大的自定义图片，而是把世界切分成更小的瓦片（tile）。

客户端只下载相关的瓦片，像拼马赛克一样把它们渲染出来。

不同缩放级别对应不同的瓦片。客户端根据自己的缩放级别选择合适的瓦片。

例如，缩小到整个世界时，只会下载一张 256x256 的瓦片，代表整个世界。

#### 面向导航算法的道路数据处理

在大多数路径规划算法中，交叉口表示为节点，道路表示为边：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/road-representation.png" alt="道路表示" width="500" />
</div>

大多数导航算法使用 Dijkstra 或 A* 算法的修改版本。

寻路性能对图的规模很敏感。为了实现大规模运行，我们不能把整个世界表示为一张图然后在上面运行算法。

相反，我们使用一种类似于切片的技术 —— 把世界细分成越来越小的图。

路径规划瓦片保存着对相邻瓦片的引用，算法在遍历相互连接的瓦片时，可以把更大的道路图拼接起来：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/routing-tiles.png" alt="路径规划瓦片" width="500" />
</div>

这项技术使我们能够显著减少内存带宽，只为给定的起点/终点对加载所需的瓦片。

然而，对于更长的路线，拼接小而详细的路径规划瓦片仍然会耗费时间和内存。因此，存在不同细节层级的路径规划瓦片，算法根据我们前往的目的地使用细节程度合适的瓦片：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/map-routing-hierarchical.png" alt="分层路径规划" width="500" />
</div>

### **粗略估算**

在存储方面，我们需要存储：
 * 世界地图 —— 根据需要存储的所有瓦片估算约 70PB，但要把非常相似的瓦片（如广袤的沙漠）的压缩考虑在内
 * 元数据 —— 大小可忽略，所以我们可以从计算中略去
 * 道路信息 —— 以路径规划瓦片的形式存储

导航请求的估算 QPS —— 10 亿 DAU，每周人均使用 35 分钟 -> 每天 50 亿分钟。
假设 GPS 更新请求是批量处理的，我们得出 20 万 QPS，峰值负载时 100 万 QPS。

---

## 第 2 步：提出高层设计并获得认可

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/high-level-design.png" alt="高层设计" width="500" />
</div>

### **位置服务**

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/location-service.png" alt="位置服务" width="500" />
</div>

它负责记录用户的位置更新：
 * 位置更新每隔 `t` 秒发送一次
 * 位置数据流可用于随时间改进服务，例如提供更准确的 ETA、监控路况数据、检测封闭道路、分析用户行为等

我们可以不在客户端一直向服务器发送位置更新，而是在客户端批量收集更新，然后按批次发送：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/location-update-batches.png" alt="批量位置更新" width="500" />
</div>

尽管做了这个优化，对于谷歌地图这种规模的系统，负载仍然很大。因此，我们可以利用为重写入优化的数据库，如 Cassandra。

我们还可以利用 Kafka 对位置更新进行高效的流处理，用于进一步分析。

位置更新请求负载示例：

```
POST /v1/locations
Parameters
  locs: JSON encoded array of (latitude, longitude, timestamp) tuples.
```

### **导航服务**

该组件负责在合理时间内在 A 地和 B 地之间找出快速路线（可以接受一点点延迟）。路线不一定是最快的，但准确性很重要。

请求负载示例：

```
GET /v1/nav?origin=1355+market+street,SF&destination=Disneyland
```

响应示例：

```json
{
  "distance": {"text":"0.2 mi", "value": 259},
  "duration": {"text": "1 min", "value": 83},
  "end_location": {"lat": 37.4038943, "Ing": -121.9410454},
  "html_instructions": "Head <b>northeast</b> on <b>Brandon St</b> toward <b>Lumin Way</b><div style=\"font-size:0.9em\">Restricted usage road</div>",
  "polyline": {"points": "_fhcFjbhgVuAwDsCal"},
  "start_location": {"lat": 37.4027165, "lng": -121.9435809},
  "geocoded_waypoints": [
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJwZNMti1fawwRO2aVVVX2yKg",
       "types" : [ "locality", "political" ]
    },
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJ3aPgQGtXawwRLYeiBMUi7bM",
       "types" : [ "locality", "political" ]
    }
  ],
  "travel_mode": "DRIVING"
}
```

路况变化和改道尚未考虑，这些将在深入设计部分处理。

### **地图渲染**

在客户端保存全部地图瓦片数据集是不可行的，因为它有 PB 级大小。

瓦片需要根据客户端的位置和缩放级别，按需从服务器获取。

什么时候应该获取新瓦片 —— 在用户放大/缩小地图时，以及在导航过程中驶向新瓦片区域时。

应该如何向客户端提供地图瓦片？
 * 可以动态构建瓦片，但这会给服务器带来巨大负载，也使缓存变得困难
 * 地图瓦片基于其地理哈希静态提供，客户端可以自己计算地理哈希。瓦片可以静态存储在 CDN 上并从 CDN 提供

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/static-map-tiles.png" alt="静态地图瓦片" width="500" />
</div>

CDN 让用户可以从最接近用户的边缘节点（POP）服务器获取地图瓦片，以最小化延迟：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/cdn-vs-no-cdn.png" alt="使用 CDN 与不使用 CDN 的对比" width="500" />
</div>

确定地图瓦片时可考虑的选项：
 * 地图瓦片的地理哈希可以在客户端计算。如果这样做，我们应该谨慎：我们要长期承诺这种瓦片计算方式，因为强制客户端升级很困难
 * 或者，我们可以提供一个简单的 API，代替客户端计算地图瓦片 URL，代价是多一次 API 调用

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/map-tile-url-calculation.png" alt="地图瓦片 URL 的计算" width="500" />
</div>

---

## 第 3 步：深入设计

### **数据模型**

我们来讨论如何存储正在处理的不同类型的数据。

#### 路径规划瓦片

初始道路数据集从不同来源获取。它会根据位置更新数据随时间不断改进。

道路数据是非结构化的。我们有一个周期性的离线处理流水线，把这些原始数据转换为应用所需的基于图的路径规划瓦片。

我们不需要数据库的任何特性，因此不把这些瓦片存入数据库，而是存储在 S3 对象存储中，同时积极地缓存它们。

我们还可以利用库将邻接表高效地压缩成二进制文件。

#### 用户位置数据

用户位置数据对于更新路况和进行各种其他分析非常有用。

由于这类数据的性质是写密集型的，我们可以使用 Cassandra 存储。

示例行：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/user-location-data-torw.png" alt="用户位置数据行" width="500" />
</div>

#### 地理编码数据库

该数据库存储经纬度坐标与地点之间的键值对。

由于读多写少，我们可以使用 Redis 以获得快速的读取访问速度。

#### 预计算的世界地图图像

如前所述，我们将预计算地图切片图像并存储在 CDN 中。

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/precomputed-map-tile-image.png" alt="预计算的地图瓦片图像" width="500" />
</div>

### **服务**

#### 位置服务

我们聚焦于该服务的数据库设计，以及用户位置的详细存储方式。

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/location-service-diagram.png" alt="位置服务架构图" width="500" />
</div>

我们可以使用 NoSQL 数据库来应对位置更新上的重写入负载。我们优先考虑可用性而非一致性，因为用户位置数据经常变化，随着新更新到来，旧数据会过时。

我们选择 Cassandra 作为数据库，因为它很好地满足了我们所有的需求。

我们要存储的示例行：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/user-location-row-example.png" alt="用户位置行示例" width="500" />
</div>

 * `user_id` 是分区键（partition key），以便快速访问某个特定用户的所有位置更新
 * `timestamp` 是聚簇键（clustering key），以便按接收位置更新的时间排序存储数据

我们还利用 Kafka 将位置更新流式传输给其他各种需要位置更新的服务：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/location-update-streaming.png" alt="位置更新流处理" width="500" />
</div>

#### 地图渲染

地图瓦片以不同缩放级别存储。在最低缩放级别，整个世界由一张 256x256 的瓦片表示。

随着缩放级别提高，地图瓦片数量变为原来的四倍：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/zoom-level-increases.png" alt="缩放级别提高" width="500" />
</div>

我们可以使用的一个优化是：不在网络上发送完整的图像信息，而是将瓦片表示为矢量（路径和多边形），让客户端动态渲染瓦片。

这将大幅节省带宽。

#### 导航服务

该服务负责寻找最快路线：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/navigation-service.png" alt="导航服务" width="500" />
</div>

我们来逐一介绍这个子系统中的每个组件。

首先，我们有地理编码服务，它把地址解析为经纬度坐标位置。

请求示例：

```
https://maps.googleapis.com/maps/api/geocode/json?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA
```

响应示例：

```json
{
   "results" : [
      {
         "formatted_address" : "1600 Amphitheatre Parkway, Mountain View, CA 94043, USA",
         "geometry" : {
            "location" : {
               "lat" : 37.4224764,
               "lng" : -122.0842499
            },
            "location_type" : "ROOFTOP",
            "viewport" : {
               "northeast" : {
                  "lat" : 37.4238253802915,
                  "lng" : -122.0829009197085
               },
               "southwest" : {
                  "lat" : 37.4211274197085,
                  "lng" : -122.0855988802915
               }
            }
         },
         "place_id" : "ChIJ2eUgeAK6j4ARbn5u_wAGqWA",
         "plus_code": {
            "compound_code": "CWC8+W5 Mountain View, California, United States",
            "global_code": "849VCWC8+W5"
         },
         "types" : [ "street_address" ]
      }
   ],
   "status" : "OK"
}
```

路径规划服务根据当前路况计算一条建议路线，针对出行时间优化。

最短路径服务在对象存储中的路径规划瓦片上运行 A* 算法的一个变体，以计算最优路径：
 * 它接收起点/终点对，将其转换为经纬度坐标对，并从这些坐标对推导地理哈希，从而得到对应的路径规划瓦片
 * 算法从初始路径规划瓦片开始，遍历它，直到找到通往目标瓦片的足够好的路径

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/shortest-path-service.png" alt="最短路径服务" width="500" />
</div>

ETA 服务由路径规划服务调用，基于机器学习算法根据路况数据预测预计到达时间。

排序器（ranker）服务负责根据用户传入的过滤器对不同的可能路径进行排序，即避开收费公路或高速公路的标志位。

更新器服务异步更新一些重要数据库，使其保持最新。

#### 改进 —— 自适应 ETA 与改道

我们可以做的一项改进是：基于新获得的路况数据，自适应地更新正在进行中的路线。

实现方法之一是：把正在某条路线上导航的用户正在经过的所有瓦片存储到数据库中。

数据可能如下所示：

```
user_1: r_1, r_2, r_3, …, r_k
user_2: r_4, r_6, r_9, …, r_n
user_3: r_2, r_8, r_9, …, r_m
...
user_n: r_2, r_10, r21, ..., r_l
```

如果某个瓦片上发生交通事故，我们可以识别出所有路径经过该瓦片的用户，并为他们重新规划路线。

为了减少数据库中存储的瓦片数量，我们可以改为存储起点路径规划瓦片以及不同分辨率层级的若干路径规划瓦片，直到把目标瓦片也包含进来：

```
user_1, r_1, super(r_1), super(super(r_1)), ...
```

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/adaptive-eta-data-storage.png" alt="自适应 ETA 数据存储" width="500" />
</div>

使用这种方式，我们只需检查用户的最终瓦片是否包含事故瓦片，就能判断该用户是否受影响。

我们还可以跟踪正在导航的用户的所有可能路线，如果有更快的改道方案可用，就通知他们。

#### 交付协议

我们有几个选项，可以让服务器主动向客户端推送数据：
 * 移动推送通知不适用，因为其负载有限，而且对 Web 应用不可用
 * WebSocket 通常是比长轮询更好的选择，因为它在服务器上的计算开销更小
 * 我们也可以使用服务器发送事件（SSE），但更倾向于 WebSocket，因为它支持双向通信，这在例如最后一公里配送功能中会很有用

---

## 第 4 步：总结

这是我们的最终设计：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/final-design.png" alt="最终设计" width="500" />
</div>

我们可以提供的另一项功能是多站点导航，可以卖给 Uber 或 Lyft 等企业客户，用于规划访问一组地点的最优路径。
