---
layout: post
title: RDD 是汇编，DataFrame 是高级语言：重新认识 Spark 基础 API
tags: ["Spark"]
---
Spark 的 API 看起来很多——RDD、DataFrame、Dataset、Column、聚合函数、读写方法……但其实只有一条主线：**让编译器多做一点，让运行时少错一点**。这一篇我们从 `SparkSession` 讲起，经过 RDD 的本质、DataFrame 的"高级语言"哲学、列表达式、聚合，到数据源——把 Spark 基础的"全景"梳理一遍。

<!-- more -->

## 序：第一次打开 Spark shell 的那个下午

2017 年某个加班的下午，我第一次打开 Spark shell，盯着屏幕发了五分钟呆。

桌面上同时打开了两个教程页：一个让我 `val rdd = sc.parallelize(Seq(1, 2, 3))`，另一个让我 `val df = spark.createDataFrame(Seq(...))`。两个教程都没说清楚——**我到底该用哪个？**

带我的同事路过瞄了眼屏幕，扔下一句话：

> "RDD 是汇编，DataFrame 是高级语言。"

我似懂非懂地点了点头。后来用了三年 Spark，回头看这句话才真正理解——**RDD 让你写 JVM 字节码级别的事，DataFrame 让你写 SQL 级别的 DSL**。当你把这句话吃透了，所有"高级 API 看起来很怪"的细节都会迎刃而解。

这一篇我想把"基础 API 全景"讲清楚：从 `SparkSession` 入口，到 RDD 的本质、DataFrame 的"高级语言"哲学、列与表达式、聚合，再到数据源——这条路径走完，你就有了读懂 Spark 所有上层 API 的底座。

## 一、`SparkSession`：所有 API 的入口

Spark 2.0 之前，每个功能要单独搞一个入口：`sc` 跑 RDD、`sqlContext` 跑 SQL、`hiveContext` 跑 Hive。每个新人都要被这"三件套"折磨一遍。

我有次带实习生，光是讲清这三个东西的区别就花了一下午。第二天实习生跟我说："我还是搞不清楚我该用哪个。"

我当时心想：**这个问题 Spark 团队肯定也意识到了**。果然——Spark 2.0 把这三个入口统一成了 `SparkSession`：

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder()
  .appName("My First App")                            // 应用名，会出现在 Spark UI 上
  .master("local[*]")                                  // 本地模式，* 表示用本机所有可用核
  .config("spark.sql.shuffle.partitions", "4")         // shuffle 后默认分区数
  .getOrCreate()                                       // 单例：同名 appName 第二次调用复用
```

老入口都被藏在 `SparkSession` 里了，需要的时候一行就能拿到：

| 老的入口 | 现在通过 `SparkSession` 怎么拿 |
|----------|--------------------------------|
| `sc`（SparkContext） | `spark.sparkContext` |
| `sqlContext`（SQLContext） | `spark.sqlContext` |
| `hiveContext`（HiveContext） | `spark.sqlContext`（Hive 支持默认开启） |

**`SparkSession.builder()` 是单例**——同一个 `appName` 第二次调用 `getOrCreate()` 会复用已有实例，不会重新初始化。我有次开了两个 `getOrCreate()`，差点为它重连了集群——后来才意识到"哦，原来它自己已经处理了"。

> 小细节：`master("local[*]")` 里的 `*` 表示"用本机所有 CPU 核"。生产环境不会这么写，会传 `spark://...` 或 `yarn`。

## 二、RDD：分布式的"数组"

RDD 这个名字第一次看到的时候，我以为是某种"高级数据结构"——其实它本质就是一个**分布在集群多台机器上的、容错的、可并行处理的元素集合**。换句话说：**RDD 是数组，跨机器版的数组**。

