---
layout: post
title: 从"函数是一等公民"开始，重新认识 Scala
tags: ["Scala"]
---
Scala 不是"Java + 函数式 lambda"，它的真正强项是把"函数作为值"贯穿到集合、模式匹配、并发模型里。这一篇我们从 `Function1` 这个"lambda 的真相"开始，经过高阶函数、偏函数、自己手写一个函数式 `Set`，最后到 `Option` 和 for 推导式——走完一条"从命令式思维转向函数式思维"的完整路径。

<!-- more -->

## 序：我第一次觉得"函数式编程"不是噱头

2019 年我在写一个 ETL 任务，需要把 200 万行日志做"按用户分组 → 每组按时间排序 → 取每组前 100 条"。

我先写了 30 行 Java 8 stream，自测通过，提交。第二天下午生产环境挂了——OOM。原因是我在 `groupingBy` 之后用 `sorted`，**而 stream 的 sorted 是把整个组加载到内存**。

我重写成 Scala，21 行：

```scala
val topPerUser = logs
  .groupBy(_.userId)
  .view
  .mapValues(_.sortBy(_.timestamp).take(100))
  .toMap
```

跑了一个小时，稳如老狗。

那一刻我才真正理解 FP 的"**惰性**"和"**不可变**"不是学术噱头——它直接决定了你的程序能不能在生产环境跑得动。

这一篇我想讲清楚：Scala 的函数式编程到底在解决什么问题，以及它怎么用"函数作为值"这把钥匙打开新世界。

## 一、`Function1` 的本质：lambda 不是语法糖，是对象

我们先看一段最基础的代码：

```scala
// 显式 new 一个 Function1
val simpleIncrementer = new Function[Int, Int] {
  override def apply(arg: Int): Int = arg + 1
}
simpleIncrementer(23)  // 24
```

`Function[Int, Int]` 实际上就是 `Function1[Int, Int]`——Scala 用 `Function0..Function22` 共 23 个 trait 表示"接受 0 到 22 个参数的函数"。每个 trait 都有一个抽象方法 `apply`，调用 `simpleIncrementer(23)` 实际就是 `simpleIncrementer.apply(23)`。

而 lambda 是什么？

```scala
// "语法糖"形式的 lambda
val incrementAlt = (x: Int) => x + 1
```

**反编译回去**，它就是：

```scala
val incrementAlt = new Function1[Int, Int] {
  override def apply(x: Int): Int = x + 1
}
```

所以 lambda 不是"函数定义的简化写法"——它是**匿名类的简化写法**。理解这一点之后，你就知道为什么 lambda 能被传递、能被存储、能被组合：**它本质上是个对象**。

### 一个延伸：`Runnable` 也能 lambda 化

既然 lambda 是 `Function0[Unit]` 的语法糖，那任何"接受一个无参方法"的接口都可以用 lambda：

```scala
val aThread = new Thread(() => println("sweet, scala!"))
// 替代了 new Thread(new Runnable { override def run(): Unit = println("...") })
```

`Runnable.run()` 接受 0 个参数返回 `Unit`，正好对应 `Function0[Unit]`。这个特性在 Akka Actors、UI 框架、回调密集型代码里到处可见。

## 二、高阶函数：接受函数、返回函数

"高阶函数"听起来很高深，其实就两件事：

1. 接受函数作为参数
2. 返回函数作为结果

```scala
// nTimes 接受一个函数 f 和次数 n，返回一个新函数
// 新函数对输入 x 调用 f 共 n 次
def nTimes(f: Int => Int, n: Int): Int => Int = {
  if (n <= 0) (x: Int) => x
  else (x: Int) => nTimes(f, n - 1)(f(x))
}

val func = (x: Int) => x + 1
val fun4 = nTimes(func, 4)   // fun4 = x => f(f(f(f(x))))
fun4(5)  // 9
```

`nTimes(func, 4)` 返回的函数是 `x => x+4`，所以 `fun4(5) = 5+4 = 9`。这种"函数工厂"是函数式编程的核心抽象。

### 真正让你每天用 HOF 的是集合 API

```scala
val aMappedList = List(1, 2, 3).map(x => x + 1)              // List(2, 3, 4)

val aFlatMappedList = List(1, 2, 3).flatMap { x => List(x, 2*x) }
// 等价于 List(1, 2, 2, 4, 3, 6)
// 对每个元素调用 f 得到一个集合，然后"拍扁"成一个

val aFilteredList = List(1, 2, 3, 4, 5).filter(_ <= 3)
// _ <= 3 是 (x: Int) => x <= 3 的简化写法（_ 占位第一个参数）
```

`flatMap` 看起来绕：它对每个元素调用 `f` 得到一个集合，然后把所有子集合"拍扁"成一个。这在"一对多"映射（一个用户多个订单、一个订单多个商品）时特别有用。

