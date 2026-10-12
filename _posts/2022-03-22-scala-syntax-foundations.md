---
layout: post
title: Scala 让你又爱又恨的语法，到底在表达什么
tags: ["Scala"]
---
Scala 的语法看起来"花样繁多"，背后其实只有一条主线：**让编译器多做一点，让运行时少错一点**。这一篇我们从 `val` 讲起，途经 case class、sealed、`@tailrec`、7 个语法糖、模式匹配 8 个 Trick，一直到自定义 `unapply`——把 Scala 入门的"语法全景"梳理一遍。

<!-- more -->

## 序：第一次被 Scala 吓到的那个晚上

2018 年某个加班的晚上，我打开同事发过来的一段 Scala 代码，看了三分钟没敢说话。

```scala
val pf1: PartialFunction[Any, String] = { case s: String => "yes" }
val pf2: PartialFunction[Any, String] = { case d: Double => "yes" }
val pf = pf1 orElse pf2
List("str", 3.14, 10) foreach { item => print(pf(item).toString + "") }
```

我心里想的是：这是什么？偏函数？`orElse`？为什么 `List foreach` 的花括号能这么写？为什么 `print` 后面那个 `+ ""` 是多余的但又不报错？

那晚我盯着屏幕一个小时，把每个看似"魔法"的语法反编译成 Java，才第一次真正理解 Scala 设计的哲学：**它不是在 Java 上面加糖，它是用类型系统重新表达程序员的意图**。

这一篇我想讲清楚：Scala 那些"看起来很怪"的语法，到底想说什么。

## 一、`val` 不是一个简单的 `final`

我们先看最基础的部分：

```scala
val array: Array[String] = new Array[String](5)   // val 声明，引用不可变
array(0) = "hello"                                // 但 array[0] 是可变的！

var stockPrice: Double = 100.0                    // var 才是真正"可重新赋值"
stockPrice = 200.0
```

从 Java 转过来的人第一反应是"哦，`val` 就是 `final`"。这话对了一半——`val` 承诺的是"**这个引用**不能再指向别处"，但它指向的对象**完全可以被修改**。

`array` 是 `val`，所以你不能写 `array = new Array[String](10)`，但 `array(0) = "hello"` 完全合法。

这跟 Java 的 `final` 一样。但 Scala 在"鼓励你用 `val`"这件事上比 Java 认真得多——社区约定是"**能用 `val` 就别用 `var`**，能用 case class（不可变数据结构）就别用普通 class"。Scala 的并发优势很大程度上来自这个约定：当你看到一个 `val`，你就知道它的引用在生命周期里是稳定的。

### 一个小细节：方法名可以是"操作符"

Scala 里那些看着像操作符的 `+`、`-`、`*`、`::`、`++`，**全都是方法名**。这意味着你可以用反引号包任何字符序列当方法名：

```scala
class Crocodile extends Animal with Carnivore with Philosopher {
  // 方法名是 ?!，能这样写是因为 Scala 里"操作符"就是方法
  override def ?!(thought: String): Unit = println(s"I was thinking: $thought")
}

val aCroc = new Crocodile
aCroc ?! "What if we could fly?"   // 调用的是 aCroc.?!("What if we could fly?")
```

这看着搞笑，但 ScalaTest 的 `should`/`must`、`Akka` 的 `!`（tell）都靠这个特性。**Scala 的"操作符重载"不是重载，是操作符就是方法。**

## 二、Case Class：你以为只是个数据类，其实它是 DSL 的地基

```scala
case class Point(x: Double = 0.0, y: Double = 0.0) {
  // shift 方法返回一个新 Point，原 Point 不变
  def shift(deltax: Double = 0.0, deltay: Double = 0.0) =
    copy(x + deltax, y + deltay)
}
```

`case class` 关键字是 Scala 最被低估的特性。一行 `case class`，编译器自动给你 7 样东西：

