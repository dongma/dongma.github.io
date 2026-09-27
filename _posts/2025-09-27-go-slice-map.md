---
layout: post
title: Go Slice介绍
date: 2025-09-27 +0800
tags: ["Go"]
---

<!-- more -->
`go`语言中的切片对标于其他编程语言中通俗意义上的“数组”. 切片中的元素存放在一块内存地址连续的区域，使用索引可以快速检索到指定位置的元素；切片长度和容量是可变的，在使用过程中可以根据需要进行扩容.
- `array`：指向了内存空间地址的起点. 由于`slice`数据存放在连续的内存空间中，后续可以根据索引`index`，在起点的基础上快速进行地址偏移，从而定位到目标元素；
- `len`：切片的长度，指的是逻辑意义上`slice`中实际存放了多少个元素；
- `cap`：切片的容量，指的是物理意义上为`slice`分配了足够用于存放多少个元素的空间. 使用`slice`时，要求`cap`永远大于等于`len`;
```go
type slice struct {
    // 指向起点的地址
    array unsafe.Pointer
    // 切片长度
    len   int
    // 切片容量
    cap   int
}
```
初始化：声明但不初始化，方式为`var s []int`，`make`初始化`slice`也分为两种方式：
- `s := make([]int,8)`，此时会将切片的长度`len`和容量`cap`同时设置为`8`；
- 第二种方式，是分别指定切片的长度`len`和容量`cap`，`s := make([]int,8,16)`；
- 初始化连带赋值，`s := []int{2,3,4}`，会将`slice`长度`len`和容量`cap`均设置为`3`，同时完成对这`3`个元素赋值；
<div>
    <img src="https://pic1.zhimg.com/v2-e524355a081ff85126fc1e82211d1f4c_1440w.jpg" width="680"/>
</div>

**slice操作**，进行`slice`内容的截取，形如`s[a:b]`的格式，其中`a` `b`代表切片的索引`index`，左闭右开；
在对切片`slice`执行截取操作时，本质上是一次引用传递操作，因为不论如何截取，底层复用的都是同一块内存空间中的数据，只不过，截取动作会创建出一个新的`slice header`实例.
<div>
    <img src="https://pica.zhimg.com/v2-6bb03e03b44fd965cd411e81ad144218_r.jpg" width="680"/>
</div>

**切片扩容**, 当`slice`当前的长度`len`与容量`cap`相等时，下一次`append`操作就会引发一次切片扩容.
- 倘若预期的新容量超过老容量的两倍，则直接采用预期的新容量；
- 倘若老容量小于`256`，则直接采用老容量的2倍作为新容量；
- 倘若老容量已经大于等于`256`，则在老容量的基础上扩容`1/4`的比例并且累加上`192`的数值；
<div>
    <img src="https://pic2.zhimg.com/v2-af8b05aebcb03a881c7b2ce47e0fdc75_1440w.jpg" width="680"/>
</div>

### slice的问题：
1. 初始化切片`s`长度和容量均为`10`在`s`的基础上追加`append`一个元素，切片`s`的内容、长度以及容量分别是什么？
```go
func Test_slice(t *testing.T){
    s := make([]int,10)
    s = append(s,10)
    t.Logf("s: %v, len of s: %d, cap of s: %d",s,len(s),cap(s))
}
```
输出，`s: [0 0 0 0 0 0 0 0 0 0 10], len of s: 11, cap of s: 20`， `make([]int, 10)`其`len`和容量`cap`均为`10`，且前`10`个元素是已经切实被分配过的（零值）。在切片原容量小于`256`的情况下，扩容时会采用原容量的`2`倍作为新的容量，于是在新切片中，长度增加为`11`，而容量则翻倍变成`20`.

2. 初始化切片`s`长度为`0`，容量为`10`，在`s`的基础上追加`append`一个元素，切片`s`的内容、长度以及容量分别是什么？
```go
func Test_slice(t *testing.T){
    s := make([]int,0,10)  
    s = append(s,10)
    t.Logf("s: %v, len of s: %d, cap of s: %d",s,len(s),cap(s))
}
```
输出，`slice_test.go:15: s: [10], len of s: 1, cap of s: 10`，`make([]int, 0, 10)` 的方式使得切片长度`0`，容量为`10`，实际上还有长度为`10`的缓存空间，append的时，切片长度从`0`增加为`1`，容量则维持为`10`不变。

