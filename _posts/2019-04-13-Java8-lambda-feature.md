---
layout: post
title: Java8语言中的新特性以及lambda表达式
---

java8提供了一个新的API（称为流 Stream）,它支持许多并行的操作，其思路和在数据库查询语言中的思路类似——用更高级的方式表达想要的东西，而由“实现”来选择最佳低级执行机制。这样就可以避免`synchronized`编写代码，这一行代码不仅容易出错，而且在多核cpu上的执行所需成本也比想象的高；

在java8中加入Stream可以看作把另外两项扩充加入java8的原因：把代码传递给方法的简洁方式(方法引用、lambda)和接口中的默认方法；java8里面将代码传递给方法的功能(同时也能返回代码并将其包含在数据结构中)还让我们能够使用一套新技巧，通常称为函数式编程；

>java8引入默认方法主要是为了支持库设计师，让他们能够写出更容易改进的接口。这一方法很重要，因为你会在接口中遇到越来越多的默认方法。由于真正需要编写默认方法的程序员较少，而且它们只是有助于程序改进。

### 1. 行为参数化:
让你的方法接受多种行为(或策略)作为参数，并在内部使用完成不同的行为。行为参数化是一个很有用的设计模式，它能够轻松的适应不断变化的需求。这种模式可以把一个行为(一段代码)封装起来，并通过传递和使用创建的行为(例如对Apple的不同谓词)将方法进行行为参数化。
 <!-- more -->

有些类似于策略设计模式，java api中的很多方法都可以用不同的行为来参数化，这些方法往往与匿名类一起使用：比如对于数据进行排序的`Comparator`接口以及用于创建java线程实现的`Runnable`接口等；行为参数化可让代码更好的适应不断变化的要求，减轻未来的工作量。

传递代码，就是将新行为参数传递给方法，在java8之前实现起来很啰嗦。为接口声明许多只用一次的实体类而造成的啰嗦，在java8之前可以用匿名类来减少。java API包括很多可以用不同行为进行参数化的方法，包括排序、线程以及GUI处理；

```java
public interface ApplePredicate { // 谓词的设计包含一个返回boolean的方法.
  boolean test(Apple apple);
}
// 使用lambda实现数组数据的排序以及runnable接口的实现.
inventory.sort((Apple a1, Apple a2) -> a1.getWight().compareTo(a2.getWight()));
Runnable t = new Thread(() -> System.out.println("hello world"));
```
lambda表达式总共包括三个部分：参数列表、用于将参数列表和lambda主体隔开的箭头、以及lambda主体；lambda中的参数检查、类型推断以及限制：类型检查是从使用lambda的上下文中推断出来的，上下文（比如接收它传递方法的参数，或者接受它的值得局部变量）中的lambda表达式需要的类型成为目标类型。` List<Apple> heavierThan150g = filter(inventory, (Apple a) -> a.getWeight() > 150); `会检查filter接受的参数是否为函数式接口，以及绑定到接口的类型；

__lambda表达式类型推断__：java编译器会从上下文(目标类型)推断出用什么函数式接口来配合lambda表达式，这意味着它可以推断出适合lambda的签名，因为函数描述符可以通过目标类型来得到。编译器可以了解lambda表达式的参数类型，这样就可以在lambda语法中省去标注参数类型。

> 换句话说，java编译器会进行类型推断。对于局部变量的限制，在lambda表达式中可以进行引用方法中定义的变量类型，但其状态必须是最终态，在lambda中不能对变量的值进行修改，这种限制的主要原因在于局部变量保存在栈上，并且隐式表示它们仅限于其所在的线程，如果允许捕获可改变状态的局部变量，就会引发造成线程不安全的新的可能性。

```java
// 参数a没有显示类型,则lambda会进行类型推断.
List<Apple> greenApple = filter(inventory, a -> "green".equals(a.getColor()));
// 没有类型推断，因为在参数列表里参数的类型已经被显示的指定出来了.
Comparator<Apple> c = (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());
```