| 自动生成 | 作用 |
|----------|------|
| 构造器 | `Point(1, 2)` 不需要 `new` |
| 伴生对象 | `Point` 既能当类型又能当工厂方法 |
| `unapply` | 模式匹配的解构器（后面详讲） |
| `equals` / `hashCode` | 基于字段值比较，不是引用 |
| `toString` | 输出 `Point(1.0, 2.0)` 这种可读形式 |
| `copy` | 命名参数做"修改"：`point.copy(y = 6.6)` |
| `Serializable` | 自动混入 |

参数还能给默认值，于是 `Point()` 是原点，`Point(1.0)` 是 `(1.0, 0.0)`。

这套机制的本质：**case class 是一种带类型安全的数据结构**，但因为自带 `unapply`，它就成了 Scala 模式匹配、for 推导式、JSON 库、ORM 库的基础。**没有 case class，Scala 生态会塌掉一半**。

## 三、`sealed`：让"忘了写某个 case"从 bug 变成编译错误

```scala
sealed abstract class Option[+A] {}   // +A 后面会讲，sealed 现在先记住
```

这一行非常不起眼，但改变了整个错误处理生态。`sealed` 翻译过来是"封闭的"——它告诉编译器：**所有继承我的子类必须写在同一个源文件里**。

这意味着编译器能看到所有可能的子类，**所以你写 `match` 的时候漏掉一个 case，编译器会警告或报错**。

对比一下普通抽象类：

```scala
abstract class Animal                  // 任何文件、任何包都能继承
sealed abstract class Animal           // 只能在本文件继承
```

这个特性的真正威力：当你有 5 个 case class 表达"业务状态"，Scala 编译器能告诉你"嘿，你 `match` 没处理 `Cancelled` 这个状态"。

### 我自己的一个教训

我曾经用普通 `abstract class` 写过一个枚举层级，三个月后加了个新 case，**没有任何编译器警告**——直到 `MatchError` 在生产环境爆了 5 分钟。从那之后我给自己立了规矩：**有限层级一律 `sealed`**。

## 四、`@tailrec`：让"忘了写累积器"从 StackOverflow 变成编译错误

递归是 FP 的灵魂，但 JVM 栈是有限的。Java 递归跑着跑着 `StackOverflowError`，Scala 给了一个温柔的护栏：

```scala
def factorial(intValue: Int): Long = {
  @tailrec   // 强制要求下面的 fact 必须是尾递归，否则编译报错
  def fact(value: Int, accumulator: Int): Long = {
    if (value <= 1) accumulator
    else fact(value - 1, value * accumulator)   // 递归调用之后没有任何其他操作 = 尾递归
  }
  fact(intValue, 1)
}
```

`@tailrec` 注解告诉编译器："这个递归**必须是**尾递归"。如果你的递归不是（比如忘了带累积器），**编译直接失败**——而不是运行时栈爆了才被发现。

注意几个细节：

- **累积器是必须的**：把"当前结果"作为参数传下去，递归调用之后**不接任何操作**。
- **嵌套函数而非递归自身**：`factorial` 内部嵌套 `fact`，外层只调用一次。
- **类型必须显式写**：因为类型推断看不到递归终止那条分支的累加器初始值。

背后是一个更大的设计哲学：**Scala 倾向于让"潜在的错误"在编译期显形**。`@tailrec`、`sealed`、case class、`Option` 都是这条主线的体现。

## 五、Dark Sugars：那些怪写法背后都是普通方法调用

讲个真实感受：第一次看到 `2 :: List(3, 4)` 的时候，我本能地以为这是"自定义的语法"。后来才知道——它就是 `List(3, 4).::(2)`。

Scala 有一整套"语法糖"让代码读起来像数学公式或自然语言。理解每个糖的反编译真相，你就能看懂 90% 的奇怪 Scala 代码。

### 糖 1：单参方法的花括号调用

```scala
def singleArgMethod(arg: Int): String = s"$arg little ducks..."

val description = singleArgMethod {
  // 花括号里的表达式会被求值后传给 singleArgMethod
  42
}   // 等价于 singleArgMethod(42)
```

花括号和小括号在调用单参方法时**可以互换**。这有什么意义？看这个：

