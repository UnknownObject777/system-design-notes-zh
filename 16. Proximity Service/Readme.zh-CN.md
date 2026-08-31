# 第 16 章：附近位置服务（Proximity Service）

## 简介
**附近位置服务（proximity service）** 用于查找附近的地点，如餐厅、酒店、加油站和其他商家。**Google Maps** 和 **Yelp** 等应用中的此类功能，能够帮助用户发现在指定半径内的场所。


## 步骤 1：理解问题并确定范围

### **功能需求**
1. **根据用户位置（纬度、经度）和搜索半径搜索商家**。
2. **允许商家所有者**添加、更新或删除商家（非实时）。
3. **在请求时提供详细的商家信息**。

### **非功能需求**
- **低延迟**：用户应获得快速响应。
- **数据隐私**：符合 GDPR 和 CCPA 法规。
- **高可用性**：应对繁忙地点的高峰期流量激增。

### **粗略估算（Back-of-the-Envelope）**
- **1 亿日活跃用户**。
- **系统中存在 2 亿商家**。
- **搜索 QPS 计算**：
  - 用户每天进行 **5 次搜索**。
  - **搜索 QPS** = (100M × 5) / 86,400 ≈ **5,000 QPS**。

---

## 步骤 2：高层设计

### **API 设计**
#### **搜索附近的商家**
GET /v1/search/nearby

- **请求参数**：
  - `latitude`：用户所在位置的纬度。
  - `longitude`：用户所在位置的经度。
  - `radius`：搜索半径（默认值：5000 米）。

#### **商家 API**
| API 端点                     | 描述                                      |
|-----------------------------------|--------------------------------------------------|
| `GET /v1/businesses/{id}`         | 获取商家的详细信息                    |
| `POST /v1/businesses`             | 新增商家                              |
| `PUT /v1/businesses/{id}`         | 更新商家详情                         |
| `DELETE /v1/businesses/{id}`      | 将商家从系统中移除               |


### **数据模型**
- 由于以下两个功能非常常用，读取量很高，因此关系型数据库（如 MySQL）是一个不错的选择。
  - 搜索附近的商家
  - 查看商家的详细信息

### **数据模式（Data Schema）**
- 关键的数据库表是商家表（business table）和地理空间索引表（geospatial index table）。
- 商家表包含商家的详细信息。

### **高层系统架构**
系统由两部分组成：基于位置的服务（Location Based Service，LBS）和商家相关服务（business related service）。

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="HLD" width="400" />
</div>

- **基于位置的服务（Location-Based Service，LBS）**：
  - 处理基于位置的搜索查询。
  - 只读服务，没有写请求。
  - QPS 很高，尤其是在人口密集地区的高峰时段，而且系统是无状态的（stateless）。
- **商家服务（Business Service）**：处理两类请求。
  - 商家所有者创建、更新或删除商家。
  - 客户查看商家的详细信息。
- **负载均衡器（Load Balancer）**：将流量路由到 LBS 和商家服务。
- **数据库集群（Database Cluster）**：
  - 针对读多的工作负载采用 **主从复制架构（primary-replica architecture）**。
  - LBS 读取的数据与主数据库写入的数据之间可能存在一些差异。
  - 这种不一致并不是问题，因为商家信息并非实时更新。


---

## 步骤 3：获取附近商家的算法

### **选项 1：二维搜索（朴素方法）**

<div style="margin-left:3rem">
    <img src="./images/2d-search.png" alt="2D" width="250" />
</div>

最直观的方法是画一个预定义半径的圆，然后找出圆内的所有商家。

**SQL 查询：**
```
SELECT business_id, latitude, longitude
FROM business
WHERE (latitude BETWEEN :lat - radius AND :lat + radius)
AND (longitude BETWEEN :long - radius AND :long + radius);
```
**存在的问题：**
- **效率低下**：需要扫描整个数据库。
- **受一维索引限制**（纬度/经度）。

一个可能的改进是在经度和纬度列上建立索引，虽然这略有改善，但仍然非常慢。