```scala
// 从一个 Scala 集合创建 RDD
val numbers = spark.sparkContext.parallelize(Seq(1, 2, 3, 4, 5))   // <-- 等于把数组"复制"到多台机器
val lines   = spark.sparkContext.textFile("path/to/text.txt")     // <-- 文件按行切成多个 partition

// RDD 的两个家族操作：Transformation（懒）和 Action（触发）
val doubled = numbers.map(_ * 2)              // <-- Transformation：返回新 RDD，不立即执行
val sum     = doubled.reduce(_ + _)           // <-- Action：真正触发计算，返回 Scala 值

println(sum)  // 30
```

理解 RDD 的三个关键词：

1. **分布式**：RDD 的数据被切成多个 partition 分到多台机器上。`parallelize` 的默认 partition 数等于 `spark.default.parallelism`（本地模式 = 本机核数）。
2. **弹性**（Resilient）：每个 partition 的数据可以通过"血统"（lineage）从父 RDD 重新算出来——所以某个 partition 丢了，可以重算而不丢数据。
3. **懒求值**：`map`、`filter`、`flatMap` 这些 transformation **不会立即执行**，只是记下"要做什么"。只有遇到 `count`、`collect`、`reduce` 这种 action 才真正触发计算。

### 我曾经踩过的坑：忘了它是"懒"的

有次我写了 `rdd.map(println)`，debug 时想在 driver 端打印所有数据。结果：

```scala
val rdd = spark.sparkContext.parallelize(Seq(1, 2, 3, 4, 5))
rdd.map(println)   // <-- 啥也没打！map 没触发执行
```

我盯着屏幕发了五分钟呆——**明明写了 `println`，为什么没输出？**

后来才搞明白：`map` 是 transformation，不触发执行。必须加个 action：

```scala
rdd.map(println).collect()  // <-- collect() 触发执行，输出 1 2 3 4 5
```

这种 bug 是 RDD 入门者的"必经之路"——记下来：**RDD 是懒的，没有 action 不会执行**。

### RDD 的局限性：类型擦除

RDD 是无 schema 的。如果你想统计一个 Person 集合的年龄平均：

```scala
case class Person(name: String, age: Int)
val people = spark.sparkContext.parallelize(Seq(Person("Alice", 30), Person("Bob", 25)))

// 编译能过，但运行时拼错字段名就 NPE
people.map(_.agee).collect()   // <-- 编译过，运行时：value agee is not a member of Person
```

编译器看不到 RDD 元素的 schema——`_.agee` 这种拼写错误，要到运行时才被发现。这就是 **RDD 是汇编**的原因：它给你全部控制权，但**所有类型检查都得你自己做**。

我曾经因为这种 typo 在生产环境 debug 了一晚上——从那以后我才真正理解 DataFrame 的价值。

## 三、DataFrame = RDD[Row] + Schema

如果说 RDD 是"跨机器的数组"，那 DataFrame 就是"跨机器的、有 schema 的表"。

```scala
import spark.implicits._

val df = people.toDF()         // <-- case class 自动转成 DataFrame
df.printSchema()                // <-- 把 schema 打印出来
/*
root
 |-- name: string (nullable = true)
 |-- age: long (nullable = true)
*/

df.filter($"age" > 26).show()   // <-- $"age" 是列表达式，filter 是 SQL 语义
/*
+-----+---+
| name|age|
+-----+---+
|Alice| 30|
+-----+---+
*/
```

DataFrame 本质上是 `RDD[Row]` + 一个 schema 描述：

```scala
// DataFrame 的内部结构
// 1. 逻辑计划：知道列名和类型（"name": String, "age": Long）
// 2. 物理执行：RDD[Row]，每个 Row 是一个按列顺序存值的数组
```

但 DataFrame 强在两件事：

1. **schema 是已知**的：列名拼错直接编译错误（或运行前就抛异常），不是运行时 NPE。
2. **Catalyst 优化器**能看到你的操作：`filter($"age" > 26)` 不是"对每行做 lambda"，而是"在执行计划里加一个 Filter 节点"——Catalyst 可以做谓词下推、列裁剪、join 重排序等优化。