### 2.java中的方法引用：
__方法引用主要有三类__，指向静态方法的方法引用，其调用模式为 ` Integer::parseInt ` 静态类的名称与静态方法的名称进行拼接；第二类为调用类型实例的方法，可以使用` String::toString `类型为实例类型::实例方法名称，在于你在引用一个对象的方法；第三类为你引用实例变量的方法的名称，其调用模式为` instance::declaredMethod `；对于构造方法的引用，可以使用` ClassName::new `创建实体对象，如果构造函数是带有参数的，则可以调用其apply方法.
```java
// 如果调用的是无参的构造函数,则引用的是其Supplier签名.
Supplier<Apple> c1 = Apple::new;
Apple a1 = c1.get();  // 调用supplier的get方法将产生一个新的apple.
// 如果构造器的签名是Apple(Integer weight)则其适合Function接口的签名.
Function<Integer, Apple> c2 = Apple::new;
Apple a2 = c2.apply(110);
// 如果你构造一个带有两个参数的构造器Apple(String color, Integer weight)则它就适合BiFunction类型.
BiFunction<String,Integer,Apple> c3 = Apple::new;
Apple a3 = c3.apply("green", 110);
```
### 3. Stream函数式数据处理：
java8中新的 __流式StreamAPI处理数据__：在java8中的集合支持一个新的stream方法，该方法会返回一个流(接口定义在java.util.stream.Stream里)。对于流简单的定义为“从支持数据处理操作的源生成所有的元素序列”。元素序列 — 就像集合一样，流也提供一个接口，可以访问特定元素类型的一组有序值，因为集合是数据结构，所以它主要目的是以特定的时间/空间复杂度存储和访问元素，但流的目的在于计算。

源 — 流会使用一个提供数据的源，如集合、数组或者输入输出资源。从有序集合生成流时会保留原有的顺序，由列表生成流，其元素顺序与列表一直。数据处理操作 — 流的数据处理功能支持类似于数据库的操作，以及函数式编程语言中的常用操作，如filter\map\reduce\find\match等。

流与集合的差异：粗略的说，集合与流之间的差异就在于什么时候进行计算，集合是一个内存中的数据结构，它包含数据结构中目前所有的值—集合中的每个元素都得先算出来才能添加到集合中。相比之下，流则是在概念上固定的数据结构(你不能删除或者添加元素)，其元素是按需计算的。这个思想就是用户仅仅从流中提取需要的值，而这些值—在用户看不见的地方之后按需生成；
和迭代器类似，流只能遍历一次，遍历完成之后这个流就已经被消费掉了。对于流的使用可以包括三件事：一个数据源(如集合)来执行一个查询；一个中间操作链，形成一条流水线；一个终端操作，执行流水线并能生成结果；`常见的中间操作有 filter、map、limit`可以形成一条流水线，`collect forEach`等的都为终端操作。
```java
List<String> names = menu.stream().filter(d -> {return d.getCalories() > 300;})
	.map(d -> { return d.getName(); })
	.limit(3)	// 限制元素的个数为3
	.collect(toList());	// 将最后返回的结果转换为list结构.
```
__如何使用数据流__，例如筛选（用谓词筛选，筛选出各不相同的元素，忽略流中的头几个元素，或者将流截短至指定长度）和切片的操作。与SQL语言中的对数据记录的去重类似，使用`distinct`关键字可以过滤掉重复的元素。在`java stream`流数据操作中，判断数据流中的两个元素是否相等时通过其`hashCode`和`equals`方法的实现来进行判断的。

__Stream聚合操作__：可以在流中使用`limit`操作对返回数据流中元素的个数进行限制，与SQL语句中的`limit`类似。`skip(n)`操作会排除返回结果集合中的前n个元素，如果结果集合中元素的个数不足n，则会返回一个空的数据流；使用map进行数据元素的映射，流支持map方法它会接受一个函数作为参数，这个函数会被应用到每个元素上，并将其转换成为一个新的元素，在其中其会创建一个新版本而不是去修改。`flatMap`方法让你把一个流中的每个值都转换成另一个流，然后把所有的流连接起来成为一个流，`Arrays.stream()`方法可以接受一个数组作为并产生一个流；

__查找和匹配__：另一种常见的数据处理套路是看看数据集中的某些元素是否匹配一个给定的属性：`Stream API`通过`allMatch, anyMatch, noneMatch, findFirst和findAll`方法提供了这样的工具。对于数据筛选中的查找元素类似于SQL中的where条件查询。`Optional<T>`类是一个容器类，其可以用来代表一个值是否存在，`java8`的库设计人员引入了`Optional<T>`这样就不用返回众所周知的null问题了。

