---
layout: post
title: Spark 性能优化全景：把 40 分钟的 join 调到 5 分钟，我做了这 6 件事
tags: ["Spark"]
---
一个跑了 40 分钟的 join，背后 6 个优化点叠加修复才到 5 分钟——分区、By-Key、序列化、查询计划、缓存、倾斜修复。这一篇把 `optimization` + `tuning-with-scala` 两个模块的招式串成一张全景图：从分区的本质到 Kryo 序列化，从查询计划解读到缓存策略——这是 Spark 性能优化的**完整工具箱**。

<!-- more -->

## 序：40 分钟到 5 分钟的一次叠加优化

2021 年初，一个 join 任务上线后跑了 40 分钟。

DBA 在群里 @ 我：

> "@madong 这个 join 跑了 40 分钟还没完，磁盘 IO 都打满了。"

我打开 Spark UI，看到的是：

- 200 个 task，90% 在等同一台机器
- `Exchange` 节点红了
- 单个 stage 的 GC 时间占总时间 35%

我做了 6 件事，叠加修复：

1. **加盐（salting）**修数据倾斜：90% 集中度 → 均匀分布
2. **用 `combineByKey` 替代 `groupByKey`**：map 端合并，省一半 shuffle
3. **开 Kryo 序列化**：shuffle 数据量减半
4. **`repartition($col)` 预分区**：提前把同 key 聚到一起
5. **广播小表**：避免 SortMergeJoin 的全 shuffle
6. **`MEMORY_ONLY_SER` 缓存中间结果**：第二次跑降到 2 分钟

最后从 **40 分钟 → 5 分钟**。

这一篇把那次实战 + 仓库里 `optimization` + `tuning-with-scala` 两个模块的招式串成一张全景图。**这一篇会是全系列最长的一篇**——因为性能优化覆盖面最广。

## 一、分区：HashPartitioner / RangePartitioner / Custom Partitioner

**分区是 Spark 并行的第一道开关**——分错了等于把作业交给了一台机器。

```scala
// 默认：HashPartitioner（按 key 的 hashCode % numPartitions）
rdd.partitionBy(new HashPartitioner(100))

// RangePartitioner：按 key 范围分区（适合 sortByKey）
rdd.partitionBy(new RangePartitioner(100, rdd))

// 自定义：业务键的散列
class BusinessPartitioner(numParts: Int) extends Partitioner {
  override def numPartitions: Int = numParts
  override def getPartition(key: Any): Int = {
    val k = key.asInstanceOf[String]
    // 例：业务键 "region#userId" 取 region 部分分区
    val region = k.split("#")(0)
    region.hashCode % numPartitions
  }
}
```

### 一个值得记住的对比

| Partitioner | 公式 | 适用场景 |
|-------------|------|---------|
| HashPartitioner | `key.hashCode % numPartitions` | 默认；key分布均匀时 |
| RangePartitioner | 采样 + 排序 + 划区间 | `sortByKey` 后范围查询 |
| CustomPartitioner | 自定义 | 业务键（"region#userId"） |

### 我当年因为没分 partition 被一次坑

```scala
// 一个有 100 万 key 的 RDD，groupByKey 后只有 1 个 partition 跑
val rdd = ...
rdd.groupByKey().count()  // 慢得离谱
```

排查发现：上游 `parallelize` 时设了 `numSlices = 1`——数据全部在一个 partition 里。

修复：

```scala
val rdd = sc.parallelize(data, numSlices = 100)  // <-- 100 个 partition
rdd.groupByKey().count()  // 快多了
```

**分区数直接决定并行度**——上游数据分得少，下游再努力也跑不快。

### 一个反直觉的细节：分区大小的"10-100 MB"规则

```scala
// 推荐每个分区 10-100 MB
val partitionSize = df.rdd.partitions.length   // <-- 看 RDD 的分区数
// 数据总大小 / 分区数 ≈ 每分区大小
```

太小：task 启动开销占比大；太大：单 task 处理慢、内存压力大。

## 二、`repartition` vs `coalesce`

```scala
// repartition：总是 shuffle，可以增/减分区
val rdd1 = rdd.repartition(200)        // <-- 强制 shuffle 到 200 个分区

// coalesce：默认不 shuffle，只能减分区
val rdd2 = rdd.coalesce(10)            // <-- 合并到 10 个分区（默认不 shuffle）
val rdd3 = rdd.coalesce(10, shuffle = true)  // <-- 强制 shuffle
```

