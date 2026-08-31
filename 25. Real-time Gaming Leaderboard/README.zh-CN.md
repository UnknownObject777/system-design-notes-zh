# 第 25 章：实时游戏排行榜

## 引言

我们将为一个在线手机游戏设计一个**排行榜**：

<div style="margin-left:3rem">
    <img src="./images/leaderboard.png" alt="排行榜" width="500" />
</div>

---

## 步骤 1：理解问题并确定设计范围

- 问：排行榜的分数是如何计算的？
- 答：每当用户赢下一场比赛，就会获得 1 分。
- 问：所有玩家都会出现在排行榜上吗？
- 答：是的。
- 问：排行榜是否有时间分段？
- 答：每个月会开始一场新的锦标赛，同时开启一个新的排行榜。
- 问：我们可以假设只关心前 10 名用户吗？
- 答：我们希望展示前 10 名用户，以及特定用户的排名。如果时间允许，我们可以讨论展示排行榜中特定用户周围的用户。
- 问：一场锦标赛中有多少用户？
- 答：500 万日活跃用户（DAU）和 2500 万月活跃用户（MAU）。
- 问：一场锦标赛期间平均进行多少场比赛？
- 答：每位玩家平均每天打 10 场比赛。
- 问：如果两名玩家分数相同，我们如何确定排名？
- 答：在这种情况下，他们的排名相同。如果时间允许，我们可以讨论如何打破平局。
- 问：排行榜需要实时更新吗？
- 答：是的，我们希望呈现实时结果，或尽可能接近实时。不接受批量呈现的历史结果。

### **功能需求**

- 显示排行榜前 10 名玩家
- 显示特定用户的排名
- 显示给定用户前后各四名的用户（加分项）

### **非功能需求**

- 分数实时更新
- 分数更新实时反映在排行榜上
- 通用的可扩展性、可用性、可靠性

### **粗略估算（Back-of-the-envelope estimation）**

按 5000 万 DAU 计算，如果在 24 小时内玩家分布均匀，平均每秒约有 50 名用户。
然而，由于分布通常是不均匀的，我们可以估计峰值在线用户为每秒 250 名用户。

用户得分的 QPS——按平均每天 10 场比赛计算，50 用户/秒 * 10 = 500 QPS。峰值 QPS = 2500。

获取前 10 名排行榜的 QPS——假设用户平均每天打开一次，QPS 为 50。

---

## 步骤 2：提出高层级设计并达成共识

### **API 设计**

我们需要的第一个 API 是更新用户分数的 API：

```
POST /v1/scores
```

这个 API 接受两个参数——`user_id` 和赢得比赛获得的 `points`（分数）。

该 API 只应允许游戏服务器访问，客户端不能直接调用。

下一个是获取排行榜前 10 名玩家的 API：

```
GET /v1/scores
```

示例响应：

```
{
  "data": [
    {
      "user_id": "user_id1",
      "user_name": "alice",
      "rank": 1,
      "score": 12543
    },
    {
      "user_id": "user_id2",
      "user_name": "bob",
      "rank": 2,
      "score": 11500
    }
  ],
  ...
  "total": 10
}
```

你还可以获取特定用户的分数：

```
GET /v1/scores/{:user_id}
```

示例响应：

```
{
    "user_info": {
        "user_id": "user5",
        "score": 1000,
        "rank": 6,
    }
}
```

### **高层级架构**

<div style="margin-left:3rem">
    <img src="./images/high-level-architecture.png" alt="高层级架构" width="500" />
</div>

- 当玩家赢下一场比赛时，客户端向游戏服务发送请求
- 游戏服务验证获胜是否有效，并调用排行榜服务更新玩家的分数
- 排行榜服务在排行榜存储中更新用户的分数
- 玩家调用排行榜服务获取排行榜数据，例如前 10 名玩家和给定玩家的排名

一个曾被考虑的替代方案是客户端直接在排行榜服务中更新自己的分数：

<div style="margin-left:3rem">
    <img src="./images/alternative-design.png" alt="替代方案" width="500" />
</div>

这个方案不安全，因为它容易受到中间人攻击。玩家可以架设代理并按自己的意愿修改分数。

另一个需要注意的地方是，对于由服务器管理游戏逻辑的游戏，客户端无需显式调用服务器来记录胜利。
服务器会根据游戏逻辑自动为客户端完成记录。

