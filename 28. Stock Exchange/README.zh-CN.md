# 第 28 章：股票交易所

## 简介
本章我们将设计一个**电子股票交易所**。

其基本功能是高效地撮合买方和卖方。

主要的股票交易所有 **NYSE**、**NASDAQ** 等。

<div style="margin-left:3rem">
    <img src="./images/world-stock-exchanges.png" alt="全球股票交易所" width="500" />
</div>

---

## 步骤 1：理解问题并确定设计范围
 * C：我们将交易哪些证券？股票、期权还是期货？
 * I：为了简单起见，只交易股票。
 * C：支持哪些订单类型 - 下单、撤单、改单？那限价单、市价单、条件单呢？
 * I：我们需要支持下单和撤单。订单类型只需要考虑限价单。
 * C：系统需要支持盘后交易吗？
 * I：不需要，只支持正常交易时段。
 * C：你能描述一下交易所的基本功能吗？
 * I：客户可以下达或取消限价单，并实时收到成交回报。他们应该能够实时查看订单簿。
 * C：交易所的规模有多大？
 * I：数万名用户同时交易，约 100 个交易品种（symbol）。每天数十亿订单。我们还需要支持合规性风险检查。
 * C：什么样的风险检查？
 * I：我们就做简单的风险检查 - 例如限制用户一天最多交易 100 万股苹果股票。
 * C：用户钱包呢？
 * I：在下单前我们需要确保客户有足够的资金。用于未完成订单的资金需要被冻结，直到订单最终确定。

### **非功能性需求**
面试官提到的规模暗示我们设计的是一个中小规模的交易所。
我们还需要确保灵活性，以便未来支持更多交易品种和用户。

其他非功能性需求：
 * 可用性 - 至少 99.99%。停机可能会损害声誉
 * 容错性 - 需要容错和快速恢复机制，以限制生产事故的影响
 * 延迟 - 端到端往返延迟应该在毫秒级别，重点关注 99 百分位数。持续较高的 99 百分位延迟会导致少数用户体验不佳
 * 安全性 - 我们应该有一个账户管理系统。出于法律合规要求，我们需要支持 KYC 以验证用户身份。我们还应该针对公共资源防范 DDoS

### **粗略估算**
 * 100 个交易品种，每天 10 亿订单
 * 正常交易时段为 09:30 至 16:00（6.5 小时）
 * QPS = 10亿 / 6.5 / 3600 = 43000
 * 峰值 QPS = 5 * QPS = 215000
 * 市场开盘时交易量显著更高

---

## 步骤 2：提出高层级设计并获得认可

### **业务知识 101**
让我们讨论一些与交易所有关的基本概念。

经纪商（broker）在交易所和终端用户之间进行中介 - 例如 Robinhood、Fidelity 等。

机构客户使用专门的交易软件进行大宗交易。他们需要特殊的对待。
例如，在大批量交易时进行订单拆分，以避免影响市场。

订单类型：
 * 限价单（Limit） - 以固定价格买入或卖出。可能不会立即找到匹配，也可能只被部分撮合。
 * 市价单（Market） - 不指定价格。立即以当前市场价格成交。

价格：
 * 买价（Bid） - 买方愿意买入股票的最高价格
 * 卖价（Ask） - 卖方愿意卖出股票的最低价格

美国市场有三层行情报价 - L1、L2、L3。

L1 市场数据包含最优买价/卖价和数量：

<div style="margin-left:3rem">
    <img src="./images/l1-price.png" alt="L1 行情" width="500" />
</div>

L2 包含更多价格档位：

<div style="margin-left:3rem">
    <img src="./images/l2-price.png" alt="L2 行情" width="500" />
</div>

L3 显示每个档位的挂单量和排队量：

<div style="margin-left:3rem">
    <img src="./images/l3-price.png" alt="L3 行情" width="500" />
</div>

K 线图（蜡烛图）显示市场的开盘价和收盘价，以及给定区间内的最高价和最低价：

<div style="margin-left:3rem">
    <img src="./images/candlestick.png" alt="K 线图" width="500" />
</div>

FIX 是一种用于交换证券交易信息的协议，被大多数厂商使用。示例证券交易：
```
8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS | 52=20071123-05:30:00.000 | 11=ATOMNOCCC9990900 | 20=3 | 150=E | 39=E | 55=MSFT | 167=CS | 54=1 | 38=15 | 40=2 | 44=15 | 58=PHLX EQUITY TESTING | 59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 | 10=128 |
```

### **高层级设计**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="高层级设计" width="500" />
</div>