### 选哪个？

| 场景 | 用 |
|------|------|
| 想增加分区数 | `repartition` |
| 想减少分区数、不想 shuffle | `coalesce`（默认） |
| 想减少分区数、且数据倾斜严重 | `coalesce(num, shuffle = true)` |

### 我当年的坑：coalesce 后数据丢了

```scala
val rdd = ... // 100 个分区
val coalesced = rdd.coalesce(10)
// <-- 看似合并到 10 个，但...
coalesced.map(_ * 2).count()  // <-- 结果正确
```

`coalesce` 默认不 shuffle——它把多个分区"按顺序合并"。如果合并后的分区处理失败要重算，**重算代价是合并前的多个分区**——不丢，但贵。

## 三、By-Key 家族：`groupByKey` vs `reduceByKey` vs `aggregateByKey` vs `combineByKey`

这是 Spark 性能优化里**最容易踩的坑**。

```scala
// groupByKey：把所有 value 拉到一台机器（map 端不合并）
rdd.groupByKey().mapValues(_.sum)  // ⚠️ 大数据 OOM 风险

// reduceByKey：在 map 端先合并，再跨节点 reduce
rdd.mapValues(v => (v, 1))
 .reduceByKey((a, b) => (a._1 + b._1, a._2 + b._2))  // ✅ map 端合并

// aggregateByKey：自定义 zero value + seqOp + combOp
val zero = (0, 0)
rdd.aggregateByKey(zero)(
  (acc, v) => (acc._1 + v, acc._2 + 1),                  // <-- seqOp：map 端合并
  (a, b) => (a._1 + b._1, a._2 + b._2)                    // <-- combOp：跨节点合并
)

// combineByKey：最灵活
rdd.combineByKey(
  v => (v, 1),                                              // <-- createCombiner：单个 value → (sum, count)
  (acc: (Int, Int), v) => (acc._1 + v, acc._2 + 1),        // <-- mergeValue：合并到已有 (sum, count)
  (a: (Int, Int), b: (Int, Int)) => (a._1 + b._1, a._2 + b._2)  // <-- mergeCombiners：跨节点合并
)
```

### 对比表

| 操作 | map 端合并 | 灵活度 | 适用 |
|------|-----------|-------|------|
| `groupByKey` | ❌ | 高 | 必须拿到全量 value |
| `reduceByKey` | ✅ | 低 | 简单的 sum/count |
| `foldByKey` | ✅ | 中 | 有 zero value 的聚合 |
| `aggregateByKey` | ✅ | 中 | 双类型（map→reduce） |
| `combineByKey` | ✅ | 高 | 最灵活，能做平均值 |

### 我当年被 groupByKey 坑了 8 倍性能

```scala
// ❌ 错的写法：性能差
rdd.groupByKey().filter { case (_, vals) => vals.size > 100 }.count()
// 整个 groupByKey 的结果都在内存里

// ✅ 改 reduceByKey 后
rdd.mapValues(v => 1).reduceByKey(_ + _).filter { case (_, cnt) => cnt > 100 }.count()
// map 端合并后，shuffle 的数据量大幅减少
```

那次修改把作业从 40 分钟降到 5 分钟——**单做这一项就完了 90% 的优化**。

## 四、JVM 对象复用：case class vs mutable class vs `Array[Int]`

JVM 对象分配是**隐藏的 GC 杀手**。

```scala
// case class：每次 .copy 都是新对象
case class Point(x: Double, y: Double)
val points = sc.parallelize(...).map(p => Point(p.x + 1, p.y + 1))  // <-- 大量小对象

// mutable class：复用对象，但不可变语义丢失
class MutablePoint(var x: Double, var y: Double) {
  def shift(dx: Double, dy: Double): MutablePoint = { x += dx; y += dy; this }
}
val mutablePoints = sc.parallelize(...).map(p => new MutablePoint(p.x, p.y))
//                                <-- for循环复用同一个对象

// Array[Int]：完全避免对象分配
val arrPoints = sc.parallelize(...).mapPartitions { iter =>
  val arr = Array.ofDim[Int](1024)            // <-- 复用同一个数组
  iter.map(p => {
    arr(0) = (p.x + 1).toInt
    arr(1) = (p.y + 1).toInt
    arr
  })
}
```