3. 初始化切片`s`长度为`10`，容量为`11`，在`s`的基础上追加`append`一个元素，切片`s``的内容、长度以及容量分别是什么？
```go
func Test_slice(t *testing.T){
    s := make([]int,10,11)
    s = append(s,10)
    t.Logf("s: %v, len of s: %d, cap of s: %d",s,len(s),cap(s))
}
```
输出,`slice_test.go:22: s: [0 0 0 0 0 0 0 0 0 0 10], len of s: 11, cap of s: 11`，由于容量大于长度，因此仍有足够的空间，这次`append`操作不会引发扩容。

4. 初始化切片`s`长度为`10`，容量为`12`，截取切片`s index = 8`往后的内容赋给`s1`，长度以及容量分别是什么？
```go
func Test_slice(t *testing.T){
    s := make([]int,10,12)  
    s1 := s[8:]
    t.Logf("s1: %v, len of s1: %d, cap of s1: %d",s1,len(s1),cap(s1))
}
```
输出, `slice_test.go:29: s1: [0 0], len of s1: 2, cap of s1: 4`，截取操作会以`s[8]`作为内存空间的起点，截取所得新切片`s1`的长度和容量强依赖于原切片`s`的长度和容量，并在此基础上减去头部`8`个未使用到的单位。

5. `s`长度为`10`，容量为`12`，截取切片`s index`为`[8,9)`范围内的元素赋给切片`s1`，长度以及容量分别是什么？
```go
func Test_slice(t *testing.T){
    s := make([]int,10,12)  
    s1 := s[8:9]
    t.Logf("s1: %v, len of s1: %d, cap of s1: %d",s1,len(s1),cap(s1))
}
```
输出`slice_test.go:36: s1: [0], len of s1: 1, cap of s1: 4`，我们需要注意虽然`s[8:9]`的截取操作限定了`s1`的右边界，但这只是长度意义上的，对于容量，`s1`仍然和`s`保持强关联性。

6. 初始化切片`s`长度为`10`，容量为`12`，截取切片`s index = 8`往后的内容赋给`s1`，修改`s1[0]`的值，修改是否会影响到`s`？ 此时`s`的内容是什么？
```go
func Test_slice(t *testing.T){
    s := make([]int,10,12)  
    s1 := s[8:]
    s1[0] = -1
    t.Logf("s: %v",s)
}
```
输出`slice_test.go:44: s: [0 0 0 0 0 0 0 0 -1 0]`，`s1`是在`s`基础上截取得到的，属于一次引用传递，底层共用同一片内存空间，其中`s[x]`等价于`s1[x+8]`. 因此修改了`s1[0]`会直接影响到`s[8]`。

7. 初始化切片`s`长度为`10`，容量为`12`，请问，访问`s[10]`是否会越界？
```go
func Test_slice(t *testing.T){
    s := make([]int,10,12)  
    s = s[10]
    // 求问，此时数组访问是否会越界
}
```
会发生`panic`，切片长度为`10`，容量为`12`. 容量是物理意义上的，但长度是逻辑意义上的，判断是否越界以逻辑意义为准，因此`index = 10`已经越界。

8. 初始化切片`s`长度为`10`，容量为`12`，截取`s`中`index = 8`后面的内容赋给`s1`，在`s1`的基础上追加`[]int{10,11,12}` `3`个元素，经过上述操作时候，访问`s[10]`是否会越界？
```go
func Test_slice(t *testing.T){
    s := make([]int,10,12)  
    s1 := s[8:]
    s1 = append(s1,[]int{10,11,12}...)
    v := s[10]
    // ...
    // 求问，此时数组访问是否会越界
}
```
会发生`panic`，在`s`的基础上截取产生了`s1`，此时`s1`和`s`会拥有两个独立的`slice header`。`s1`扩容后，会被迁移到新的空间地址，此时`s1`已经和`s`做到真正意义上的完全独立，意味着修改`s1`不再会影响到`s`
`s`继续维持原本的长度值`10`和容量值`12`，因此访问`s[10]`会`panic`。

9. 初始化切片`s`长度为`10`，容量为`12`，截取切片`s index = 8`往后的内容赋给`s1`，在方法`changeSlice`中，对`s1[0]`进行修改，经过上述操作之后，`s`的内容是什么？
```go 
func Test_slice(t *testing.T){
    s := make([]int,10,12)  
    s1 := s[8:]
    changeSlice(s1)
    t.Logf("s: %v",s)
}
func changeSlice(s1 []int){
  s1[0] = -1
}
```
输出`slice_test.go:58: s: [0 0 0 0 0 0 0 0 -1 0]`，切片在传递时属于引用传递，且`s1[0]`和`s[8]`指向同一个元素. 因此在局部方法中，修改了`s1[0]`会直接影响到`s[8]`的内容.

10. 初始化切片`s`长度为`10`，容量为`12`，截取切片`s index = 8`往后的内容赋给`s1`，在方法`changeSlice`中，对`s1`进行`apend`追加操作，`s`以及`s1`的内容、长度和容量分别是什么？
```go
func Test_slice(t *testing.T){
    s := make([]int,10,12)  
    s1 := s[8:]
    changeSlice(s1)
	// slice_test.go:69: s: [0 0 0 0 0 0 0 0 0 0], len of s: 10, cap of s: 12
    t.Logf("s: %v, len of s: %d, cap of s: %d",s, len(s), cap(s))
    // slice_test.go:70: s1: [0 0], len of s1: 2, cap of s1: 4
    t.Logf("s1: %v, len of s1: %d, cap of s1: %d",s1, len(s1), cap(s1))
}

