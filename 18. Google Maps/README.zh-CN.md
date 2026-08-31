# 第 18 章：Google Maps

## 简介

我们将设计一个简化版的 **Google Maps**。

关于 Google Maps 的一些事实：
 * 于 2005 年上线。
 * 提供各种服务——卫星图像、街道地图、实时交通状况、路线规划。
 * 到 2021 年，拥有 10 亿日活跃用户，覆盖全球 99% 的地区，每天有 2500 万次实时位置信息更新。

---

## 步骤 1：理解问题并确定设计范围

面试者与面试官之间的问答示例：
 * C：我们要面对多少日活跃用户？
 * I：10 亿 DAU。
 * C：我们应该关注哪些功能？
 * I：位置更新、导航、ETA（预计到达时间）、地图渲染。
 * C：道路数据有多大？我们能获得这些数据吗？
 * I：我们从各种来源获得了道路数据，有 TB 级的原始数据。
 * C：我们应该把交通状况考虑进去吗？
 * I：是的，为了准确估算时间，我们应该考虑。
 * C：不同的出行方式怎么办——步行、骑行、驾车？
 * I：我们应该支持这些方式。
 * C：多目的地路线怎么办？
 * I：在面试范围内我们先不关注这个。
 * C：商家地点和照片呢？
 * I：好问题，但不需要考虑这些。

我们将重点放在三个关键功能上——用户位置更新、包括 ETA 在内的导航服务、地图渲染。

### **非功能需求**

- **准确性**：用户不应得到错误的指引。
- **导航流畅**：用户应享受流畅的地图渲染体验。
- **数据和电池消耗**：客户端应尽可能少地消耗数据和电池。这对移动设备很重要。
- 通用可用性和可扩展性要求。

### **地图基础（Map 101）**

在进入设计之前，我们需要了解一些与地图相关的概念。

#### 定位系统

世界是一个绕轴旋转的球体。位置由纬度（你在南北方向上的位置）和经度（你在东西方向上的位置）定义：

<div style="margin-left:3rem">
    <img src="./images/partitioning-system.png" alt="partitioning-system" width="500" />
</div>

#### 从 3D 到 2D

将点从 3D 转换到 2D 平面的过程称为"地图投影（map projection）"。

有多种不同的实现方式，每种方式各有利弊。几乎所有的投影方式都会扭曲实际的几何形状。

<div style="margin-left:3rem">
    <img src="./images/map-projections.png" alt="map-projections" width="500" />
</div>

Google Maps 选择了 Mercator 投影的一种改进版本，称为"Web Mercator"。

#### 地理编码（Geocoding）

地理编码是将地址转换为地理坐标的过程。

相反的过程称为"反向地理编码"。

实现这一点的一种方法是使用插值（interpolation）——利用来自不同来源（例如 GIS）的数据，将街道网络映射到地理坐标空间。

#### GeoHash 编码（Geohashing）

GeoHash 是一种将地理区域编码为字母和数字字符串的编码系统。

它把世界描绘成一个平面，并递归地将其细分为四个象限：

<div style="margin-left:3rem">
    <img src="./images/geohashing.png" alt="geohashing" width="500" />
</div>

#### 地图渲染

地图渲染通过瓦片化（tiling）完成。不是把整个地图渲染成一张巨大的定制图像，而是把世界分解成更小的瓦片。

客户端只下载相关的瓦片，并像拼接马赛克一样进行渲染。

不同缩放级别对应不同的瓦片。客户端根据自身的缩放级别选择合适的瓦片。

例如，缩放到整个世界，只需要下载一个代表整个世界的 256x256 瓦片。

#### 面向导航算法的道路数据处理

在大多数路由算法中，交叉口被表示为节点，道路被表示为边：

<div style="margin-left:3rem">
    <img src="./images/road-representation.png" alt="road-representation" width="500" />
</div>

大多数导航算法使用 Dijkstra 或 A* 算法的改进版本。

寻路性能对图的大小很敏感。要在大规模下工作，我们无法把整个世界表示成一个图并对其运行算法。

相反，我们使用一种类似于瓦片化的技术——我们把世界细分为越来越小的图。