```java
// 使用一个返回boolean值的函数作为谓词对元素进行筛选，最后将筛选的结果以list的形式进行呈现。
List<Dish> vegetarianMenu = menu.stream().filter(Dish::isVegetarian).collect(toList());
// 可以创建一个包含重复元素的数组，然后获取得到其中所有的偶数.
List<Integer> numbers = Arrays.asList(1, 2, 1, 3, 3, 2, 4);
numbers.stream().filter(i -> i%2 == 0).distinct().forEach(System.out::println);
// 使用limit限制只会返回3个结果.
List<Dish> dishes = menu.stream().filter(d -> d.getCalories() > 300).limit(3).collect(toList());
// 跳过返回结果集合中的前2个元素.
List<Dish> dishes = menu.stream().filter(d -> d.getCalories() > 300).skip(2).collect(toList());
// 使用map操作得到得到菜单流中每个菜单的名称.
List<String> dishNames = menu.stream().map(Dish::getName).collect(toList());
// 使用扁平的数据流flatMap进行数据处理,flatMap让你把流中的每个值都转换为另一个流，然后把所有的流连接起来成为一个流。
String[] arrayOfWords = {"goodbye", "world"};
List<String> uniqueCharacters = words.stream().map(w -> w.split("")).flatMap(Arrays::stream)
	.distinct().collect(Collections.toList());
// 查找与数据匹配(anyMatch只要有任意一个元素匹配就会返回`true`).
if(menu.Stream().anyMatch(Dish::isVegetable)) {
  System.out.println("the vegetable is (somewhat) vegetarian friendly");
}
// 检查谓词是否匹配所有的元素.
boolean idHealthy = menu.stream().allMatch(d -> d.getCalories() <1000);
// noneMatch其中没有任何一个元素匹配.
boolean isHealthy = menu.stream().noneMatch(d -> d.getCalories() >= 1000);
// 使用findAny对数据进行过滤和筛选,返回一个Optional<Dish>,如果其包含元素则打印出元素的内容.
menu.stream().filter(Dish::isVegetarian).findAny().isPresent(d -> System.out.println(d.getName()));
```
有些流有一个出现顺序来指定流中项目出现的逻辑顺序(比如由`List`排序好的数据列生成的流)。对于这种流，你可能就像找到第一个元素，为此存在一个findAny的方法；到目前为止，见到过的终端操作都是返回一个`boolean`值。对于将一个流中所有的元素都结合起来的操作可以使用；

`reduce`操作来表达更复杂的查询，比如“计算菜单中的总卡路里”或者“菜单中卡路里最高的菜是哪一个”此类的查询。此类操作需要将所有元素反复结合起来得到一个值。这样的查询可以被归类为规约操作(将流规约为一个数值)。

__规约方法的优势与并行化__，使用`reduce`的好处在于这里的迭代被内部迭代抽象掉了，这让内部实现得以选择并行reduce操作。而迭代式求和例子中要更新共享变量`sum`这并不是并行化的。如果加入同步的话，很可能会发现线程竞争抵消了并行本应带来的性能提升，这种计算是通过引入一种`fork/join`的模式对任务进行计算的；数值范围是一个常用的东西，其可以代替java中的for循环并且语法也更加的简单。
```java
List<Integer> someNumbers = Arrays.alList(1, 2, 3, 4, 5);
Option<Integer> firstSquareDivisibleByThree = someNumbers.stream().map(x -> x*x)
	.filter(x -> x%3 == 0).findFirst();
// 对流中所有的元素进行求和.
int sum = numbers.stream().reduce(0, (a, b) -> a+b);
// 对于求最大值和最小值的操作，你可以使用Integer.max以及min的方法.
Optional<Integer> max = numbers.stream().reduce(Integer::max);
Optional<Integer> min = numbers.stream().reduce(Integer::min);
// 计算0~100中有多少个偶数,首先生成一个范围.并向控制台打印出偶数元素的个数.
IntStream evenNumbers = IntStream.rangeClosed(1, 100).filter(n -> n%2 == 0);
System.out.println(evenNumbers.count());
// 构建java流的方式.
Stream<String> stream = Stream.of("java 8", "lambda", "in", "action");
// 通过数组创建流.
int[] numbers = {2, 3, 5, 7, 11, 13};
int sum = Arrays.stream(numbers).sum();
```
__用流收集数据__：将流中的元素积累成一个汇总结果，具体的做法就是通过定义新的`Collector`接口来定义的，因此区分`Collection、Collector和collect`是很重要的。关于使用collect和收集器可以做什么：对于一个交易列表按货币进行分组，获得该货币的所有交易额总和（返回一个`Map<Currency, Integer>`）;将交易列表分为两组，贵的和不贵的返回一个`Map<Boolean, List<Transaction>>`；可以使用`counting()`工厂方法返回收集器，统计流中元素的个数；