```scala
List(1, 2, 3).map { x => x + 1 }   // 比 List(1, 2, 3).map(x => x + 1) 看着舒服
```

很多 DSL 风格 API 都会利用这一点。

### 糖 2：单抽象方法的 lambda 化

```scala
trait Action {
  def act(x: Int): Int
}

val aFunkyInstance: Action = (x: Int) => x + 1   // 编译器知道 Action 只有一个抽象方法

val aSweetThread = new Thread(() => println("sweet, scala!"))
// 替代了 new Thread(new Runnable { override def run(): Unit = ... })
```

只要一个 trait（或 Java interface）只有一个抽象方法，就可以用 lambda 直接实现——因为编译器知道"这个 lambda 就是要实现那个唯一的抽象方法"。Java 8 之后也支持这个特性（叫 SAM），但 Scala 早了好多年。

### 糖 3：`::` 操作符的右结合

```scala
val prependedList = 2 :: List(3, 4)
// 不是 2.::(List(3, 4))，而是 List(3, 4).::(2)
// 因为以 : 结尾的方法是右结合的

val chain = 1 :: 2 :: 3 :: List(4, 5)
// 实际执行顺序：List(4, 5).::(3).::(2).::(1)
```

Scala 明确规定：**以 `:` 结尾的方法调用时是右结合的**。这让 cons 操作（向列表头部追加元素）的链式调用 `1 :: 2 :: 3 :: List(4, 5)` 读起来就像"从右往左"自然地往 list 前加元素。

### 糖 4：多词方法名

```scala
class TeenGirl(name: String) {
  // 反引号包起来的任意字符串都能当方法名
  def `and then said`(gossip: String) = println(s"$name said $gossip")
}

val lilly = new TeenGirl("Lilly")
lilly `and then said` "Scala is so sweet"   // 调用是 lilly.`and then said`("...")
```

看着搞笑，但 ScalaTest 的 `should`/`must`/`when`、Akka 的 `!`（tell）都靠这个特性写出"自然语言风格"的测试和 DSL。

### 糖 5：中缀类型

```scala
class Composite[A, B]
val composite: Int Composite String = ???

class -->[A, B]
val towards: Int --> String = ???
```

类型名也能用符号。这在函数式编程库（如 Cats 的 `Either[A, B]`）里偶尔出现，让某些领域特定的类型读起来更像数学公式。

### 糖 6：`apply` 和 `update` 的括号语法

```scala
val anArray = Array(1, 2, 3)
anArray(2)        // 编译为 anArray.apply(2)，读取元素
anArray(2) = 7    // 编译为 anArray.update(2, 7)，写入元素
```

`apply` 是"用括号调用方法"的语法糖，`update` 是"用括号赋值"的语法糖。`Array`、`MutableMap` 等可变集合的整个 API 都靠这两个方法。

### 糖 7：setter 语法

```scala
class Mutable {
  private var internalMember: Int = 0   // 真正的存储

  def member = internalMember           // getter

  def member_=(value: Int): Unit =      // setter，方法名以 _= 结尾
    internalMember = value
}

val a = new Mutable
a.member = 42   // 编译为 a.member_=(42)
```

只要方法名以 `_=` 结尾，就可以用等号赋值。这是 Scala getter/setter 模式的基础——也是 Scala 写可变容器时"看上去像属性访问，但其实是方法调用"的核心。

## 六、模式匹配 8 个 Trick：远超 switch 的解构器

终于到了 Scala 真正的"杀手锏"——模式匹配。很多人把 `match` 理解为"加强版的 switch"，这低估了它。**Scala 的模式匹配本质是一个编译器驱动的解构器**：它会根据值的结构（`List(_, 2, _)`、`Person(name, age)`）做解构 + 提取，所有这些都在一个 `match` 里完成。

下面这 8 个 Trick 来自实战，每个对应一类真实场景。

### Trick 1：列表按位置解构