路由瓦片持有对相邻瓦片的引用，算法在遍历相互连接的瓦片时可以拼接出更大的道路图：

<div style="margin-left:3rem">
    <img src="./images/routing-tiles.png" alt="routing-tiles" width="500" />
</div>

这种技术使我们能够显著减少内存带宽，并且只加载给定起点/终点对所需的瓦片。

然而，对于更长距离的路线，拼接小而详细的路由瓦片仍然很耗时/耗内存。相反，存在不同详细程度的路由瓦片，算法会根据我们要前往的目的地使用适当详细程度的瓦片：

<div style="margin-left:3rem">
    <img src="./images/map-routing-hierarchical.png" alt="map-routing-hierarchical" width="500" />
</div>

### **粗略估算（Back-of-the-Envelope）**

对于存储，我们需要存储：
 * 世界地图——根据我们需要存储的所有瓦片估算约为 70PB，但考虑到了对非常相似瓦片（例如广阔的沙漠）的压缩。
 * 元数据——大小可忽略不计，因此我们可以将其从计算中省略。
 * 道路信息——以路由瓦片的形式存储。

导航请求的估算 QPS——10 亿 DAU，每周使用 35 分钟 → 每天 50 亿分钟。
假设 GPS 更新请求是批量发送的，我们得到 20 万 QPS，峰值负载时为 100 万 QPS。

---

## 步骤 2：提出高层设计并达成共识

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

### **位置服务（Location service）**

<div style="margin-left:3rem">
    <img src="./images/location-service.png" alt="location-service" width="500" />
</div>

它负责记录用户的位置更新：
 * 位置更新每 `t` 秒发送一次。
 * 位置数据流可用来随时间改进服务，例如提供更准确的 ETA、监控交通数据、检测封闭道路、分析用户行为等。

我们可以不必一直向服务器发送位置更新，而是在客户端批量累积这些更新，然后批量发送：

<div style="margin-left:3rem">
    <img src="./images/location-update-batches.png" alt="location-update-batches" width="500" />
</div>

尽管有这样的优化，对于 Google Maps 这种规模的系统来说，负载仍然很大。因此，我们可以利用针对大量写入进行优化的数据库，例如 Cassandra。

我们还可以利用 Kafka 对位置更新进行高效的流式处理，用于进一步的分析。

位置更新请求负载示例：

```
POST /v1/locations
Parameters
  locs: JSON encoded array of (latitude, longitude, timestamp) tuples.
```

### **导航服务（Navigation service）**

该组件负责在合理的时间内找到 A 和 B 之间的快速路线（稍微有点延迟是可以的）。路线不必是最快的，但准确性很重要。

请求负载示例：

```
GET /v1/nav?origin=1355+market+street,SF&destination=Disneyland
```

响应示例：

```json
{
  "distance": {"text":"0.2 mi", "value": 259},
  "duration": {"text": "1 min", "value": 83},
  "end_location": {"lat": 37.4038943, "Ing": -121.9410454},
  "html_instructions": "Head <b>northeast</b> on <b>Brandon St</b> toward <b>Lumin Way</b><div style=\"font-size:0.9em\">Restricted usage road</div>",
  "polyline": {"points": "_fhcFjbhgVuAwDsCal"},
  "start_location": {"lat": 37.4027165, "lng": -121.9435809},
  "geocoded_waypoints": [
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJwZNMti1fawwRO2aVVVX2yKg",
       "types" : [ "locality", "political" ]
    },
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJ3aPgQGtXawwRLYeiBMUi7bM",
       "types" : [ "locality", "political" ]
    }
  ],
  "travel_mode": "DRIVING"
}
```

交通变化和重新路由暂不考虑，这些将在深入设计部分处理。

### **地图渲染（Map rendering）**

在客户端保存全部地图瓦片数据集是不可行的，因为其规模达到 PB 级。

它们需要根据客户端的位置和缩放级别按需从服务器获取。

什么时候应该获取新瓦片——用户放大/缩小地图时，以及导航中驶向新瓦片的过程中。

地图瓦片应该如何提供给客户端？
 * 可以动态构建，但这会给服务器带来巨大负载，而且使缓存变得困难。
 * 地图瓦片基于客户端可以计算的 geohash 静态提供。它们可以静态存储并从 CDN 提供。

