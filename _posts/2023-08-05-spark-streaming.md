---
layout: post
title: Spark 流式进化史：Watermark 之前，我以为自己已经懂了流式
tags: ["Spark"]
---
Spark 流式处理有两条完全不同的路：DStream 是 RDD 的流式封装，Structured Streaming 是把流当成表。我用了一年 DStream 之后被通知迁移——那次迁移让我意识到 **Spark 流式只有最新一代才适合做"事件时间"窗口**。这一篇讲清楚 DStream 与 Structured Streaming 的架构差异、事件时间 vs 处理时间、Watermark 的语义、流与批的 JOIN、Kafka 集成——这是一条"从流式入门到理解 Spark 流式灵魂"的路径。

<!-- more -->

## 序：用了一年 DStream 之后，我被通知迁移

2019 年我在一家电商公司处理实时订单流。技术栈是：

```
Kafka → Spark Streaming (DStream) → Cassandra
```

我自以为了解 Spark 流式——直到产品经理找我：

> "我们要把订单事件的处理时间从 '处理时刻' 改成 '业务时刻'，不然用户下单 30 秒后才看到订单状态，体验不对。"

那一刻我才意识到：**我处理的是"处理时间"，不是"事件时间"**。DStream 窗口是基于"消息到达 Spark 的时间"，不是"消息实际产生的时间"。如果用户下单时网络延迟 5 秒，我的窗口聚合就错过了正确的事件。

那次迁移让我从 DStream 转到 Structured Streaming。迁移完我才看明白：**DStream 是 RDD 的流式封装，Structured Streaming 是把流当成无界表**。前者按"处理时间"切窗，后者可以按"事件时间"切窗 + Watermark 容错。

这一篇把那次踩坑 + 迁移的经验讲给你：从 DStream 到 Structured Streaming 的架构差异、窗口演进、Watermark 灵魂、Kafka 集成。

## 一、流式的两条路：DStream 与 Structured Streaming

Spark 流式处理有**两套完全独立的 API**：

| 维度 | DStream（低阶） | Structured Streaming（高阶） |
|------|-----------------|----------------------------|
| 抽象 | RDD 的流式封装 | 无界表（unbounded table） |
| 时间模型 | 处理时间 | 事件时间 + 处理时间 |
| 容错 | Checkpoint | Checkpoint + WAL |
| 数据源 | 多种（含 KafkaUtils） | 多种（含 kafka format） |
| 输出 | Action 风格 | Output mode（append/complete/update） |
| 状态管理 | `updateStateByKey`、`mapWithState` | 任意复杂状态（groupBy + agg） |
| 与 DataFrame 集成 | ❌ | ✅（直接是 DataFrame） |

### 一个值得记住的对比

```scala
// DStream：基于 RDD 流
val lines = streamingContext.socketTextStream("localhost", 9999)
val words = lines.flatMap(_.split(" "))
val counts = words.map(x => (x, 1)).reduceByKey(_ + _)
counts.print()

// Structured Streaming：基于表
val df = spark.readStream.format("socket").option("host", "localhost").option("port", 9999).load()
val words = df.select(explode(split($"value", " ")).as("word"))
val counts = words.groupBy("word").count()
val query = counts.writeStream.outputMode("complete").format("console").start()
query.awaitTermination()
```

**同样的功能，Structured Streaming 写起来像 SQL**，DStream 像 RDD。

### 我当年因为 DStream 折腾的事

DStream 的 API 跟 RDD 像，但**跟 DataFrame 完全不通**。如果你想用 DataFrame 的优化（Catalyst、Tungsten），用 DStream 就要绕一圈：先把流转成 DataFrame、再 join、再 back 到流。

Structured Streaming 直接就是 DataFrame——**优化全继承**。

## 二、DStream：低阶 API

DStream（Discretized Stream，离散化流）把流数据按批（batch interval）切成"小 RDD"：