### 更好的方法
- 上一个方法的问题在于，数据库索引只能在一维上提高搜索速度。
- 最优的方法是使用地理空间索引（geospatial indexing）将二维数据表示为一维数据。
  - 哈希（Hash）：均匀网格（Even Grid）、GeoHash
  - 树（Tree）：四叉树（Quadtree）、Google S2、RTree

  <div style="margin-left:3rem">
    <img src="./images/geospatial-index-types.png" alt="2D" width="500" />
  </div>


### **选项 2：均匀划分的网格**

  <div style="margin-left:3rem">
    <img src="./images/even-grid.png" alt="Even Grid" width="400" />
  </div>

- **将世界划分为固定大小的网格**。
- **问题**：商家分布不均匀（城市密度高，农村地区稀疏）。

### **选项 3：GeoHash**
- 沿本初子午线（prime meridian）和赤道（equator）将地球划分为四个象限。然后将每个网格再划分为四个更小的网格。
- 每个网格可以通过交替使用经度和纬度比特位来表示。
- 重复这一细分过程。

  <div style="margin-left:3rem">
    <img src="./images/geohash.png" alt="Geohash" width="300" />
    <img src="./images/geohash-1.png" alt="Geohash" width="285" />
  </div>


- **将纬度和经度编码为单个字母数字字符串**。它有 12 个精度（级别）。
- **分层网格结构**可实现高效的搜索。
- 根据表格选择最小的 geohash 长度来确定合适的精度。
  <div style="margin-left:3rem">
    <img src="./images/geohash-radius-mapping.png" alt="Geohash Radius" width="400" />
  </div>
- GeoHash 保证两个 geohash 之间共享的前缀越长，它们就越接近。

- **挑战**：
  <div style="margin-left:3rem">
    <img src="./images/boundary-issue.png" alt="Boundary Issue" width="300" />
  </div>

  - **边界问题**（靠近网格边缘的商家可能被排除在外）。
    - 两个位置可能非常接近，但完全没有共享前缀（可能位于赤道两侧）。
    - 两个位置可能有很长的共享前缀，但属于不同的 geohash。
  - 解决方案：需要搜索相邻的网格。


### **选项 4：四叉树（Quadtree）**

  四叉树是一种树形数据结构，它递归地将二维空间划分为四个象限，每个内部节点恰好有四个子节点，分别代表该空间的四个子区域。
  - 四叉树是一种内存中的数据结构，运行在每个 LBS 服务器上，并在服务器启动时构建。

  <div style="margin-left:3rem">
    <img src="./images/quadtree.png" alt="Quadtree" width="500" />
  </div>

  - 根节点被递归地分解为 4 个象限，直到每个节点中的商家数量都不超过 x 个（此处为 100 个）为止。

  <div style="margin-left:3rem">
    <img src="./images/building-quadtree.png" alt="Building Quadtree" width="500" />
  </div>

- 四叉树索引占用的内存不多（通常在 GB 级别），可以轻松放入一台服务器。
- 由于构建树的时间复杂度为 nlogn，构建整棵树可能需要几分钟。
- **适合 k 最近邻（k-nearest）搜索查询**（例如，找到最近的加油站）。

  <div style="margin-left:3rem">
    <img src="./images/realworld-quadtree.png" alt="Real World Quadtree" width="400" />
  </div>

#### 运维考量
 - 对于大约 2 亿商家，在服务器启动时构建四叉树可能需要几分钟。
 - 构建四叉树期间，服务器无法提供流量，因此新版本应增量发布到一部分服务器上。
 - 更新商家或新增商家时，最简单的方法是增量重建四叉树（会导致大量缓存失效）。
 - 也可以在线更新四叉树，但实现起来更复杂（需要锁机制）。

### **选项 5：Google S2**
它基于希尔伯特曲线（Hilbert curve）将球面映射到一维索引。在希尔伯特曲线上彼此接近的两个点，在 1D 空间中也是接近的。


  <div style="margin-left:3rem">
    <img src="./images/hilbert-curve.png" alt="Hilbert curve" width="300" />
    <img src="./images/geofence.png" alt="Geofence" width="355" />
  </div>

