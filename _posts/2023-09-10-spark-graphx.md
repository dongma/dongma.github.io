---
layout: post
title: GraphX 实战：揪出三人团伙——一次风控场景的图算法拆解
tags: ["Spark"]
---
图算法不是"难"，是没分清 `aggregateMessages` 和 `Pregel` 这两个原语。一次风控场景里"揪出三人团伙"的需求，其实 GraphX 用 `aggregateMessages` + `mergeVertices` 三步就能搞定。这一篇从那次风控实战讲起，把 GraphX 的属性图、`triplets`、`aggregateMessages`、`Pregel` API 一次讲透。

<!-- more -->

## 序：一次风控场景的图算法拆解

2020 年初，我在一家风控公司做事后欺诈检测。一个常见的需求摆在桌上：

> "假设我们有一张图，节点是人，边表示'认识'关系。请找出所有'三人成伙'的团伙（即三个人两两都有边相连）。"

脑子里第一个反应是"遍历"——但这种暴力遍历在大图上跑不动。几亿条边、三轮嵌套循环——凌晨跑都跑不完。

我盯着那张社交关系图看了半小时，跟同事说："GraphX 三步就能搞定。"

```scala
// 1. 找出所有"两人相连"的对
val pairRDD = graph.triplets
  .flatMap { case Triplet(e, s, d) =>
    // 对每个 triplet，找出 src 和 dst 的所有"两两组合"
    ...
  }

// 2. 合并两人对，看谁出现两次（说明三人两两相连）
val mergedRDD = pairRDD.reduceByKey(_ ++ _)

// 3. 过滤出三人组
val triangles = mergedRDD.filter { case (k, v) => v.length == 2 }
```

那一周我们写完了这个算法并接入了风控告警链路。这一篇我把那次图算法拆给你看：从 GraphX 的入门，到 `aggregateMessages` 和 `Pregel` 这两个原语的差异，再到 CrimeGangRunner 实战。

## 一、GraphX 入门：VertexRDD / EdgeRDD / Graph

GraphX 用三个 RDD 表达图：

```scala
import org.apache.spark.graphx._

// 1. VertexRDD：节点 (id, 属性)
val vertices: RDD[(VertexId, String)] = sc.parallelize(Seq(
  (1L, "Alice"),
  (2L, "Bob"),
  (3L, "Charlie")
))

// 2. EdgeRDD：边 (srcId, dstId, 属性)
val edges: RDD[Edge[String]] = sc.parallelize(Seq(
  Edge(1L, 2L, "friend"),    // <-- Alice -> Bob
  Edge(2L, 3L, "friend"),    // <-- Bob -> Charlie
  Edge(1L, 3L, "friend")     // <-- Alice -> Charlie
))

// 3. Graph：组合起来
val graph = Graph(vertices, edges)
// 类型：Graph[String, String] = Graph[VD, ED]
//                                VD: vertex 属性类型
//                                ED: edge 属性类型
```

### 一个反直觉的细节：默认属性

```scala
val graph = Graph(vertices, edges, "default_value")
//                                       <-- 不存在的节点/边会拿到这个默认值
```

有向图 + 默认值很关键——`outerJoinVertices` 时，未匹配的节点会拿到默认属性。

## 二、属性图：给 vertex/edge 挂上业务属性

```scala
// 顶点属性：人名 + 年龄
case class Person(name: String, age: Int)
val vertices: RDD[(VertexId, Person)] = sc.parallelize(Seq(
  (1L, Person("Alice", 30)),
  (2L, Person("Bob", 25))
))

// 边属性：关系强度
val edges: RDD[Edge[Double]] = sc.parallelize(Seq(
  Edge(1L, 2L, 0.8)         // <-- Alice 和 Bob 的关系强度 0.8
))

val graph: Graph[Person, Double] = Graph(vertices, edges)
graph.vertices.collect().foreach(println)
// (1, Person(Alice, 30))
// (2, Person(Bob, 25))
```

### 一个值得记住的细节：`mapVertices` / `mapEdges` / `mapTriplets`

```scala
// 修改顶点属性
graph.mapVertices { case (id, person) => person.copy(age = person.age + 1) }
//       <-- 返回新 Graph[Person, ED]，原 graph 不变

// 修改边属性
graph.mapEdges(e => e.attr * 2)

// 修改三元组（triplet）属性
graph.mapTriplets(t => t.attr + t.dstAttr.age - t.srcAttr.age)
```

`mapVertices`、`mapEdges`、`mapTriplets` 都是**返回新 Graph**，原 graph 不变——GraphX 是不可变的。

## 三、triplets：从一条边看两端

```scala
val triplets = graph.triplets.collect()
// 返回 RDD[Triplet[VD, ED]]
// 每个 Triplet 含：edge + srcAttr + dstAttr

triplets.foreach(println)
// Triplet(Edge(1, 2, 0.8), Person(Alice, 30), Person(Bob, 25))
```

