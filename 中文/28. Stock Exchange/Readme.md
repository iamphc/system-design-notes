# 第 28 章：证券交易所

## 引言
本章将设计一个**电子化证券交易所（electronic stock exchange）**。

它的基本功能是高效地撮合买家和卖家。

主要的证券交易所有 **NYSE**、**NASDAQ** 等。

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/world-stock-exchanges.png" alt="全球证券交易所" width="500" />
</div>

---

## 第 1 步：理解问题并确立设计范围
 * C：我们要交易哪些证券？股票、期权还是期货？
 * I：为简单起见，只做股票
 * C：支持哪些订单类型——下单、撤单、改单？限价单、市价单、条件单呢？
 * I：我们需要支持下单和撤单。订单类型只需考虑限价单。
 * C：系统需要支持盘后交易吗？
 * I：不需要，只做正常交易时段
 * C：你能描述一下交易所的基本功能吗？
 * I：客户可以下单或撤销限价单，并实时收到撮合成交。他们还应该能实时看到订单簿。
 * C：交易所的规模有多大？
 * I：同时有数万用户在交易，约 100 个标的（symbol）。每天数十亿笔订单。我们还需要支持合规方面的风险检查。
 * C：什么样的风险检查？
 * I：做简单的风险检查——例如限制一个用户一天内只能交易 100 万股苹果股票。
 * C：用户钱包的资金联动怎么考虑？
 * I：我们需要确保客户在下单前有足够资金。用于未完成订单的资金需要被冻结，直到订单最终确定。

### **非功能需求**
面试官提到的规模暗示我们要设计一个中小型交易所。
我们还需要保证灵活性，以便未来支持更多标的和用户。

其他非功能需求：
 * 可用性——至少 99.99%。停机时间会损害声誉
 * 容错性——需要容错能力和快速恢复机制，以限制生产事故的影响
 * 延迟——往返延迟应在毫秒级，并重点关注第 99 百分位。持续偏高的 99 分位延迟会给一小部分用户带来糟糕体验
 * 安全性——我们应该有一个账户管理系统。出于法律合规，我们需要支持 KYC 来验证用户身份。我们还应针对 DDoS 保护公共资源。

### **粗略估算**
 * 100 个标的，每天 10 亿笔订单
 * 正常交易时段为 09:30 到 16:00（6.5 小时）
 * QPS = 1bil / 6.5 / 3600 = 43000
 * 峰值 QPS = 5*QPS = 215000
 * 开盘时交易量会显著升高

---

## 第 2 步：提出高层设计并获得认可

### **业务知识入门（Business Knowledge 101）**
先来讨论一些与交易所相关的基本概念。

经纪商（broker）在交易所和终端用户之间牵线搭桥——例如 Robinhood、Fidelity 等。

机构客户使用专用交易软件进行大额交易。它们需要特殊对待。
例如大额交易时拆单，以避免冲击市场。

订单类型：
 * 限价单（Limit）——以固定价格买入或卖出。它可能不会立刻成交，也可能部分成交。
 * 市价单（Market）——不指定价格。立即按当前市场价格成交。

价格：
 * 买价（Bid）——买家愿意买入一只股票的最高价格
 * 卖价（Ask）——卖家愿意卖出一只股票的最低价格

美国市场有三层行情报价——L1、L2、L3。

L1 行情数据包含最优买/卖价和数量：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/l1-price.png" alt="L1 行情" width="500" />
</div>

L2 包含更多价格档位：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/l2-price.png" alt="L2 行情" width="500" />
</div>

L3 展示各档位以及每个档位排队的数量：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/l3-price.png" alt="L3 行情" width="500" />
</div>

K 线（candlestick）展示某个时间区间内的开盘价、收盘价以及最高价和最低价：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/candlestick.png" alt="K 线图" width="500" />
</div>

FIX 是一种用于交换证券交易信息的协议，大多数厂商都在使用。证券交易报文示例：
```
8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS | 52=20071123-05:30:00.000 | 11=ATOMNOCCC9990900 | 20=3 | 150=E | 39=E | 55=MSFT | 167=CS | 54=1 | 38=15 | 40=2 | 44=15 | 58=PHLX EQUITY TESTING | 59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 | 10=128 |
```

### **高层设计**

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/high-level-design.png" alt="高层设计" width="500" />
</div>

交易流程：
 * 客户通过交易接口下单
 * 经纪商把订单发送到交易所
 * 订单通过客户端网关进入交易所，网关负责校验、限流、认证等。订单被转发给订单管理器。
 * 订单管理器根据风险管理器设定的规则执行风险检查
 * 通过风险检查后，订单管理器核实钱包中有足够资金来支付该订单
 * 订单被发送到撮合引擎。找到撮合匹配后，撮合引擎为买方和卖方各产生一条成交（execution，也叫 fill）。两个订单都会被排序，以保证确定性。
 * 成交结果返回给客户。