```scala
import org.apache.spark.streaming.{Seconds, StreamingContext}

val ssc = new StreamingContext(spark.sparkContext, Seconds(5))
//                                                            <-- 每 5 秒切一批

// socket 数据源
val lines = ssc.socketTextStream("localhost", 9999)
//                <-- 从 socket 读，按行切

// 经典 wordcount
val words = lines.flatMap(_.split(" "))
val pairs = words.map(w => (w, 1))
val counts = pairs.reduceByKey(_ + _)

counts.print()         // <-- 打印到 driver 控制台
ssc.start()            // <-- 启动流
ssc.awaitTermination() // <-- 阻塞直到流结束
```

理解 DStream 的核心：**每个 batch interval 是一次 mini RDD 操作**。每 5 秒 Spark 把这 5 秒内的数据当成一个 RDD 算一次。

### 一个反直觉的细节：DStream 是 lazy 的吗

```scala
val lines = ssc.socketTextStream(...)                   // <-- Transformation，lazy
val counts = lines.flatMap(...).reduceByKey(...)        // <-- Transformation，lazy
counts.print() // <-- Action，但 print 是注册一个输出，不是触发计算
ssc.start()                                             // <-- 启动流后才真正开始计算
```

DStream 的 transformation 是 lazy，但 `start()` 之后就开始持续计算，每个 batch interval 触发一次。

## 三、窗口：`window` / `reduceByKeyAndWindow` / 滑动窗口

DStream 提供四种窗口操作：

```scala
// 1. window：把多个 batch 的数据合并成 RDD
val windowedWords = words.window(Seconds(30))    // <-- 30 秒的窗口

// 2. countByWindow：直接数行数
val count = words.countByWindow(Seconds(30))    // <-- 30 秒内的单词总数

// 3. reduceByWindow：自定义 reduce
val sum = words.countByWindow(Seconds(30))      // <-- 类似

// 4. reduceByKeyAndWindow：按 key 的窗口聚合
val pairCounts = pairs.reduceByKeyAndWindow(
  (a: Int, b: Int) => a + b,                   // <-- reduce 函数
  Seconds(30),                                 // <-- 窗口长度
  Seconds(10)                                  // <-- 滑动步长
)
```

### 滑动窗口 vs 滚动窗口

```scala
// 滚动窗口（步长 = 窗口长度）：每个窗口独立
val t1 = words.window(Seconds(30))             // <-- [0-30s] 一次
val t2 = words.window(Seconds(30))             // <-- [30-60s] 又一次

// 滑动窗口（步长 < 窗口长度）：窗口有重叠
val t3 = words.window(Seconds(30), Seconds(10))
// <-- [0-30s], [10-40s], [20-50s]...
```

实际项目里**滑动步长一般比窗口短**，比如窗口 1 分钟、步长 10 秒——每 10 秒输出一次过去 1 分钟的数据。

### 反向函数：优化窗口增量计算

```scala
val pairs = ...

// 加一个反向函数，Spark 知道怎么减（避免全量重算）
val pairCounts = pairs.reduceByKeyAndWindow(
  (a: Int, b: Int) => a + b,            // <-- 加：新数据进来
  (a: Int, b: Int) => a - b,            // <-- 减：旧数据移出窗口
  Seconds(30),                            // <-- 窗口
  Seconds(10)                             // <-- 步长
)
```

反向函数让 Spark **只算增量**，不重算整个窗口——性能提升 5-10 倍。我的电商项目用了这个优化，CPU 降了一半。

## 四、Structured Streaming：高阶 API

Structured Streaming 把流当成无界表——**每个新事件是表的新行**：

```scala
// 读 socket 数据源
val df = spark.readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()                                  // <-- 返回 DataFrame[String]，每行是一行 socket 数据

// SQL 风格的 groupBy
val words = df.select(explode(split($"value", " ")).as("word"))
val counts = words.groupBy("word").count()

// 输出
val query = counts.writeStream
  .outputMode("complete")               // <-- 输出完整结果（区别于 append/update）
  .format("console")                    // <-- 输出到 console
  .start()

query.awaitTermination()
```

