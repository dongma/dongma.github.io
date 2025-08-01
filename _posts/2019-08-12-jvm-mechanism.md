---
layout: post
title: 深入理解Java虚拟机
---

`Java`与`C++`之间有一堵由内存分配和垃圾回收技术所围成的“高墙”，墙外面的人想进去，墙里面的人却想出来。对于`C`、`C++`程序开发人员来说，在内存管理领域，它们既是拥有最高权力的皇帝又是从事最基础工作的劳动人民。拥有每一个对象的所有权，也有担负着每一个对象生命开始到终结的维护责任。

> 对`Java`程序员来说，在虚拟机自动内存管理机制的帮助下，不再需要为每一个`new`操作写配对的`delete`、`free`代码，因有虚拟机管理内存，不容易出现内存泄漏和内存溢出的问题。
<!-- more -->
### 1. 虚拟机内存结构：

`jvm`会把它管理的内存划分为若干个不同的数据区域。这些区域都有各自的用途，以及创建和销毁的时间。有的区域随着虚拟机进程的启动而存在，有些区域则依赖于用户线程的启动和结束而建立和销毁。

![1575427884538](../../../../resource/2019/1575427884538.png)

程序计数器：程序计数器是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器，在虚拟机的概念模型里，字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令。分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器完成。


`java`方法栈(`java method stack`)也是线程私有的，它的声明周期也是与线程相同。虚拟机栈描述的是`java`方法执行的内存模型：每个在执行的时候都会创建一个栈帧(`stack frame`)用于创建局部变量表、操作数栈、动态链接、方法出口信息。当退出当前执行的方法时，`java`虚拟机均会弹出当前线程的当前栈针，并将之舍弃。

本地方法栈(`native method stack`)：本地方法栈与`java`方法栈发挥的作用是非常相似的，它们之间的区别不过是为虚拟机执行`java`方法服务，而本地方法栈则为虚拟机使用到的`native`方法服务。

`java`堆(`java heap`)：`java`堆是`java`虚拟机中所管理内存中最大的一块，`java`堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都要在堆上分配内存。`java`堆是垃圾收集器管理的主要区域，因此很多时候也被称为"`GC`堆(`garbage collected heap`)"。

方法区(`method area`)与`java`堆一样，是各个线程共享的内存区域，它用于加载已被虚拟机加载的类信息、常量、静态变量、即时编译器后的代码等数据。虽然`java`虚拟机规范把方法去描述为堆的一个逻辑部分，但是它却有一个别名叫做`non-heap`非堆，目的是与`java`堆区分开。很多人愿意把方法去称为"永久代(`permanent generation`)"，本质上两者并不等价，仅仅是因为`hotspot`虚拟机的设计团队选择把`gc`分代收集扩展至方法区，或者说永久代来实现方法区而已，这样`hotspot`的垃圾收集器可以像管理`java`堆一样管理这部分内存，能够省去专门为方法区编写内存管理代码的工作。

运行时常量池：运行时常量池(`runtime constant pool`)是方法区的一部分，`class`文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项是常量池(`constant pool table`)，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池存放。

直接内存(`direct memory`)：直接内存并不是虚拟机运行时数据区的一部分，也不是`java`虚拟机规范中定义的内存区域。但是这部分内存也被频繁地使用，而且也可能导致`outOfMemoryError`异常。显然本机直接内存的分配不会受到`java`堆大小的限制，但是既然是内存，肯定还是会受到本机总内存(包括`swap`以及`raw`区或者分页文件大小)以及处理器寻址空间的限制。

### 2. java虚拟机是如何加载java类的：
从虚拟机的视角来看，执行`java`代码首先需要将它编译而成的`class`文件加载到`java`虚拟机中。加载后的`java`类会被存放于方法区，实际执行时，虚拟机会执行方法区的代码。

![1575429252338](../../../../resource/2019/1575429252338.png)

**加载阶段**是"类加载"过程的一个阶段，在加载阶段虚拟机主要完成以下3件事情：通过一个类的全限定名来获取定义此类的二进制字节流。将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。在内存中生成一个代表这个类的`java.lang.Class`对象，作为方法区这个类的各种数据的访问入口。