- **使用希尔伯特曲线将地球划分为小单元（cell）**。
- 非常适合地理围栏（geofencing），因为它能以不同的层级覆盖任意区域。
- 地理围栏还允许定义围绕感兴趣区域的参数。
- 另一个优势是：无需固定的精度级别，我们可以指定 S2 中的最小、最大级别和最大单元数。


## 权衡对比

#### GeoHash
- 易于使用和实现——无需构建/重建树。
- 支持固定半径的结果。
- 更新索引很容易。
- 无法根据人口密度动态调整网格大小。

#### 四叉树
- 实现起来稍难一些。
- 支持获取 k 最近邻商家。
- 可以根据人口密度动态调整网格大小。
- 更新索引更复杂，因为可能需要重建整棵树。

---

## 步骤 4：扩展数据库与缓存策略

### **扩展商家表**
- **按商家 ID 分片（sharding）** 可确保数据均匀分布。
- 表中每个商家都有单独的行。

| GeoHash | 商家 ID |
|---------|------------|
| 9q9hvu  | 343        |
| 9q9hvu  | 347        |
| 9q9hvu  | 112        |

### **扩展地理空间索引**
- 可能不太适合 geohash 表。在这种情况下，所有内容都可以放入一台服务器，因此没有分片的技术理由。
- 更好的方法是使用只读副本（read-replica）来分担读取负载。



---

### **缓存策略**
最显而易见的缓存键选择是位置坐标，但它有几个问题：
 - GPS 的位置坐标并不精确。
 - 用户可能会移动，导致位置坐标发生变化。
 - 更好的键是 geohash。

| 缓存键  | 缓存值 |
|------------|------------|
| `geohash`  | 该网格中的商家 ID 列表 |
| `business_id` | 商家详细信息（名称、地址、评论等） |

---

## 步骤 5：部署策略与最终架构

### **区域与可用区**
- 在 **多个区域** 部署 LBS 和商家服务。

### **处理实时更新**
- **商家更新每天批量处理**。

### **最终系统架构**


  <div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="Final Design" width="500" />
  </div>


最终算法如下：

## 获取附近商家的步骤
1. **用户请求：**
   - 用户搜索 **500 米** 范围内的餐厅。
   - 客户端将 **纬度（37.776720）、经度（-122.416730）和半径（500 米）** 发送到 **负载均衡器**。

2. **请求转发：**
   - **负载均衡器（LB）** 将请求转发到 **基于位置的服务（LBS）**。

3. **计算 GeoHash：**
   - LBS 确定与半径匹配的 **geohash 长度**。
   - 根据对照表，**500 米对应 geohash 长度为 6**。

4. **获取相邻的 GeoHash：**
   - LBS 计算 **相邻的 geohash**，以包含附近区域。
   - 结果是列表：
     ```
     [my_geohash, neighbor1_geohash, neighbor2_geohash, ..., neighbor8_geohash]
     ```

5. **从 Redis 获取商家 ID：**
   - 对于列表中的每个 geohash，LBS 查询 **GeoHash Redis 服务器** 以获取 **商家 ID**。
   - 使用并行查询以最大限度降低延迟。

6. **获取并排序商家：**
   - LBS 从 **商家信息 Redis 服务器** 获取 **完整的商家详情**。
   - 商家按与用户位置的 **距离排序**。
   - **排序后的结果** 返回给客户端。

## 关键优化
- **并行 Redis 调用**：缩短响应时间。
- **GeoHash 索引**：确保高效的空间查询。
- **缓存**：加速商家数据的查找和检索。

该方案可确保以 **低延迟、可扩展** 的方式获取用户位置附近的商家。

---

### **选择最佳索引方法**
| 索引方法 | 优点 | 缺点 |
|----------------|------|------|
| **GeoHash** | 易于实现，适合附近位置搜索 | 边界问题，网格大小固定 |
| **四叉树** | 根据密度动态调整，支持 k 最近邻查询 | 更复杂，需要重新平衡树 |
| **Google S2** | 先进的地理围栏，用于 Google Maps | 更难实现 |

---

## 参考资料
1. [Geohash Algorithm](https://www.movable-type.co.uk/scripts/geohash.html)
2. [Quadtree Indexing](https://en.wikipedia.org/wiki/Quadtree)
3. [Google S2 Geometry](https://s2geometry.io/)