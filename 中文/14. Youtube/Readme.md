# 第 14 章：设计 YouTube

## 引言
YouTube 是一个大型视频流媒体平台，支持视频上传、播放以及各种互动。本章聚焦于设计一个可扩展的视频流媒体系统，它具备以下核心特性：
- **快速的视频上传**
- **流畅的视频流媒体播放**
- **能够切换视频画质**
- **低廉的基础设施成本**
- **高可用性与可靠性**

### 关键数据（2020 年）
- **20 亿月活跃用户**
- **每天观看 50 亿个视频**
- **移动互联网流量的 37% 来自 YouTube**
- 提供 **80 种语言**
- 2019 年广告收入达 **151 亿美元**

---

## 第 1 步：理解问题与范围

### 核心功能
1. 上传视频
2. 观看视频

### 支持的平台
- 移动应用、网页浏览器和智能电视

### 假设
- **日活跃用户（DAU）：** 500 万
- **平均视频大小：** 300 MB
- **上传限制：** 每个视频最大 1 GB
- **每日存储需求：** 150 TB
- **CDN 成本：** 500 万 * 5 个视频 * 0.3GB * $0.02 = $150,000/天（使用 Amazon CloudFront）

---

## 第 2 步：高层设计

### 组件

<div style="margin-left:3rem">
    <img src="../../14.%20Youtube/images/high-level-design.png" alt="高层设计" width="400">
</div>

1. **客户端（Client）：** 智能手机、电脑和电视等设备。
2. **CDN（内容分发网络）：** 存储并流媒体传输视频。
3. **API 服务器（API Servers）：** 处理除视频流媒体传输之外的所有用户交互（例如上传、元数据更新）。
4. **元数据数据库（Metadata Database）：** 存储视频元数据（例如标题、描述、大小）。
5. **原始存储（Original Storage）：** 用于已上传视频的 blob 存储。
6. **转码服务器（Transcoding Servers）：** 将视频转换为多种分辨率和格式。
7. **转码后存储（Transcoded Storage）：** 用于转码后视频的 blob 存储。


---

### 核心工作流程
#### 1. 视频上传流程
- **并行流程：**
  1. 将视频上传到原始存储。
  2. 在数据库中更新视频元数据。

- **视频上传（步骤）：**

    <div style="margin-left:3rem">
        <img src="../../14.%20Youtube/images/video-uploading-flow.png" alt="视频上传流程" width="500">
    </div>

    - [1] 视频被上传到 blob 存储。 
    - [2] 转码服务器将视频转换为多种格式。
    - [3] 一旦转码完成，以下两步并行执行。
        - [3a] 转码后的视频被发送到转码后存储。
        - [3b] 转码完成事件被放入完成队列（completion queue）。 
    - [3a.1] 视频被分发到 CDN。 
    - [3b.1] 完成处理器（completion handler）更新元数据并通知用户。
    


- **元数据上传（步骤）：**

    <div style="margin-left:3rem">
        <img src="../../14.%20Youtube/images/metadata-upload.png" alt="元数据上传" height="500">
    </div>

    - 客户端并行发送一个更新视频元数据的请求。 
    - 该请求包含视频元数据，包括文件名、大小、格式等。
    
       


#### 2. 视频流媒体播放流程

<div style="margin-left: 3em;">
  <img src="../../14.%20Youtube/images/video-streaming-flow.png" alt="视频流媒体播放流程" height="400">
</div>

- 视频使用边缘服务器（edge server）直接从 CDN 流媒体传输，以最小化延迟。
- 一些流行的流媒体协议有 MPEG-DASH、Apple HLS、Adobe HDS。
-  *不同的流媒体协议支持不同的视频编码和播放播放器。*


---

## 第 3 步：深入设计

### 视频转码
#### 重要性
1. 原始视频占用大量存储空间。它可减少存储空间占用。
2. 确保跨设备和浏览器的兼容性。
3. 使视频画质能根据网络状况自适应。

#### 组成
- **容器（Container）：** 封装视频、音频和元数据（例如 MP4、AVI）。
- **编解码器（Codecs）：** 压缩与解压缩算法（例如 H.264、VP9）。

#### 有向无环图（DAG）模型
<div style="margin-left: 3em;">
    <img src="../../14.%20Youtube/images/dag-video-transcoding.png" alt="DAG 视频转码" width="600">
</div>