### `for` 推导式就是 `flatMap + map` 的语法糖

最有意思的是 `for` 推导式：

```scala
val pairs = for {
  x <- List(1, 2, 3)
  y <- List(4, 5, 6)
} yield (x, y)
// List((1,4), (1,5), (1,6), (2,4), (2,5), (2,6), (3,4), (3,5), (3,6))
```

**它完全等价于**：

```scala
List(1, 2, 3).flatMap(x => List(4, 5, 6).map(y => (x, y)))
```

"双层循环 + 收集结果"的模式是处理嵌套集合的利器。编译器帮你展开了 `flatMap` + `map`，代码读起来像数学公式，但执行起来和手写 `flatMap` 链完全一致。

## 三、偏函数 `PartialFunction`：只对部分输入有定义的"半函数"

讲个真实场景：你在写一个聊天机器人，"hello" 回复 A，"goodbye" 回复 B，其他输入我们想抛错或者返回 "I don't understand"。

普通函数（`Function1`）对**所有**输入都得有定义——你得在函数体里写 `if/else`。但有些函数在"语义上"就只对部分输入有意义，比如"把数字 1/2/3 转成英文，其它抛异常"。

Scala 的解法是 `PartialFunction`：

```scala
val aPartialFunction: PartialFunction[Int, Int] = {
  case 1 => 42
  case 2 => 56
  case 3 => 999
}

aPartialFunction.isDefinedAt(67)  // false，判断输入是否在定义域内
aPartialFunction(2)               // 56，正常调用
// aPartialFunction(57273)        // 抛 MatchError
```

`PartialFunction[A, B]` 是 `Function1[A, B]` 的子 trait，多了 `isDefinedAt(x): Boolean` 方法。它的好处是**把"我接受什么输入"作为类型的一部分**。

### `lift` 和 `orElse` 让偏函数变得实用

```scala
// lift：把半函数提升成"全函数"，未匹配的输入返回 None 而不是抛异常
val lifted = aPartialFunction.lift
lifted(2)   // Some(56)
lifted(98)  // None

// orElse：把多个偏函数拼起来，类似 match 的 |
val pfChain = aPartialFunction.orElse[Int, Int] {
  case 45 => 67
}
pfChain(2)   // 56
pfChain(45)  // 67
```

### 真实案例：聊天机器人

```scala
val chatbot: PartialFunction[String, String] = {
  case "hello"    => "hi! my name is HAL9000"
  case "goodbye"  => "Once you start talking to me, there is no return, human!"
  case "call mom" => "Unable to find your phone without your credit card"
}

scala.io.Source.stdin.getLines().map(chatbot).foreach(println)
// 用户的每行输入被 chatbot 处理后打印
```

注意 `map(chatbot)`——`List.map` 接受 `A => B`，但这里传的是 `PartialFunction[String, String]`，依然能编译运行。Scala 在编译器层面允许你把偏函数"当作"全函数传给需要全函数的 API（内部用 `isDefinedAt` 做过滤），**这种自然的"鸭子类型"在 `PartialFunction` 上得到了体现**。

### 实战经验：什么时候用 `PartialFunction`

- **错误处理**：把"能处理的异常"包装成偏函数，未匹配的异常走默认分支。
- **路由匹配**：Web 框架里把"路径 → 处理函数"映射成偏函数，未匹配的路径返回 404。
- **状态机转移**：只对部分状态有效的转换。

但**不要因为"PartialFunction 这个词听起来高大上"就到处用**——它本质是"`Function1` + `isDefinedAt`"，什么时候需要"**我需要知道我能不能处理这个输入**"就用它。

## 四、实战：自己实现一个纯函数式 `Set`

讲了这么多 FP 概念，让我们用代码把"FP 思维"做一次完整的具象化。我们用 70 行实现一个完整的函数式 `Set` 抽象数据类型。

### 接口设计

```scala
trait FuncSet[A] extends (A => Boolean) {
  // Set 本身就是一个函数：set(elem) 等价于 set.contains(elem)
  def apply(elem: A): Boolean = contains(elem)

  def contains(elem: A): Boolean
  def +(elem: A): FuncSet[A]                    // 加一个元素，返回新 Set
  def ++(anotherSet: FuncSet[A]): FuncSet[A]    // 拼接两个 Set

  def map[B](f: A => B): FuncSet[B]
  def flatmap[B](f: A => FuncSet[B]): FuncSet[B]
  def filter(predict: A => Boolean): FuncSet[A]
  def foreach(f: A => Unit): Unit
}
```