<div style="margin-left:3rem">
    <img src="./images/static-map-tiles.png" alt="static-map-tiles" width="500" />
</div>

CDN 使用户能够从离用户最近的接入点服务器（point-of-presence，POP）获取地图瓦片，从而最大程度地降低延迟：

<div style="margin-left:3rem">
    <img src="./images/cdn-vs-no-cdn.png" alt="cdn-vs-no-cdn" width="500" />
</div>

确定地图瓦片时可考虑的选择：
 * 地图瓦片的 geohash 可以在客户端计算。如果是这样，我们应注意，既然强制客户端更新很难，就要从长远考虑长期坚持使用这种瓦片计算方式。
 * 或者，我们可以提供一个简单的 API，代客户端计算地图瓦片 URL，代价是多一次额外的 API 调用。

<div style="margin-left:3rem">
    <img src="./images/map-tile-url-calculation.png" alt="map-tile-url-calculation" width="500" />
</div>

---

## 步骤 3：深入设计

### **数据模型**

让我们讨论如何存储我们面对的各种类型的数据。

#### 路由瓦片（Routing tiles）

初始道路数据集来自不同的来源。随着时间推移，它基于位置更新数据不断改进。

道路数据是非结构化的。我们有一个周期性的离线处理流水线，将这些原始数据转换为我们的应用所需的基于图的路由瓦片。

由于我们不需要任何数据库特性，因此不把这些瓦片存储在数据库中。我们可以将它们存储在 S3 对象存储中，同时积极缓存它们。

我们还可以利用一些库将邻接表高效地压缩为二进制文件。

#### 用户位置数据

用户位置数据对于更新交通状况和进行各种其他分析非常有用。

这类数据可以使用 Cassandra 存储，因为它的本质是写多负载。

示例行：

<div style="margin-left:3rem">
    <img src="./images/user-location-data-torw.png" alt="user-location-data-row" width="500" />
</div>

#### 地理编码数据库

该数据库存储经纬度对与地点之间的键值对。

由于我们读操作频繁而写操作不频繁，可以使用 Redis，因为它的读访问速度很快。

#### 预计算的世界地图图像

正如我们所讨论的，我们将预计算地图瓦片图像并存储在 CDN 中。

<div style="margin-left:3rem">
    <img src="./images/precomputed-map-tile-image.png" alt="precomputed-map-tile-image" width="500" />
</div>

### **服务（Services）**

#### 位置服务

让我们重点关注该服务的数据库设计，以及用户位置存储方式的细节。

<div style="margin-left:3rem">
    <img src="./images/location-service-diagram.png" alt="location-service-diagram" width="500" />
</div>

我们可以使用 NoSQL 数据库来应对位置更新上的大量写入负载。我们优先考虑可用性而非一致性，因为用户位置数据经常变化，并且会随着新更新的到来而变得过期。

我们选择 Cassandra 作为数据库，因为它完全符合我们的所有要求。

我们将存储的示例行：

<div style="margin-left:3rem">
    <img src="./images/user-location-row-example.png" alt="user-location-row-example" width="500" />
</div>

 * `user_id` 是分区键（partition key），以便快速访问特定用户的所有位置更新。
 * `timestamp` 是聚类键（clustering key），以便按收到位置更新的时间对数据进行排序存储。

我们还利用 Kafka 将位置更新流式传输给出于各种目的需要位置更新的其他服务：

<div style="margin-left:3rem">
    <img src="./images/location-update-streaming.png" alt="location-update-streaming" width="500" />
</div>

#### 渲染地图

地图瓦片以不同的缩放级别存储。在最低缩放级别下，整个世界用单个 256x256 瓦片表示。

随着缩放级别的增加，地图瓦片数量以四倍增加：

<div style="margin-left:3rem">
    <img src="./images/zoom-level-increases.png" alt="zoom-level-increases" width="500" />
</div>

我们可以采用的一个优化是，不在网络上传输完整的图像信息，而是将瓦片表示为矢量（路径与多边形），让客户端动态渲染瓦片。

这将大幅节省带宽。

#### 导航服务

该服务负责查找最快的路线：