交易流程：
 * 客户通过交易界面下达订单
 * 经纪商将订单发送到交易所
 * 订单通过客户端网关（client gateway）进入交易所，网关进行验证、限流、身份认证等。订单被转发给订单管理器（order manager）。
 * 订单管理器根据风险管理器（risk manager）设定的规则执行风险检查
 * 通过风险检查后，订单管理器验证钱包中是否有足够的资金来执行订单
 * 订单被发送到撮合引擎。当找到匹配时，撮合引擎为买方和卖方各产生一次执行结果（称为成交，fill）。两个订单都被排序，以保证确定性。
 * 执行结果返回给客户端。

市场数据流程（M1-M3）：
 * 撮合引擎生成执行结果流，发送给市场数据发布者（market data publisher）
 * 市场数据发布者构建 K 线图并将其发送给数据服务（data service）
 * 市场数据存储在专门的存储中，用于实时分析。经纪商连接到数据服务以获取及时的市场数据。

报告流程（R1-R2）：
 * 报告器（reporter）从订单和执行结果中收集所有必要的报告字段，并写入数据库
 * 报告字段 - client_id、price、quantity、order_type、filled_quantity、remaining_quantity

交易流程位于关键路径（critical path）上，而其他流程不在，因此它们之间的延迟要求有所不同。

#### 交易流程
交易流程位于关键路径上，因此应针对低延迟进行高度优化。

撮合引擎是它的核心，也被称为交叉引擎（cross engine）。主要职责：
 * 维护每个交易品种的订单簿 - 某个交易品种的买单/卖单列表。
 * 撮合买单和卖单 - 一次撮合产生两个执行结果（成交），买入方和卖出方各一个。该功能必须快速且准确
 * 将执行结果流作为市场数据分发
 * 撮合结果必须以确定性顺序生成。这是高可用性的基础

接下来是排序器（sequencer） - 它是使撮合引擎具备确定性的关键组件，通过为每个入站订单和出站成交盖上序列 ID 实现：

<div style="margin-left:3rem">
    <img src="./images/sequencer.png" alt="排序器" width="500" />
</div>

我们为入站订单和出站成交盖章有几个原因：
 * 及时性和公平性
 * 快速恢复/重放
 * 精确一次（exactly-once）保证

从概念上讲，我们可以使用 Kafka 作为我们的排序器，因为它本质上是一个入站和出站消息队列。然而，我们将自己实现它，以实现更低的延迟。

订单管理器管理订单状态。它还与撮合引擎交互 - 发送订单并接收成交结果。

订单管理器的职责：
 * 发送订单进行风险检查 - 例如验证用户的交易量是否小于 100 万股
 * 根据用户钱包检查订单，并验证是否有足够的资金执行它
 * 将订单发送给排序器，进而发送到撮合引擎。为了减少带宽，只将必要的订单信息传递给撮合引擎
 * 从排序器接收执行结果（成交），然后通过客户端网关发送给经纪商

实现订单管理器的主要挑战是状态转换管理。事件溯源是一个可行的解决方案（将在深潜部分讨论）。

最后，客户端网关从用户处接收订单并将其发送给订单管理器。其职责：

<div style="margin-left:3rem">
    <img src="./images/client-gateway.png" alt="客户端网关" width="500" />
</div>

由于客户端网关位于关键路径上，它应该保持轻量。

可以为不同的客户端设置多个客户端网关。例如，托管引擎（colo engine）是一种交易引擎服务器，由经纪商租用并放置在交易所的数据中心内：

<div style="margin-left:3rem">
    <img src="./images/client-gateways.png" alt="客户端网关（多个）" width="500" />
</div>

#### 市场数据流程
市场数据发布者从撮合引擎接收执行结果，并从执行结果流构建订单簿/K 线图。

该数据被发送到数据服务，数据服务负责向订阅者展示聚合数据：

<div style="margin-left:3rem">
    <img src="./images/market-data.png" alt="市场数据" width="500" />
</div>

#### 报告流程
报告器不在关键路径上，但它仍然是一个重要的组件。

<div style="margin-left:3rem">
    <img src="./images/reporting-flow.png" alt="报告流程" width="500" />
</div>

它负责交易历史、税务报告、合规报告、结算等。
延迟对报告流程不是关键要求。准确性和合规性更为重要。

### **API 设计**
客户端通过经纪商与股票交易所交互，以下单、查看执行结果、获取市场数据、下载历史数据进行分析等。

我们在客户端网关和经纪商之间使用 RESTful API 进行通信。

对于机构客户，使用专有协议来满足他们的低延迟要求。