注意 `FuncSet[A] extends (A => Boolean)`——**让 `Set` 本身就是一个函数**。`set(elem)` 等价于 `set.contains(elem)`。这是 Scala 函数式编程里"数据即函数、函数即数据"的体现。

### 两个实现类

```scala
class EmptySet[A] extends FuncSet[A] {
  def contains(elem: A) = false
  def +(elem: A) = new NonEmptySet[A](elem, this)
  def ++(anotherSet) = anotherSet
  def map[B](f: A => B) = new EmptySet[B]
  def flatmap[B](f: A => FuncSet[B]) = new EmptySet[B]
  def filter(p: A => Boolean) = this    // 任何 predicate 在空集上都是空集
  def foreach(f: A => Unit) = ()
}

class NonEmptySet[A](head: A, tail: FuncSet[A]) extends FuncSet[A] {
  def contains(elem: A) = elem == head || tail.contains(elem)
  def +(elem: A) = if (this contains elem) this else new NonEmptySet[A](elem, this)
  def ++(anotherSet) = tail ++ anotherSet + head   // 递归拼接
  def map[B](f: A => B) = (tail map f) + f(head)   // 递归映射
  def flatmap[B](f: A => FuncSet[B]) = (tail flatmap f) ++ f(head)
  def filter(p: A => Boolean) = {
    val filteredTail = tail filter p
    if (p(head)) filteredTail + head else filteredTail
  }
  def foreach(f: A => Unit) = {
    f(head)
    tail foreach f
  }
}
```

注意几个真正 FP 风格的设计：

1. **`+` 和 `++` 都是不可变操作**——返回新 `Set`，原 `Set` 不变。这是 FP 的"持久化数据结构"思想。
2. **`map` / `flatMap` / `filter` 全递归实现**——没有一行循环。
3. **伴生对象工厂方法**：

```scala
object FuncSet {
  // 工厂方法：把 varargs 构造为 FuncSet
  def apply[A](values: A*): FuncSet[A] = {
    @tailrec
    def buildSet(valSeq: Seq[A], acc: FuncSet[A]): FuncSet[A] =
      if (valSeq.isEmpty) acc
      else buildSet(valSeq.tail, acc + valSeq.head)
    buildSet(values.toSeq, new EmptySet[A])
  }
}
```

调用方可以这样用：

```scala
val s = FuncSet(1, 2, 3, 4)
s + 5 ++ FuncSet(-1, -2) + 3 map (x => x * 10) filter (_ % 2 == 0) foreach println
```

整个调用链没有任何命令式循环，全部由 HOF 串联起来。**理解了这段代码，你就理解了 FP 的核心思维方式：递归 + 不可变 + 组合**。

## 五、内置集合操作：那些你以为懂但其实不太懂的细节

```scala
val list1 = List("programming", "scala")

// :: 向右结合，依次向 list 头部追加元素
val list2 = "People" :: "should" :: "read" :: list1
// list2 = List("People", "should", "read", "programming", "scala")

// Nil 是 List[Nothing]，因为 Nothing 是所有类型的子类型
val list3 = "Programming" :: "Scala" :: Nil

// ++ 拼接两个 list，顺序与左操作数一致
val list4 = Seq("people", "should", "read") ++ list1
```

**三个反直觉但很重要的细节**：

1. **`::` 是右结合的**：上面讲过，链式调用按"从右往左"自然地往 list 前加元素。
2. **`Nil` 是 `List[Nothing]`**：所以 `Nil` 可以被加到任何类型的 `List` 后面。
3. **`++` 和 `::` 都不会修改原集合**：每次都返回新集合，原集合完全不变。

### `Vector` vs `List` 的选择

```scala
// Vector 的所有操作都是 O(1)，List 头部 O(1) 但访问第 n 个是 O(n)
val vector = "people" +: "should" +: "read" +: Vector.empty
```

如果你的场景是"经常追加 / 索引访问"，用 `Vector`；如果是"频繁 head / tail 递归"，用 `List`。

### `fold` vs `reduce`

```scala
val listElements = List(1, 2, 3, 4)
listElements reduce (_ + _)         // 10，从第一个元素开始
listElements.fold(10) (_ * _)       // 240，初始值 10 参与第一次计算
```

`reduce` 在空集合上会抛异常，`fold` 提供初始值。

注意一个坑：`fold` 在 `List` 上是 `foldLeft`（从左到右），但在 `Set`/`Map` 上**顺序不定**——所以对无序集合只能用"满足结合律"的操作（求和、拼接），不能用"对顺序敏感"的操作。

### Map 的"加减"运算符

```scala
val stateCapitals = Map("Alabama" -> "Montgomery", "Alaska" -> "Juneau")
val stateCapitals2 = stateCapitals + ("Virginia" -> "Richmond")  // 加元素
val elements = stateCapitals2 - "Wyoming" - "Alabama"             // 减元素（链式）
```