这就是 **DataFrame 是高级语言**的本质：你写的是 DSL，编译器把它翻译成最优的 RDD 操作。

### 一个反直觉的细节

```scala
import org.apache.spark.sql.Row
import org.apache.spark.sql.types.{StructType, StructField, StringType}

// 手写一个 DataFrame
val rdd = spark.sparkContext.parallelize(Seq(Row("Alice", 30), Row("Bob", 25)))   // <-- 数据用 Row 装
val schema = StructType(Seq(                                                       // <-- schema 单独定义
  StructField("name", StringType),                                                 // <-- 一列：name 是 String
  StructField("age", LongType)                                                     // <-- 一列：age 是 Long
))
val df = spark.createDataFrame(rdd, schema)                                         // <-- 数据 + schema = DataFrame
```

`createDataFrame` 需要两个东西：一个 `RDD[Row]`（数据），一个 `StructType`（schema）。这就是为什么我说 **DataFrame = RDD[Row] + Schema**。

## 四、列与表达式：`col` / `expr` / `when`

DataFrame 的"列"不是值，是**表达式**。这一点第一次理解的时候，我有种"啊哈"的感觉——它解释了很多看似神奇的事情。

```scala
import org.apache.spark.sql.functions._

// 引用列的四种方式（任何一种都行，看团队约定）
df.select(col("name"))               // <-- col 函数
df.select(column("name"))            // <-- column 是 col 的别名
df.select($"name")                   // <-- $ 需要 import spark.implicits._（隐式转换）
df.select(df.col("name"))             // <-- 通过 DataFrame 的 col
```

`col` 返回的是一个 `Column` 对象。`Column` 是表达式的载体——你可以对它做算术、函数、case when：

```scala
// 加一列
df.withColumn("age_plus_one", $"age" + 1)            // <-- $"age" + 1 是表达式，不是值

// 字符串函数
df.select(upper($"name").as("upper_name"))           // <-- upper() 返回 Column

// case when
df.select(
  when($"age" >= 30, "senior")                       // <-- 第一个分支
    .when($"age" >= 20, "middle")                    // <-- 第二个分支
    .otherwise("junior")                             // <-- 默认分支
    .as("level")                                      // <-- 给结果列起名
)

// 聚合函数
df.select(sum($"age"), avg($"age"), count($"name"))  // <-- 多个聚合列
```

### 为什么 `expr` 很强大

```scala
// 用 SQL 风格的字符串写表达式
df.select(expr("age + 1 as new_age"), expr("upper(name) as upper_name"))
```

`expr` 接收一串 SQL 表达式，返回 `Column`。它适合"用 SQL 写一段复杂逻辑、然后接到 DataFrame API 里"的场景。我经常这样写：

```scala
// 复杂的 SQL 表达式用 expr，简单的用 DataFrame API
df.select(
  $"user_id",
  expr("CASE WHEN age < 18 THEN 'minor' WHEN age >= 65 THEN 'senior' ELSE 'adult' END").as("category")
)
```

### 一个值得记住的细节：`===` 不是 `==`

```scala
// 错的写法
df.filter($"age" == 30)        // <-- Scala 的 ==，触发 null 比较的特殊语义

// 对的写法
df.filter($"age" === 30)       // <-- Column 的 ===，生成 SQL 的 =
df.filter($"age".equalTo(30))  // <-- 等价的显式调用
```

`$"age" == 30` 在某些版本下会触发 Scala `Any.==` 的特殊语义（对 null 的处理不同），而 `$"age" === 30` 直接生成 SQL `= 30`。**DataFrame 里过滤比较，一律用 `===`**。我当年被这个 bug 坑过一次——查了半小时才意识到是 `==` vs `===` 的问题。

## 五、聚合：`groupBy` + `agg`