行情数据流（M1-M3）：
 * 撮合引擎产生成交流，发送给行情发布器
 * 行情发布器构建 K 线图，并把它们发送给数据服务
 * 行情数据存储在专门的存储中，用于实时分析。经纪商连接到数据服务以获取及时行情。

报送流程（R1-R2）：
 * 报送器从订单和成交中收集所有必要的报文字段，并写入数据库
 * 报文字段——client_id、price、quantity、order_type、filled_quantity、remaining_quantity

交易流程位于关键路径上，而其他流程不在关键路径上，因此它们的延迟要求不同。

#### 交易流程
交易流程位于关键路径上，因此它必须为低延迟而高度优化。

撮合引擎是其核心，也称为交叉引擎（cross engine）。主要职责：
 * 维护每个标的的订单簿——一个标的的买入/卖出订单列表
 * 撮合买入和卖出订单——一次撮合产生两条成交（fill），买卖双方各一条。这个函数必须快速且准确
 * 把成交流作为行情数据分发出去
 * 撮合必须按确定性顺序产生。这是高可用的基础

接下来是排序器（sequencer）——它是让撮合引擎具有确定性的关键组件，通过为每一个入站订单和出站成交打上序列 ID。

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/sequencer.png" alt="排序器" width="500" />
</div>

我们为入站订单和出站成交打上序列，有以下几个原因：
 * 时效性与公平性
 * 快速恢复/重放
 * 恰好一次保证

从概念上讲，我们可以用 Kafka 作为排序器，因为它实际上就是一个入站和出站消息队列。但为了实现更低的延迟，我们打算自己实现它。

订单管理器管理订单状态。它还与撮合引擎交互——发送订单、接收成交。

订单管理器的职责：
 * 把订单送去做风险检查——例如核实用户的交易量小于 100 万
 * 对照用户钱包检查订单，确认有足够资金执行
 * 它把订单发送给排序器，再送往撮合引擎。为减少带宽，只有必要的订单信息被传给撮合引擎
 * 成交（fill）从排序器接收回来，然后通过客户端网关发送给各经纪商

实现订单管理器的主要挑战是状态流转管理。事件溯源是一种可行方案（在深入设计中讨论）。

最后，客户端网关接收来自用户的订单，并把它们发送给订单管理器。它的职责：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/client-gateway.png" alt="客户端网关" width="500" />
</div>

由于客户端网关位于关键路径上，它应该保持轻量。

不同客户可以有多个客户端网关。例如 colo 引擎是一台交易引擎服务器，由经纪商租用在交易所的数据中心内：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/client-gateways.png" alt="多个客户端网关" width="500" />
</div>

#### 行情数据流
行情发布器从撮合引擎接收成交，并基于成交流构建订单簿/K 线图。

这些数据被发送给数据服务，数据服务负责向订阅者展示聚合后的数据：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/market-data.png" alt="行情数据" width="500" />
</div>

#### 报送流程
报送器不在关键路径上，但仍然是一个重要组件。

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/reporting-flow.png" alt="报送流程" width="500" />
</div>

它负责交易历史、税务报送、合规报送、结算等。报送流程对延迟没有严格要求，准确性和合规性更重要。

### **API 设计**
客户通过经纪商与证券交易所交互，用于下单、查看成交、行情数据、下载历史数据做分析等。

客户端网关与经纪商之间使用 RESTful API 通信。

对于机构客户，使用私有协议来满足其低延迟要求。

创建订单：
```
POST /v1/order
```

参数：
 * symbol——股票标的。String
 * side——买入或卖出。String
 * price——限价单的价格。Long
 * orderType——限价或市价（我们的设计只支持限价单）。String
 * quantity——订单数量。Long

响应：
 * id——订单的 ID。Long
 * creationTime——订单的系统创建时间。Long
 * filledQuantity——已成功执行的数量。Long
 * remainingQuantity——尚待执行的数量。Long
 * status——new/canceled/filled。String
 * 其余属性与输入参数相同

获取成交：
```
GET /execution?symbol={:symbol}&orderId={:orderId}&startTime={:startTime}&endTime={:endTime}
```

参数：
 * symbol——股票标的。String
 * orderId——订单的 ID。可选。String
 * startTime——查询起始时间，epoch [11]。Long
 * endTime——查询结束时间，epoch。Long