```scala
val aList = List(1, 2, 3)
val listDescription = aList match {
  case List(_, 2, _) => "List containing 2 on its second position"
  case _             => "unknown list"
}
```

在解析 CSV、协议头、配置文件时，"按位置匹配列表"特别有用。

### Trick 2：Haskell 风格的 `::`

```scala
val numberList = List(1, 2, 3, 42)

// 模式 1 :: tail 既解构（head=1, tail=List(2,3,42)），又判断（列表非空）
val startsWithOne = numberList match {
  case 1 :: tail => s"list starts with one, tail is $tail"
}

def process(aList: List[Int]): String = aList match {
  case Nil              => "List is empty"
  case head :: tail     => s"list starts with $head, tail is $tail"
}
```

如果你写过 Haskell/Racket，这套语法应该很熟悉。

### Trick 3：变长模式 `_*`

```scala
val dontCareAboutTheRest = numberList match {
  case List(_, 2, _*) => "I only care about second number being 2"
  //      ^^^^^^^^
  //      _* 表示"任意多个元素，名字我不关心"
}
```

判断"列表至少有几个元素 / 某个位置必须是某值"时必备。

### Trick 4：中缀模式 + 末尾追加

```scala
val mustEndWithTheMeaningOfLife = numberList match {
  // List(1, 2, _) :+ 42 是个中缀表达式
  // 含义：一个 List(1, 2, 任意) 后面追加 42
  case List(1, 2, _) :+ 42 => "that's right, I have a meaning"
}
```

`:+` 是 List 的"末尾追加"操作符。

### Trick 5：按类型匹配

```scala
def gimmeAValue(): Any = 45

val gimmeTheType: String = gimmeAValue() match {
  case _: String => "I have a string"
  case _: Int    => "I have an int"
  case _         => "Something else"
}
```

Java 的 `instanceof` + 强制类型转换要写 5 行，Scala 一行搞定。处理 JSON 反序列化、配置解析、协议包时是杀手锏。

### Trick 6：命名绑定 `p@`

```scala
val bobInfo: String = bob match {
  // p@Person(name, age) 把整个对象绑定到 p，同时把字段解构到 name、age
  case p @ Person(name, age) => s"$name's info: ${requestInfo(p)}"
}
```

当你**既需要整个对象（传给其他方法），又需要解构其中的字段**时，这个 trick 必不可少。

### Trick 7：守卫（guards）

```scala
val ordinal2: String = aNumber match {
  case 1 => "first"
  case 2 => "second"
  case 3 => "third"
  case n if n % 10 == 1 => n + "st"   // 守卫：只有这个条件成立才算匹配
  case n if n % 10 == 2 => n + "nd"
  case n if n % 10 == 3 => n + "rd"
  case _ => aNumber + "th"
}
```

"按区间匹配"或"按某种规则计算"时非常方便。

### Trick 8：备选模式 `|`

```scala
val myOptionalList = numberList match {
  case List(1, _*) | List(_, _, 3, _*) => "I like this list"
  //                ^
  //                | 表示"或者"，只要左或右任一匹配就算匹配
  case _ => "I hate this list"
}
```

在"有多个合法结构"的场景下非常实用。

### 模式匹配不是 switch

回头看这 8 个 Trick，你会发现模式匹配至少做了 4 件事：

1. **解构**（case class、tuple、list）
2. **类型判断**（`_ : Type`）
3. **命名绑定**（`p@`）
4. **条件筛选**（guards + `|`）

每件事单独看都不复杂，但**它们能组合**——这才是 Scala 模式匹配真正的威力。

## 七、进阶 `unapply`：自己写"模式"

`case class` 的解构能力靠编译器自动生成的 `unapply` 在背后撑腰。但 Scala 允许你**完全自定义** `unapply`——这让模式匹配的能力扩展到了没有 case class 的类。

