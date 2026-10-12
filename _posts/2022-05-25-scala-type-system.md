---
layout: post
title: Scala 类型系统 —— 让编译器替你做正确性检查
tags: ["Scala"]
---
Scala 2 的类型系统是图灵完备的——你可以在编译期写"图灵机级别"的证明和约束。这一篇我们从协变 / 不变 / 逆变开始，理解 `+T` 和 `-T` 表达的类型关系；然后看 `Nothing` 这个"所有类型的子类型"如何在协变泛型里做"空集"；最后用 Self-Type 解决"为了约束而继承"的设计臭味。掌握这 3 个核心特性，你就能读懂 Cats/Scalaz 的源码。

<!-- more -->

## 序：我第一次理解"Scala 类型系统是图灵完备"的那一刻

2020 年我在读 Cats 源码，看到一段完全看不懂的代码：

```scala
val comparison: _1 < _3 = <[_1, _3]
```

我盯着看了 5 分钟：**这不是值，是类型**。`_1 < _3` 是个**类型**，它表示"1 小于 3 这个证明"。编译器在编译期通过类型推导链"证明"了 1 < 3 成立，**整个过程没有任何运行时计算**。

如果换成 `<[_3, _2]`，编译直接报错。

那一刻我才真正理解 Martin Odersky 说的"**Scala 类型系统是图灵完备的**"是什么意思——你可以在编译期写"图灵机级别"的证明和约束。

日常开发里你不需要写这么深，但理解 3 个核心特性会让你读懂 Cats/Scalaz 的源码：

1. **协变 / 不变 / 逆变**（variance）——`List[+T]` / `Vet[-T]`
2. **`Nothing` / `Null` / `Unit` / `None` / `Nil` 五者区别**
3. **Self-Type**——`trait Diet { self: Person => ... }`

这一篇我们逐一拆解。

## 一、协变 / 不变 / 逆变：用 `+T` 和 `-T` 表达类型关系

讲个真实场景：你在写一个动物收容所系统，需要一个 `List[Animal]` 存所有动物，也需要 `List[Dog]`、`List[Cat]`。那么问题来了：

- `List[Dog]` 是不是 `List[Animal]` 的子类？——也就是 `val myDogs: List[Animal] = List(dog1, dog2)` 能不能编译？

直觉上"是的，狗是动物，狗的列表也应该是动物的列表"。这种关系叫**协变（covariance）**。

但**直觉有时候会骗你**。我们看代码：

```scala
class Animal
class Dog(name: String) extends Animal

val lassie = new Dog("Lassie")
val anAnimal: Animal = lassie        // Dog <: Animal，OK
val myDogs: List[Animal] = List(lassie)  // List 是协变的，OK
```

`List[+T]` 是协变的——`List[Dog] <: List[Animal]`。但有些类型**不能**是协变的。

### 三种 variance 的标记

```scala
// +T 协变：如果 Dog <: Animal，则 MyList[Dog] <: MyList[Animal]
class MyList[+T]

// T 不变：MyInvariantList[Animal] 和 MyInvariantList[Dog] 没关系
class MyInvariantList[T]
// val myDogs2: MyInvariantList[Animal] = new MyInvariantList[Dog]  // 编译错误

// -T 逆变：如果 Dog <: Animal，则 MyContravarianceList[Animal] <: MyContravarianceList[Dog]
class MyContravarianceList[-T]
val myDogs2: MyContravarianceList[Dog] = new MyContravarianceList[Animal]  // OK
```

`Vet[-T]` 是逆变最经典的例子：

```scala
// Vet 接受 Animal 作为参数，-T 表示 Vet 是逆变的
trait Vet[-T] {
  def heal(animal: T): Boolean
}

def gimmeAVet(): Vet[Dog] = new Vet[Animal] {
  override def heal(animal: Animal): Boolean = {
    println("you'll be fine")
    true
  }
}

val myDog = new Dog("Buddy")
val anyVet: Vet[Dog] = gimmeAVet()
anyVet.heal(myDog)  // "you'll be fine"，true
```

直觉上：能治"动物"的医生一定能治"狗"，所以 `Vet[Animal] <: Vet[Dog]`——`Vet` 是逆变的。

### 为什么 `Vet` 必须是逆变？

