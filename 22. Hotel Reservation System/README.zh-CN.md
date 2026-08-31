# 第 22 章：酒店预订系统

## 引言
在本章中，我们将设计一个**酒店预订系统**，类似于万豪国际（Marriott International）。

该设计也适用于其他类型的系统——Airbnb、机票预订、电影票预订。

---

## 步骤 1：理解问题并确定设计范围
在深入设计系统之前，我们应该向面试官提问以澄清范围：
 - C：系统的规模是多大？
 - I：我们正在为一个拥有 5000 家酒店和 100 万间客房的连锁酒店构建网站。
 - C：客户是在预订时付费，还是在到达酒店时付费？
 - I：他们在预订时全额付款。
 - C：客户只能通过网站预订酒店客房吗？我们是否需要支持其他预订方式，例如电话？
 - I：他们只能通过网站或应用程序预订。
 - C：客户可以取消预订吗？
 - I：可以。
 - C：还有其他需要考虑的事情吗？
 - I：是的，我们允许超售 10%。酒店会出售比实际更多的客房。酒店这样做是预料到客户会取消预订。
 - C：由于时间不多，我们将专注于——显示酒店相关页面、酒店客房详情页面、预订客房、管理后台、支持超售。
 - I：听起来不错。
 - I：还有一件事——酒店的价格一直在变化。假设酒店客房的价格每天都变。
 - C：好的。

### **非功能需求**
 - 支持高并发——旺季时可能会有大量客户尝试预订同一家酒店。
 - 中等延迟——用户预订时最好有低延迟，但系统花几秒钟处理也是可以接受的。

### **粗略估算**
 - 总共 5000 家酒店和 100 万间客房
 - 假设 70% 的客房被占用，平均入住时长为 3 天
 - 估计每日预订量——1mil * 0.7 / 3 = 每天大约 24 万次预订
 - 每秒预订量——24万 / 一天的 10^5 秒 = 约 3。平均预订 TPS 很低。

让我们来估算 QPS。如果我们假设到达预订页面需要三个步骤，并且每个页面有 10% 的转化率，
我们可以估算，如果有 3 次预订，那么预订页面必须有 30 次浏览，酒店客房详情页面必须有 300 次浏览。

<div style="margin-left:3rem">
    <img src="./images/qps-estimation.png" alt="qps-估算" width="500" />
</div>

---

## 步骤 2：提出高层设计并取得共识
我们将探讨——API 设计、数据模型、高层设计。

### **API 设计**
这个 API 设计聚焦于核心端点（使用 RESTful 实践），我们需要这些端点来支持一个酒店预订系统。

一个功能完备的系统需要更全面的 API，以支持基于大量条件搜索客房，但我们不会在本节中关注这一点。
原因是它们在技术上并不具有挑战性，所以不在范围内。

**酒店相关 API**
 - `GET /v1/hotels/{id}` - 获取酒店的详细信息
 - `POST /v1/hotels` - 添加一家新酒店。仅对运营人员可用
 - `PUT /v1/hotels/{id}` - 更新酒店信息。仅对运营人员可用
 - `DELETE /v1/hotels/{id}` - 删除一家酒店。该 API 仅对运营人员可用

**客房相关 API**
 - `GET /v1/hotels/{id}/rooms/{id}` - 获取客房的详细信息
 - `POST /v1/hotels/{id}/rooms` - 添加一间客房。仅对运营人员可用
 - `PUT /v1/hotels/{id}/rooms/{id}` - 更新客房信息。仅对运营人员可用
 - `DELETE /v1/hotels/{id}/rooms/{id}` - 删除一间客房。仅对运营人员可用

**预订相关 API**
 - `GET /v1/reservations` - 获取当前用户的预订历史
 - `GET /v1/reservations/{id}` - 获取预订的详细信息
 - `POST /v1/reservations` - 进行新的预订
 - `DELETE /v1/reservations/{id}` - 取消预订

以下是一个发起预订的示例请求：