**链接阶段**是指将创建的类合并至`java`虚拟机中，使之能够执行的过程。它分为验证准备、准备和解析三个阶段。验证阶段目的是为了确保`class`文件的字节流中包含的信息符合当前虚拟机的要求。验证阶段主要包括：文件格式的验证，是否以魔数开头、主次版本还是否在当前虚拟机处理的范围之内等。元数据的验证，第二阶段主要是对类的元数据信息进行语义校验，保证不存在不符合`java`语言规范的元数据信息。该类是否继承自`java.lang.Object`这个类是否继承了不允许被继承的类（被`final`修饰的类）。字节码验证，其主要的目的是通过数据流和控制流分析确定程序语义是合法符合逻辑的。

准备阶段正式为变量分配内存并设置类变量的初始值阶段，这些变量所使用的内存都将在方法区中进行分配。这个阶段中有两个容易产生混淆的概念。这会在对象实例化时随着对象一起分配在`java`堆中，其次这里所说的初始值"通常情况"下是数据类型的零值。

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。

**初始化阶段**是类加载过程的最后一步，前面的类加载过程中除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作都是完全由虚拟机主导和控制。到了初始化阶段，才真正的执行类中定义的`java`代码。

**类加载器的双亲委派模型**：从`java`虚拟机的角度来看，只存在两种不同的类加载器：一种是启动类加载器(`Bootstrap Classloader`)这个类加载器使用`C++`语言实现，是虚拟机自身的一部分；另一种是由`java`语言实现的类加载器，其全都继承自抽象类`java.lang.ClassLoader`。双亲委派模型的工作过程是，如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此。

启动类加载器负责加载最为基础、最为重要的类，比如存放在`JRE`的`lib`目录下的`jar`包，除了启动类加载器外另外两个重要的类加载器是扩展类加载器（`extension class loader`）和应用类加载器（`application class loader`），均由`java`的核心类库提供。扩展类加载器负责加载相对次要、但又通用的类，比如存放在`JRE`的`lib/ext`目录下`jar`包中的类（以及由系统变量`java.ext.dirs`指定的类）。应用类加载器主要负责加载程序路径下的类（径包括`classpath`、系统变量`java.class.path`和环境变量`classpath`的路径）。

### 3. jvm中的方法调用机制：

`Java`虚拟机识别方法的关键在于类名、方法名以及方法描述符（`method descriptor`），对于方法描述符其是由方法的参数类型以及返回类型所构成。在同一个类中，如果同时出现多个名字相同且描述符也相同的方法，那么`java`虚拟机会在验证阶段报错。

对于重载方法的区分在编译阶段已经完成，我们可以认为`java`虚拟机不存在重载这一概念。因此，在一些文章中，重载也被称为静态绑定（`static binding`）,或者编译时多台（`compile-time polymorphism`），而重写则被称为动态绑定（`dynamic binding`）。确切的说，`java`虚拟机中的静态绑定指的是在解析时便能够直接识别目标方法的情况，而动态绑定则指的是需要在运行过程中根据调用者动态类型来识别目标方法的情况。

具体来说，`java`字节码中与调用相关的指令共有五种：`invokestatic`用于调用静态方法、`invokespecial`用于调用私有实例方法、构造器，以及使用`super`关键字调用父类实例的方法、`invokevirtual`用于调用非私有实例的方法、`invokeinterface`用于调用接口方法、`invokedynamic`用于调用动态方法。

```java
interface Customer {
  boolean isVIP();
}

class Merchant {
  public double priceAfterDiscount(double oldPrice, Customer customer) {
    return oldPrice * 0.8d;
  }
}

class Traitor extends Merchant {
  @Override
  public double priceAfterDiscount(double oldPrice, Customer customer) {
    if (customer.isVIP()) {                         // invokeinterface
      return oldPrice * 价格歧视();                  // invokestatic
    } else {
      return super.priceAfterDiscount(oldPrice, customer);  // invokespecial
    }
  }
  public static double 价格歧视() {
    return new Random()                          // invokespecial
           .nextDouble()                         // invokevirtual
        + 0.8d;
   }
}
```

在类加载机制的链接部分中，在类加载的准备阶段，它除了为静态字段分配内存之外，还会构造与该类相关联的方法表。这个数据结构便是`java`虚拟机实现动态绑定的关键所在。方法调用指令中的符号引用会在执行之前被解析成实际引用，对于静态绑定方法调用而言，实际引用则是方法表的索引值。在执行过程中，`java` 虚拟机将获取调用者的实际类型，并在该实际类型的虚方法表中，根据索引值获得目标方法。这个过程便是动态绑定。