`Triplet` 是 `aggregateMessages` 的基础——你能在一次"看一条边"时拿到两端。

### 一个反直觉的细节：边的方向

```scala
// Edge(1, 2, 属性) 是有向边：1 -> 2
val e = Edge(1L, 2L, "friend")
// e.srcId = 1, e.dstId = 2

// 访问时：
t.srcAttr  // <-- 1 号节点（Alice）
t.dstAttr  // <-- 2 号节点（Bob）

// 想反方向遍历？用 .reverse
graph.reverse.triplets  // <-- 所有边都反过来
```

## 四、`aggregateMessages`：只发你想发的消息

`aggregateMessages` 是 GraphX 的核心原语——**只发你想发的消息，不发多余的**：

```scala
import org.apache.spark.graphx._

val messages = graph.aggregateMessages[String](
  triplet => {
    // 1. 对每个 triplet 决定发什么消息、发给谁
    triplet.sendToDst(s"I'm from ${triplet.srcAttr.name}")  // <-- 给 dst 发消息
    triplet.sendToSrc(s"I'm from ${triplet.dstAttr.name}")  // <-- 给 src 发消息
  },
  // 2. 多个消息怎么合并
  (a, b) => a + " | " + b
)
// 返回 VertexRDD[String]，每个节点一条聚合消息

messages.collect().foreach(println)
// (1, "I'm from Bob | I'm from Charlie")
// (3, "I'm from Alice | I'm from Bob")
```

### 一个值得记住的细节：`sendToSrc` vs `sendToDst`

```scala
// sendToDst(msg)：把消息发给边的目的节点（dst）
// sendToSrc(msg)：把消息发给边的源节点（src）

// 一条边 (1 -> 2)：
//   sendToDst(msg) -> 节点 2 收到 msg
//   sendToSrc(msg) -> 节点 1 收到 msg
```

这是"消息方向"的核心——**边 (1, 2) 中，1 是 src、2 是 dst**。

## 五、CrimeGangRunner 实战：揪出三人团伙

回到风控场景的需求：找三人成伙的团伙。

```scala
// 第一步：找出所有"两两相连"的对（按某一边聚合）
val pairMessages: VertexRDD[Seq[(VertexId, VertexId)]] = graph.aggregateMessages[Seq[(VertexId, VertexId)]](
  triplet => {
    // 对每个 triplet (src -> dst)，假设 src < dst（标准化）
    val (s, d) = if (triplet.srcId < triplet.dstId) (triplet.srcId, triplet.dstId)
                 else (triplet.dstId, triplet.srcId)
    triplet.sendToDst(Seq((s, d)))    // <-- 给 dst 发"自己这对的 src"
  },
  (a, b) => a ++ b                    // <-- 合并消息
)

// 第二步：统计每个节点参与的对数
// 对找到 2 次以上对的节点 = 三人团伙成员

val mergedPairs = pairMessages.mapValues(_.distinct)
val candidates = mergedPairs.filter { case (id, pairs) => pairs.length >= 2 }
// <-- 至少 2 对 = 至少 2 个不同邻居 = 三角关系

candidates.collect().foreach(println)
// (1, [(2, 3), (3, 2)])  <-- 1 跟 2、3 都相连，是团伙成员
// (2, [(1, 3), (3, 1)])
// (3, [(1, 2), (2, 1)])
```

### 一个反直觉的细节：合并"双向"边

```scala
// 如果原图是 (1, 2) 和 (2, 1) 两条边，分别处理：
//   (1, 2) 发给 dst=2：(1, 2)
//   (2, 1) 发给 dst=1：(2, 1)
// 节点 1 收到 [(2, 1)]
// 节点 2 收到 [(1, 2)]
// 但 "1 和 2 是相连的" 是同一条边！

// 解决：在发送前标准化（小 id 在前）
val (s, d) = if (srcId < dstId) (srcId, dstId) else (dstId, srcId)
sendToDst(Seq((s, d)))   // <-- (1, 2) 和 (2, 1) 都发 (1, 2)
```

我当年写 CrimeGangRunner 踩过这个坑——不算标准化会出现"假团伙"。

## 六、Pregel API：迭代式图计算（BSP）

`aggregateMessages` 是一次性聚合——所有消息一次性发完、合并、结束。

Pregel 是**迭代传播**：每轮每个节点根据收到的消息更新状态，再发新消息。**直到收敛**。

```scala
import org.apache.spark.graphx.Pregel

// 最短路径：固定起点 1，求所有节点到 1 的最短距离
val sssp = Pregel(
  graph.mapVertices((id, _) => if (id == 1L) 0.0 else Double.PositiveInfinity),  // <-- 初始值
  Double.PositiveInfinity,                                                          // <-- 初始消息
  10 // <-- 最多 10 轮
)(
  // vprog：节点收到消息后怎么更新自己的属性
  (id, attr, msg) => math.min(attr, msg),

  // sendMsg：节点想给哪些邻居发消息
  triplet => {
    if (triplet.srcAttr + triplet.attr < triplet.dstAttr) {
      Iterator((triplet.dstId, triplet.srcAttr + triplet.attr))   // <-- 给 dst 发新距离
    } else Iterator.empty                                          // <-- 否则不发
  },

  // mergeMsg：多个消息怎么合并
  (a, b) => math.min(a, b)
)

sssp.vertices.collect().foreach(println)
// (1, 0.0)
// (2, 1.5)
// (3, 2.3)
```

