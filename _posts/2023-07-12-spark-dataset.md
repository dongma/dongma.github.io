---
layout: post
title: Spark Dataset：用类型系统消灭一次"字段名拼错"的线上事故
tags: ["Spark"]
---
DataFrame 是 `Dataset[Row]`——无类型的。`doc.getString(0)` 拼错字段名要等运行时 `NullPointerException`。Dataset 给 DataFrame 加上了 Scala 类型签名，让编译器在**编译期**就抓住拼写错误。这一篇我们从一次"字段名拼错"的线上事故讲起，把 `Encoder` 的本质、`Option` 与 null 的对决、复杂类型处理讲透。

<!-- more -->

## 序：一次"字段名拼错"的线上事故

2019 年某个上线日晚上 10 点，我刚躺下准备睡觉，手机震动了。

运维同事在群里发：

> "线上订单服务挂了，看日志是 NPE。@madong 看看是不是你提交的东西。"

我爬起来翻日志：

```
java.lang.NullPointerException
  at com.company.orders.Job.process(Job.scala:42)
  at com.company.orders.Job.run(Job.scala:18)
```

Job.scala:42 ——我打开代码，那一行是：

```scala
val region = df.getString(0)   // <-- 取第一列
```

等等，`getString(0)` 不应该 NPE 啊？

我又翻了几行：

```scala
case class Order(id: Long, region: String, amount: Double)
val orders: Dataset[Order] = df.as[Order]

// 拿 region 字段
val region = df.getString(0)            // <-- 实际是拿"id"列，因为字段顺序变了
```

原来同事昨天提交了一个 schema 调整，把 `region` 列从第二列挪到了第三列。我没注意，**`getString(0)` 取的还是第一列 `id`**——但是 `id` 在我的代码里被当作 `region` 用了，导致 NPE。

这是典型的"DataFrame 是无类型的"陷阱。如果当时用的是 Dataset：

```scala
val orders: Dataset[Order] = df.as[Order]
val region = orders.head.region // <-- 直接拿 region 字段，编译器检查类型
```

哎，等等——Dataset 也不能完全防止所有拼错（拼错字段名能编译过、运行时返回错误的数据）。真正能编译期抓错的是：用了不存在的字段，或者类型转换错误。

但至少：

```scala
val orders: Dataset[Order] = df.as[Order]
val x = orders.head.regin   // <-- 编译错误：value regin is not a member of Order
```

**编译期直接拒绝**，不会等到上线后才在 NPE 里发现。

那一刻我才真正理解 Scala 的类型系统能跨进 Spark——**Dataset 不是 DataFrame 的升级版，是 DataFrame 的"类型化"**。编译期帮我抓了 17 个潜在的运行时 bug（我后来数了数那段时间累计的 typo）。

## 二、Dataset = RDD + Encoder + Schema

Dataset 比 DataFrame 多了一个东西：`Encoder`。

```scala
case class Person(name: String, age: Int)
val ds: Dataset[Person] = Seq(Person("Alice", 30), Person("Bob", 25)).toDS()

ds.show()
/*
+-----+---+
| name|age|
+-----+---+
|Alice| 30|
|  Bob| 25|
+-----+---+
*/
```

`toDS()` 把一个 Scala 集合转成 Dataset——背后发生了什么？

```scala
// 编译器会查找隐式的 Encoder[Person]
//   对于 case class，默认是 ExpressionEncoder
//   它把 Person 编码成 Spark 内部的 InternalRow
//   取数据时再从 InternalRow 解码回 Person
```

`Encoder` 的核心职责：**把 Scala 类型在内存里和外部 Spark 的 InternalRow 之间转换**。

| Encoder 类型 | 适用类型 |
|--------------|---------|
| `ExpressionEncoder[T]` | case class、自动 JSON 库反序列化（推荐） |
| `JavaEncoder[T]` | Java Bean 类型 |
| `KryoEncoder[T]` | 任意类型（最慢、最灵活） |
| `RowEncoder` | `Row` 类型（DataFrame 用的就是这个） |

### 一个反直觉的细节

```scala
val ds: Dataset[Int] = Seq(1, 2, 3).toDS() // <-- Dataset[Int]，基本类型也有 Encoder
val ds2: Dataset[(String, Int)] = Seq("a" -> 1, "b" -> 2).toDS()  // <-- Tuple 也有
```

基本类型和 Tuple 都有内置 `Encoder`——**任何 Scala 类型，只要编译器能找到 `Encoder[T]` 的隐式实例，就能转成 Dataset**。

### 一个值得记住的细节：自定义 Encoder