仔细看 `Vet[T].heal(animal: T)`——这里 `T` 在**输入位置**（方法参数）。如果 `Vet` 是协变（`Vet[+T]`），那 `heal` 方法的签名就变成 `(animal: T)` 在协变类里接收输入——**这叫 variance 污染**，编译器会直接拒绝。

规律：

| T 的"位置" | 允许的 variance |
|------------|---------------|
| 方法返回值（输出位置） | 协变（`+T`）或不 |
| 方法参数（输入位置） | 逆变（`-T`）或不 |
| 都没出现 | 协变 / 逆变 / 不变都行 |

`List[+T]` 的 `head: T` 是输出位置，所以可以协变；`Vet[-T]` 的 `heal(animal: T)` 是输入位置，所以必须逆变。

### 实战踩坑：variance 污染

```scala
class MyList[+T] {
  def add(elem: T): MyList[T] = ???  // 编译错误！covariant type T occurs in contravariant position
}
```

为什么？因为 `add` 接收 `T`（输入位置），但 `MyList` 是协变的。

解决办法：

```scala
class MyList[+T] {
  def add[U >: T](elem: U): MyList[U] = ???  // OK
}
```

加一个"更宽"的类型参数 U（T 的父类），让方法本身变得"更通用"——这是 Scala 库作者经常用的小 trick。

## 二、`Nothing` 是所有类型的子类型

讲个笑话：刚学 Scala 的时候，我以为 `null`、`Nil`、`None`、`Nothing` 是同一个东西——都是"空"嘛。**完全不是**。

我们看 `Nothing` 在实际代码里的关键应用：

```scala
abstract class MyList[+T]  // 协变
class NonEmptyList[+T](head: T, tail: MyList[T])
object EmptyList extends MyList[Nothing]  // Nothing 是所有类型的子类型

val listOfString:    MyList[String]    = EmptyList
val listOfIntegers:  MyList[Integer]   = EmptyList
val preciousList:    MyList[MyPrecious] = EmptyList
```

`EmptyList` 是 `MyList[Nothing]`，因为 `Nothing` 是所有类型的子类型，**所以 `MyList[Nothing]` 也是所有 `MyList[T]` 的子类型**——这让一个 `EmptyList` 实例能"扮演"任何 `MyList[T]` 的空集。

**这就是协变泛型里"空集"的根本原理**。

### `throw` 表达式的类型是 `Nothing`

```scala
def gimmeNumber(): Int = throw new NoSuchElementException
def gimmeString(): String = throw new NoSuchElementException
def gimmePrecious(): MyPrecious = throw new NoSuchElementException
```

`throw new ...` 表达式的类型是 `Nothing`——`Nothing` 是 `Int`/`String`/`MyPrecious` 的子类型，所以这些函数可以"返回"任何声明类型。编译器在 throw 之后认为代码"永远跑不到"，优化也随之而来。

`???` 也是一个 `Nothing` 类型的占位：

```scala
def someUnimplementedMethod(): String = ???
// 等价于 def someUnimplementedMethod(): String = throw new NotImplementedError
```

### `null` / `Nil` / `None` / `Unit` / `Nothing` 速查

| 名称 | 是什么 | 用途 |
|------|--------|------|
| `null` | Java 的空引用 | 兼容 Java，**Scala 不鼓励** |
| `Null` | 所有 `AnyRef` 子类型都能"装" null 的类型 | 主要在类型推导时出现 |
| `Nil` | `List[Nothing]` 的实例 | 空 List 哨兵 |
| `None` | `Option` 的"空"实例 | 表示"没有值" |
| `Unit` | 只有一个值 `()` 的类型 | 替代 Java `void` |
| `Nothing` | 所有类型的子类型 | 协变空集、`throw` 类型、`???` |

**最容易踩的坑**：`Null` 只能"装"引用类型（`AnyRef` 的子类型），不能装 `Int`、`Double` 等值类型：

```scala
val theNullReference: Null = null
val noString:  String    = theNullReference  // OK
val noPerson:  Person    = theNullReference  // OK
val noList:    List[Int] = theNullReference  // OK

// val noInt: Int = theNullReference  // 编译错误！Null 不能装值类型
```

