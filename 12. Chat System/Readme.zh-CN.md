# 第 12 章：设计一个聊天系统

## 简介
**聊天系统**（chat system）支持用户之间的实时消息传递。本章重点设计一个聊天应用，包含以下功能：
- **一对一聊天**
- **群聊（最多 100 人）**
- **在线状态指示**
- **多设备支持**
- **推送通知**

该系统目标支持 **5000 万日活跃用户（DAU）**，并永久保存聊天记录。

---

## 步骤 1：理解问题

### 需求
1. **功能：**
   - 一对一聊天和群聊（最多 100 名成员）。
   - 基于文本的消息（最多 100,000 个字符）。
   - 在线/离线状态指示。
   - 支持多设备。
   - 推送通知。
2. **规模：** 按 5000 万 DAU 进行设计。
3. **存储：** 永久保存聊天记录。

---

## 步骤 2：高层设计

### 通信协议
1. **发送方：** 使用 HTTP 发送消息，利用持久连接（persistent connection）来提高效率。

      <div style="margin-left:2rem">
      <img src="./images/basic-design.png" alt="基础设计" width="500">    
      <div>

2. **接收方：**
   - **轮询（Polling）：**
      - 客户端定期询问服务器是否有新消息可用。
      - 由于频繁的冗余请求，效率较低。

         <img src="./images/polling.png" alt="轮询" width="400">    

   - **长轮询（Long Polling）：**
      - 保持连接打开，直到有消息到达。
      - 对不活跃用户来说效率较低。

         <img src="./images/long-polling.png" alt="长轮询" width="400">

   - **WebSocket：**
      - 一种用于实时通信的双向持久连接，发送和接收消息均采用此方式。
      - 使用 WebSockets（ws）协议发送和接收消息。

         <img src="./images/websocket.png" alt="Websocket"  width="400" >    
   
---

### 组件

<div style="margin-left:5rem">
   <img src="./images/high-level-stateless-arch.png" alt="高层架构" height="350">    
   <img src="./images/high-level-statefull-arch.png" alt="高层架构" height="350" width="550">
</div>

1. **无状态服务（Stateless Services）：**
   - 处理注册、登录和用户资料管理。
   - 与服务发现（service discovery）集成，以推荐最佳聊天服务器。
2. **有状态服务（Stateful Services）：**
   - 聊天服务器维护持久的 WebSocket 连接。
   - 负责消息的投递和同步。
3. **第三方集成：**
   - 推送通知服务通知用户有新消息。
   - 通知功能的实现请参考"通知系统"章节。

---
### 设计

客户端与聊天服务器之间保持一条持久的 WebSocket 连接，用于实时消息传递。

<div style="margin-left:3rem">
      <img src="./images/high-level-design.png" alt="高层设计" width="450"> 
</div>

- 聊天服务器负责消息的发送/接收。
- 在线状态服务器（presence server）管理在线/离线状态。
- API 服务器处理所有事务，包括用户登录、注册、修改资料等。
- 通知服务器发送推送通知。
- 最后，使用键值存储（key-value store）来保存聊天记录。选择键值存储作为聊天记录数据库的原因如下：
   - 它易于水平扩展。
   - 键值存储访问数据时延迟非常低。
   - 关系型数据库无法很好地处理数据的长尾。当索引变大时，随机访问成本很高。
   - 键值存储已被其他经过验证的可靠聊天应用所采用。例如 Facebook Messenger 和 Discord 都使用键值存储。

以下是一对一聊天和群聊的数据模型。
   - 主键是消息 ID，它有助于确定消息的先后顺序。
   - 对于群聊，复合主键为 (channel_id, message_id)。
      - ID 可以使用全局 64 位序列号生成器（如 Snowflake）来生成。
      - 更好的方法是使用本地序列号生成器。所谓"本地"，是指 ID 仅在某个群组内部唯一。
      - 之所以本地 ID 可行，是因为在某个一对一频道或群聊频道内维护消息顺序就已经足够了。
      
      <img src="./images/one-to-one-chat.png" alt="一对一聊天设计" width="300">   
      <img src="./images/group-chat.png" alt="群聊设计" width="300">   


