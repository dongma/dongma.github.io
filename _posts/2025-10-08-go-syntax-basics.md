---
layout: post
title: Go基础语法介绍
date: 2025-10-08 +0800
tags: ["Go"]
---

<!-- more -->
> `go`语言变量的声明、`if`分支、循环、`switch`语法、数组、`slice`和`map`的使用

### 变量声明
`go`变量声明的语法是：`var name type = expression`，如下声明一个`s`的字符串， 当变量未赋值时，会使用零值，对字符串则是空串，而`slice`、指针、`map`、`chan`类型则是`nil`，`go`常见的类型有：整数、浮点数、复数、`bool`和字符串类型。

经常混淆的一点是，在`scala`中`var`声明的变量值是可以改变的，但是`val`声明的变量表示常量，其在赋值后，变量的值不能改变。
```go
// 变量未赋值时，会提供默认的零值
var s string
// 多变量声明，会根据具体值推断类型，声明变量未使用时，编译器会报错
var b, f, v = true, 2.3, "four"
```
短变量的声明使用`:=`，例如：`i := 100`声明了一个变量`i`为`100`，另外，将整型变量转换为字符串，可以使用`strconv.Itoa`方法：
```go
i := 100
fmt.Println("short variable: " + strconv.Itoa(i))
// strconv.FormatInt将整型转换为对应进制
fmt.Println("strconv.FormatInt(val,10): " + strconv.FormatInt(int64(i), 10))
// 将100转换成了对应的ascii码值，100 -> d
var char = string(100)
```
谈谈`var`、`:=`、`new()`和`make()`的区别，`new()`会按类型分配空间，返回变量地址，而`make()`常用于`chan`、`map`和`slice`的初始化：
```go
// go创建变量方式：var, :=, new , make的区别，new()分配内存的函数，返回内存地址
p := new(int)
*p = 29
fmt.Printf("val: %v \n", *p)
// make()分配内存的函数，用于chan、map和slice，返回的数据类型是3个类型本身
array := make([]string, 5)
array[0] = "hello"
fmt.Printf("array val: %v \n", array)
```
### 条件分支、循环流程
`go`中分支条件判断与`java`类似，不过其支持在`if`条件表达式中声明临时变量，如下：
```go
// 声明临时变量plus，score为方法中传递的参数
if plus := score + 3; plus >= 80 {
    return true
}
```
`go`中`for`循环遍历列表、`map`元素有两种方式，一种是传统的`for..i`遍历，另一种是`for range`循环，循环中的第一个参数表示下标，表示元素的索引，当不使用此变量时，可用下划线`_`替代，这一点和`scala`的语法很类似。
```go
// sum for i..循环实现对数组元素累加，传统的方式for..累加
func sum(elems []int64) int64 {
	var sumVal int64 = 0
	for i := 0; i < len(elems); i++ {
		sumVal += elems[i]
	}
	return sumVal
}
// sum for range循环实现对数组元素累加
func sumForRange(elems []int64) int64 {
    var sumVal int64 = 0
    for _, val := range elems {
        sumVal += val
    }
    return sumVal
}
```
### 函数的声明
`go`中函数声明的语法如下，每个函数声明都包含一个名字、一个形参列表、一个可选的返回列表及函数体，如下，是一个判断成绩是否为优秀的方法：
```go
func name(parameter-list) (result-list) {
	body
}
// isGood score为入参分数，根据分数是否达到80分，判断其是否优秀(有5分的加分项)
func isGood(score int64) bool {
    if plus := score + 3; plus >= 80 {
        return true
    }
    return false
}
```
### 数组、切片和映射
与`java`不同，`go`中的数组是值传递类型，数组声明的方式有`4`种，第`4`种可以根据元素下标指定该位置的值:
```go
// 第一种: 
var arr1 [3]int = [3]int{3, 6, 9}
// 第二种:
var arr2 = [3]int{1, 4, 7}
// 第三种:
var arr3 = [...]int{4, 5, 6, 7}
// 第四种，可以根据元素下标->指定元素的值
var arr4 = [...]int{2:66, 0:33, 1:99, 3:88}
```
创建切片的方式有`3`种，可以引用一个数组，或通过`make`方法创建一个切片，或切片直接引用一个数组。切片结构体可任务包含`3`个属性：`指向数组的ptr`、`切片的元素数`、`切片的容量`。
```go
// 方式一：直接引用一个数组
var intarr [6]int = [6]int{3, 6, 9, 1, 4, 7}
slice := intarr[1:3]
// 方式二：通过make内置函数创建切片，基本语法：var 切片名[type] = make([], len, [cap])
// make参数说明，1:切片类型、2:切片长度 3:切片的容量
slice2 := make([]int, 4, 20)
// 方式三: 直接引用一个数组
slice3 := []int{1, 4, 7}
```
创建`map`有`3`种不同的方式，通过`make`声明变量不指定容量大小时，默认为`1`，也可以通过`{}`初始化数据：
```go
// 方式1: 定义map变量，只声明map，内存没有分配内存空间，必须通过make函数进行初始化，才会分配空间
var amap map[int]string
// 方式2: 使用make进行创建，然后设置key-value的值，在不指定cap时，map的容量为1
bmap := make(map[int]string)
bmap[200909001] = "张三"
// 方式3: 直接初始化值
cmap := map[int]string{
    20095452: "张三",
    20098765: "李四",
}
```
`map`中删除元素用`delete`方法，查找元素值用`cmap[key]`方法，若找到则`found`为`true`：
```go
// map中删除元素
delete(cmap, 20098766)
fmt.Printf("after delete 20098766, val: %v \n", cmap)
// map中查找元素，会返回两个结果，found表示元素是否存在? val表示值是否存在？
val, found := cmap[20095452]
```
`map`中`for-range`对元素的遍历是无序的（每次返回的结果都不同），原因：在`mapiterinit`方法中生成了随机数，用于决定从哪里开始循环迭代。