内联缓存是一种加快动态绑定的优化技术，它能够缓存虚方法调用中调用者的动态类型，以及该类型对应的目标方法。在之后的执行过程中，如果碰到已缓存的类型，内联缓存便会直接调用该类型所对应的目标方法。如果没有碰到已缓存的类型，内联缓存则会退化至使用基于方法表的动态绑定。

### 4. jvm中的垃圾回收机制：

垃圾回收，顾名思义就是将已经分配出去的，但不再使用的内存回收回来以便能够再次分配。如何判断一个对象是否已经死亡？可以使用引用计数法，其做法是为每个对象添加一个引用计数器，用来统计该对象的引用个数。一旦某个对象的引用计数器为0，则说明该对象已经死亡，便可以被回收。但是，其存在缺陷是无法解决对象之前的循环引用问题。

目前`java`虚拟机的主流垃圾回收器采取的是可达性分析算法，这个算法的实质在于将一系列`GC Root`作为初始的存活对象集（`live set`）。然后从该集合出发，探索所有能够被该集合引用到的对象，并将其添加到该集合中，这个过程我们称之为标记（`mark`）。最终，未被探索到的对象便是死亡的，是可以回收的。

![1575476764337](../../../../resource/2019/1575476764337.png)

目前`java`虚拟机将堆划分为新生代和老年代。其中，新生代又被划分为`Eden`区以及两个大小相同的`Survivor`区，可在应用启动时通过参数`-XX:SurvivorRatio`来调整`Eden`区和`Survivor`区的比例。当使用`new`指令时，它会在`Eden`区中划出一块作为存储对象的内存。由于堆空间是线程共享的，因此直接在这里边化空间是同步的。

当`Eden`区的空间耗尽的时，`java`虚拟机便会触发一次`Minor GC`来收集新生代的垃圾（使用标记-复制算法）。存活下来的对象则会被送到`Survivor`区。`java`虚拟机会记录`Survivor`区中的对象一共被来回复制了几次，如果一个对象被复制的次数为`15`（对应虚拟机参数`-XX:MaxTenuringThreshold`），那么该对象将被晋升至老年代。另外，如果单个`Survivor`区已经被占用了`50%`（对应虚拟机参数`-XX:TargetSurvivorRatio`），那么较高复制次数的对象也被晋升至老年代。

**标记-清除算法**：该算法如同它的名字一样，算法分为“标记”和“清除”两个阶段。首先标记出所有需要回收的对象，在返回标记完成后统一回收所有被标记的对象。该算法的不足主要表现在：一个是效率问题，标记和清除两个过程的效率都不高。另一个是空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后再程序运行过程中需要分配较大对象时，无法找到足够连续的内存而不得不提前触发另一次垃圾收集动作。

**复制算法**：该算法将可用内存按照容量分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另一块上面，然后再把已经使用过的内存一次清理掉。`IBM`公司专门研究表明新生代中的对象`98%`是"朝生夕死"的。其将内存分为一块较大的`eden`空间和两块较小的`survivor`空间，每次使用`eden`和其中一块`survivor`区。当回收时，将`eden`和`survivor`中还存活着的对象一次性地复制到另外一块`survivor`空间上，最后清理掉`eden`和刚才用过的`survivor`空间，`hotspot`虚拟机默认`eden`和`survivor`的大小比例为`8:1`。

**标记-整理算法**：复制收集算法在对象存活率较高时就要进行较多的复制操作，效率将会变低。更关键的是，如果不想浪费`50%`的空间就需要有额外的内存进行空间担保，以应对被使用的内存中所有对象都是`100%`存活的极端情况，所以在老年代中一半不能直接选用该算法。整理算法在标记之后并不是将已经标记的对象进行清理而是让所有存活的对象都向一端移动，然后直接清理掉边界以外的内存。

**分代收集算法**：当前商业虚拟机大多数的垃圾收集器都是分代收集(`Generation Collection`)算法，该算法只是根据对象的存活周期的不同将内存划分为几块。一般是把`java`堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适合的收集算法。在新生代中，每次垃圾收集都会发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存户对象的复制成本就可以完成收集。而老年代中因为对象存活率高，没有额外的空间对其进行担保，就必须使用"标记-整理"或者"标记-清理"的算法进行回收。

**Minor GC**与**Major GC**有什么不一样？新生代`GC(Minor GC)`指发生在新生代的垃圾收集动作，因为`java`对象大多都具备朝生夕灭的特性，所以其非常频繁，一般回收速度也比较快。老年代`GC(Major GC)`指的是发生在老年代的`GC`，出现了`Major GC`经常会伴随着至少一次的`Minor GC`但并非绝对，`Major GC`的速度一般`Minor GC`慢`10`倍以上。