## 步骤 3：深入设计

### 服务发现（Service Discovery）

<div style="margin-left:3rem">
   <img src="./images/zookeeper.png" alt="Zookeeper" width="400">   
</div>

- 服务发现的主要作用是根据地理位位置、服务器容量等标准，为客户端推荐最佳的聊天服务器。
- 使用 **Apache Zookeeper**，根据地理位置和服务器容量等标准分配聊天服务器。
- 确保高效的负载分配并最大限度地降低延迟。


### 消息流程
#### 一对一聊天


1. 用户 A 向聊天服务器 1 发送一条消息。
2. 聊天服务器 1 为该消息分配一个唯一的消息 ID，并将其存储在键值存储中。
3. 如果用户 B 在线，消息会被转发到聊天服务器 2，该服务器维护着一条持久的 WebSocket 连接。
4. 如果用户 B 离线，则发送一条推送通知。



#### 群聊

<div style="margin-left:3rem">
   <img src="./images/group-chat-flow.png" alt="群聊流程" width="400">  
</div>

- 消息会被复制到群组中每个收件人的个人收件箱（inbox）中。
- 这简化了同步，但对于较大的群组来说代价较高。
- 在收件人这一端，一个收件人可以从多个用户那里接收消息。每个收件人都有一个收件箱（消息同步队列），其中包含来自不同发件人的消息。

---

#### 消息同步

许多用户拥有多个设备。我们需要跨设备同步消息。
每个设备维护一个名为 cur_max_message_id 的变量，用于跟踪该设备上最新的消息 ID。满足以下两个条件的消息被视为新消息：

<div style="margin-left:3rem">
   <img src="./images/message-synchronization.png" alt="消息同步"  width="400">  
</div>

- 收件人 ID 等于当前登录的用户 ID。
- 键值存储中的消息 ID 大于 cur_max_message_id。

---

### 在线状态（Online Presence）
1. **心跳机制（Heartbeat Mechanism）：**
   <div style="margin-left:3rem">
      <img src="./images/heartbeat-mechanism.png" alt="心跳机制" width="400"> 
   </div>
   
   - 客户端定期向在线状态服务器发送心跳，以表明自己在线。
   - 如果在某个阈值时间内（例如 x = 30 秒）没有收到心跳，该用户将被标记为离线。

     

2. **扇出模型（Fanout Model）：**

   <div style="margin-left:3rem">
      <img src="./images/fanout-presence.png" alt="在线状态扇出" width="400"> 
   </div>

   - 在线状态更新通过发布-订阅（publish-subscribe）模型推送给好友，其中每一对好友维护一个频道（channel）。
   - 当用户 A 的在线状态改变时，它会把该事件发布到三个频道：A-B、A-C 和 A-D。
   - 这三个频道分别被用户 B、C 和 D 订阅，他们因此能获得在线状态更新。
   - 上述设计在用户群规模较小时比较有效。


---

## 补充考虑
### 可扩展性
- **水平扩展：** 随着用户数量增长，增加服务器。
- **负载均衡：** 将流量均匀地分配到各服务器。
- **缓存：** 减轻数据库负载并改善延迟。

### 错误处理
- **重试机制：** 通过重试和排队来处理消息投递失败。
- **服务器故障：** 使用服务发现在发生故障时分配新的服务器。

### 未来扩展
1. **媒体支持：** 增加对照片和视频的处理，包括压缩和云存储。
2. **端到端加密：** 确保消息的隐私性。
3. **客户端缓存：** 减少数据传输以获得更好的性能。
4. **缩短加载时间：** 使用地理上分布式的缓存网络。