响应：
 * executions——数组，包含范围内的每条成交（属性见下）。Array
 * id——成交的 ID。Long
 * orderId——订单的 ID。Long
 * symbol——股票标的。String
 * side——买入或卖出。String
 * price——成交价。Long
 * orderType——限价或市价。String
 * quantity——成交量。Long

获取订单簿：
```
GET /marketdata/orderBook/L2?symbol={:symbol}&depth={:depth}
```

参数：
 * symbol——股票标的。String
 * depth——每侧的订单簿深度。Int

响应：
 * bids——数组，包含价格和挂单量。Array
 * asks——数组，包含价格和挂单量。Array

获取 K 线：
```
GET /marketdata/candles?symbol={:symbol}&resolution={:resolution}&startTime={:startTime}&endTime={:endTime}
```

参数：
 * symbol——股票标的。String
 * resolution——K 线窗口长度，单位秒。Long
 * startTime——窗口开始时间，epoch。Long
 * endTime——窗口结束时间，epoch。Long

响应：
 * candles——数组，包含每条 K 线数据（属性见下）。Array
 * open——每条 K 线的开盘价。Double
 * close——每条 K 线的收盘价。Double
 * high——每条 K 线的最高价。Double
 * low——每条 K 线的最低价。Double

### **数据模型**
我们的交易所中有三类主要数据：
 * 产品、订单、成交
 * 订单簿
 * K 线图

#### 产品、订单、成交
产品描述一个交易标的的属性——产品类型、交易代码、UI 展示代码等。

这类数据不常变化，主要用于 UI 渲染。

订单表示一条买入/卖出指令。成交是出站的撮合结果。

数据模型如下：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/product-order-execution-data-model.png" alt="产品-订单-成交数据模型" width="500" />
</div>

我们在三个流程中都会遇到订单和成交：
 * 在关键路径上，它们在内存中处理以获得高性能。它们被持久化到排序器中并可从中恢复。
 * 报送器把订单和成交写入数据库，用于报送场景
 * 成交被转发给行情数据侧，用于重建订单簿和 K 线图

#### 订单簿
订单簿是一个交易工具的买入/卖出订单列表，按价格档位组织。

一个高效的数据结构需要满足：
 * 常数时间查找——获取某价格档位或价格档位之间的挂单量
 * 快速的添加/成交/取消操作
 * 查询最优买/卖价
 * 遍历各价格档位

订单簿执行示例：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/order-book-execution.png" alt="订单簿执行示例" width="500" />
</div>

完成这笔大额订单后，由于买卖价差扩大，价格上升。

订单簿实现的伪代码示例：
```
class PriceLevel{
    private Price limitPrice;
    private long totalVolume;
    private List<Order> orders;
}

class Book<Side> {
    private Side side;
    private Map<Price, PriceLevel> limitMap;
}

class OrderBook {
    private Book<Buy> buyBook;
    private Book<Sell> sellBook;
    private PriceLevel bestBid;
    private PriceLevel bestOffer;
    private Map<OrderID, Order> orderMap;
}
```

为了更高效地实现，我们可以用双向链表代替标准列表：
 * 下新订单是 O(1)，因为我们把订单加到列表尾部。
 * 撮合订单是 O(1)，因为我们从头部删除订单。
 * 取消订单意味着从订单簿中删除订单。我们利用 `orderMap` 实现 O(1) 查找和 O(1) 删除（因为 `Order` 持有列表中前一个元素的引用）。

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/order-book-impl.png" alt="订单簿实现" width="500" />
</div>

这个数据结构也被用在行情数据服务中，用于重建订单簿。

#### K 线图
K 线数据是在行情数据服务中，基于一个时间区间内处理的订单计算出来的：
```
class Candlestick {
    private long openPrice;
    private long closePrice;
    private long highPrice;
    private long lowPrice;
    private long volume;
    private long timestamp;
    private int interval;
}

class CandlestickChart {
    private LinkedList<Candlestick> sticks;
}
```

一些避免占用过多内存的优化：
 * 使用预分配的环形缓冲区持有 K 线，以减少分配次数
 * 限制内存中的 K 线数量，其余持久化到磁盘

我们将使用内存列式数据库（例如 KDB）做实时分析。收盘后，数据持久化到历史数据库。

---

## 第 3 步：深入设计
关于现代交易所，有一个值得注意的有趣事实：与大多数其他软件不同，它们通常把所有组件都跑在一台巨型服务器上。

我们来深入探讨细节。

### **性能**
对交易所而言，所有百分位上的整体延迟都很重要。

如何降低延迟？
 * 减少关键路径上的任务数量
 * 通过减少网络/磁盘使用和/或缩短任务执行时间，来缩短每个任务耗时