### 5. jvm中的即时编译：

在部分的商用虚拟机(`Sun HotSpot`)中`java`程序员最初是通过解释器(`interceptor`)进行解释执行的，当虚拟机发现某个方法或代码块的运行特别频繁的时候，就会把这些代码认定为“热点代码”。为了提高热点代码的执行效率，在运行时虚拟机将会把这些代码编译成为与本地平台无关的机器码，并进行各种层次的优化，完成这个任务的编译器称为即时编译器(`just intime compiler`)。

即时编译时一项用来提升应用程序运行效率的技术。通常而言，代码会先被`java`虚拟机解释执行，之后反复执行的热点代码则会被即时编译为机器码，直接运行在底层硬件之上。`HotSpot`虚拟机包括多个即时编译器`C1`、`C2`和`Graal`。在`java7`以前，我们需要根据程序的特性选择对应的即时编译器，对于执行时间较短的或者对启动性能有要求的程序，我们采用编译效率较快的`C1`，对应参数`-client`。而对于执行时间较长的，或者对峰值性能有要求的程序，我们采用生成代码执行效率较快的`C2`，对应参数为`-server`。

即时编译是根据方法的调用次数以及循环回边的执行次数来触发的，具体是由`-XX:CompileThreshold`指定的阀值（使用`C1`时，该值为`1500`；使用`C2`时，该职位`10000`）。除了以方法为单位的即时编译外，`java`虚拟机还存在着另一种以循环为单位的即时编译，叫做`On-Stack-Replacement (OSR)`编译，循环回边计数器是用来触发这种类型的编译的。

`java`虚拟机是通过`synchronized`实现同步机制，在其对应的字节码中包含`monitorenter`和`monitorexit`。重量级锁是`java`虚拟机中最为基础的实现，在这种状态下，`java`虚拟机会阻塞加锁失败的线程，并且在目标锁被释放的时候唤醒这些线程。`java`线程的阻塞以及唤醒都是依赖于操作系统实现的，举例来说，对于符合`posix`接口的操作系统（如`mocos`和绝大部分的`linux`），上述操作是通过`pthread`的互斥（`mutex`）来实现的。为了避免昂贵的线程阻塞、唤醒操作，`java`虚拟机会在线程进入阻塞状态之前，以及被唤醒后的竞争不到锁的情况下会进入自旋状态，在处理器上空跑并且轮询锁是否已经被释放。

在对象内存布局中曾对对象头中的标记字段（`mark word`）中的最后两位便被用来表示该对象的锁状态，其中`00`表示轻量级锁、`01`表示无锁（或偏移锁）、`10`表示重量级锁、`11`则跟垃圾回收算法的标记有关。`java`虚拟机会尝试用`CAS`操作，比较锁对象的标记字段的值是否为当前锁记录的地址。如果是，则替换为锁记录中的值，也就是锁对象原本标记字段。此时，该线程已经成功释放了这把锁。

具体来说，在线程进行加锁时，如果该锁对象支持偏向锁，那么`java`虚拟机会通过`cas`操作，将当前线程的地址记录在锁对象的标记之中，并且将标记字段最后三位设置为`101`。每当有线程请求这把锁时，`java`虚拟机只需判断锁对象标记字段中：最后三位是否为 `101`，是否包含当前线程的地址，以及`epoch`值是否和锁对象的类的 `epoch`值相同。如果都满足，那么当前线程持有该偏向锁，可以直接返回。

```java
public void foo(Object lock) {
     synchronized (lock) {
         lock.hashCode();
     }
}
// 上面的Java代码将编译为下面的字节码
public void foo(java.lang.Object);
Code:
	3: monitorenter
	4: aload_1
    5: invokevirtual java/lang/Object.hashCode:()I
    8: pop
    9: aload_2
   10: monitorexit
Exception table:
	from to target type
    4 	 11   14   any
    14 	 17   14   any
```


### 6. jvm中的代码优化：
`jvm`中的方法内联：是指在编译过程中遇到方法调用时，将目标方法的方法体纳入编译范围之中，并取代原方法调用的优化手段。方法内联不仅可以消除调用本身带来的性能开销，还可以进一步触发更多的优化。因此，它可以算是编译器优化中最重要的一环。