使用`maxBy和minBy`获取得到数据流中的最大值和最小值；java 8实现了大多数的规约操作，但是仍然有一些操作需要我们进行自定义，这也就是reducing规约存在的意义。在使用reducing进行统计的时候第一个参数的值代表的是规约操作的起始值，第二个参数就是你在6.2节中使用的函数，将菜肴转换成表示其所含热量的int。第三个参数是一个`BinaryOperator`将两个项目累积成一个同类型的值，这里它就是两个Int求和的结果；分组，一个常见的数据库操作就是根据一个或多个属性对集合中的项目进行分组，就像前面讲到的对货币进行分组一样。

在java8之前实现此类的操作略显复杂，但如果使用Java 8所推崇的函数式风格来重写的话，就很容易转化为一个非常容易看懂的语句。分区是分组的特殊情况，由一个谓词作为分类函数，它成为分区函数。分区函数返回一个布尔值，这意味着得到的分组map的键类型是`Boolean`，因而正常情况下分组只会有两种结果，true是一组false是一组结果。分区的好处在于保留了分区函数返回`true`和`false`的两套流元素列表。
```java
// 1.统计菜单列表中菜品的总数.
long howManyDishes = menu.stream().collect(Collectors.counting());
long howManyDishes = menu.stream().count();
// 2.查找流中的最大值和最小值.
Comparator<Dish> dishCaloriesComparator = Comparator.comparing(Dish::getCalories);
// 返回的Optional类是为了解决Java中的空指针问题.
Optional<Dish> mostCalorieDish = menu.stream().collect(maxBy(dishCaloriesComparator));
// 3.统计数据流中所有元素总和summingInt
int totalCalories = menu.stream().collect(summing(Dish::getCalories));
// 4.对于求数据流中元素的平均值来说，可以使用averagingInt来进行处理.
double avgCalories = menu.stream().collect(averagingInt(Dish::getCalories));
// 5.使用joining对数据流中的每个元素调用其toString方法进行拼接.
String shortMenu = menu.stream().map(Dish::getName).collect(joining());
// 6.使用reducing规约进行操作.
Optional<Dish> mostCalorieDish = menu.stream().collect(reducing(d1, d2) -> d1.getCalories() > d2.getCalories() ? d1 : d2));
// 使用reducing规约来计算你菜单的总热量.
int totalCalories = menu.stream().collect(reducing(0, Dish::getCalories, (i, j) -> i + j));
// 7.对数据流进行分组group by,将菜单中的菜肴按照类型进行分类.给groupingBy提供了一个传递Function，它提取了流中每一道Dish的Dish.Type，我们将这个函数叫做分类函数.
Map<Dish.Type,List<Dish>> dishesType = menu.stream().collect(groupingBy(Dish::getType));
// 多级分组对于groupingBy工厂方法创建的收集器中，它除了普通的分类函数外，还可以接受collector类型的第二个参数。要么进行二级分组的话，我们可以将一个内层的groupingBy传递给外层的groupingBy.
Map<Dish.Type, Map<CalorieLevel, List<Dish>> dishesByTypeCaloriesLevel = 
	menu.stream().collect(Dish::getType,
	groupingBy(dish->{
      if(dish.getCalories() <= 400) return CaloricLevel.DIET;
      else if (dish.getCalories() <= 700) return CaloricLevel.NORMAL;
      else return CaloricLevel.FAT;
	})
)
```
__并行处理数据集合时，你需要考虑的事情__：你得明确的把包含数据的数据结构分为若干子部分。第二，你要给每个子部分分配一个独立的线程。第三，你需要在恰当的时候对它们进行同步来避免不希望出现的竞争条件。等待所有线程完成，最后把这些部分结果合并起来，java 7版本的时候引入了fork/join的多线程框架用于处理此类任务。
引入Stream流操作之后，它允许你声明性的将顺序流变为并行流。在现实中对顺序流调用parallel方法并不意味着流本身有任何实际的变化，它在内部实际上就是设置了一个`boolean`标志，表示你想让调用parallel之后进行的所有操作都并行执行。内部迭代让你可以并行的处理一个流，而无需在代码中显示使用和协调不同的线程。分支/合并框架让你得以用递归的方式将可以并行的任务拆分成更小的任务，在不同的线程上执行，然后将各个子任务的结果合并起来生成整体的结果。
```java
// 将顺序流转换成为并行流进行计算.
Stream.iterate(1L, i -> i+1).limit(n).parallel().reduce(0L, Long::sum);
```