为了实现第一个目标，我们把关键路径上所有无关的职责都剥离掉，甚至连日志都移除，以获得最优延迟。

如果沿用最初的设计，会有几个瓶颈——服务间的网络延迟和排序器的磁盘使用。

这样的设计我们只能做到几十毫秒的端到端延迟，而我们想要的是几十微秒。

因此，我们把所有组件放在一台服务器上，进程之间通过 mmap 作为事件存储来通信：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/mmap-bus.png" alt="mmap 总线" width="500" />
</div>

另一个优化是使用应用主循环（while 循环执行关键任务），并把它绑定到同一个 CPU 上，避免上下文切换：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/application-loop.png" alt="应用主循环" width="500" />
</div>

使用应用主循环的另一个附带好处是没有锁竞争——即多个线程争抢同一资源。

现在来看看 mmap 的工作原理——它是一个 UNIX 系统调用，把磁盘上的文件映射到应用的内存中。

一个小技巧是把文件创建在 `/dev/shm` 中，它代表「共享内存」。这样我们就完全没有磁盘访问了。

### **事件溯源**
事件溯源在[数字钱包章节](../chapter28)中有深入讨论。所有细节请参考该章。

简而言之，我们不存储当前状态，而是存储不可变的状态流转：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/event-sourcing.png" alt="事件溯源" width="500" />
</div>

 * 左侧——传统模式
 * 右侧——事件溯源模式

到目前为止，我们的设计如下：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/design-so-far.png" alt="迄今的设计" width="500" />
</div>

 * 外部域通过 FIX 协议与我们的客户端网关交互
 * 订单管理器接收新订单事件，校验它，并把它加入内部状态。然后订单被发送到撮合核心。
 * 如果订单被撮合，`OrderFilledEvent` 被生成并通过 mmap 发送
 * 其他组件订阅事件存储，各自完成自己的处理

另一个优化——所有组件都持有一份订单管理器的副本，订单管理器被打包成一个库，以避免管理订单时的额外调用。

在这个设计中，排序器不再是事件存储，而是一个单写者，在把事件转发到事件存储之前先对其排序：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/sequencer-deep-dive.png" alt="深入排序器" width="500" />
</div>

### **高可用性**
我们的目标是 99.99% 可用性——每天只有 8.64 秒停机时间。

为实现这一点，我们必须识别交易所架构中的单点故障：
 * 为关键服务（例如撮合引擎）设置处于待命状态的备用实例
 * 大力推进故障检测自动化和到备用实例的故障转移

无状态服务（如客户端网关）可以通过增加服务器轻松地水平扩展。

对于有状态组件，如果我们不是 leader，就可以处理入站事件，但不能发布出站事件：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/leader-election.png" alt="leader 选举" width="500" />
</div>

为检测主副本是否宕机，我们可以发送心跳来检测它是否失效。

这个机制只在单台服务器的边界内有效。如果想扩展，可以把整台服务器设置为热/温副本，并在故障时故障转移。

为了在副本之间复制事件存储，我们可以使用可靠 UDP 以获得更快的通信。

### **容错性**
如果连温备实例都宕机了怎么办？这是低概率事件，但我们应该做好准备。

大型科技公司通过把核心数据复制到多个城市的数据中心来应对这个问题，以缓解例如自然灾害的影响。

需要考虑的问题：
 * 如果主实例宕机，我们如何以及何时故障转移到备用实例？
 * 我们如何在备用实例中选择 leader？
 * 需要多长的恢复时间（RTO——恢复时间目标）？
 * 需要恢复哪些功能？我们的系统能否在降级状态下运行？

如何应对这些问题：
 * 系统可能因 bug 而宕机（影响主实例和副本），我们可以使用混沌工程来暴露这类边缘情况和灾难性后果
 * 不过一开始，我们可以手动执行故障转移，直到积累了足够多关于系统故障模式的知识
 * 可以使用 leader 选举（例如 Raft）来确定主实例宕机后由哪个副本成为 leader

跨不同服务器的复制工作方式示例：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/replication-across-servers.png" alt="跨服务器复制" width="500" />
</div>

leader 选举术语示例：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/leader-election-terms.png" alt="leader 选举术语" width="500" />
</div>