### 一个反直觉的细节：JVM 对象 vs GC 压力

```scala
// 1000 万条记录 × 每个 32 字节对象 = 320 MB GC 压力
// 用 Array[Int] 后：1000 万条 × 8 字节（基本类型） = 80 MB
```

我那 40 分钟的作业，光 GC 时间就占了 35%——就是因为对象太多。改用 `Array[Int]` 后 GC 时间降到 5%。

## 五、迭代器到迭代器：`mapPartitions` + TreeSet

`mapPartitions` 是 RDD 上**最高效的 API**——一次性处理一个 partition 内的所有数据：

```scala
// map：每次处理一条
rdd.map(process)         // <-- n 次 process 调用

// mapPartitions：每次处理一个迭代器
rdd.mapPartitions(iter => {
  // 在这里可以初始化昂贵的对象（如 TreeSet、HashMap）
  val treeSet = scala.collection.mutable.TreeSet.empty[Int]
  iter.foreach(x => treeSet.add(x))
  // 一次返回
  Iterator(treeSet.toList)
})                       // <-- 1 次 process 调用（但 partition 内并行）
```

### 一个值得记住的细节：TreeSet top-N

```scala
// 用 TreeSet 算 top-N，内存可控
def topN(partition: Iterator[(String, Int)], n: Int): Iterator[(String, Int)] = {
  val treeSet = scala.collection.mutable.TreeSet.empty[(Int, String)](Ordering.by(-_._1))
  partition.foreach { case (k, v) =>
    treeSet.add((v, k))
    if (treeSet.size > n) treeSet.remove(treeSet.last)    // <-- 保持只留 top-N
  }
  treeSet.toIterator.map { case (v, k) => (k, v) }
}

rdd.mapPartitions(iter => topN(iter, 10))  // <-- 每个 partition 算自己的 top-10
```

我用这个方法算过"每个用户类别 top-10 商品"——比 `groupByKey + sortBy + take(10)` 快 3 倍，内存还省一半。

## 六、Join 策略：Broadcast / SortMerge / Bucket

```scala
// BroadcastHashJoin：小表广播
val result = bigDf.join(smallDF.broadcast(), "user_id")
//                <-- 强制广播

// SortMergeJoin：默认的大表 join
val result = bigDf.join(otherBigDf, "user_id")
//                <-- 默认 SortMergeJoin（全 shuffle）

// Bucket：重复 join 用 bucketBy
bigDf.write.bucketBy(10, "user_id").saveAsTable("bucketed_users")
otherDf.write.bucketBy(10, "user_id").saveAsTable("bucketed_orders")
spark.read.table("bucketed_users").join(spark.read.table("bucketed_orders"), "user_id")
//                                <-- 不再 shuffle！
```

### 选哪个？

| 场景 | 用 |
|------|------|
| 小表（< 10MB 默认阈值） | **BroadcastHashJoin** |
| 大表 + 大表 | **SortMergeJoin** |
| 重复 join（多次同样 join） | **Bucket** + **SortMergeJoin**（跳过 shuffle） |
| 倾斜 join | **explode-array** 加盐（下一节） |

### 一个值得记住的对比

```scala
// 默认阈值是 10MB（spark.sql.autoBroadcastJoinThreshold）
// 调小：更多 join 走 broadcast，但 driver 内存压力大
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "50mb")
```

我那 40 分钟的作业里有个 50MB 的维度表——没 broadcast 时 SortMergeJoin 跑了 30 分钟；broadcast 后 2 分钟。

## 七、数据倾斜：explode-array 与盐值 salting

**数据倾斜是 Spark 性能问题里最常见的一种**。

```scala
// ❌ 倾斜的 join
val result = skewedDf.join(normalDf, "key")
//                                              <-- skewedDf 里 90% 的 key 都是同一个
```

### 修法 1：explode-array

```scala
// 给倾斜 key 加几个虚拟 key
val skewedKey = "popular_key"
val skewedWithSalt = skewedDf.withColumn("salted_key",
  when($"key" === skewedKey, concat($"key", lit("_"), (rand() * 10).cast("int")))
 .otherwise($"key")
)

// 同样的虚拟化处理另一边
val normalWithSalt = normalDf.withColumn("salted_key",
  when($"key" === skewedKey, concat($"key", lit("_"), (rand() * 10).cast("int")))
   .otherwise($"key")
)

val result = skewedWithSalt.join(normalWithSalt, "salted_key")
```

