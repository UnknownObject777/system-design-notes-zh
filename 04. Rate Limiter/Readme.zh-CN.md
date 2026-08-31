# 第 4 章：设计限流器（Rate Limiter）

## 简介
本章探讨限流器（rate limiter）的设计与实现。限流器是一种系统组件，用于控制客户端或服务发送的流量速率。限流器对于防止滥用、降低成本以及确保服务器资源稳定至关重要。其应用示例包括限制发帖、创建账号和领取奖励等。

## 限流的好处
- **防止 DoS 攻击：** 阻止过量调用，避免资源耗尽。
- **降低成本：** 限制不必要的请求，减少服务器开销。
- **防止过载：** 过滤掉过多的请求，稳定服务器性能。

## 步骤 1：理解问题
### 关键特性
- 服务端 API 限流器。
- 支持多条限流规则。
- 能够处理分布式环境中的大规模系统。
- 可选择独立服务或应用级代码实现。
- 当请求被限流时通知用户。

### 需求
- 精确的请求限流。
- 极低的延迟。
- 低内存占用。
- 支持分布式。
- 清晰的异常处理。
- 高容错性。

## 步骤 2：高层设计
### 部署位置选项
<div style="margin-left:2rem">
    <img src="./images/rate_limiter_architecture.png"  alt="限流中间件架构" width="550">
</div>

1. **客户端实现：** 不可靠，因为可能被滥用。
2. **服务端实现：** 更受青睐，因为便于控制和更可靠。
3. **中间件（API 网关）：** 一种灵活的方案，可集成限流功能。


### 部署位置的指导原则
- 评估现有技术栈并选择高效的方案。
- 根据业务需求选择合适的算法。
- 如果采用微服务架构，可使用 API 网关。
- 如果资源有限，可选择商业解决方案。

## 步骤 3：限流算法
### 1. 令牌桶（Token Bucket）
<div style="margin-left:2rem">
  <img src="./images/token-bucket.png"  alt="令牌桶算法" width="550">
</div>

- **描述：** 以固定速率向桶中添加令牌；每个请求消耗一个令牌。
- **参数：** 桶容量和补充速率。
- **优点：** 易于实现、节省内存、支持流量突发。
- **缺点：** 需要仔细调参。



### 2. 漏桶（Leaking Bucket）
<div style="margin-left:2rem">
  <img src="./images/leaking-bucket.png"  alt="漏桶算法" width="550">
</div>

- **描述：** 使用 FIFO 队列以固定速率处理请求。
- **优点：** 节省内存、流出速率稳定。
- **缺点：** 流量突发可能会延迟新到的请求。
  

  示例：https://github.com/uber-go/ratelimit



### 3. 固定窗口计数器（Fixed Window Counter）
<div style="margin-left:2rem">
  <img src="./images/fixed-window-counter.png"  alt="固定窗口计数器" width="550">
</div>

- **描述：** 将时间划分为固定的时间窗口，并使用计数器限制请求数量。
- **优点：** 实现简单，对特定场景高效。
- **缺点：** 窗口边界处的流量尖峰可能超出限制。

- 在时间窗口边界出现突发流量
会导致超过允许配额的请求通过。

  <img src="./images/fixed-window-issue.png"  alt="固定窗口问题" width="550">


### 4. 滑动窗口日志（Sliding Window Log）
<div style="margin-left:2rem">
  <img src="./images/sliding-window-log.png"  alt="滑动窗口日志" width="550">
</div>

- **描述：** 跟踪时间戳，实现滚动的时间窗口。
- **优点：** 限流精确。
- **缺点：** 内存消耗高。



### 5. 滑动窗口计数器（Sliding Window Counter）
<div style="margin-left:2rem">
  <img src="./images/sliding-window-counter.png"  alt="滑动窗口计数器" width="550">
</div>

- **描述：** 结合固定窗口与滑动日志两种方法，以平滑流量尖峰。
- **优点：** 节省内存、能够处理流量突发。
- **缺点：** 近似计算可能不够严格。



## 高层架构
<div style="margin-left:2rem">
  <img src="./images/architecture.png" style="margin-left: 40px; margin-top: 40px; margin-bottom: 20px;" alt="架构" width="550">
</div>

- **数据存储：** 使用内存缓存（如 Redis）实现快速的计数器操作。
- **流程：**
  1. 客户端向中间件发送请求。
  2. 中间件检查 Redis 中的计数器。
  3. 根据限制条件处理或拒绝请求。


## 进阶考量
### 分布式环境
- **挑战：** 竞态条件、同步问题。
- **解决方案：** 使用 Redis 中的锁、Lua 脚本或有序集合（sorted set）。采用集中式数据存储实现同步。

### 性能优化
- 采用多数据中心架构以降低延迟。
- 使用最终一致性模型进行同步。

### 监控
- 定期分析以确保算法的有效性，并根据需要调整规则。