关于 Raft 的工作原理详情，[点这里查看](https://thesecretlivesofdata.com/raft/)

最后，我们还需要考虑丢失容忍度——在情况变得危急之前我们能丢失多少数据？这将决定我们多久备份一次数据。

对证券交易所而言，数据丢失是不可接受的，因此我们必须频繁备份数据，并依靠 Raft 的复制来降低数据丢失概率。

### **撮合算法**
稍作偏离，用伪代码看一下撮合如何工作：
```
Context handleOrder(OrderBook orderBook, OrderEvent orderEvent) {
    if (orderEvent.getSequenceId() != nextSequence) {
        return Error(OUT_OF_ORDER, nextSequence);
    }

    if (!validateOrder(symbol, price, quantity)) {
        return ERROR(INVALID_ORDER, orderEvent);
    }

    Order order = createOrderFromEvent(orderEvent);
    switch (msgType):
        case NEW:
            return handleNew(orderBook, order);
        case CANCEL:
            return handleCancel(orderBook, order);
        default:
            return ERROR(INVALID_MSG_TYPE, msgType);

}

Context handleNew(OrderBook orderBook, Order order) {
    if (BUY.equals(order.side)) {
        return match(orderBook.sellBook, order);
    } else {
        return match(orderBook.buyBook, order);
    }
}

Context handleCancel(OrderBook orderBook, Order order) {
    if (!orderBook.orderMap.contains(order.orderId)) {
        return ERROR(CANNOT_CANCEL_ALREADY_MATCHED, order);
    }

    removeOrder(order);
    setOrderStatus(order, CANCELED);
    return SUCCESS(CANCEL_SUCCESS, order);
}

Context match(OrderBook book, Order order) {
    Quantity leavesQuantity = order.quantity - order.matchedQuantity;
    Iterator<Order> limitIter = book.limitMap.get(order.price).orders;
    while (limitIter.hasNext() && leavesQuantity > 0) {
        Quantity matched = min(limitIter.next.quantity, order.quantity);
        order.matchedQuantity += matched;
        leavesQuantity = order.quantity - order.matchedQuantity;
        remove(limitIter.next);
        generateMatchedFill();
    }
    return SUCCESS(MATCH_SUCCESS, order);
}
```

这个撮合算法使用 FIFO 算法来决定同一价格档位中先撮合哪些订单。

### **确定性**
功能确定性通过我们使用的排序器技术来保证。

事件实际发生的时间并不重要：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/determinism.png" alt="确定性" width="500" />
</div>

延迟确定性则是我们必须跟踪的。我们可以通过监控第 99 或 99.99 百分位延迟来计算它。

可能导致延迟尖峰的事情，例如 Java 中的垃圾回收事件。

### **行情发布器优化**
行情发布器从撮合引擎接收撮合结果，并基于它们重建订单簿和 K 线图。

由于内存有限，我们只保留一部分 K 线。客户可以选择想要多细粒度的信息：更细粒度的信息可能需要更高的价格：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/market-data-publisher.png" alt="行情发布器" width="500" />
</div>

环形缓冲区（ring buffer，也称循环队列）是一个固定大小队列，头尾相连。空间预分配以避免运行时分配。该数据结构也是无锁的。

优化环形缓冲区的另一个技巧是填充（padding），它确保序列号永远不会与其他数据落在同一个缓存行中。

### **行情分发公平性与组播**
我们需要确保订阅者同时收到数据，因为如果一个订阅者比另一个先收到数据，它就获得了关键的市场情报，并可能用来操纵市场。

为实现这一点，我们在向订阅者发布数据时，可以使用基于可靠 UDP 的组播（multicast）。

数据在互联网上的传输方式有三种：
 * 单播（Unicast）——一个源，一个目的地
 * 广播（Broadcast）——一个源，发到整个子网
 * 组播（Multicast）——一个源，发到不同子网上的一组主机

理论上，使用组播后，所有订阅者应该同时收到数据。

然而 UDP 不可靠，数据可能无法送达所有人。不过可以通过重传来增强它。

### **主机托管（Colocation）**
交易所为经纪商提供在交易所同一数据中心托管其服务器的服务。

这能大幅降低延迟，可以被视为一项 VIP 服务。

### **网络安全**
由于有一些面向公网的服务，DDoS 对交易所是一个挑战。我们的选项：
 * 把公共服务和数据与私有服务隔离，使 DDoS 攻击不会影响最重要的客户
 * 使用缓存层存储不常更新的数据
 * 加固 URL 以防 DDoS，例如优先使用 `https://my.website.com/data/recent` 而不是 `https://my.website.com/data?from=123&to=456`，因为前者更容易被缓存
 * 需要有效的白名单/黑名单机制
 * 可以使用限流来缓解 DDoS

---

## 第 4 步：总结
其他有趣的备注：
 * 并非所有交易所都把所有东西放在一台大服务器上，但确实有一些仍然这样做
 * 现代交易所更多地依赖云基础设施，也依赖自动做市商（AMM），以避免维护订单簿