### 三种输出模式

| 模式 | 行为 | 适用 |
|------|------|------|
| `append` | 只输出新增的行（默认） | 大多数场景 |
| `complete` | 输出完整结果（全量重写） | 聚合（`groupBy` + `count`） |
| `update` | 输出变化的行 | 聚合 + 状态变化 |

```scala
// append：聚合不能用（必须是有界聚合）
df.select($"value").writeStream.outputMode("append").start()

// complete：聚合用 complete
df.groupBy("word").count().writeStream.outputMode("complete").start()
```

## 五、事件时间 vs 处理时间

**这是 Spark 流式最重要的概念区分**。

```scala
// 处理时间：Spark 收到事件的时间（毫秒时间戳）
df.groupBy(window($"processingTime", "10 minutes")).count()

// 事件时间：事件实际产生的时间（业务时间戳）
df.groupBy(window($"eventTime", "10 minutes")).count()
```

### 我当年被这个坑过

2019 年的电商项目，我用 DStream 算"过去 5 分钟订单数"：

```scala
// DStream
val orders = KafkaUtils.createStream(...)
orders.countByWindow(Seconds(300))           // <-- 默认按处理时间
```

问题：用户 10:00:00 下单、Kafka 延迟 10 秒、Spark 10:00:10 才收到。如果窗口是"10:00 - 10:05"，这条订单算到了 10:00 之后那个窗口。

DStream 没有"事件时间"概念——**只有按"消息到达 Spark 时间"切窗**。

迁移到 Structured Streaming：

```scala
val orders = spark.readStream.format("kafka")
  .option("subscribe", "orders")
  .load()
  .select(from_json($"value".cast("string"), schema).as("data"))
  .select($"data.*", $"data.eventTime".cast("timestamp").as("event_time"))

orders.groupBy(window($"event_time", "5 minutes")).count()
  .writeStream.outputMode("complete").format("console").start()
```

**事件时间是业务时间，处理时间是 Spark 时间**。分析时必须用事件时间。

## 六、Watermark：让迟到数据有边界

事件时间解决了"按业务时间切窗"，但还有个问题：**迟到数据**。

```
10:00 产生的事件，因为 Kafka 重试，10:30 才到 Spark
这属于"迟到数据"
```

无 Watermark 时：Spark 会无限等下去，10:30 收到就重新计算 10:00-10:05 的窗口——**状态会无限增长**。

Watermark 是"我愿意等多长时间"的声明：

```scala
import org.apache.spark.sql.streaming.Trigger

val orders = spark.readStream.format("kafka")...

orders
  .withWatermark("event_time", "10 minutes") // <-- 最多等 10 分钟
  .groupBy(window($"event_time", "5 minutes"))
  .count()
  .writeStream
  .outputMode("complete")
  .trigger(Trigger.ProcessingTime("10 seconds"))      // <-- 每 10 秒触发一次微批
  .option("checkpointLocation", "/path/checkpoint")
  .start()
```

`withWatermark("event_time", "10 minutes")` 意思是：**事件时间 T 的数据，最多等到 T + 10 分钟。超过就丢弃**。

### 一个值得记住的细节：Watermark 的语义

```scala
// 事件时间 = 10:00
// Watermark 延迟 = 10 分钟
// Spark 等待到 10:10
// 10:10 之后，10:00 这个事件就当作"过期"处理

// 实际触发条件：
// 当前最大事件时间 = 10:15
// Watermark = 10:15 - 10 = 10:05
// 10:05 之前的窗口被认为"完整"
```

Watermark 不是"等 N 分钟"，是"事件时间推进 N 分钟后，旧的窗口被认定完整"。我第一次看官方文档理解错了——以为"延迟 10 分钟就是等 10 分钟"，实际上是"事件时间推进后才推进"。