### 一个值得记住的细节：Pregel 的三个函数

```scala
Pregel(graph, initialMsg, maxIter)(
  vprog,    // Vertex Program：收到消息后更新属性
  sendMsg,  // Send Message：节点给哪些邻居发消息
  mergeMsg  // Merge Message：多个消息怎么合并
)
```

三个函数一起决定算法行为。`aggregateMessages` 没有 vprog（一次性）、Pregel 有 vprog（多轮迭代）。

### 一个反直觉的细节：Pregel 是 BSP 模型

```scala
// BSP = Bulk Synchronous Parallel
// 每轮：
//   1. 所有活跃节点并行运行 vprog，更新自己的属性
//   2. 所有活跃节点并行运行 sendMsg，发消息
//   3. 所有消息按 dstId 分组，运行 mergeMsg 合并
//   4. 没消息可发的节点被标记为"非活跃"，不再参与下一轮
// 直到没有活跃节点 或 达到 maxIter
```

BSP 模型的优势：**全局同步，状态确定**。每轮之间没有 race condition。

## 七、ShortestPath 实战

```scala
import org.apache.spark.graphx.Pregel

// 固定起点 = 1L，求所有节点到 1 的最短路径
val sourceId: VertexId = 1L

val initialGraph = graph.mapVertices((id, _) =>
  if (id == sourceId) 0.0 else Double.PositiveInfinity
)

val sssp = Pregel(initialGraph, Double.PositiveInfinity, maxIter = 10)(
  // vprog：拿到消息后，更新为 min(当前, 消息)
  (id, attr, msg) => math.min(attr, msg),

  // sendMsg：src 的距离 + 边权重 < dst 当前距离，就发
  triplet => {
    if (triplet.srcAttr != Double.PositiveInfinity) {
      val newDist = triplet.srcAttr + triplet.attr
      if (newDist < triplet.dstAttr) Iterator((triplet.dstId, newDist))
      else Iterator.empty
    } else Iterator.empty
  },

  // mergeMsg：取最小
  (a, b) => math.min(a, b)
)

println(sssp.vertices.collect().mkString("\n"))
```

### 我当年因为 maxIter 太小 debug 半天

```scala
Pregel(graph, initMsg, maxIter = 3) // <-- 只跑 3 轮
```

最长路径长度是 5，我只跑 3 轮——结果部分节点还是 Infinity。

**经验：maxIter 设为"图直径"或更大**。直径 = 最长最短路径长度。

## 八、`aggregateMessages` vs Pregel 选择表

| 维度 | `aggregateMessages` | `Pregel` |
|------|--------------------|---------|
| 轮数 | 1 轮 | 多轮（最多 maxIter） |
| 状态更新 | 无（只发消息） | 有（vprog 更新属性） |
| 适用 | 邻居聚合、度数统计 | 最短路径、PageRank、连通分量 |
| 性能 | 更快 | 稍慢（多轮迭代） |

```scala
// 一次性聚合：用 aggregateMessages
graph.aggregateMessages[String](...)

// 迭代传播：用 Pregel
Pregel(graph, ...)(vprog, sendMsg, mergeMsg)
```

## 本篇小结

这一篇我们覆盖了 GraphX 的 8 个话题——入门、属性图、triplets、aggregateMessages、CrimeGangRunner、Pregel、ShortestPath、选择表——它们都围绕一个中心思想：

> **GraphX 的两个核心原语——`aggregateMessages`（单跳聚合）和 `Pregel`（迭代传播）——覆盖了 90% 的图算法场景。**

- GraphX 用 VertexRDD / EdgeRDD / Graph 三个抽象表达图。
- `triplets` 是"看一条边时拿两端属性"的入口。
- `aggregateMessages` 一次性聚合，按 `sendToSrc` / `sendToDst` 决定消息方向。
- Pregel 三件套：`vprog` / `sendMsg` / `mergeMsg`，多轮迭代直到收敛。
- 一次性聚合用 `aggregateMessages`，迭代传播用 `Pregel`。
- 处理"两两相连"或"三方关系"时，**先标准化边方向**——避免重复计数。

## 🔜 下篇预告

这一篇我们讲了图算法——把"实体间的关系"作为第一公民。

但所有的招式都要落在性能上——下一篇讲 **Spark 性能优化全景**。我会把一个跑了 40 分钟的 join 调到 5 分钟的实战拆给你看：分区、By-Key、序列化、查询计划、缓存、倾斜修复——这 6 把刀怎么叠加使用，最终把作业从 40 分钟推到 5 分钟。