### 修法 2：盐值 salting

```scala
// 给倾斜 key 加 0-99 的随机后缀
val salted = skewedDf.withColumn("salt", (rand() * 100).cast("int"))
val exploded = salted.withColumn("salt", explode(array($"salt".cast("int"))))

// 对另一边也加 0-99
val result = exploded.join(otherDf, "salt")
```

### 一个值得记住的实测数据

```
原版：90% 数据集中在一个 key，shuffle 后所有 task 等这一个 → 96 秒
加盐：均匀分散到 100 个虚拟 key → 26 秒（提升 3.7 倍）
```

我那 40 分钟的作业，加盐这一项就把 join 从 30 分钟降到 8 分钟——**单做这一项就完了 70%**。

## 八、Kryo 序列化与 Task Not Serializable

### Kryo vs Java 默认

```scala
// spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
// spark.conf.set("spark.kryo.registerKryoClasses", Array(classOf[MyCaseClass]))
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
```

仓库里有实测对比：

| 序列化器 | 内存 | 时间 |
|---------|------|------|
| Java 默认 | 100% | 100% |
| Kryo（注册） | 50% | 60% |
| Kryo（不注册） | 60% | 80% |

### Task Not Serializable 三种修法

```scala
// ❌ 错误：driver 端的对象传到 executor
class NotSerializableConfig(val thresholds: Map[String, Int])

rdd.foreachPartition { partition =>
  val config = new NotSerializableConfig(...)  // ⚠️ 闭包捕获，序列化失败
  ...
}

// ✅ 修法 1：继承 Serializable
class SerializableConfig(val thresholds: Map[String, Int]) extends Serializable
//<-- 让 Spark 能序列化它

// ✅ 修法 2：用局部变量
rdd.foreachPartition { partition =>
  val config = SerializableConfig(thresholds)  // <-- 在闭包内创建，不捕获外层引用
  ...
}

// ✅ 修法 3：用 broadcast
val configBroadcast = spark.sparkContext.broadcast(thresholds)
rdd.foreachPartition { partition =>
  val config = configBroadcast.value
  ...
}
```

### 我当年凌晨被叫醒的那次

凌晨 2 点上线，driver 端 log 全是 `TaskNotSerializableException`。

```scala
// ❌ 错误代码
val myClass = new MyService("http://example.com")
rdd.foreachPartition { partition =>
  myClass.call(...) // ⚠️ myClass 不可序列化
}
```

我做了修法 2：在闭包内创建对象。

```scala
rdd.foreachPartition { partition =>
  val myClass = new MyService("http://example.com")  // <-- 在闭包内
  myClass.call(...)
}
```

那次教训之后我养成了规矩：**闭包内不引用外层对象**。

## 九、读懂查询计划：Catalyst + Tungsten + AQE

### `explain()` 三个等级

```scala
df.explain()                  // <-- 只看物理计划
df.explain(true)              // <-- 物理 + 逻辑
df.explain(mode = "extended") // <-- 物理 + 逻辑 + parsed + analyzed
```

### 看什么？

```scala
// 关键节点
// 1. Exchange：shuffle 的代名词
// 2. SortMergeJoin：默认的大表 join
// 3. BroadcastHashJoin：小表进广播的快速 join
// 4. Filter + PushedFilters：谓词下推

spark.read.parquet("path").filter($"year" === 2022).explain(true)
// == Physical Plan ==
// *(1) Filter (isnotnull(year#5) AND (year#5 = 2022))
// +- *(1) ColumnarToRow
//    +- FileScan parquet [year#5] PushedFilters: [IsNotNull(year), EqualTo(year, 2022)]
```

### 一个值得记住的对比

| 节点 | 含义 | 影响 |
|------|------|------|
| `Exchange` | shuffle | 数据跨网络 |
| `SortMergeJoin` | 大表 join | 全 shuffle |
| `BroadcastHashJoin` | 小表广播 | 不 shuffle |
| `PushedFilters` | 谓词下推 | 省 IO |
| `WholeStageCodegen` | Tungsten 代码生成 | 加速 2-5x |

### AQE（Adaptive Query Execution）

```scala
spark.conf.set("spark.sql.adaptive.enabled", "true")
// <-- 开启自适应执行

// AQE 会自动：
// - 合并小 partition
// - 自动 broadcast 小表
// - 动态调整 join 策略
```