## 七、流与批、流与流的 JOIN

Structured Streaming 支持两类流式 JOIN：

### 流-批 JOIN（流 join 静态维度表）

```scala
// 这是流（订单）
val orders = spark.readStream.format("kafka")...

// 这是静态维度（用户表）
val users = spark.read.parquet("path/to/users").cache()

// JOIN：给订单补上用户信息
val result = orders.join(users, "city")
result.writeStream.format("console").start()
```

流-批 JOIN 简单：流每次触发时，去查一次维度表（Spark 会 broadcast）。

### 流-流 JOIN（流 join 流）

```scala
val clicks = spark.readStream.format("kafka").option("topic", "clicks").load()...
val impressions = spark.readStream.format("kafka").option("topic", "impressions").load()...

// 流-流 JOIN 需要 Watermark
clicks
  .withWatermark("clickTime", "5 minutes")
  .join(
    impressions.withWatermark("impressionTime", "5 minutes"),
    expr("clickerId = impressionerId AND clickTime BETWEEN impressionTime AND impressionTime + INTERVAL 1 HOUR"),
    "leftOuter"
  )
  .writeStream.start()
```

流-流 JOIN 的硬性要求：**两个流都必须有 Watermark**。否则 Spark 不知道"两个流之间怎么对齐时间"。

## 八、Kafka 集成

Structured Streaming 集成 Kafka 非常简单：

```scala
val df = spark.readStream
  .format("kafka")                                    // <-- 用 kafka 数据源
  .option("kafka.bootstrap.servers", "host1:9092,host2:9092")
  .option("subscribe", "topic1")                      // <-- 订阅 topic
  .option("startingOffsets", "latest")                // <-- 从最新开始（earliest = 从头）
  .load()                                              // <-- DataFrame: key | value | topic | partition | offset | timestamp | timestampType

// value 是二进制，需要解析
val parsed = df.selectExpr("CAST(value AS STRING)").select(from_json($"value", schema).as("data"))
```

写 Kafka：

```scala
val query = parsed
  .selectExpr("to_json(struct(*)) AS value")          // <-- 序列化成 JSON
  .writeStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:9092")
  .option("topic", "topic2")
  .option("checkpointLocation", "/path/checkpoint")
  .start()

query.awaitTermination()
```

### 一个反直觉的细节：Checkpoint 必填

Structured Streaming 的 Kafka 集成**必须**指定 `checkpointLocation`——否则重启后会丢失状态（watermark、聚合结果）。

```scala
.option("checkpointLocation", "/path/checkpoint")   // <-- 必填
```

我用第一次忘了加，重启后 watermark 从 0 开始算——所有过期数据都重算了一次。

## 本篇小结

这一篇覆盖了 Spark 流式的 8 个话题——两条路、DStream、窗口、Structured Streaming、事件时间、Watermark、JOIN、Kafka——它们都围绕一个中心思想：

> **DStream 是 RDD 的流式封装，Structured Streaming 是把流当成表——后者才是 Spark 流式的未来。**

- DStream 按"处理时间"切窗，Structured Streaming 可以按"事件时间"切窗。
- DStream 的窗口用 `window` / `reduceByKeyAndWindow`；Structured Streaming 用 SQL 风格的 `groupBy`。
- Watermark 是"等多久"的声明——超过 Watermark 的事件被丢弃。
- 流-批 JOIN 简单；流-流 JOIN 必须双 Watermark。
- Kafka 集成直接用 `format("kafka")`，Checkpoint 必填。

## 🔜 下篇预告

这一篇我们讲了 Spark 流式——把数据流当成表。

但有些问题天然不是表，而是**图**：社交网络、推荐系统、欺诈检测——这些场景里"实体之间的关系"才是核心。下一篇讲 **GraphX**：用图算法揪出三人成伙的犯罪团伙。我会从一个真实的面试题开场，把 `aggregateMessages` 和 `Pregel` 的差异讲透。