创建订单：
```
POST /v1/order
```

参数：
 * symbol - 股票代码。String
 * side - 买入或卖出。String
 * price - 限价单的价格。Long
 * orderType - 限价或市价（在我们的设计中只支持限价单）。String
 * quantity - 订单的数量。Long

响应：
 * id - 订单的 ID。Long
 * creationTime - 订单的系统创建时间。Long
 * filledQuantity - 已成功成交的数量。Long
 * remainingQuantity - 仍需成交的数量。Long
 * status - new/canceled/filled。String
 * 其余属性与输入参数相同

获取执行结果：
```
GET /execution?symbol={:symbol}&orderId={:orderId}&startTime={:startTime}&endTime={:endTime}
```

参数：
 * symbol - 股票代码。String
 * orderId - 订单的 ID。可选。String
 * startTime - 查询开始时间（epoch）\[11\]。Long
 * endTime - 查询结束时间（epoch）。Long

响应：
 * executions - 数组，包含范围内的每个执行结果（见下方属性）。Array
 * id - 执行结果的 ID。Long
 * orderId - 订单的 ID。Long
 * symbol - 股票代码。String
 * side - 买入或卖出。String
 * price - 执行价格。Long
 * orderType - 限价或市价。String
 * quantity - 成交数量。Long

获取订单簿：
```
GET /marketdata/orderBook/L2?symbol={:symbol}&depth={:depth}
```

参数：
 * symbol - 股票代码。String
 * depth - 每侧的订单簿深度。Int

响应：
 * bids - 数组，包含价格和数量。Array
 * asks - 数组，包含价格和数量。Array

获取 K 线：
```
GET /marketdata/candles?symbol={:symbol}&resolution={:resolution}&startTime={:startTime}&endTime={:endTime}
```

参数：
 * symbol - 股票代码。String
 * resolution - K 线图窗口长度（秒）。Long
 * startTime - 窗口的开始时间（epoch）。Long
 * endTime - 窗口的结束时间（epoch）。Long

响应：
 * candles - 数组，包含每个 K 线数据（见下方属性）。Array
 * open - 每个 K 线的开盘价。Double
 * close - 每个 K 线的收盘价。Double
 * high - 每个 K 线的最高价。Double
 * low - 每个 K 线的最低价。Double

### **数据模型**
我们的交易所中有三种主要类型的数据：
 * 产品（Product）、订单（order）、执行结果（execution）
 * 订单簿（order book）
 * K 线图（candlestick chart）

#### 产品、订单、执行结果
产品描述交易品种的属性 - 产品类型、交易代码、UI 展示代码等。

这些数据不会频繁变化，主要用于 UI 渲染。

订单（order）表示一个买入/卖出指令。执行结果（execution）是出站的撮合结果。

以下是数据模型：

<div style="margin-left:3rem">
    <img src="./images/product-order-execution-data-model.png" alt="产品-订单-执行结果数据模型" width="500" />
</div>

我们在所有三个流程中都会遇到订单和执行结果：
 * 在关键路径上，它们在内存中处理以获得高性能。它们从排序器存储并恢复。
 * 报告器将订单和执行结果写入数据库，用于报告场景
 * 执行结果被转发给市场数据服务，以重建订单簿和 K 线图

#### 订单簿
订单簿是一种金融工具的买卖订单列表，按价格档位组织。

针对该模型的高效数据结构需要满足：
 * 恒定时间的查找 - 获取某个价格档位或价格档位之间的成交量
 * 快速的增加/成交/取消操作
 * 查询最优买价/卖价
 * 遍历价格档位

示例订单簿执行过程：

<div style="margin-left:3rem">
    <img src="./images/order-book-execution.png" alt="订单簿执行" width="500" />
</div>

在完成这个大宗订单后，随着买卖价差的扩大，价格上涨。

订单簿的示例实现伪代码：
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

为了更高效的实现，我们可以使用双向链表（doubly-linked list）替代标准链表：
 * 放置新订单是 O(1)，因为我们在链表尾部添加订单。
 * 撮合订单是 O(1)，因为我们从链表头部删除订单
 * 取消订单意味着从订单簿中删除一个订单。我们利用 `orderMap` 实现 O(1) 查找和 O(1) 删除（因为 `Order` 持有对链表中前一个元素的引用）。

<div style="margin-left:3rem">
    <img src="./images/order-book-impl.png" alt="订单簿实现" width="500" />
</div>

这种数据结构也用于市场数据服务中重建订单簿。

#### K 线图
K 线数据是在市场数据服务中基于时间区间内的订单处理计算得出的：
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

