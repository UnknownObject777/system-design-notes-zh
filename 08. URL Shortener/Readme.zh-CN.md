# 第 8 章：设计 URL 短链接服务

## 引言
本章讨论如何设计类似 TinyURL 的 URL 短链接服务。系统的主要目标包括**URL 缩短**、**重定向**以及处理大流量时的**高可扩展性**。

### 需求
- 生成的短 URL 必须**唯一**，并且**尽可能短**。
- 每天处理 **1 亿个 URL 生成请求**，并支持 10 年的容量。
- 支持**高效的读操作**，读写比例约为 10:1。
- 存储 3650 亿条记录，10 年内大约需要 **365 TB** 的存储空间。

---

## 步骤 1：高层设计

### API 端点
1. **URL 缩短：**  
   - 端点：`POST api/v1/data/shorten`  
   - 参数：`{longUrl: longURLString}`  
   - 返回：`shortURL`

2. **URL 重定向：**  
   - 端点：`GET api/v1/shortUrl`  
   - 返回：用于重定向的 `longURL`。

    <p align="center">
    <img src="./images/url-redirection.png" alt="URL 重定向" width="600">
    </p>

### URL 重定向
- **301 重定向：** 301 重定向表示所请求的 URL 已“永久”移动到长 URL。浏览器会缓存该响应，后续对同一 URL 的请求将不再发送到短链接服务。
- **302 重定向：** 临时重定向；适用于统计点击次数等分析场景。

### URL 缩短
<p align="center">
    <img src="./images/url-shortening.png" alt="URL 缩短" width="400">
</p>

- 使用**哈希函数**生成短 URL，将长 URL 映射到唯一的短链接版本。
- 哈希函数必须满足以下要求：
    - 每个 longURL 必须被哈希成唯一的 hashValue。
    - 每个 hashValue 都能映射回对应的 longURL。
    

---

## 步骤 2：深入设计

### 数据模型
将 `<shortURL, longURL>` 映射存储在关系型数据库中，以优化内存使用。表结构包括：
- `id`（主键）、
- `shortURL`、
- `longURL`。

    <img src="./images/table-schema.png" alt="表结构" width="300">

### 哈希函数
#### 1. Base 62 转换：
- 使用字符 `[0-9, a-z, A-Z]` 对数字进行编码，共提供 **62 个可用字符**。
- 进制转换是 URL 短链接服务中常用的一种方法。
- 可以为短 URL 分配一个唯一 ID，然后将该 ID 进行 Base 62 转换，从而得到短 URL。
- 7 位字符的哈希最多支持 **3.5 万亿个唯一 URL**，足以满足 3650 亿个 URL 的需求。

**示例：**  
将 ID `2009215674938` 转换为 Base 62：
- `2009215674938` → `zn9edcu`。

#### 2. 哈希 + 冲突解决：
- 使用 CRC32、MD5 或 SHA-1 等哈希函数。

    <img src="./images/hash-function.png" alt="哈希函数" width="500">

- 一种方法是取哈希值的前 7 个字符；不过这种方法可能导致哈希冲突。
- 为了解决冲突，可以递归地追加一个预定义的字符串，直到不再发生冲突，但这种方法成本较高。
- 使用**布隆过滤器（Bloom Filter）**高效地解决冲突。

    <p align="center">
    <img src="./images/url-lookup.png" alt="URL 查找" width="500">
    </p>

### 方案对比

-  **哈希 + 冲突解决：**
    - 短 URL 长度固定
    - 不需要唯一 ID 生成器
    - 可能发生冲突，需要解决冲突
    - 无法预知下一个可用的短 URL，因为它不依赖 ID

- **Base 62 转换**
    - 长度不固定，会随着 ID 增长而增加
    - 需要唯一 ID 生成器
    - 不会发生冲突
    - 如果 ID 按 1 递增，很容易找到下一个短 URL（这可能带来安全隐患）


---

### URL 缩短流程

<p align="center">
    <img src="./images/url-shortening-flow.png" alt="URL 缩短流程" width="500">
</p>

1. 检查 `longURL` 是否已存在于数据库中。
2. 如果已存在，返回现有的 `shortURL`。
3. 否则：
   - 使用**分布式 ID 生成器**生成唯一 ID。
   - 使用 Base 62 将 ID 转换为 `shortURL`。
   - 将 `<id, shortURL, longURL>` 映射存储到数据库中。



---

### URL 重定向流程
<p align="center">
    <img src="./images/url-redirecting-flow.png" alt="URL 重定向流程" width="600">
</p>

1. 用户点击某个 `shortURL`。
2. 查询 `<shortURL, longURL>` 映射：
   - 先检查**缓存**以获得更快的访问速度。
   - 如果缓存中不存在，则查询数据库。
3. 将用户重定向到 `longURL`。


---

## 其他考虑因素
### 限流器（Rate Limiter）
- 通过设置每个 IP 的请求限制来防止滥用。

### 可扩展性
1. **Web 层：** 无状态，可通过添加/移除 Web 服务器进行扩展。
2. **数据库层：** 使用复制（replication）和分片（sharding）。

### 分析
- 收集点击率、来源和时间戳等数据，用于业务洞察。

### 高可用性与可靠性
- 通过数据库复制和容错设计，确保服务的一致性和可靠性。