```scala
class Person(val name: String, val age: Int)

object Person {
  // 让 Person 也能参与模式匹配，并且加入"未成年不能匹配"的业务规则
  def unapply(person: Person): Option[(String, Int)] =
    if (person.age < 21) None        // 返回 None 表示"模式不匹配"
    else Some((person.name, person.age))

  // 重载：让 Person 这个 object 也能匹配 Int 类型的输入
  def unapply(age: Int): Option[String] =
    Some(if (age < 21) "minor" else "major")
}

val bob = new Person("Bob", 25)
val greeting = bob match {
  case Person(n, a) => s"Hi, my name is $n and I am $a year."
}
// greeting == "Hi, my name is Bob and I am 25 year."

val legalStatus = bob.age match {
  case Person(status) => s"my legal status is $status"
  //               ^^^^^^
  //      bob.age 是 Int，匹配的是 unapply(age: Int) 那个重载
}
// legalStatus == "my legal status is major"
```

注意 `Person.unapply` 返回 `None` 而不是抛异常——这就是"模式不匹配"和"运行时错误"的区别。`unapply` 返回 `None` 时编译器会继续尝试下一个 `case`，整个 `match` 都不匹配才抛 `MatchError`。

### 一个超优雅的 trick：`unapply` 可以返回 `Boolean`

```scala
// even 和 singleDigit 不是值，是返回 Boolean 的 unapply object
object even       { def unapply(arg: Int): Boolean = arg % 2 == 0 }
object singleDigit { def unapply(arg: Int) = arg > -10 && arg < 10 }

val n = 8
val mathProperty = n match {
  case singleDigit() => "single digit"   // 匹配 predicate
  case even()        => "an even number"
  case _             => "no property"
}
```

`even` 不是一个值，是一个 `unapply` 返回 `Boolean` 的 object。`case even()` 在 `n` 是偶数时匹配。

**这是 Scala 模式匹配里少有人知道但极其实用的 trick**——它让你把"某种谓词判断"包装成"模式"，复用模式匹配的语法。

### 极致自定义：`Wrapper[T]`

```scala
// unapply 的"标准签名"是 Option[T]
// 但编译器只要求返回类型有 isEmpty: Boolean 和 get: T 两个方法
abstract class Wrapper[T] {
  def isEmpty: Boolean
  def get: T
}

object PersonWrapper {
  def unapply(person: Person): Wrapper[String] = new Wrapper[String] {
    def isEmpty = false
    def get = person.name
  }
}

println(bob match {
  case PersonWrapper(n) => s"This person's name is $n"
  case _               => "An alien"
})
```

这给了框架作者（JSON 库、ORM 库）极大的自由度——可以构造"懒求值"或"抛错时给出更详细信息"的 `unapply`。

最后还有 `unapplySeq`，让你的自定义列表也能用 `List(1, 2, _*)` 这种 vararg 模式匹配——这是 `List`、`Stream`、`Vector` 都能匹配 vararg 的根本原因。

## 本篇小结

如果你让我用一句话总结 Scala 语法的设计哲学，我会说：

> **让编译器多做一点事，让运行时少错一点。**

- `val` 让"重新赋值"在编译期被禁止。
- `sealed` 让"忘记处理某个 case"在编译期被警告。
- `@tailrec` 让"忘了写累积器导致栈爆"在编译期被拒绝。
- 模式匹配让"判断 + 解构 + 提取"在一个表达式里完成。
- Dark Sugars 让代码读起来像自然语言，但**每一个糖都有反编译的真相**。

## 🔜 下篇预告

这一篇我们站在"语法层面"看 Scala——它的所有怪语法，背后都是为了让"潜在的错误"在编译期显形。

但 Scala 真正的灵魂在**函数式编程**。下一篇我会讲一个我亲身经历的故事：2019 年我写了一个 ETL 任务，Java 8 写法在生产环境 OOM 了，换成 Scala 21 行就稳如老狗——那一刻我才真正理解 "**函数是一等公民**" 不是噱头，是性能、正确性、可读性的全方位提升。

我们将从 `Function1` 这个"lambda 的真相"开始，到高阶函数、偏函数、自己手写一个函数式 `Set`，最后到 `Option` 和 for 推导式——这是一条"从命令式思维转向函数式思维"的完整路径。