__java 8提高编程的效率__：相比较于匿名类，lambda表达式可以帮助我们用更紧凑的方式描述程序的行为，如果希望将一个既有的方法作为参数传递给另一个方法，那么方法引用无意是我们推荐的方法，利用这种方式我们能够写出非常简介的代码。跟之前的版本相比较，java 8的新特性也可以帮助提升代码的可读性：使用java 8你可以减少冗长的代码，代码更易于理解。通过方法引用和`Stream API`你的代码会更加直观。

__代码的重构__ 主要有3种简单的方式：重构代码，用`lambda`表达式代替匿名内部类，用方法引用重构lambda表达式，用`Stream API`重构命令式的数据处理。

* 需要注意的地方，在有些情况下将匿名类转换成为lambda表达式可能是一个比较复杂的过程。匿名类中的`this`和`super`的含义与lambda中的含义不同，在lambda表达式中`this`指代的的是外部所在类，而不是匿名类中的自身。另外一点，匿名类可以屏蔽外部类中的变量名称，而lambda表达式则不能因为其会导致编译错误。
* 可以通过方法引用将lambda表达式中的内容抽取到一个单独的方法中，将其作为参数传递给`groupingBy`方法。在使用方法引用的时候还应尽量的参考`comparing、maxBy`等方法。
* 从命令式的数据处理转换到`Stream`，java 8中的流式操作能够更加清晰的表达数据处理管道的意图，除此之外，通过短路和延迟加载以及之前介绍的现代计算机的多核架构，在内部可以对Stream进行优化。
```java
Runnable r1 = new Runnable() {
  public void run() { System.out.println("hello"); }
}
// 1.新的方式，使用lambda表达式代替内名内部类。
Runnable r2 = () -> { System.out.println("hello"); }
// 2.按照Dish的level等级将菜单中的菜品进行分组(使用方法引用代替lambda中的判断逻辑).
Map<CaloricLevel, List<Dish>> dishesByCaloricLevel = menu.stream().collect(groupingBy(Dish::getCaloricLevel));
// 3.使用现代计算机中的多核架构parallel代替命令式的数据流处理.
menu.parallelStream().filter(d -> d.getCalories() > 300).map(Dish::getName).collect(toList());
```

使用java8中的lambda表达式对于设计模式中的重构：对于策略设计模式我们使用lambda表达式直接传递代码避免了僵尸代码的出现，对于给定的接口使用lambda表达式进行实现。对于模板设计模式的优化也是将代码传递到了方法参数中，不再需要对基类进行继承。
### 4. java8中的默认方法：
在java中接口将相关的方法按照约定组合到一起，实现接口的类必须为接口中定义的每个方法提供一个实现，或者从父类中继承它的实现。但是一旦类库的设计者需要更新接口向其中添加新的方法，这种方式就会出现问题。java8为了解决这个问题引入了一种新的设计机制，java8的接口现在支持在声明方法的同时提供实现。

可以通过两种方式进行实现：一种为java8允许在接口内声明静态方法。其二是java8中引入了一个新的功能叫做默认方法，通过默认方法可以指定接口的默认实现，也就是接口能够提供方法的默认实现。默认方法在java8中已经大量的使用了，如`Collection`类的`stream`方法就是默认方法，`List`接口的`sort`方法以及之前介绍的很多函数式接口`Predicate、Function以及Comparator`也引入了新的默认方法。