### 结构体定义
`go`中结构体声明使用`type`类型名`struct`，属性名首字母为大写时，外界可以访问这个属性(`public`)。
```go
type Teacher struct {   // 声明一个Teacher类型
	Name string
	age int
	School string
}
```
创建结构体的实例、对象、变量，`(*t).Name = "John"`、`(*t).Age = 23`，一种，创建`struct`实例可使用`new`关键字。另一种，也可以用`&`取地址符，然后按属性声明的顺序赋字段值。
```go
var t *Teacher = new(Teacher)
(*t).Name = "John"  // *的作用，根据地址取值
t.Age = 23  // go编译器优化的部分，将t.Age优化成 (*t).Age = 23
// 创建结构体变量，使用&取地址符
var t2 *Teacher = &Teacher{"John", 23}
```
`go`中类型转换，必要显示的指定类型名称，`Student(p)`但需要显示的指定类型。结构体中的方法，对于普通类型`(p Person)`是值传递的（复制），而对于`(p *Person)`则是地址传递的。另外，如果一个类型实现了`String()`这个方法，则`fmt.Println`默认就回调用类型的`String()`进行输出。
```go
type Student struct {
	Age int
}
type Person struct {
	Age int
}
// 给Person绑定方法test，访问权限控制：方法首字母小写，则只能在本包内访问，方法首字母大写，则可以在本包和其它包内访问。
func (p Person) test() {
	fmt.Println(p.Name)
}

var p Student = Student{10}
var p Person = Person{10}
// 实际使用，执行类型转换
s = Student(p)
// 当对象有多个属性时，创建字段也可以按字段称指定字段值
var p2 Student = Student{Age: 10}
```
`go`语言中的继承，类似于类型组合。在继承类型中包含基础类型，此时继承类型中就有了基础类型的所有属性字段，这样`SeniorStu`就有了`Person`和`Student`的属性。
```go
type SeniorStu struct {
    Person
	Student
} 
```
`go`中的接口，接口的定义：定义规则、定义规范、定义某种能力。实现声明的接口，只需要实现方法的内容即可。在`go`中，多态是通过接口实现的，可以按照同一个接口来调用不同的实现，这时接口变量就呈现不同的形态。
```go
type SayHello interface {
    sayHello()  // 声明没有实现的方法
} 
type Chinese struct {}  // 中国人，实现sayHello方法
func (p Chinese) sayHello() {
	fmt.Println("你好")
}
type American struct {} // 美国人，实现sayHello方法
func (p American) sayHello() {
    fmt.Println("hi")
}
```
类型断言，`value, ok = element.(T)`，这里的`value`就是变量的值，`ok`是一个`bool`类型，`element`是`interface`变量，`T`是断言的类型。
```go
var ch = Chinese{}
am, ok := ch.(American) // 显然ch不是American类型，故ok为false
// 此外，switch语法也可以
switch s.(type) {
    case type1:
		// xxx
	case type2:
		// xxx
}
```