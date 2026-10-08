# 📚 系统设计面试笔记 · 简体中文版

> **System Design Interview — An Insider's Guide（第 1、2 卷）** 章节笔记的全量中文翻译。

本仓库将《[System Design Interview - An Insider's Guide](https://www.goodreads.com/book/show/54109255-system-design-interview-an-insider-s-guide)》（第 1、2 卷）的学习笔记**完整翻译为简体中文**。每个章节都包含：需求梳理、高层设计、深入设计细节与延伸阅读，并**保留原始架构图、数据表格与示例代码**。

英文原版笔记索引见 [Readme.md](./Readme.md)。

按本地知识库的讲解规则整理的短版入口见[系统设计 ELI5 学习版](./学习版索引.md)。学习卡用于建立心智模型；原章节继续保留完整设计过程、图示和来源。

---

## ✨ 仓库特色

- **28 个章节全覆盖**：从「从零扩展到百万级用户」到「股票交易所」，覆盖系统设计面试全部高频主题
- **关键术语双语对照**：如 一致性哈希（Consistent Hashing）、限流器（Rate Limiter）、幂等键（Idempotency Key）
- **完整保留图解**：所有架构图、时序图、表格与代码块均未改动，图片路径保持原样
- **中英对照阅读**：同一章节目录下同时提供 `Readme.md`（英文）与 `Readme.zh-CN.md`（中文），便于对照学习

## 📖 章节导航

| 章节 | 主题 | 中文笔记 | 英文笔记 |
|:---:|---|---|---|
| 01 | 从零扩展到百万级用户 | [中文](./01.%20Scaling/Readme.zh-CN.md) | [英文](./01.%20Scaling/Readme.md) |
| 02 | 粗略估算（Back-of-the-Envelope Estimation） | [中文](./02.%20Back%20Of%20the%20Envelope%20Estimation/Readme.zh-CN.md) | [英文](./02.%20Back%20Of%20the%20Envelope%20Estimation/Readme.md) |
| 03 | 系统设计面试的框架 | [中文](./03.%20System%20Design%20Framework/Readme.zh-CN.md) | [英文](./03.%20System%20Design%20Framework/Readme.md) |
| 04 | 设计限流器（Rate Limiter） | [中文](./04.%20Rate%20Limiter/Readme.zh-CN.md) | [英文](./04.%20Rate%20Limiter/Readme.md) |
| 05 | 设计一致性哈希（Consistent Hashing） | [中文](./05.%20Consistent%20Hashing/Readme.zh-CN.md) | [英文](./05.%20Consistent%20Hashing/Readme.md) |
| 06 | 设计键值存储（Key-Value Store） | [中文](./06.%20Key-Value%20Store/Readme.zh-CN.md) | [英文](./06.%20Key-Value%20Store/Readme.md) |
| 07 | 在分布式系统中设计唯一 ID 生成器 | [中文](./07.%20Unique-Id%20Generator/Readme.zh-CN.md) | [英文](./07.%20Unique-Id%20Generator/Readme.md) |
| 08 | 设计 URL 短链接服务 | [中文](./08.%20URL%20Shortener/Readme.zh-CN.md) | [英文](./08.%20URL%20Shortener/Readme.md) |
| 09 | 设计网络爬虫（Web Crawler） | [中文](./09.%20Web%20Crawler/Readme.zh-CN.md) | [英文](./09.%20Web%20Crawler/Readme.md) |
| 10 | 设计通知系统 | [中文](./10.%20Notification%20System/Readme.zh-CN.md) | [英文](./10.%20Notification%20System/Readme.md) |
| 11 | 设计信息流系统（News Feed System） | [中文](./11.%20News%20Feed%20System/Readme.zh-CN.md) | [英文](./11.%20News%20Feed%20System/Readme.md) |
| 12 | 设计聊天系统 | [中文](./12.%20Chat%20System/Readme.zh-CN.md) | [英文](./12.%20Chat%20System/Readme.md) |
| 13 | 设计搜索自动补全系统 | [中文](./13.%20Search%20Autocomplete/Readme.zh-CN.md) | [英文](./13.%20Search%20Autocomplete/Readme.md) |
| 14 | 设计 YouTube | [中文](./14.%20Youtube/Readme.zh-CN.md) | [英文](./14.%20Youtube/Readme.md) |
| 15 | 设计 Google Drive | [中文](./15.%20Google%20Drive/Readme.zh-CN.md) | [英文](./15.%20Google%20Drive/Readme.md) |
| 16 | 附近位置服务（Proximity Service） | [中文](./16.%20Proximity%20Service/Readme.zh-CN.md) | [英文](./16.%20Proximity%20Service/Readme.md) |
| 17 | 附近好友（Nearby Friends） | [中文](./17.%20Nearby%20Friends/README.zh-CN.md) | [英文](./17.%20Nearby%20Friends/README.md) |
| 18 | 设计 Google Maps | [中文](./18.%20Google%20Maps/README.zh-CN.md) | [英文](./18.%20Google%20Maps/README.md) |
| 19 | 分布式消息队列 | [中文](./19.%20Distributed%20Message%20Queue/README.zh-CN.md) | [英文](./19.%20Distributed%20Message%20Queue/README.md) |
| 20 | 指标监控与告警系统 | [中文](./20.%20Metrics%20Monitoring%20and%20Alerting%20System/README.zh-CN.md) | [英文](./20.%20Metrics%20Monitoring%20and%20Alerting%20System/README.md) |
| 21 | 广告点击事件聚合 | [中文](./21.%20Ad%20Click%20Event%20Aggregation/README.zh-CN.md) | [英文](./21.%20Ad%20Click%20Event%20Aggregation/README.md) |
| 22 | 酒店预订系统 | [中文](./22.%20Hotel%20Reservation%20System/README.zh-CN.md) | [英文](./22.%20Hotel%20Reservation%20System/README.md) |
| 23 | 分布式邮件服务 | [中文](./23.%20Distributed%20Email%20Service/README.zh-CN.md) | [英文](./23.%20Distributed%20Email%20Service/README.md) |
| 24 | 类 S3 对象存储 | [中文](./24.%20S3-like%20Object%20Storage/README.zh-CN.md) | [英文](./24.%20S3-like%20Object%20Storage/README.md) |
| 25 | 实时游戏排行榜 | [中文](./25.%20Real-time%20Gaming%20Leaderboard/README.zh-CN.md) | [英文](./25.%20Real-time%20Gaming%20Leaderboard/README.md) |
| 26 | 支付系统 | [中文](./26.%20Payment%20System/README.zh-CN.md) | [英文](./26.%20Payment%20System/README.md) |
| 27 | 数字钱包 | [中文](./27.%20%20Digital%20Wallet/README.zh-CN.md) | [英文](./27.%20%20Digital%20Wallet/README.md) |
| 28 | 股票交易所 | [中文](./28.%20Stock%20Exchange/README.zh-CN.md) | [英文](./28.%20Stock%20Exchange/README.md) |

## 🗂 目录结构

```
system-design-notes-zh/
├── Readme.md            # 英文导航索引
├── Readme.zh-CN.md      # 中文导航与说明（本文件）
├── 01. Scaling/         # 第 1 章：从零扩展到百万级用户
│   ├── Readme.md        #   英文笔记
│   ├── Readme.zh-CN.md  #   中文笔记
│   └── images/          #   架构图（中英文共用）
└── ...                  # 第 2~28 章结构相同
```

## 🤝 使用与贡献

- 本仓库仅用于学习交流，可自由浏览与参考
- 欢迎提交 **Pull Request** 修正翻译、补充备注或改进排版
- 如发现翻译错误或链接失效，欢迎提 **Issue** 反馈
- 翻译时遵循以下约定：保留代码块与图片路径；关键术语首次出现附英文对照

## 📚 附加资源（Additional Resources）

### 限流（Rate Limiting）
- [Circuit Breaker Algorithm](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Uber Rate Limiter](https://github.com/uber-go/ratelimit/blob/master/ratelimit.go)

### 一致性哈希（Consistent Hashing）
- [Consistent Hashing](https://tom-e-white.com/2007/11/consistent-hashing.html)
- [CS168: Introduction and Consistent Hashing:]( http://theory.stanford.edu/~tim/s16/l/l1.pdf)
- [Apache Cassandra](http://www.cs.cornell.edu/Projects/ladis2009/papers/Lakshman-ladis2009.PDF)
- [Scaling Discord](https://blog.discord.com/scaling-elixir-f9b8e1e7c29b)
- [Google Maglev](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/44824.pdf)

### 键值存储（Key-Value Store）
- [Amazon Dynamo](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Cassandra Architecture](https://docs.datastax.com/en/archived/cassandra/3.0/cassandra/architecture/archIntro.html)
- [Google BigTable Architecture](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)
- [Amazon Dynamo DB Internals](https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html)
- [Design Patterns in Amazon Dynamo DB](https://www.youtube.com/watch?v=HaEPXoXVf2k)
- [Internals of Amazon Dynamo DB](https://www.youtube.com/watch?v=yvBR71D0nAQ)

### 唯一 ID 生成器（Unique-ID Generator）
- [Ticket Servers: Distributed Unique Primary Keys on the Cheap](https://code.flickr.com/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap)
- [Snowflake](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake.html)

### 网络爬虫（Web Crawler）
- [Web Crawling](http://infolab.stanford.edu/~olston/publications/crawling_survey.pdf)
- [Google Dynamic Rendering](https://developers.google.com/search/docs/guides/dynamic-rendering)

### 聊天系统（Chat Systems）
- [How Discord stores billions of messages](https://discord.com/blog/how-discord-stores-billions-of-messages)
- [Flannel: An Application-Level Edge Cache to Make Slack Scale](https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale/)

### 搜索自动补全（Search Autocomplete）
- [How We Built Prefixy](https://medium.com/@prefixyteam/how-we-built-prefixy-a-scalable-prefix-search-service-for-powering-autocomplete-c20f98e2eff1)
- [Prefix Hash Tree](https://people.eecs.berkeley.edu/~sylvia/papers/pht.pdf)

### YouTube
- [YouTube Architecture](http://highscalability.com/youtube-architecture)
- [YouTube scalability 2012](https://www.youtube.com/watch?v=w5WVu624fY8)
- [Transcoding Videos at Scale](https://www.egnyte.com/blog/2018/12/transcoding-how-we-serve-videos-at-scale/)
- [Facebook Video Broadcasting](https://engineering.fb.com/ios/under-the-hood-broadcasting-live-video-to-millions/)
- [Netflix Video Encoding at Scale](https://netflixtechblog.com/high-quality-video-encoding-at-scale-d159db052746)
- [Netflix Shot based encoding](https://netflixtechblog.com/optimized-shot-based-encodes-now-streaming-4b9464204830)

### Google Drive
- [Differential Synchronization](https://neil.fraser.name/writing/sync/)
- [Differential Synchronization Video](https://www.youtube.com/watch?v=S2Hp_1jqpY8)
- [How We've Scaled Dropbox](https://www.youtube.com/watch?v=PE4gwstWhmc&feature=youtu.be)

---

## ⚖️ 致谢

- 笔记内容基于 **Alex Xu** 的 *System Design Interview* 系列书籍（第 1、2 卷）
- 英文原版笔记：[github.com/liquidslr/system-design-notes](https://github.com/liquidslr/system-design-notes)
- 在线浏览：https://pagefy.io/system-design/system-design-interview-by-alex-xu

**注意：** 这些笔记仍在不断完善中。