<div style="margin-left:3rem">
    <img src="./images/navigation-service.png" alt="navigation-service" width="500" />
</div>

让我们逐一介绍该子系统中的每个组件。

首先，我们有地理编码服务，它将地址解析为经纬度对的位置。

请求示例：

```
https://maps.googleapis.com/maps/api/geocode/json?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA
```

响应示例：

```json
{
   "results" : [
      {
         "formatted_address" : "1600 Amphitheatre Parkway, Mountain View, CA 94043, USA",
         "geometry" : {
            "location" : {
               "lat" : 37.4224764,
               "lng" : -122.0842499
            },
            "location_type" : "ROOFTOP",
            "viewport" : {
               "northeast" : {
                  "lat" : 37.4238253802915,
                  "lng" : -122.0829009197085
               },
               "southwest" : {
                  "lat" : 37.4211274197085,
                  "lng" : -122.0855988802915
               }
            }
         },
         "place_id" : "ChIJ2eUgeAK6j4ARbn5u_wAGqWA",
         "plus_code": {
            "compound_code": "CWC8+W5 Mountain View, California, United States",
            "global_code": "849VCWC8+W5"
         },
         "types" : [ "street_address" ]
      }
   ],
   "status" : "OK"
}
```

路线规划器（route planner）服务根据当前交通状况计算一条针对出行时间优化的建议路线。

最短路径（shortest-path）服务对对象存储中的路由瓦片运行 A* 算法的一种变体，以计算最优路径：
 * 它接收起点/终点对，将其转换为经纬度对，并从这些坐标对推导出 geohash，进而确定路由瓦片。
 * 算法从起始路由瓦片开始遍历，直到找到一条通往目标瓦片的足够好的路径。

<div style="margin-left:3rem">
    <img src="./images/shortest-path-service.png" alt="shortest-path-service" width="500" />
</div>

路线规划器调用 ETA 服务，基于机器学习算法获取预计时间，根据交通数据预测 ETA。

排序（ranker）服务负责根据用户传递的筛选条件（例如避开收费公路或高速公路的标记）对不同的可行路径进行排序。

更新（updater）服务异步更新一些重要的数据库，使其保持最新。

#### 改进——自适应 ETA 与重新路由

我们可以做的一项改进是根据新获得的交通数据自适应地更新行进中的路线。

一种实现方法是将当前正在沿路线导航的用户存储在数据库中，即存储他们将要经过的所有瓦片。

数据可能如下所示：

```
user_1: r_1, r_2, r_3, …, r_k
user_2: r_4, r_6, r_9, …, r_n
user_3: r_2, r_8, r_9, …, r_m
...
user_n: r_2, r_10, r21, ..., r_l
```

如果某个瓦片发生交通事故，我们可以找出所有路径经过该瓦片的用户，并为他们重新路由。

为了减少数据库中存储的瓦片数量，我们可以改为存储起点路由瓦片，以及不同分辨率级别的若干路由瓦片，直到也包含目标瓦片为止：

```
user_1, r_1, super(r_1), super(super(r_1)), ...
```

<div style="margin-left:3rem">
    <img src="./images/adaptive-eta-data-storage.png" alt="adaptive-eta-data-storage" width="500" />
</div>

使用这种方法，我们只需要检查用户的最终瓦片是否包含交通事故瓦片，即可判断用户是否受到影响。

我们还可以跟踪导航用户的全部可能路线，如果存在更快的重新路线，则通知他们。

#### 推送协议（Delivery protocols）

我们有多种可选方案，支持我们从服务器主动向客户端推送数据：
 * 移动推送通知不可行，因为负载受限，而且不适用于 Web 应用。
 * WebSocket 通常比长轮询（long-polling）更优，因为它在服务器上的计算开销更小。
 * 我们也可以使用服务器发送事件（server-sent events，SSE），但更倾向于 WebSocket，因为它支持双向通信，这在对例如最后一公里配送这样的功能时很有用。

---

## 步骤 4：总结

这是我们最终的设计：

<div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="final-design" width="500" />
</div>

我们还可以提供一个附加功能——多目的地导航，它可以出售给 Uber 或 Lyft 等企业客户，用于确定游览一组地点的最优路径。