一些避免消耗过多内存的优化：
 * 使用预分配的环形缓冲区（ring buffer）来持有 K 线，以减少分配次数
 * 限制内存中 K 线的数量，并将其余部分持久化到磁盘

我们将使用内存列式数据库（例如 KDB）进行实时分析。市场收盘后，数据被持久化到历史数据库中。

---

## 步骤 3：设计深潜
关于现代交易所，有一点值得注意：与大多数其他软件不同，它们通常将所有东西运行在一台巨大的服务器上。

让我们探讨其中的细节。

### **性能**
对于交易所来说，在所有百分位上都有良好的整体延迟非常重要。

我们如何降低延迟？
 * 减少关键路径上的任务数量
 * 通过减少网络/磁盘使用和/或减少任务执行时间来缩短每个任务所花费的时间

为了实现第一个目标，我们从关键路径上剥离了所有无关的职责，甚至移除了日志记录，以实现最佳延迟。

如果我们遵循原始设计，会有几个瓶颈 - 服务之间的网络延迟和排序器的磁盘使用。

使用这样的设计，我们可以实现数十毫秒的端到端延迟。而我们希望实现数十微秒。

因此，我们将所有东西放在一台服务器上，进程之间通过 mmap 作为事件存储进行通信：

<div style="margin-left:3rem">
    <img src="./images/mmap-bus.png" alt="mmap 总线" width="500" />
</div>

另一个优化是使用应用程序循环（执行关键任务循环的 while 循环），并将其固定到同一个 CPU 以避免上下文切换：

<div style="margin-left:3rem">
    <img src="./images/application-loop.png" alt="应用程序循环" width="500" />
</div>

使用应用程序循环的另一个附带好处是没有锁争用 - 即多个线程争夺同一个资源。

现在让我们探讨 mmap 是如何工作的 - 它是一个 UNIX 系统调用，将磁盘上的文件映射到应用程序的内存中。

我们可以使用的一个技巧是将文件创建在 `/dev/shm` 中，它代表"共享内存"。因此，我们完全没有磁盘访问。

### **事件溯源**
事件溯源在[数字钱包一章](../chapter28)中有深入讨论。所有细节请参考该章。

简而言之，我们不存储当前状态，而是存储不可变的状态转换：

<div style="margin-left:3rem">
    <img src="./images/event-sourcing.png" alt="事件溯源" width="500" />
</div>

 * 左边 - 传统模式
 * 右边 - 事件溯源模式

到目前为止，我们的设计如下所示：

<div style="margin-left:3rem">
    <img src="./images/design-so-far.png" alt="目前的设计" width="500" />
</div>

 * 外部域通过 FIX 协议与我们的客户端网关交互
 * 订单管理器接收新订单事件，验证它并将其添加到其内部状态。然后订单被发送到撮合核心
 * 如果订单被撮合，生成 `OrderFilledEvent` 并通过 mmap 发送
 * 其他组件订阅事件存储并完成各自的处理部分

另一个额外的优化 - 所有组件都持有一份订单管理器的副本，该管理器被打包为库，以避免为管理订单而额外调用

这种设计中的排序器变为不再是事件存储，而是一个单一写入者，在将事件转发到事件存储之前对事件进行排序：

<div style="margin-left:3rem">
    <img src="./images/sequencer-deep-dive.png" alt="排序器深潜" width="500" />
</div>

### **高可用性**
我们追求 99.99% 的可用性 - 每天只有 8.64 秒的停机时间。

为了实现这一点，我们必须识别交易所架构中的单点故障：
 * 为关键服务（例如撮合引擎）设置处于待命状态的备份实例
 * 积极地自动化故障检测和故障切换到备份实例

诸如客户端网关之类的无状态服务可以通过增加更多服务器轻松进行水平扩展。

对于有状态组件，我们可以处理入站事件，但如果不是领导者，则不发布出站事件：

<div style="margin-left:3rem">
    <img src="./images/leader-election.png" alt="领导者选举" width="500" />
</div>

为了检测主副本宕机，我们可以发送心跳来检测它是否无法正常工作。

这种机制只在单个服务器的边界内有效。
如果我们想扩展它，我们可以将整个服务器设置为热/温副本，并在发生故障时进行故障切换。

为了在副本之间复制事件存储，我们可以使用可靠的 UDP 以实现更快的通信。

### **容错性**
如果连温实例也宕机了怎么办？这是一个低概率事件，但我们应该做好准备。

大型科技公司通过将核心数据复制到多个城市的数据中心来解决这个问题，例如以减轻自然灾害的影响。