func changeSlice(s1 []int){
  s1 = append(s1, 10)
}
```
虽然切片是引用传递，但是在方法调用时，传递的会是一个新的`slice header`，因此在局部方法`changeSlice`中，虽然对`s1`进行了`append`操作，但这这会在局部方法中这个独立的`slice header`中生效，不会影响到原方法`Test_slice`当中的`s`和`s1`的长度和容量。

11. 初始化切片`s`，内容为`[]int{0,1,2,3,4}`，截取`s`中`index = 2`前面的内容（不含`s[2]`），并在此基础上追加`index = 3`后面的内容，经过上述操作后，`s`的内容、长度和内容分别是什么？此时访问`s[4]`是否会越界？
```go
func Test_slice(t *testing.T){
    s := []int{0,1,2,3,4}
    s = append(s[:2],s[3:]...)
	// slice_test.go:82: s: [0 1 3 4], len: 4, cap: 5
    t.Logf("s: %v, len: %d, cap: %d", s, len(s), cap(s))
    v := s[4] 
    // 是否会数组访问越界
}
```
会`panic`，执行完上述`append`操作之后，`s`的实际长度为`4`，容量维持不变为`5`. 此时访问`s[4]`会发生数组越界的错误。

12. 初始化切片`s`长度和容量均为`512`，在`s`的基础上追加`append`一个元素，切片`s`的内容、长度以及容量分别是什么？
```go
func Test_slice(t *testing.T){
    s := make([]int,512)  
    s = append(s,1)
	// slice_test.go:91: len of s: 513, cap of s: 848
    t.Logf("len of s: %d, cap of s: %d",len(s),cap(s))
}
```
由于切片`s`原有容量为`512`，已经超过了阈值`256`，因此对其进行扩容操作会采用的计算共识为`512 * (512 + 3*256)/4 = 832`。再终，在`mallocgc`流程中，我们为扩容后的新切片分配到了`6784 byte`的空间，于是扩容后实际的新容量为`cap = 6784/8 = 848`.

### 相关扩展阅读：
- [Go Slice切片](https://zhuanlan.zhihu.com/p/634412423)
- [Go map实现原理](https://zhuanlan.zhihu.com/p/597483155)
- [sync.Map技术原理](https://zhuanlan.zhihu.com/p/781392782)
- [一致性hash的原理和实现](https://mp.weixin.qq.com/s/V8nOmxWchnK97aJLEeT6WA)
- [从零到一落地一致性哈希算法](https://zhuanlan.zhihu.com/p/654778311)
- [布隆过滤器原理与实现](https://mp.weixin.qq.com/s/5zHQbDs978OoA3g83NaVmw)