DataFrame 的聚合分两步：先 `groupBy` 分组，再 `agg` 算。简单到你可能会问"为什么这么设计"——其实是为了让你**像 SQL 一样思考**。

```scala
df.groupBy("name").count().show()                              // <-- 直接 count
df.groupBy("name").agg(sum("age"), avg("age")).show()          // <-- agg 多聚合
```

聚合函数都来自 `org.apache.spark.sql.functions`：

| 函数 | 作用 |
|------|------|
| `count` | 行数（不去重） |
| `countDistinct` | 去重计数 |
| `sum` / `avg` / `min` / `max` | 数值聚合 |
| `approx_count_distinct` | 近似去重计数（HLL++），大表友好 |
| `first` / `last` | 任意一个分组内的第一条/最后一条 |
| `collect_list` / `collect_set` | 把分组的值收集成数组（⚠️ 小数据用） |
| `stddev` / `variance` | 标准差 / 方差 |

### `agg` 接受三种写法

```scala
// 写法 1：直接传聚合函数
df.groupBy("name").agg(sum("age"), avg("age"))

// 写法 2：用 expr 传 SQL 风格的字符串
df.groupBy("name").agg(expr("sum(age)"), expr("avg(age)"))

// 写法 3：用 sql 函数 + .as 起名
df.groupBy("name").agg(sum("age").as("total_age"), avg("age").as("avg_age"))
```

三种写法生成的执行计划完全一样——DataFrame 和 SQL 真的就是同一种表达形式。我通常喜欢写法 3，因为它能给聚合列起清晰的别名。

### 一个小坑：`groupBy` 后过滤用 `having`

```scala
df.groupBy("name").agg(count("age").as("cnt"))
  .filter($"cnt" > 1)   // <-- 实际效果跟 SQL 的 having 一样，因为 filter 在 agg 之后
  .show()
```

Spark 2.0 之后，`filter` / `where` 在 `agg` 之后用，效果等同 SQL 的 `HAVING`。但在更复杂的查询里，**用 SQL 写 HAVING 会更清晰**——下一篇讲 Spark SQL 时会展开。

## 六、数据源：JSON / CSV / Parquet / JDBC

我刚开始用 Spark 的时候，每次读数据都要查文档："JSON 怎么读来着？Parquet 怎么写来着？"后来发现所有数据源的 API 是**完全镜像**的——读用 `spark.read`，写用 `df.write`。

```scala
// 读
val df = spark.read
  .format("json")                  // <-- 数据格式
  .option("inferSchema", "true")   // <-- 自动推断 schema（代价是再读一遍文件）
  .option("multiline", "true")     // <-- 多行 JSON（数组跨行）
  .load("path/to/file.json")       // <-- 实际路径

// 写
df.write
  .format("parquet")
  .mode("overwrite")               // <-- 覆盖写（默认是 error）
  .partitionBy("year", "month")    // <-- 分区列，物理上生成 year=2022/month=01/ 目录
  .save("path/to/output")
```

| 数据源 | 读 | 写 | 关键 option |
|--------|----|----|-------------|
| **JSON** | `spark.read.json(path)` | `df.write.json(path)` | `multiline`、`inferSchema` |
| **CSV** | `spark.read.csv(path)` | `df.write.csv(path)` | `header`、`inferSchema`、`sep` |
| **Parquet** | `spark.read.parquet(path)` | `df.write.parquet(path)` | 内置压缩（snappy 默认） |
| **JDBC** | `spark.read.jdbc(url, table, props)` | `df.write.jdbc(...)` | `numPartitions`、`partitionColumn`、`lowerBound`、`upperBound` |

### Parquet 是默认推荐格式

为什么？因为：

1. **列式存储**：每列独立压缩 / 编码，分析场景只读需要的列，省 IO。
2. **自带 schema**：JSON / CSV 都要 `inferSchema`，Parquet 直接读出 schema。
3. **支持谓词下推**：读取时根据 filter 自动跳过不满足条件的 row group，省 IO。