```
{
  "startDate":"2021-04-28",
  "endDate":"2021-04-30",
  "hotelID":"245",
  "roomID":"U12354673389",
  "reservationID":"13422445"
}
```

注意，`reservationID` 是一个幂等键，用于避免重复预订。详细信息在[并发问题](#concurrency-issues)一节中说明。

### **数据模型**
在选择使用什么数据库之前，让我们考虑一下访问模式。

我们需要支持以下查询：
 - 查看酒店的详细信息
 - 在给定日期范围内查找可用的客房类型
 - 记录一次预订
 - 查询一次预订或过去的预订历史

从我们的估算来看，我们知道系统的规模不大，但我们需要为流量激增做好准备。

基于这些信息，我们将选择关系型数据库，因为：
 - 关系型数据库非常适合读密集、写不太密集的系统。
 - NoSQL 数据库通常针对写入进行了优化，但我们知道写入不会很多，因为访问网站的用户中只有一小部分会进行预订。
 - 关系型数据库提供 ACID 保证。这些对于这样的系统很重要，因为如果没有它们，我们将无法防止诸如负余额、重复收费等问题。
 - 关系型数据库可以轻松地对数据进行建模，因为结构非常清晰。

以下是我们的表结构设计：

<div style="margin-left:3rem">
    <img src="./images/schema-design.png" alt="表结构设计" width="500" />
</div>

大多数字段不言自明。唯一值得一提的字段是 `status` 字段，它表示给定客房的状态机：

<div style="margin-left:3rem">
    <img src="./images/status-state-machine.png" alt="状态状态机" width="500" />
</div>

这种数据模型非常适合 Airbnb 这样的系统，但不适合酒店，因为用户预订的不是某一间特定的客房，而是一种客房类型。
他们预订一种客房类型，而具体的房号在预订时选择。

这个缺点将在[改进的数据模型](#improved-data-model)一节中解决。

### **高层设计**
我们为这个设计选择了微服务架构。它在近几年中广受欢迎：

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="高层设计" width="500" />
</div>

 - **用户（Users）**：在手机或电脑上预订酒店客房
 - **管理员（Admin）**：执行管理功能，例如退款/取消支付等
 - **CDN**：缓存静态资源，如 JS 包、图片、视频等
 - **公共 API 网关（Public API Gateway）**：托管服务，支持限流、身份验证等。
 - **内部 API（Internal APIs）**：仅对授权人员可见。通常由 VPN 保护。
 - **酒店服务（Hotel service）**：提供酒店和客房的详细信息。酒店和客房数据是静态的，因此可以积极缓存。
 - **房价服务（Rate service）**：提供未来不同日期的客房价格。关于这个领域有一个有趣的说明是，价格取决于某一天酒店的入住率。
 - **预订服务（Reservation service）**：接收预订请求并预订酒店客房。同时在预订/取消时跟踪客房库存。
 - **支付服务（Payment service）**：处理支付，并在成功时更新预订状态。
 - **酒店管理服务（Hotel management service）**：仅对授权人员可用。允许某些管理功能，用于管理和查看预订、酒店等。

服务间通信可以通过 RPC 框架（如 gRPC）来实现。

---

## 步骤 3：深入设计
让我们深入探讨：
 - 改进的数据模型
 - 并发问题
 - 可扩展性
 - 解决微服务中的数据不一致问题

### **改进的数据模型**
如前文所述，我们需要修改 API 和表结构，以支持预订客房类型而非某一间特定的客房。

对于预订 API，我们不再预订 `roomID`，而是预订 `roomTypeID`：

```
POST /v1/reservations
{
  "startDate":"2021-04-28",
  "endDate":"2021-04-30",
  "hotelID":"245",
  "roomTypeID":"12354673389",
  "roomCount":"3",
  "reservationID":"13422445"
}
```

以下是更新后的表结构：

<div style="margin-left:3rem">
    <img src="./images/updated-schema.png" alt="更新后的表结构" width="500" />
</div>

 - **room**：包含客房的信息
 - **room_type_rate**：包含给定客房类型的价格信息
 - **reservation**：记录客户的预订数据
 - **room_type_inventory**：存储酒店客房的库存数据。

让我们来看看 `room_type_inventory` 的列，因为那个表更有意思：
 - **hotel_id**：酒店的 id
 - **room_type_id**：客房类型的 id
 - **date**：单个日期
 - **total_inventory**：客房总数减去暂时从库存中移除的客房数量。
 - **total_reserved**：给定（hotel_id, room_type_id, date）下已预订的客房总数

还有其他设计这个表的方式，但每个（hotel_id, room_type_id, date）一行可以使预订管理和查询更容易。

表中的行通过每日 CRON 作业预先填充。

示例数据：
| hotel_id | room_type_id | date       | total_inventory | total_reserved |
|----------|--------------|------------|-----------------|----------------|
| 211      | 1001         | 2021-06-01 | 100             | 80             |
| 211      | 1001         | 2021-06-02 | 100             | 82             |
| 211      | 1001         | 2021-06-03 | 100             | 86             |
| 211      | 1001         | ...        | ...             |                |
| 211      | 1001         | 2023-05-31 | 100             | 0              |
| 211      | 1002         | 2021-06-01 | 200             | 16             |
| 2210     | 101          | 2021-06-01 | 30              | 23             |
| 2210     | 101          | 2021-06-02 | 30              | 25             |

检查某种客房类型可用性的示例 SQL 查询：

```
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}
```

如何利用这些数据检查指定数量客房的可用性（注意我们支持超售）：

```
if (total_reserved + ${numberOfRoomsToReserve}) <= 110% * total_inventory
```

现在让我们对存储容量做一些估算。
 - 我们有 5000 家酒店。
 - 每家酒店有 20 种客房类型。
 - 5000 * 20 * 2（年）* 365（天）= 7300 万行

7300 万行并不是很多数据，一个数据库服务器就可以处理。
然而，设置读复制（可能跨不同区域）以实现高可用性是合理的。

追问——如果预订数据对单个数据库来说太大，你会怎么做？
 - 只存储当前和未来的预订数据。预订历史可以移到冷存储中。
 - 数据库分片——我们可以按 `hash(hotel_id) % servers_cnt` 对数据进行分片，因为我们在查询中总是会选中 `hotel_id`。

### **并发问题**
另一个需要解决的重要问题是重复预订。

有两个问题需要解决：
 - 同一个用户两次点击"预订"
 - 多个用户同时尝试预订一间客房

下面是第一个问题的可视化：

<div style="margin-left:3rem">
    <img src="./images/double-booking-single-user.png" alt="单用户重复预订" width="500" />
</div>

有两种方法可以解决这个问题：
 - 客户端处理——前端可以在点击后禁用预订按钮。然而，如果用户禁用了 javascript，他们就不会看到按钮变灰。
 - 幂等 API——向 API 添加一个幂等键，使用户可以一次性执行一个操作，无论端点被调用多少次：

<div style="margin-left:3rem">
    <img src="./images/idempotency.png" alt="幂等性" width="500" />
</div>

这个流程如下：
 - 在填写详细信息并进行预订的过程中，会生成一个预订订单（reservation order）。预订订单使用全局唯一标识符生成。
 - 使用上一步生成的 `reservation_id` 提交预订 1。
 - 如果第二次点击"完成预订"，会发送相同的 `reservation_id`，后端会检测到这是一次重复预订。
 - 通过让 `reservation_id` 列具有唯一约束来避免重复，防止多个具有该 id 的记录被存储在数据库中。

<div style="margin-left:3rem">
    <img src="./images/unique-constraint-violation.png" alt="唯一约束冲突" width="500" />
</div>

如果有多个用户进行相同的预订怎么办？

<div style="margin-left:3rem">
    <img src="./images/double-booking-multiple-users.png" alt="多用户重复预订" width="500" />
</div>

 - 假设事务隔离级别不是可串行化的
 - 用户 1 和用户 2 同时尝试预订同一间客房。
 - 事务 1 检查是否有足够的客房——有
 - 事务 2 检查是否有足够的客房——有
 - 事务 2 预订了客房并更新了库存
 - 事务 1 也预订了客房，因为它仍然看到 100 间中有 99 间 `total_reserved`。
 - 两个事务都成功提交了更改

这个问题可以通过某种形式的锁机制来解决：
 - 悲观锁（Pessimistic locking）
 - 乐观锁（Optimistic locking）
 - 数据库约束

以下是我们用于预订客房的 SQL：

```sql
# 步骤 1：检查客房库存
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}

# 对于步骤 1 返回的每一行
if((total_reserved + ${numberOfRoomsToReserve}) > 110% * total_inventory) {
  Rollback
}

# 步骤 2：预订客房
UPDATE room_type_inventory
SET total_reserved = total_reserved + ${numberOfRoomsToReserve}
WHERE room_type_id = ${roomTypeId}
AND date between ${startDate} and ${endDate}

Commit
```

#### 选项 1：悲观锁
悲观锁通过在记录被更新时对记录加锁来防止同时更新。

在 MySQL 中，这可以通过使用 `SELECT... FOR UPDATE` 查询来完成，该查询会锁定查询所选中的行，直到事务提交。

<div style="margin-left:3rem">
    <img src="./images/pessimistic-locking.png" alt="悲观锁" width="500" />
</div>

优点：
 - 防止应用程序更新正在被修改的数据
 - 易于实现，并通过串行化更新来避免冲突。在数据争用严重时很有用。

缺点：
 - 当多个资源被锁定时，可能会发生死锁。
 - 这种方法不可扩展——如果事务被锁定太久，会影响所有其他试图访问该资源的事务。
 - 当查询选中大量资源且事务生命周期较长时，影响会很严重。

作者由于可扩展性问题不推荐这种方法。

#### 选项 2：乐观锁
乐观锁允许多个用户同时尝试更新一条记录。

有两种常见的实现方式——版本号和时间戳。推荐使用版本号，因为服务器时钟可能不准确。

<div style="margin-left:3rem">
    <img src="./images/optimistic-locking.png" alt="乐观锁" width="500" />
</div>

 - 在数据库表中添加一个新的 `version` 列
 - 在用户修改数据库行之前，读取版本号
 - 当用户更新该行时，版本号加 1 并写回数据库
 - 如果新版本号没有超过之前的版本号，数据库校验会阻止插入

乐观锁通常比悲观锁更快，因为我们没有锁定数据库。
然而，当并发度很高时，它的性能往往会下降，因为这会导致大量回滚。

优点：
 - 防止应用程序编辑过期数据
 - 我们不需要在数据库中获取锁
 - 数据争用较低时（即很少发生更新冲突）是首选方案

缺点：
 - 数据争用较高时性能较差

乐观锁对我们的系统是一个不错的选择，因为预订 QPS 并不是特别高。

#### 选项 3：数据库约束
这种方法与乐观锁非常相似，但防护通过数据库约束来实现：

```
CONSTRAINT `check_room_count` CHECK((`total_inventory - total_reserved` >= 0))
```

<div style="margin-left:3rem">
    <img src="./images/database-constraint.png" alt="数据库约束" width="500" />
</div>

优点：
 - 易于实现
 - 数据争用较小时效果很好

缺点：
 - 与乐观锁类似，数据争用较高时性能较差
 - 数据库约束不能像应用程序代码那样容易地进行版本控制
 - 并非所有数据库都支持约束

由于实现简单，这也是酒店预订系统的另一个好选择。

### **可扩展性**
通常，酒店预订系统的负载并不高。

然而，面试官可能会问你如何处理系统被用于更大、更受欢迎的旅游网站（如 booking.com）的情况。
在这种情况下，QPS 可能会大 1000 倍。

当存在这种情况时，重要的是了解我们的瓶颈在哪里。所有服务都是无状态的，因此可以通过复制轻松扩展。

然而，数据库是有状态的，如何扩展它并不是那么明显。

一种扩展方式是实现数据库分片——我们可以将数据拆分到多个数据库中，每个数据库包含一部分数据。

我们可以基于 `hotel_id` 进行分片，因为所有查询都基于它进行过滤。
假设 QPS 为 30,000，将数据库分成 16 个分片后，每个分片处理 1875 QPS，这在一个 MySQL 集群的负载能力范围内。

<div style="margin-left:3rem">
    <img src="./images/database-sharding.png" alt="数据库分片" width="500" />
</div>

我们还可以通过 Redis 对客房库存和预订进行缓存。我们可以设置 TTL，使过去日期的旧数据能够过期。

<div style="margin-left:3rem">
    <img src="./images/inventory-cache.png" alt="库存缓存" width="500" />
</div>

我们存储库存的方式基于 `hotel_id`、`room_type_id` 和 `date`：

```
key: hotelID_roomTypeID_{date}
value: 给定酒店 ID、客房类型 ID 和日期下可用客房的数量。
```

数据一致性是异步发生的，并通过使用 CDC 流式传输机制进行管理——数据库更改被读取并应用到另一个系统。
Debezium 是用于将数据库更改与 Redis 同步的流行选择。

使用这种机制，缓存和数据库在一段时间内可能存在不一致的可能性。
在我们的场景中这是可以的，因为数据库会阻止我们进行无效的预订。

这会在 UI 上造成一些问题，因为用户不得不刷新页面才能看到"没有更多客房了"，
但无论如何，这种情况都可能发生，例如如果一个人在预订前犹豫了很久。

缓存优点：
 - 减少数据库负载
 - 高性能，因为 Redis 在内存中管理数据

缓存缺点：
 - 维护缓存与数据库之间的数据一致性很难。我们需要考虑不一致性如何影响用户体验。

### **服务间的数据一致性**
单体应用使我们能够使用共享的关系型数据库来确保数据一致性。

在我们的微服务设计中，我们选择了混合方案，一些服务是独立的，
但预订和库存 API 由同一个服务处理。

这样做是因为我们希望利用关系型数据库的 ACID 保证来确保一致性。

然而，面试官可能会质疑这种方法，因为它不是纯粹的微服务架构——在纯微服务架构中，每个服务都有自己的数据库：

<div style="margin-left:3rem">
    <img src="./images/microservices-vs-monolith.png" alt="微服务 vs 单体" width="500" />
</div>

这可能会导致一致性问题。在单体服务器中，我们可以利用关系型数据库的事务能力来实现原子操作：

<div style="margin-left:3rem">
    <img src="./images/atomicity-monolith.png" alt="单体原子性" width="500" />
</div>

然而，当操作跨越多个服务时，要保证这种原子性就更有挑战性：

<div style="margin-left:3rem">
    <img src="./images/microservice-non-atomic-operation.png" alt="微服务非原子操作" width="500" />
</div>

有一些众所周知的技术可以处理这些数据不一致问题：
 - **两阶段提交（Two-phase commit）**：一种数据库协议，保证跨多个节点的原子事务提交。
   然而，它的性能不好，因为单个节点的延迟会导致所有节点阻塞该操作。
 - **Saga**：一系列本地事务，如果工作流中的任何步骤失败，就会触发补偿事务。这是一种最终一致的方法。

值得注意的是，解决微服务之间的数据不一致是一个具有挑战性的问题，它提高了系统的复杂性。
考虑到我们更务实的方法——在同一个关系型数据库中封装依赖的操作——值得考虑这种成本是否值得。

---

## 步骤 4：总结
我们提出了一个酒店预订系统的设计。

这些是我们经历的步骤：
 - 收集需求并进行粗略估算，以了解系统的规模
 - 我们在高层设计中提出了 API 设计、数据模型和系统架构
 - 在深入设计中，我们随着需求的变化探讨了备选的数据库表结构设计
 - 我们讨论了竞态条件并提出了解决方案——悲观/乐观锁、数据库约束
 - 通过数据库分片和缓存来扩展系统的方法
 - 最后我们解决了如何跨多个微服务处理数据一致性的问题