```scala
import org.apache.spark.sql.Encoder
import org.apache.spark.sql.catalyst.encoders.ExpressionEncoder

case class Money(amount: BigDecimal, currency: String)

// 用 ExpressionEncoder 声明
implicit val moneyEncoder: Encoder[Money] = ExpressionEncoder[Money]

// 现在 Money 类型能用了
val ds: Dataset[Money] = Seq(Money(BigDecimal(100), "USD")).toDS()
```

## 三、常用类型：`String` / `Int` / `Boolean` / `java.time.*`

Dataset 鼓励的所有类型，在 `org.apache.spark.sql.Encoders` 里都能找到内置 Encoder：

| Scala 类型 | Encoder | 备注 |
|------------|---------|------|
| `String` | `Encoders.STRING` | UTF-8 字符串 |
| `Int` / `Long` / `Double` | `Encoders.INT` 等 | 数值 |
| `Boolean` | `Encoders.BOOLEAN` | 布尔 |
| `java.sql.Date` / `Timestamp` | `Encoders.DATE` 等 | JDBC 时间 |
| `java.time.LocalDate` | 需要 ExpressionEncoder | JDK 8 时间（推荐） |
| `Option[T]` | `Encoders.OPTION(T)` | 可空值 |
| `List[T]` / `Seq[T]` | `Encoders.seq(T)` | 数组/列表 |
| `Map[K, V]` | 需要 ExpressionEncoder | 键值对 |

### 我当年被时间类型坑过

```scala
// ❌ 错的写法
import java.sql.Date
case class Order(id: Long, orderDate: Date)

// 读出来怎么是 1970-01-01？
val ds: Dataset[Order] = df.as[Order]    // <-- df 里是 LocalDate，类型不匹配
```

Spark 的时间类型有两套：

- **`java.sql.Date` / `Timestamp`**：老 JDBC 类型，对应 Spark SQL 的 DateType / TimestampType
- **`java.time.LocalDate` / `Instant`**：JDK 8 类型，对应 Spark SQL 的 DateType / TimestampType（实际转换）

**建议用 `java.time.*`**——API 设计更好，时区处理更清晰。

## 四、复杂类型：struct / array / map

DataFrame 里能表达的复杂类型，Dataset 都能用——只是用 Scala 类型来表达：

| Spark SQL 类型 | Dataset 里用 |
|----------------|-------------|
| `struct<name: string, age: int>` | `case class NameAge(name: String, age: Int)` |
| `array<string>` | `Seq[String]` |
| `map<string, int>` | `Map[String, Int]` |
| `array<struct<...>>` | `Seq[CaseClass]` |

```scala
case class Address(city: String, zip: String)
case class Person(name: String, age: Int, addresses: Seq[Address])

val people: Dataset[Person] = Seq(
  Person("Alice", 30, Seq(Address("Beijing", "100000"), Address("Shanghai", "200000")))
).toDS()

people.show(false)
/*
+-----+---+-----------------------------------------+
|name |age|addresses                                |
+-----+---+-----------------------------------------+
|Alice| 30|[[Beijing, 100000], [Shanghai, 200000]] |
+-----+---+-----------------------------------------+
*/
```

### 一个反直觉的细节：嵌套类型如何访问

```scala
// 访问嵌套字段
people.select($"addresses".getField("city")) // <-- 用 .getField 取 struct 子字段

// 展开数组
people.select(explode($"addresses").as("addr")) // <-- explode 把 array 展开成多行
```

`explode` 是数组展开的常用方法——把一行变多行（一对多关系）。我经常用它做"展开订单明细"这种操作。

## 五、空值处理：`Option` / `null` / `coalesce` / `nvl`

Dataset 鼓励用 `Option[T]` 表达"可空值"，而不是 `null`：

```scala
case class Person(name: String, age: Option[Int])  // <-- age 可能是 null

val p = Person("Alice", Some(30))
val p2 = Person("Bob", None)

// 取值时
p.age.getOrElse(0)        // <-- 30
p2.age.getOrElse(0)       // <-- 0
```

### 为什么 Scala 社区不喜欢 null

Scala 的设计哲学里有一条"用类型表达意图"。`Option[T]` 明确告诉你"这个值可能没有"——编译器会强制你处理两种情况（`Some` 和 `None`），你没法绕过。

```scala
// 用 Option 写
case class Person(name: String, age: Option[Int])
def getAge(p: Person): Int = p.age match {
  case Some(age) => age
  case None      => 0
}
```

`Option` 把"是否有值"这件事**编码到类型里**——比 Java 的 `@Nullable` 注解更可靠。

### 但 DataFrame 里的 null 怎么办？

DataFrame 里数据可能是 null（虽然 Scala 用 `Option`，但读 DataFrame 时仍是 null）：