另一个考虑是，我们是否应该在游戏服务器和排行榜服务之间放置一个消息队列。如果其他服务对游戏结果感兴趣，这会很有用，但到目前为止，面试中并没有明确要求，因此设计中没有包含它：

<div style="margin-left:3rem">
    <img src="./images/message-queue-based-comm.png" alt="基于消息队列的通信" width="500" />
</div>

### **数据模型**

让我们讨论用于存储排行榜数据的可选方案——关系型数据库、Redis、NoSQL。

NoSQL 方案将在深入探讨部分讨论。

#### 关系型数据库方案

如果规模不重要、用户数量也不多，关系型数据库就能很好地满足我们的需求。

我们可以从一个简单的排行榜表开始，每个月一张表（个人备注——这不太合理。你只需添加一个 `month` 列，就能避免每个月维护新表的麻烦）：

<div style="margin-left:3rem">
    <img src="./images/leaderboard-table.png" alt="排行榜表" width="500" />
</div>

表中还有额外的数据，但与我们要执行的查询无关，因此省略了。

当用户赢得一分时会发生什么？

<div style="margin-left:3rem">
    <img src="./images/user-wins-point.png" alt="用户赢得一分" width="500" />
</div>

如果表中尚不存在该用户，我们需要先插入他们：

```
INSERT INTO leaderboard (user_id, score) VALUES ('mary1934', 1);
```

在后续调用中，我们只需更新他们的分数：

```
UPDATE leaderboard set score=score + 1 where user_id='mary1934';
```

我们如何找到排行榜的前几名玩家？

<div style="margin-left:3rem">
    <img src="./images/find-leaderboard-position.png" alt="查找排行榜名次" width="500" />
</div>

我们可以运行以下查询：

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC;
```

但这并不高效，因为它需要对数据库表中的所有记录做全表扫描来排序。

我们可以通过在 `score` 上添加索引并使用 `LIMIT` 操作来优化，避免扫描所有数据：

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC
LIMIT 10;
```

然而，如果用户不在排行榜顶部，而你想定位他们的排名，这种方法就无法很好地扩展。

#### Redis 方案

我们希望找到一个即使面对数百万玩家也能良好工作的方案，而无需依赖复杂的数据库查询。

Redis 是一个内存数据存储，由于在内存中工作因此速度很快，并且拥有适合我们需求的数据结构——有序集合（sorted set）。

有序集合是一种类似于编程语言中集合的数据结构，它允许你按照给定条件保持数据结构有序。
在内部，它使用哈希映射来维护键（user_id）和值（score）之间的映射，并使用跳表（skip list）将分数按排序顺序映射到用户：

<div style="margin-left:3rem">
    <img src="./images/sorted-set.png" alt="有序集合" width="500" />
</div>

跳表是如何工作的？
- 它是一种允许快速搜索的链表
- 它由一个有序链表和多级索引组成

<div style="margin-left:3rem">
    <img src="./images/skip-list.png" alt="跳表" width="500" />
</div>

这种结构使我们能够在数据集足够大时快速搜索特定值。
在下面的例子中（64 个节点），要找到给定值，基础链表需要遍历 62 个节点，而跳表只需要遍历 11 个节点：

<div style="margin-left:3rem">
    <img src="./images/skip-list-performance.png" alt="跳表性能" width="500" />
</div>

有序集合比关系型数据库性能更高，因为数据始终保持有序，代价是添加和查找操作的复杂度为 O(logN)。

相比之下，下面是我们在关系型数据库中查找给定用户排名时需要运行的嵌套查询示例：

```
SELECT *,(SELECT COUNT(*) FROM leaderboard lb2
WHERE lb2.score >= lb1.score) RANK
FROM leaderboard lb1
WHERE lb1.user_id = {:user_id};
```

在 Redis 中运行排行榜需要哪些操作？
- **ZADD**——如果用户不存在，则将用户插入集合。否则，更新分数。时间复杂度为 O(logN)。
- **ZINCRBY**——将用户的分数增加给定数值。如果用户不存在，分数从零开始。时间复杂度为 O(logN)。
- **ZRANGE/ZREVRANGE**——按分数排序获取一定范围的用户。我们可以指定顺序（升序/降序）、偏移量和结果大小。时间复杂度为 O(logN+M)，其中 M 是结果大小。
- **ZRANK/ZREVRANK**——按升序/降序获取给定用户的排名。时间复杂度为 O(logN)。