方法内联的条件：方法内联能够触发更多的优化。通常而言，内联越多生成代码的执行效率越高。然而，对于即时编译器来说，内联越多编译时间也就越长，而程序达到峰值性能的时刻也就会被推迟。此外，内联越多也将会导致生成的机器码越长。生成的机器码时间越长，在`java`虚拟机里，编译生成的机器码会被部署到`CacheCode`中。这个`CacheCode`是由`java`虚拟机参数`-XX:ReservedCodeCacheSize`控制，当`CacheCode`被填满时，会出现即时编译器被关闭的警告信息（`CacheCode is full，Compiler has been disabled`）。

即时编译器的去虚化方式可分为完全去虚化以及条件去虚化，完全去虚化是通过类型推导或类层次分析（`class hierarchy analysis`）识别虚拟方法调用的唯一目标，从而将其转换为直接调用的一个优化手段。它的关键在于证明虚方法调用的目标方法是唯一的。条件去虚化则是将虚方法调用转换为若干个类型测试以及直接调用的一种优化手段。

逃逸分析是“一种确定指针动态范围的静态分析，它可以分析在程序的哪些地方可以访问到指针”。在`java`虚拟机的即时编译语境下，逃逸分析将判断新建的对象是否逃逸。即时编译器判断对象是否逃逸的依据，一是对象是否被存入堆中（静态字段或者堆中对象的实例字段），二是对象是否被传入未知代码中。

### 7. java虚拟机监控及诊断工具：

`jps command`：`jps`命令用于打印所有正在运行的`java`进程相关信息，可选参数：`-l` 将打印模块名以及包名、`-v`将打印`java`虚拟机参数、`-m`将打印传递给主类的参数。

```shell
sam@elementoryos:~$ jps -mlv
5524 eureka-0.0.1.jar
55677 sun.tools.jps.Jps -mlv
-Denv.class.path=.:/home/sam/jdk1_8/jdk1_8_0_231/lib:/home/sam/jdk1_8/jdk1_8_0_231/jre/lib -Dapplication.home=/home/sam/jdk1_8/jdk1_8_0_231 -Xms8m
```

`jstat command`：`jstat`命令可以用来打印目标`java`的性能数据，它包括多个参数信息：`-class`将打印出类加载数据、`-compiler`和`-printcompliation`将打印即时编译相关的数据，其它一些以`-gc`为前缀的子命令，它们将打印垃圾回收相关的数据。

```shell
sam@elementoryos:~$  jstat -gc 5524 1s 4
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
9216.0 512.0   0.0    0.0   294400.0 11918.3   63488.0    23750.9   56408.0 53351.8 7808.0 7199.2     21    0.567   4      0.411    0.978
9216.0 512.0   0.0    0.0   294400.0 11918.3   63488.0    23750.9   56408.0 53351.8 7808.0 7199.2     21    0.567   4      0.411    0.978
9216.0 512.0   0.0    0.0   294400.0 11918.3   63488.0    23750.9   56408.0 53351.8 7808.0 7199.2     21    0.567   4      0.411    0.978
9216.0 512.0   0.0    0.0   294400.0 11918.3   63488.0    23750.9   56408.0 53351.8 7808.0 7199.2     21    0.567   4      0.411    0.978
```

`jmap command`：`jmap`命令用于分析`java`堆中的对象，`jmap`同样包括多条子命令：`-clstats`用于打印被加载类的信息、`-finalizerinfo`用于打印所有待`finalize`的对象、`-histo`用于统计各个类的实例数据及占用内存，并按照内容使用量从多到少的顺序排序、`-dump`将导出`java`虚拟机堆的快照，`-dump:live`只保存堆中存活的对象。

`jinfo command`：`jinfo`命令可用来查看目标`java`进程的参数，如传递给`java`虚拟机的`-X`（即输出中的 `jvm_args`）、`-XX`参数（即输出中的`VM Flags`）。

```shell
sam@elementoryos:~$ jinfo 5524
Attaching to process ID 5524, please wait...
Debugger attached successfully.
VM Flags:
Non-default VM flags: -XX:CICompilerCount=2 -XX:InitialHeapSize=60817408 -XX:MaxHeapSize=958398464 -XX:MaxNewSize=319291392 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=19922944 -XX:OldSize=40894464 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
Command line:
```

`jstack command`：`jstack`命令用于可以用来打印`java`进程中各个线程的栈轨迹，以及这些线程所持有的锁。`jstack`的其中一个应用场景便是死锁检测，可以用`jstack`获取一个已经死锁了的`java`程序的栈信息。