```scala
// 读取 Parquet 时，filter 自动下推（你可以用 explain 看）
spark.read.parquet("path/to/data.parquet").filter($"year" === 2022).explain(true)
// 物理计划里会出现：
// PushedFilters: [IsNotNull(year), EqualTo(year, 2022)]
```

这段我们下一篇 SQL 与查询计划会展开讲——但记住：**Parquet 是分析型工作负载的最佳格式**。如果你在生产环境读 JSON，我的建议是先转 Parquet 再用。

## 七、窄依赖 vs 宽依赖：Spark 并行的底层原理

最后一节讲一个看似基础、实际影响所有性能优化的概念：**依赖类型**。

Spark 的 transformation 分两种依赖：

```scala
// 窄依赖：每个父 partition 最多被一个子 partition 用（不需要 shuffle）
val rdd1 = numbers.map(_ + 1)       // <-- 1 对 1 映射，窄
val rdd2 = rdd1.filter(_ > 2)       // <-- 1 对 1 过滤，窄

// 宽依赖：父 partition 被多个子 partition 用（需要 shuffle）
val rdd3 = rdd1.groupBy(_ % 3)      // <-- 同 key 必到同 partition，要 shuffle
```

| 类型 | 是否 shuffle | 例子 |
|------|--------------|------|
| 窄依赖 | 否 | `map`、`filter`、`flatMap`、`union` |
| 宽依赖 | 是 | `groupByKey`、`reduceByKey`、`join`、`repartition` |

**窄依赖可以在同一个 stage 内流水线执行**，不跨节点。**宽依赖必须 shuffle 跨节点**，是性能瓶颈的主要来源。

我有个简单的判断方法：**有没有"按 key 重排"**。只要操作里带 key 分桶（group、join、partition by），就是宽依赖。其他都是窄依赖。

这也是为什么我们最后一篇讲性能优化时要重点讲 `Exchange` 节点——**Exchange 就是宽依赖在执行计划里的代号**。当你 UI 上看到 Exchange 红了，十有八九是宽依赖出问题了。

## 本篇小结

这一篇我们覆盖了 Spark 基础 API 的 7 个话题——`SparkSession`、RDD、DataFrame、列表达式、聚合、数据源、依赖类型——它们都围绕一个中心思想：

> **DataFrame 是 RDD 的进化——你写的是 DSL，编译出来的是最优的 RDD 操作。**

- `SparkSession` 是所有 API 的统一入口，老的 `sc` / `sqlContext` 都被它包了。
- RDD 是分布式的元素集合，但**无 schema**，类型检查要自己负责。
- DataFrame = `RDD[Row]` + Schema，Catalyst 优化器能看到你的操作。
- 列是表达式，不是值——理解这一点，你就理解了 DataFrame 的全部"魔法"。
- 聚合分两步：`groupBy` 分组 + `agg` 算；函数来自 `org.apache.spark.sql.functions`。
- 数据源首选 Parquet：列式存储 + 自带 schema + 谓词下推。
- 窄依赖不 shuffle，宽依赖必须 shuffle——后者是性能瓶颈的主要来源。

## 🔜 下篇预告

这一篇我们站在"基础 API"层面看 Spark——RDD 是汇编，DataFrame 是高级语言。

但 Spark 还有一种更直接的方式写复杂查询：**Spark SQL**。它把 DataFrame 当成 SQL 表来查询，让你能用 SQL 写多表 JOIN、动态拼接 WHERE 条件，甚至把 30 张 Postgres 表搬进 Warehouse——下一篇我会讲一个 2018 年的真实项目：30 张表、5 层嵌套的 DataFrame、最后用 50 行 SQL 解决。

我们将从 `createOrReplaceTempView` 讲起，到程序化 SQL、Warehouse、JDBC——这是一条"用 SQL 把 DataFrame 嵌套地狱变成一行流"的完整路径。