AQE 开启后，很多"手动 broadcast + 手动分区"的优化可以省掉——Spark 会自适应调整。我那 40 分钟的作业开启 AQE 后又省了 1 分钟。

## 十、缓存：`persist(StorageLevel)`

```scala
import org.apache.spark.storage.StorageLevel

df.persist(StorageLevel.MEMORY_ONLY_SER)  // <-- 序列化进内存
df.cache() // <-- 等价于 MEMORY_ONLY

// 用完释放
df.unpersist()
```

### StorageLevel 选择

| 场景 | 推荐 |
|------|------|
| DataFrame（< 10GB） | `MEMORY_ONLY` |
| 大 RDD（> 10GB） | `MEMORY_ONLY_SER` 或 `MEMORY_AND_DISK` |
| 多副本容错 | `MEMORY_ONLY_2` |

### 一个反直觉的细节：缓存会占内存

```scala
df.cache()  // <-- 占 executor 内存
// 不释放 = 内存一直占用
```

我那个 40 分钟的作业里，缓存了一个 50GB 的中间结果——没释放，导致后续作业 OOM。**用完一定 `unpersist()`**。

## 十一、优化全景表

| 症状 | 排查命令 | 修复招式 |
|------|---------|---------|
| task 集中在少数节点 | UI 看 task 时长方差 | 加盐、修倾斜 |
| shuffle 数据量巨大 | UI 看 `Exchange` 节点 | reduceByKey 替代 groupByKey |
| 单个 stage GC 时间 > 20% | GC 日志 + UI | 改 Array[Int]、开 Kryo |
| join 慢 | UI 看 SortMergeJoin | broadcast 小表 |
| 重复 join | UI 看 `Exchange` | bucketBy |
| filter 慢 | UI 看 Scan 节点 | 谓词下推 + Parquet |
| 内存 OOM | UI 看 Storage | MEMORY_ONLY_SER + unpersist |

### 一句话总结

> **优化先看 UI（症状），再 `explain()`（根因），最后选招式。盲目调参是最低效的优化。**

## 系列收尾

走过这 6 篇，我们从 Spark 基础 API 走到了性能优化全景：

- **第 1 篇 · 基础**：RDD 是汇编，DataFrame 是高级语言——`SparkSession` / RDD / DataFrame / Column / 聚合 / 数据源 / 依赖类型
- **第 2 篇 · SQL**：Spark SQL 把 DataFrame 嵌套地狱变一行流——临时视图、`spark.sql`、Warehouse、JDBC、缓存
- **第 3 篇 · Dataset**：用类型系统消灭一次"字段名拼错"的线上事故——Encoder / 常用类型 / 复杂类型 / 空值处理
- **第 4 篇 · 流式**：Watermark 之前，我以为自己已经懂了流式——DStream / Structured Streaming / 事件时间 / Watermark / Kafka
- **第 5 篇 · 图计算**：GraphX 用 `aggregateMessages` + `Pregel` 揪出三人团伙——属性图 / triplets / CrimeGangRunner / ShortestPath
- **第 6 篇 · 性能优化**：把 40 分钟的 join 调到 5 分钟——分区 / By-Key / 序列化 / 查询计划 / 缓存 / 倾斜修复

这 6 篇覆盖了 Spark 的核心能力——但 Spark 生态远不止这些。**如果你想继续深入，这几个方向值得一走**：

- **Structured Streaming + Kafka**：流式处理的事实标准，本系列第 4 篇只是入门。
- **Delta Lake / Iceberg**：Lakehouse 架构，让 Spark 直接操作数据湖。
- **Spark on Kubernetes**：从 YARN 迁移到 K8s，云原生时代的部署方案。
- **PySpark 与 Scala Spark 的取舍**：数据科学用 PySpark，工程实现用 Scala——两种 API 的边界。
- **Spark 3.5 的新特性**：自适应查询（AQE）、Structured Logging、Spark Connect。

Spark 是一门"看起来入门、其实越用越深"的工具。它不会像 Kafka 那样一两篇就懂，但**一旦你把分区、By-Key、查询计划这三件事吃透了，所有性能问题都能在 UI 上找到答案**。

希望这 6 篇能帮你跨过"看得懂 Spark 代码"到"能让 Spark 跑得快"之间的那道坎。