需要考虑的问题：
 * 如果主实例宕机，我们如何以及何时切换到备份实例？
 * 我们如何在备份实例中选择领导者？
 * 需要的恢复时间是多少（RTO - 恢复时间目标，recovery time objective）？
 * 需要恢复哪些功能？我们的系统能否在降级条件下运行？

如何解决这些问题：
 * 系统可能因为一个 bug 而宕机（影响主实例和副本），我们可以使用混沌工程来暴露这样的边界情况和灾难性结果
 * 不过一开始，我们可以手动执行故障切换，直到我们对系统的故障模式有了足够的了解
 * 可以使用领导者选举（例如 Raft）来确定在主实例宕机时哪个副本成为领导者

复制在不同服务器之间如何工作的示例：

<div style="margin-left:3rem">
    <img src="./images/replication-across-servers.png" alt="跨服务器复制" width="500" />
</div>

领导者选举任期示例：

<div style="margin-left:3rem">
    <img src="./images/leader-election-terms.png" alt="领导者选举任期" width="500" />
</div>

有关 Raft 如何工作的详细信息，[请查看此处](https://thesecretlivesofdata.com/raft/)

最后，我们还需要考虑损失容忍度 - 在情况变得严重之前，我们可以丢失多少数据？
这将决定我们备份数据的频率。

对于股票交易所来说，数据丢失是不可接受的，因此我们必须频繁备份数据，并依赖 Raft 的复制来降低数据丢失的概率。

### **撮合算法**
稍微绕道，看看撮合如何通过伪代码工作：
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

该撮合算法使用 FIFO 算法来确定在某个价格档位上撮合哪些订单。

### **确定性**
功能确定性通过我们使用的排序器技术来保证。

事件发生的实际时间并不重要：

<div style="margin-left:3rem">
    <img src="./images/determinism.png" alt="确定性" width="500" />
</div>

延迟确定性是我们必须追踪的。我们可以基于监控 99 或 99.99 百分位延迟来计算。

可能引起延迟尖峰的因素是例如 Java 中的垃圾回收器事件。

### **市场数据发布者优化**
市场数据发布者从撮合引擎接收撮合结果，并根据它们重建订单簿和 K 线图。

我们只保留部分 K 线，因为我们没有无限的内存。客户可以选择他们想要的粒度信息。更细粒度的信息可能需要更高的价格：

<div style="margin-left:3rem">
    <img src="./images/market-data-publisher.png" alt="市场数据发布者" width="500" />
</div>

环形缓冲区（也称为 circular buffer）是一种固定大小的队列，头部与尾部相连。空间被预分配以避免分配。该数据结构也是无锁的。

优化环形缓冲区的另一种技术是填充（padding），它确保序列号永远不会与任何其他东西处于同一条缓存行（cache line）中。

### **市场数据的分发公平性与多播**
我们需要确保订阅者同时收到数据，因为如果一个人比另一个人更早收到数据，就会给他们提供关键的市场洞察，他们可以利用这些洞察来操纵市场。

为了实现这一点，在向订阅者发布数据时，我们可以使用基于可靠 UDP 的多播。

数据可以通过三种方式通过互联网传输：
 * 单播（Unicast） - 一个源，一个目的地
 * 广播（Broadcast） - 一个源到整个子网络
 * 多播（Multicast） - 一个源到不同子网络上的一组主机

理论上，通过使用多播，所有订阅者应该同时收到数据。

然而，UDP 是不可靠的，数据可能无法到达每个人。不过，它可以通过重传进行增强。

### **托管（Colocation）**
交易所为经纪商提供在交易所的同一数据中心托管其服务器的能力。

这大大降低了延迟，可以被认为是一项 VIP 服务。

### **网络安全**
DDoS 对交易所来说是一个挑战，因为有一些面向互联网的服务。以下是我们的选择：
 * 将公共服务和数据与私有服务隔离，这样 DDoS 攻击不会影响最重要的客户
 * 使用缓存层来存储更新不频繁的数据
 * 强化 URL 以抵御 DDoS，例如优先使用 `https://my.website.com/data/recent` 而不是 `https://my.website.com/data?from=123&to=456`，因为前者更易于缓存
 * 需要有效的允许列表/阻止列表机制。
 * 可以使用速率限制（rate limiting）来缓解 DDoS

---

## 步骤 4：总结
其他有趣的说明：
 * 并非所有交易所都依赖将所有东西放在一台大服务器上，但有些仍然这样做
 * 现代交易所更依赖云基础设施，也依赖自动做市商（AMM，automated market maker），以避免维护订单簿