当用户得分时会发生什么？

```
ZINCRBY leaderboard_feb_2021 1 'mary1934'
```

每个月都会创建一个新的排行榜，而旧的排行榜会被移到历史存储中。

当用户获取前 10 名玩家时会发生什么？

```
ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES
```

示例结果：

```
[(user2,score2),(user1,score1),(user5,score5)...]
```

用户获取自己的排行榜名次呢？

<div style="margin-left:3rem">
    <img src="./images/leaderboard-position-of-user.png" alt="用户的名次" width="500" />
</div>

已知用户的排行榜名次后，通过以下查询可以轻松实现：

```
ZREVRANGE leaderboard_feb_2021 357 365
```

用户的排名可以使用 `ZREVRANK <user-id>` 获取。

让我们探讨一下我们的存储需求：
- 假设最坏情况是给定月份全部 2500 万 MAU 都参与游戏
- ID 是 24 个字符的字符串，分数是 16 位整数，我们需要 26 字节 * 2500 万 = 约 650MB 的存储空间
- 即使因为跳表的开销使存储成本翻倍，这仍然可以轻松放进现代的 redis 集群中

另一个需要考虑的非功能需求是支持每秒 2500 次更新。这完全在单个 Redis 服务器的能力范围内。

其他注意事项：
- 我们可以启动一个 Redis 副本，避免 redis 服务器崩溃时丢失数据
- 我们仍然可以利用 Redis 持久化来在崩溃时避免数据丢失
- 我们需要在 MySQL 中建两个辅助表来获取用户详细信息（如用户名、显示名等），并存储用户何时赢得比赛等数据
- MySQL 中的第二个表可用于在基础设施故障时重建排行榜
- 作为一项小的性能优化，我们可以缓存前 10 名玩家的用户信息，因为它们会被频繁访问

---

## 步骤 3：深入设计

### **是否使用云服务提供商**

我们可以选择自行部署和管理自己的服务，也可以使用云服务提供商来为我们管理。

如果我们选择自行管理服务，我们会使用 redis 存储排行榜数据，用 mysql 存储用户资料，如果我们要扩展数据库，可能还需要一个缓存来存储用户资料：

<div style="margin-left:3rem">
    <img src="./images/manage-services-ourselves.png" alt="自行管理服务" width="500" />
</div>

或者，我们可以使用云服务产品来为我们管理大量服务。例如，我们可以使用 AWS API Gateway 将 API 调用路由到 AWS Lambda 函数：

<div style="margin-left:3rem">
    <img src="./images/api-gateway-mapping.png" alt="API 网关映射" width="500" />
</div>

AWS Lambda 使我们无需自行管理或配置服务器即可运行代码。它只在需要时运行，并自动扩展。

用户得分的示例：

<div style="margin-left:3rem">
    <img src="./images/user-scoring-point-lambda.png" alt="使用 Lambda 的用户得分" width="500" />
</div>

用户获取排行榜的示例：

<div style="margin-left:3rem">
    <img src="./images/user-retrieve-leaderboard.png" alt="用户获取排行榜" width="500" />
</div>

Lambda 是无服务器架构的一种实现。我们无需管理扩展和环境配置。

作者建议，如果我们从零开始构建这款游戏，应采用这种方法。

### **Redis 的扩展**

按 5000 万 DAU 计算，无论是从存储量还是 QPS 的角度，单个 Redis 实例都能满足需求。

然而，如果我们设想用户基数增长 10 倍至 5 亿 DAU，那么我们需要 65GB 的存储空间，QPS 也会达到 25 万。

这种规模需要分片（sharding）。

实现分片的一种方式是按范围对数据进行分区：

<div style="margin-left:3rem">
    <img src="./images/range-partition.png" alt="范围分区" width="500" />
</div>

在这个例子中，我们根据用户的分数进行分片。我们将在应用程序代码中维护 user_id 与分片之间的映射。
我们可以通过 MySQL 或另一个缓存来维护这个映射。

要获取前 10 名玩家，我们会查询分数最高的分片（`[900-1000]`）。