- 转码一段视频在计算上代价高昂且耗时。
- DAG 模型定义了诸如编码、缩略图生成和水印等任务。
- 允许视频处理高度并行。


- 原始视频被拆分为视频、音频和元数据。 
    - 视频编码：视频被转换以支持不同的分辨率、编解码器和码率。
    - 缩略图：它既可以由用户上传，也可以由系统自动生成。
    - 水印：叠加在视频之上的图像，包含关于该视频的识别信息。

---

### 视频转码架构

<div style="margin-left: 3em;">
<img src="../../14.%20Youtube/images/video-transcoding-architecture.png" alt="视频转码" width="600">
</div>

1. **预处理器（Preprocessor）：** 将视频拆分为更小的数据块（按 GOP 对齐）。它有 4 项职责。

    <div style="margin-left: 3em;">
        <img src="../../14.%20Youtube/images/dag-config.png" alt="DAG 配置" width="500">
    </div>

    - 视频切分：视频流被切分或进一步切分为更小的图像组（GOP，Group of Pictures）对齐单元。
    - 它按 GOP 对齐切分视频，以兼容旧客户端。
    - 它根据客户端程序员编写的配置文件生成 DAG。 
    - 它将 GOP 和元数据存储在临时存储中，以防编码失败时，系统可以使用已持久化的数据进行重试。


2. **DAG 调度器（DAG Scheduler）：** 将任务组织成顺序或并行的阶段。
    <div style="margin-left: 3em;">
        <img src="../../14.%20Youtube/images/dag-scheduler.png" alt="DAG 调度器" width="500">
    </div>

    - 它将一个 DAG 图拆分为若干任务阶段，并把它们放入资源管理器中的任务队列。 
    - 阶段 1：视频、音频和元数据。
    - 视频文件在阶段 2 中进一步拆分为两个任务：视频编码和缩略图。 


3. **资源管理器（Resource Manager）：** 负责管理资源分配的效率。它包含 3 个队列和一个任务调度器。
    <div style="margin-left: 3em;">
        <img src="../../14.%20Youtube/images/resource-manager.png" alt="资源管理器" width="700">
    </div>

    - 任务队列：包含待执行任务的优先级队列。
    - 工作进程队列：包含工作进程利用率信息的优先级队列。
    - 运行队列：包含当前正在运行的任务以及运行这些任务的工作进程。
    - 任务调度器：挑选最优的任务/工作进程，并指示选中的工作进程执行该作业。


4. **任务工作进程（Task Workers）：** 执行转码及其他操作。
    <div style="margin-left: 3em;">
        <img src="../../14.%20Youtube/images/task-worker.png" alt="任务工作进程" width="250">
   </div>

    - 不同的任务工作进程可能运行不同的任务。 


5. **临时存储（Temporary Storage）：** 存储用于重试的中间数据。
    - 存储系统的选择取决于数据类型、数据大小、访问频率、数据生命周期等因素。 
6. **输出（Output）：** 可用于分发的转码后视频。


---

## 系统优化

### 速度优化
1. **并行视频上传：** 将视频拆分为更小的数据块，以实现更快、可断点续传的上传。

    <img src="../../14.%20Youtube/images/video-split.png" alt="视频切分" width="600">

2. **分布式上传中心：** 将 CDN 用作靠近用户的上传枢纽。
3. **并行处理：** 使用消息队列解耦各模块，以实现高度并行。

    <img src="../../14.%20Youtube/images/message-queue1.png" alt="消息队列" width="600">
    <img src="../../14.%20Youtube/images/message-queue2.png" alt="消息队列" height="170" width="500">

### 安全性优化
1. **预签名 URL（Pre-Signed URLs）：** 将视频上传限制为授权用户。

    <img src="../../14.%20Youtube/images/pres-signed-urls.png" alt="预签名 URL" width="500">

2. **保护视频：**
   - **DRM 系统**（例如 Apple FairPlay、Google Widevine）。
   - **AES 加密。**
   - **水印。**

### 成本节约优化
1. 仅通过 CDN 分发热门视频；较冷门的视频从大容量服务器提供。
2. 对极少访问的视频按需编码。
3. 根据热度对视频分发进行区域化。
4. 自建 CDN 并与 ISP 合作，以降低带宽成本。

---

## 错误处理
### 可恢复错误
- 重试失败的上传、转码或资源分配任务。

### 不可恢复错误
- 停止处理格式错误的视频，并返回错误码。