### 互递归：`TailCalls` 的存在意义

普通尾递归要求"函数自己调用自己"才能优化。但如果你有两个函数**互相递归**（比如奇偶判断），普通尾递归就没法处理了，因为每次都会新建栈帧。

```scala
import scala.util.control.TailCalls._

// TailRec 把每次递归包成"延续"对象，让 JVM 在不增长栈的情况下完成互递归
def isEven(xs: List[Int]): TailRec[Boolean] =
  if (xs.isEmpty) done(true) else tailcall(isOdd(xs.tail))

def isOdd(xs: List[Int]): TailRec[Boolean] =
  if (xs.isEmpty) done(false) else tailcall(isEven(xs.tail))
```

`TailRec` 是 Scala 在 JVM 限制下做出的一种妥协——**为了让你能用函数式风格写"互相调用"的递归**。

## 六、`Option` / `Some` / `None`：消灭"空指针"的类型方案

讲个故事：我曾经用 Java 写过一个订单查询接口，`Map.get("userId")` 返回 `null` 时我忘了判空，直接 `.toUpperCase()`——**线上 NPE 爆了 8 分钟我才反应过来**。

Scala 的 `Option` 专门为这个问题而生：

```scala
val dogBreedsOptional = List(
  Some("doberman"), None, Some("yorkshire terrier"),
  Some("dachshund"), None, Some("scottish terrier"), None
)

// 在 for 推导式里用 <- 自动解开 Option
// breedOption 是 Option[String]，breed 是 String
// None 会被自动跳过，不会抛异常
for {
  breedOption <- dogBreedsOptional
  breed       <- breedOption            // Some(breed) 自动解构，None 跳过
  upcasedBreed = breed.toUpperCase
} print(upcasedBreed + " ")
```

第二个 `<-` 后面跟的是 `breedOption`（`Option[String]`），不是 `List`——这在 Scala 里完全合法。`<-` 在 for 推导式里会"自动调用 `Option.flatMap`"——如果 `breedOption` 是 `Some(breed)`，就解开；如果是 `None`，这个元素就被跳过。

如果你更倾向于"显式模式匹配"，可以这样写：

```scala
for {
  Some(breed) <- dogBreedsOptional   // None 直接被过滤掉
  upcasedBreed = breed.toUpperCase
} print(upcasedBreed + " ")
```

`Some(breed)` 是个模式——它只在 `dogBreedsOptional` 的元素是 `Some` 时匹配成功，`None` 直接被过滤掉。

### 为什么 `Map.get` 返回 `Option`？

```scala
val stateCapitals = Map("Alabama" -> "Montgomery", ...)
stateCapitals.get("Alabama")                // Some("Montgomery")
stateCapitals.get("unknown")                // None
stateCapitals.get("unknown").getOrElse("Oops!")  // "Oops!"
```

为什么 `Map.get` 不返回 `null`？因为 `null` 没法强制你处理"key 不存在"的情况。返回 `Option` 后，**编译器让你没法直接 `.toUpperCase`**——你必须先决定怎么处理 `None`（用 `getOrElse`、用 `match`、用 for 推导式）。

这是 Scala 解决"十亿美元错误"（`NullPointerException`）的核心方案：**用类型表达"可能有、可能没有"，让编译器帮你强制处理**。

## 本篇小结

这一篇覆盖的 6 个话题——`Function1`、HOF、`PartialFunction`、`FuncSet`、内置集合、Option——它们都围绕一个中心思想：

> **函数是一等公民，集合是函数的舞台，类型帮你消灭运行时错误。**

- 函数是值，所以可以传、可以存、可以组合。
- HOF + for 推导式是处理集合的"FP 流水线"。
- `PartialFunction` 把"我能处理什么"作为类型的一部分。
- `FuncSet` 演示了 FP 思维：递归 + 不可变 + 组合。
- `Option` 用类型系统消灭"空指针"。

## 🔜 下篇预告

这一篇我们讲了"函数是值"和"集合是舞台"——Scala 函数式编程的两条主线。

但 Scala 最"魔法"的部分，是它的**隐式机制**。下一篇我会讲一个让人抓狂的小故事：2019 年我在代码 review 里看到 `count.isEven()`，盯着 `Int` 的源码翻了 10 分钟——`Int` 根本没有 `isEven`！那为什么能编译？

答案藏在一个 `implicit class` 关键字背后，**它让 21 行的 Scala 拥有了"运行时不存在的能力"**。我们将从 4 种隐式机制（参数、转换、Type Class、Phantom Types）逐一拆解，最后用"用 Scala 给 Java API 注入泛型"这个实战案例收尾——这一篇会让你真正理解为什么 Scala 让人"又爱又恨"。
