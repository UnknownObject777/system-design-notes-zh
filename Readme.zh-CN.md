# [System Design Interview - An Insider's Guide (Vol 1 and 2)](https://bytebytego.com/courses/system-design-interview)
这些笔记基于 System Design Interview 系列书籍——[Vol 1 and Vol 2 2nd Ed](https://www.goodreads.com/book/show/54109255-system-design-interview-an-insider-s-guide)

在此查看笔记：https://pagefy.io/system-design/system-design-interview-by-alex-xu

**注意：** 这些笔记仍在不断完善中。


 * [第 1 章——从零扩展到百万级用户](./01.%20Scaling/)
 * [第 2 章——粗略估算（Back-of-the-Envelope Estimation）](./02.%20Back%20Of%20the%20Envelope%20Estimation/)
 * [第 3 章——系统设计面试的框架](./03.%20System%20Design%20Framework/)
 * [第 4 章——设计限流器（Rate Limiter）](./04.%20Rate%20Limiter//)
 * [第 5 章——设计一致性哈希（Consistent Hashing）](./05.%20Consistent%20Hashing/)
 * [第 6 章——设计键值存储（Key-Value Store）](./06.%20Key-Value%20Store/)
 * [第 7 章——在分布式系统中设计唯一 ID 生成器](./07.%20Unique-Id%20Generator/)
 * [第 8 章——设计 URL 短链接服务](./08.%20URL%20Shortener/)
 * [第 9 章——设计网络爬虫（Web Crawler）](./09.%20Web%20Crawler/)
 * [第 10 章——设计通知系统](./10.%20Notification%20System/)
 * [第 11 章——设计信息流系统（News Feed System）](./11.%20News%20Feed%20System/)
 * [第 12 章——设计聊天系统](./12.%20Chat%20System/)
 * [第 13 章——设计搜索自动补全系统](./13.%20Search%20Autocomplete/)
 * [第 14 章——设计 YouTube](./14.%20Youtube/)
 * [第 15 章——设计 Google Drive](./15.%20Google%20Drive/)
 * [第 16 章——附近位置服务（Proximity Service）](./16.%20Proximity%20Service/)
 * [第 17 章——附近好友（Nearby Friends）](./17.%20Nearby%20Friends/)
 * [第 18 章——设计 Google Maps](./18.%20Google%20Maps/)
 * [第 19 章——分布式消息队列](./19.%20Distributed%20Message%20Queue/)
 * [第 20 章——指标监控与告警系统](./20.%20Metrics%20Monitoring%20and%20Alerting%20System/)
 * [第 21 章——广告点击事件聚合](./21.%20Ad%20Click%20Event%20Aggregation/)
 * [第 22 章——酒店预订系统](./22.%20Hotel%20Reservation%20System/)
 * [第 23 章——分布式邮件服务](./23.%20Distributed%20Email%20Service/)
 * [第 24 章——类 S3 对象存储](./24.%20S3-like%20Object%20Storage/)
 * [第 25 章——实时游戏排行榜](./25.%20Real-time%20Gaming%20Leaderboard/)
 * [第 26 章——支付系统](./26.%20Payment%20System/)
 * [第 27 章——数字钱包](./27.%20%20Digital%20Wallet/)
 * [第 28 章——股票交易所](./28.%20Stock%20Exchange/)


# 附加资源（Additional Resources）

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
- [Ticket Servers: Distributed Unique Primary Keys on the Cheap](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap)
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
- [How We’ve Scaled Dropbox](https://www.youtube.com/watch?v=PE4gwstWhmc&feature=youtu.be)