## 三、Self-Type：用类型约束代替继承

讲个设计问题：你在写一个饮食系统，`Diet`（饮食）应该只能被 `Person`（人）使用。如果让 `Diet extends Person`，`Diet` 就成了"一种人"——语义不对（"饮食"不是"人"的子类）。

`Self-Type` 是 Scala 提供的解法：

```scala
trait Edible   // "可食用" 的标识
trait Person {
  def hasAllergiesTo(thing: Edible): Boolean   // 是否有某种过敏
}
trait Child extends Person
trait Adult extends Person

// 关键：self: Person =>
// 含义："任何混入 Diet 的 trait/class 必须也混入 Person"
trait Diet { self: Person =>
  def eat(thing: Edible): Boolean = {
    if (self.hasAllergiesTo(thing)) false   // 通过 self 访问 Person 的方法
    else 42 > 10
  }
}

trait Carnivore  extends Diet with Person
trait Vegetarian extends Diet with Person
```

`self: Person =>` 翻译成白话是："**任何混入 `Diet` 的 trait/class 必须也混入 `Person`**"。这里的 `self` 是个别名，在 `Diet` 内部可以用 `self.hasAllergiesTo(...)` 访问 Person 的方法。

使用示例：

```scala
class VegetarianAthlete extends Vegetarian with Adult {
  // 必须实现 Person 的 hasAllergiesTo（因为 Adult 没实现）
  override def hasAllergiesTo(thing: Edible): Boolean = false
}
```

编译器会强制 `VegetarianAthlete` 必须混入 `Person`（直接或通过 `Adult`），否则编译失败。**这就是用类型约束代替"为了约束而继承"的设计**。

### 三种方案的对比

**方案 1：让 `Diet extends Person`**——简单但有"为了约束而继承"的味道，违反 OO 设计原则（"is-a" 关系被滥用）。

**方案 2：加类型参数 `trait Diet[+T <: Person]`**——灵活但每个使用方都要带类型参数，调用时 `Diet[Adult]`、`Diet[Child]` 写起来很啰嗦。

**方案 3：Self-Type `trait Diet { self: Person => ... }`**——既能约束，又不污染继承链，调用方零负担。

**Self-Type 是 Scala 在"约束"和"灵活"之间找到的平衡**。在 Akka、Cats 这类库里，Self-Type 经常被用来表达"需要混入某些其他能力"——比如 `trait JsonCodec { self: Logger => ... }` 表达"JSON 编解码需要日志能力"。

## 系列收尾

走过这 4 篇，我们从"看得懂 Scala 代码"升级到了"理解 Scala 设计哲学"：

- **第 1 篇**：**让编译器多做一点，让运行时少错一点**。从 `val` 到 `sealed` 到模式匹配，所有怪语法都是为了让"潜在的错误"在编译期显形。
- **第 2 篇**：**函数是一等公民，集合是函数的舞台，类型帮你消灭运行时错误**。从 `Function1` 到 for 推导式到 `Option`，Scala 把"函数作为值"贯穿到了集合和错误处理里。
- **第 3 篇**：**隐式机制是 Scala 的"宏能力"**。`implicit` 关键字背后藏着 4 种完全不同的能力——参数、转换、Type Class、Phantom Types。用好了威力巨大，滥用就是给自己挖坑。
- **第 4 篇**：**Scala 类型系统是图灵完备的**。`+T`/`-T` 表达类型关系，`Nothing` 做协变空集，Self-Type 做混入约束——这些是读懂 Cats/Scalaz 源码的钥匙。

如果你想继续深入，这 4 篇之外还有几个方向：

- **Cats / Cats-Effect**：Scala 函数式编程生态的事实标准库，看完这 4 篇你就能直接读它的源码。
- **Higher-kinded types / Type members / Path-dependent types**：Scala 类型系统更深的能力。
- **Scala 3 (Dotty)**：新版本里 trait 可以带构造器参数、枚举更自然、语法更简洁，值得关注。

Scala 是一门"小众但强大"的语言。它不会像 Java 那样人手一把，但它在你需要"用类型系统做正确性证明"时，是少数能让你做到这件事的语言之一。

希望这 4 篇能帮你跨过"看得懂"到"能写出来"之间的那道坎。