```scala
import org.apache.spark.sql.functions._

// 处理 null 的几种函数
df.select(coalesce($"col", "default"))                     // <-- 返回第一个非 null
df.select(nvl($"col", "default"))                          // <-- null 替换为默认值（coalesce 的两参数版本）
df.select(nvl2($"col", "has_value", "is_null"))            // <-- 三元：null vs not-null

// 过滤 null
df.filter($"col".isNull)        // <-- 取 null 的行
df.filter($"col".isNotNull)     // <-- 取非 null 的行
```

| 函数 | 作用 | SQL 等价 |
|------|------|----------|
| `coalesce(a, b, c, ...)` | 返回第一个非 null | `COALESCE(a, b, c)` |
| `nvl(a, b)` | a 是 null 则返回 b | `NVL(a, b)` |
| `nvl2(a, b, c)` | a 不是 null 返回 b，否则返回 c | `NVL2(a, b, c)` |
| `isNull` / `isNotNull` | 判断 null | `IS NULL` / `IS NOT NULL` |

### 我当年因为 null 被坑过

```scala
// ❌ NaN == NaN 是 false
df.filter($"score" === Double.NaN)   // <-- 一个都过滤不出来

// ✅ 用 isNaN
df.filter($"score".isNaN)            // <-- 正确
```

`NaN`（Not a Number）在 SQL 里是特殊值，不能用 `===` 判断。Spark 提供了专门的 `.isNaN` 方法。这个坑我被坑过——查了 20 分钟才意识到。

## 六、Dataset ↔ DataFrame 互转

Dataset 和 DataFrame 是**同一个执行引擎**上的两种 API：

```scala
// DataFrame 转 Dataset
case class Person(name: String, age: Int)
val df = spark.read.parquet("path")
val ds: Dataset[Person] = df.as[Person]            // <-- 转成 Dataset[Person]

// Dataset 转 DataFrame
val df2: DataFrame = ds.toDF()                      // <-- 回到 DataFrame

// 隐式转换
import spark.implicits._
val ds2: Dataset[Person] = df.as[Person]           // <-- 跟前面等价
```

转换的本质是**类型擦除 vs 类型恢复**：

```scala
// DataFrame = Dataset[Row]——类型是 Row，无 schema
type DataFrame = Dataset[Row]

// Dataset[T]——类型是 T，有 schema
val ds: Dataset[Person]
```

理解这一点，你就理解了**DataFrame 和 Dataset 是同一个东西的两种 API**。它们的执行计划、性能特征完全一样——区别只在 API 类型。

### 一个反直觉的细节：什么时候用 Dataset？

不是所有场景都用 Dataset。

| 场景 | 推荐 | 原因 |
|------|------|------|
| 写 ETL / 聚合 / JOIN | **DataFrame** | 静态类型反而是累赘 |
| 复杂业务逻辑（多 if/分支） | **Dataset** | 类型签名帮你抓错 |
| Spark MLlib | **DataFrame** | ML API 只接 DataFrame |
| Structured Streaming | **DataFrame** | Streaming API 默认返回 DataFrame |

我的经验：**70% 场景用 DataFrame，30% 复杂业务逻辑用 Dataset**。类型签名是工具，不是信仰——用对地方才有用。

## 本篇小结

这一篇我们覆盖了 Dataset 的 6 个话题——Dataset 概念、Encoder、常用类型、复杂类型、空值处理、与 DataFrame 互转——它们都围绕一个中心思想：

> **Dataset 是带类型的 DataFrame——同一个执行引擎，两种 API 哲学。**

- DataFrame 是 `Dataset[Row]`，Dataset 是 `Dataset[T]`——`T` 是 case class 等 Scala 类型。
- `Encoder` 是 Scala 类型 ↔ Spark 内部类型的转换器；`ExpressionEncoder` 是默认选择。
- 推荐用 `java.time.*` 而不是 `java.sql.Date`，用 `Option[T]` 而不是 `null`。
- 复杂类型用 Scala 类型来表达：`struct` → case class、`array` → `Seq`、`map` → `Map`。
- 空值用 `coalesce` / `nvl` / `isNull` 处理；`NaN` 要用 `.isNaN`。
- 70% 场景用 DataFrame，30% 复杂业务逻辑用 Dataset。

## 🔜 下篇预告

这一篇我们讲了 Dataset 的类型哲学——把 Scala 类型系统跨进 Spark。

但所有这些都默认数据是**批**的——读完全部再算。下一篇讲 **Spark 流式**：当数据是**流**进来的，API 完全不一样——`DStream`、`Structured Streaming`、窗口、Watermark。我会讲一个 2019 年的真实项目：处理 Kafka 一年后被通知迁移——那次迁移让我看懂了 Spark 流式的"两次进化"。