java8中抽象类和抽象接口之间的区别：在继承关系上一个类只能继承一个抽象类，但是一个类可以实现多个接口。其次，一个抽象类可以通过实例变量保存一个通用的状态，而接口是不能够有实例变量的。
```java
// 在jdk1.8中为List接口新增的默认方法，函数方法sort前面的修饰符default能够知道一个方法是否为默认方法.
default void sort(Comparator<? super E> c) {
  Collections.sort(this, c);
}
```
null带来的种种问题，首先`NullPointerException`是目前java程序开发中最典型的异常。此外在代码中进行着空指针的检查会使得你的代码可读性糟糕透顶。`null`自身是没有任何的语义，尤其是它代表的是在静态语言类型中以一种错误的方式对缺失变量值得建模。其它语言对于`null`的处理。

在Groovy语言中通过引入安全导航操作符可以安全的访问可能为`null`的变量。其语法解释为当某个属性的值为null的时候，语法分析将不会再继续往后处理。`person`可能没有`car`的属性，在调用链中如果遭遇了`null`时将`null`引用沿着调用链传递下去，返回一个`null`的值。在spring El表达式中也存在于groovy类似的语法用于对对象的属性进行索引。
`def carInsuranceName = person?.car?.insurance?.name;`

```java
// java中的spring El表达式其也采用了与groovy类似的语法用于获取对象的属性值.
String city = parser.parseExpression("PlaceOfBirth?.City").getValue(context, String.class);
System.out.println(city); // Smiljan
```
汲取Haskell和Scale的灵感，java 8中引入了一个新的类`java.util.Optional<T>`，有时候还可以通过此类判断当前jdk的版本(spring core中使用了这种方法)。这是一个封装`Optional`值得类。当变量存在的时候`Optional`类只是对类的简单封装。变量不存在的时候，缺失的值会进行自动建模成一个“空”的Optional对象，由方法`Option.empty()`返回。

应用`Optional`的几种模式：使用map从`Optional`对象中提取和转换值，map操作会将提供的函数应用于流中的每个元素，可以将`Optional`当做一个特殊的集合，它至多包含一个元素。如果是递归调用操作调用属性值的话，则不能使用`map`应该使用扁平化的数据流`flatMap`进行操作；

默认行为以及解引用`Optional`对象：get()是这些方法中最简单但又不安全的方法，如果变量存在则返回变量的值否则抛出`NoSuchElementException`的异常；orElse()操作允许你在`Optional`对象不存在的时候提供一个默认值；`ifPresent(Consumer<? super T>)`能让变量在存在的时候执行一个以参数传递进来的方法。

```java
// 声明一个空的Optional,通过其静态方法创建一个空的Optional对象.
Optional<Car> optCar = Optional.empty();
// 依据一个非空值创建Optional，我们可以使用Optional.of依据一个非空值创建一个Optional对象.如果car的值为null的话则会立即抛出NullPointerException.
Optional<Car> optCar = Optional.of(car);
// 可接受null的Optional,可以使用ofNullable方法创建一个允许null值得Optional对象.
Optional<Car> optCar = Optional.ofNullable(car);
// 通过map方法获取得到Optional对象中的属性值.
Optional<Insurance> optInsurance = Optional.ofNullable(insurance);
Optional<String> name = optInsurance.map(Insurance::getName);
// 使用flatMap操作属性.属性的值.如果Optional的结果值为空设置默认值.
person.flatMap(Person::getCar).flatMap(Car::getInsurance)
	.map(Insurance::getName)
	.orElse("Unknown");
```
java8中新引入的`CompletableFuture`接口构建异步的应用，其弥补了之前`Future`接口在这些方面的不足：将两个异步计算合并为一个—这两个异步计算之间相互独立，同时第二个又依赖于第一个的结果。等待`Future`集合中的所有任务都完成。仅等待Future集合中最快结束的任务完成，并返回它们的结果。通过编程的方式完成一个`Future`任务的执行(即以手工设定异步操作结果的方式)。应对Future的完成事件(即当Future的完成事件发生时会收到通知，并使用Future计算的结果进行下一步的操作，不只是简单地阻塞等待结果)。
```java
public Future<Double> getPriceAsync(String product) {
  CompletableFuture<Double> futurePrice = new CompletableFuture<>();
  new Thread(() -> {
  	double price = calculatePrice(product);
  	futurePrice.complete(price);
  }).start();
}
// 当在客户端调用该方法的时候回字节返回future结果,等其它操作结束之后可以调用future.get()获取计算的结果.如果价格未知，
// 则get方法会一直处于阻塞的状态直至方法调用结束.
double price = future.get();
```







