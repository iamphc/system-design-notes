# 第 10 章：设计通知系统

## 引言
**通知系统（notification system）** 对现代应用至关重要，它能提供及时的更新，例如商品通知、活动、优惠和提醒。通知可以通过以下方式发送：
1. **推送通知（push notification）**（移动端或桌面端），
2. **短信（SMS）**，以及
3. **电子邮件（Email）**。

本章聚焦于设计一个可扩展的系统，使其能够每天发送数百万条通知。

---

## 第 1 步：理解问题
### 需求
- **通知类型：** 推送通知、短信和电子邮件。
- **投递：** 软实时系统，延迟极低。
- **平台：** iOS、Android 和桌面端。
- **触发方式：** 通知可以由客户端应用触发，也可以在服务器上定时调度。
- **规模：**
  - **推送通知：** 1000 万条/天，
  - **短信：** 100 万条/天，
  - **电子邮件：** 500 万条/天。
- **退订支持：** 用户可以禁用特定类型的通知。

---

## 第 2 步：高层设计

### 组件

1. **通知类型：**
   - **iOS 推送通知：** 使用 **Apple 推送通知服务（APNS，Apple Push Notification Service）**。
   - **Android 推送通知：** 使用 **Firebase 云消息服务（FCM，Firebase Cloud Messaging）**。
   - **短信：** 使用 Twilio 或 Nexmo 等第三方服务。
   - **电子邮件：** 使用 SendGrid 或 Mailchimp 等商业邮件服务。

2. **联系信息采集：**
   <div style="margin-left:3rem">
      <img src="../../10.%20Notification%20System/images/contact-info-gathering.png" alt="联系信息采集" width="500">
   </div>

   - 在应用安装或注册时采集设备令牌（device token）、电话号码或电子邮件地址。
   - 将联系信息存入数据库：
     - **设备令牌表（Device Tokens Table）：** 用于推送通知。
     - **用户表（User Table）：** 用于电子邮件和电话号码。


3. **通知发送流程：**

   <div style="margin-left:3rem">
      <img src="../../10.%20Notification%20System/images/high-level-design.png" alt="高层设计" width="500">
   </div>

   - **触发服务（Trigger Services）：**
      - 生成事件以发起通知（例如账单提醒、物流更新）。
      - 一个服务可以是微服务、定时任务（cron job），或一个触发通知发送事件的分布式系统。
   - **通知服务器（Notification Server）：** 
      - 提供 API 供各服务发送通知。 
      - 执行基本校验，以验证电子邮件和电话号码。
      - 查询数据库或缓存，获取渲染一条通知所需的数据。
   - **第三方服务：** 将通知投递给用户。

     

### 初始设计中的挑战
- **单点故障（SPOF）：** 一台通知服务器崩溃就可能拖垮整个系统。
- **可扩展性问题：** 难以独立地扩展数据库、缓存和处理组件。
- **性能瓶颈：** 发送通知对资源的需求很高。

### 改进后的设计

   <div style="margin-left:3rem">
      <img src="../../10.%20Notification%20System/images/improved-design.png" alt="改进后的设计" width="500">
   </div>

- 将数据库和缓存移出通知服务器。
- 引入**水平扩展（horizontal scaling）**，部署多台通知服务器。
- 使用**消息队列（message queue）**来解耦系统组件。
   - 当需要大批量发送通知时，消息队列充当缓冲区。
- 增加工作进程（worker），从消息队列中拉取通知事件，并将其发送到相应的第三方服务。

   

---

## 第 3 步：深入设计

### 可靠性
1. **防止数据丢失：** 
   <div style="margin-left:3rem">
   <img src="../../10.%20Notification%20System/images/data-loss.png" alt="数据丢失" width="400">
   </div>

   - 将通知数据持久化到数据库，并实现重试机制。 
   - 通知日志数据库（Notification log database）用于数据持久化。


2. **去重（Deduplication）：** 
   - 检查事件 ID，避免发送重复通知。
   - 当一个通知事件首次到达时，通过检查事件 ID 判断它是否之前已见过。若已见过则丢弃，否则发出该通知。 


### 附加组件
   <div style="margin-left:3rem">
   <img src="../../10.%20Notification%20System/images/events-tracking.png" alt="事件追踪" width="400">
   </div>

1. **通知模板（Notification Templates）：** 预先格式化好的模板，用于一致且高效地发送通知。
2. **通知设置（Notification Settings）：**
   - 用户可以选择开启或关闭特定渠道（推送、短信或电子邮件）的通知。
   - 存储在专门的通知设置表中。
3. **限流（Rate Limiting）：** 限制发送给用户的通知频率。
4. **重试机制（Retry Mechanism）：** 当第三方服务失败时重试发送通知。
5. **监控队列（Monitoring Queues）：** 跟踪队列中的通知，以动态扩展工作进程。
6. **事件追踪（Event Tracking）：** 采集打开率、点击率和参与度等指标。


### 安全性
- 使用 **AppKey** 和 **AppSecret** 进行认证，并保护推送通知的 API。

### 通知流程

   <div style="margin-left:3rem">
   <img src="../../10.%20Notification%20System/images/updated-design.png" alt="更新后的设计" width="500">
   </div>

1. 触发服务调用 API 发送通知。
2. 通知服务器校验请求，并从缓存或数据库中获取元数据。
3. 通知事件被发送到消息队列。
4. 工作进程处理事件，并与第三方服务交互。
5. 第三方服务将通知投递给用户。


---

## 关键优化
1. **水平扩展：** 增加更多通知服务器以分担负载。
2. **消息队列：** 解耦处理过程，以应对高并发量。
3. **缓存：** 缓存频繁访问的数据，以降低延迟。
4. **分布式爬取：** 在地理上优化消息投递，以获得更好的性能。

  