要获取用户的排名，我们需要计算用户在其分片内的排名，再加上其他分片中分数更高的所有用户。
后者是一个 O(1) 操作，因为可以通过 info keyspace 命令快速访问每个分片的总记录数。

另一种选择是使用 Redis Cluster 进行哈希分区。它是一个代理，根据类似于一致性哈希（consistent hashing）的分区方式（但不完全相同）将数据分布到 redis 节点上：

<div style="margin-left:3rem">
    <img src="./images/hash-partition.png" alt="哈希分区" width="500" />
</div>

在这种设置下，计算前 10 名玩家很有挑战性。我们需要获取每个分片的前 10 名玩家，并在应用程序中合并结果：

<div style="margin-left:3rem">
    <img src="./images/top-10-players-calculation.png" alt="前 10 名玩家的计算" width="500" />
</div>

哈希分区存在一些限制：
- 如果我们需要获取前 K 名用户，而 K 很大，延迟可能会增加，因为我们需要从所有分片获取大量数据
- 延迟会随着分片数量的增加而增加
- 没有直接的方法来确定用户的排名

基于以上原因，作者倾向于在这个问题上使用固定分区。

其他注意事项：
- 一个最佳实践是，对于写入量大的 redis 节点，分配所需内存的两倍，以便在需要时容纳快照
- 我们可以使用名为 Redis-benchmark 的工具来跟踪 redis 设置的性能，并做出基于数据的决策

### **替代方案：NoSQL**

另一个值得考虑的替代方案是使用合适的 NoSQL 数据库，针对以下需求进行优化：
- 大量写入
- 在同一分区内高效地按分数排序

DynamoDB、Cassandra 或 MongoDB 都是很好的选择。

在本章中，作者决定使用 DynamoDB。它是一个全托管的 NoSQL 数据库，提供可靠的性能和出色的可扩展性。
当我们需要查询不属于主键的字段时，它还能使用全局二级索引（global secondary index）。

<div style="margin-left:3rem">
    <img src="./images/dynamo-db.png" alt="DynamoDB" width="500" />
</div>

让我们从一张存储国际象棋游戏排行榜的表开始：

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-1.png" alt="国际象棋游戏排行榜表 1" width="500" />
</div>

这样工作良好，但如果我们需要按分数查询任何内容，它的扩展性就不佳。因此，我们可以把分数作为排序键（sort key）：

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-2.png" alt="国际象棋游戏排行榜表 2" width="500" />
</div>

这个设计的另一个问题是按月份分区。这会形成热点分区，因为最近月份会被不均匀地访问，其他月份则较少被访问。

我们可以使用一种叫做写入分片（write sharding）的技术，即通过 `user_id % num_partitions` 计算，为每个键附加一个分区号：

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-3.png" alt="国际象棋游戏排行榜表 3" width="500" />
</div>

一个重要的权衡是应该使用多少个分区：
- 分区越多，写入扩展性越高
- 但读取扩展性会受到影响，因为我们需要查询更多分区来收集聚合结果

使用这种方法要求我们采用前面见过的"扇出（scatter-gather）"技术，随着分区数量的增加，其时间复杂度会增长：

<div style="margin-left:3rem">
    <img src="./images/scatter-gather-2.png" alt="扇出 2" width="500" />
</div>

要就分区的数量做出好的评估，我们需要进行一些基准测试。

这种 NoSQL 方法仍然有一个主要缺点——很难计算用户的特定排名。

如果我们的规模大到需要分片，那么也许我们可以告诉用户他们处于分数的哪个"百分位"。

一个 cron 作业可以定期运行来分析分数分布，据此确定用户的百分位，例如：

```
10th percentile = score < 100
20th percentile = score < 500
...
90th percentile = score < 6500
```

---

## 步骤 4：总结

如果时间允许，还可以讨论的其他事项：
- **更快的检索**——我们可以通过 Redis 哈希缓存用户对象，映射关系为 `user_id -> user object`。与查询数据库相比，这能实现更快的检索。
- **打破平局**——当两名玩家分数相同时，我们可以根据他们最近一次参与比赛的时间进行排序来打破平局。
- **系统故障恢复**——在发生大规模 Redis 宕机时，我们可以遍历 MySQL WAL 记录，并通过一个临时脚本重建排行榜。