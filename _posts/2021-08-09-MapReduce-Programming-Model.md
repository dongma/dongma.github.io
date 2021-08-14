---
layout: post
title: 大数据三架马车之MapReduce
---
> `Hadoop`是`Apache`的一个开源的分布式计算平台，以`HDFS`分布式文件系统和`MapReduce`计算框架为核心，为用户提供一套底层透明的分布式基础设施。

`MapReduce`提供简单的`API`，允许用户在不了解底层细节的情况下，开发分布式并行程序。利用大规模集群资源，解决传统单机无法解决的大数据处理问题，其设计思想起源于`MapReduce Paper`。

### MapReduce编程模型
`MapReduce`是一种用于处理和生成大型数据集的编程模型和相关实现，用户指定一个`map()`函数接收处理`key/value`对，同时产生另外一组临时`key/value`集合，`reduce()`函数合并相同`intermediate key`关联的`value`数据，以这种函数式方风格写的程序会自动并行化并在大型商品机器集群上运行。

在`Paper`发布之前的几年，`Jeffrey Dean`及`Google`的一些工程师已经实现了数百个用于处理大量原始数据且特殊用途的计算程序，数据源如抓取的文档、`Web`日志的请求等，来计算各种派生数据，像倒排索引、`Web`文档图结构的各种表示、每个主机爬取的页面数汇总等。

看个统计单词在文章中出现的次数的例子，`map()`函数`emit`每个单词及其出现次数，`reduce()`函数统计按单词统计其出现的总次数，伪代码如下：
```java
map(String key, String value):
  // key: document name
  // value: document contents for each word w in value:
  EmitIntermediate(w, "1");

reduce(String key, Iterator values): // key: a word
  // values: a list of counts
  int result = 0;
  for each v in values: result += ParseInt(v);
      Emit(AsString(result));
```
<!-- more -->
谈一些`MapReduce`程序在`Google`的应用：
* 反向索引（`Inverted Index`），`map()`函数解析文档中的每个单词，对`(word, document ID)`进行`emit`，`reduce`端依据`word`将二元组`pair`中的`document id`进行合并，最终`emit(word, list(document ID))`的数据。
* 统计`URL`的访问频次，`map()`函数输出从`log`中获取`web request`的信息，将`(URL, 1)`的二元组进行`output`，`reduce()`端将具有相同`URL`的请求进行归集，以`(URL, total count)`的方式进行`emit`。
* 反转`Web-Link Graph`，`map()`函数针对网页中存在的超链接，按`(target, source)`的格式输出二元组，`reduce()`函数将相同`target`对应的`source`拼接起来，按`(target, list(source))`的方式进行`emit`。

### Execution Overview
在输入数据切分成数份后，`map`函数会被自动分发到多台机器上，输入数据切分可以在多不同的机器上并行执行。`partition()`函数（`hash(key) mod R`）会将中间`key`切分为`R`片，`reduce()`函数会根据切分结果分到不同机器。

<img src="../../../../resource/2021/mapreduce/mapreduce_execution_overview.png" width="800" alt="mapreduce执行概览"/>

1. 用户程序中的`mapreduce library`会将输入的文件切分成`16MB`～`64MB`的文件块，然后它在`cluster`中启动多个副本。
2. 在`cluster`上跑的多个程序中有一个是特殊的，其为`master`节点，剩余的为`worker`节点。`master`节点向`worker`节点分配任务，当`worker`节点有空闲时，会向其分配`map`或`reduce`任务。
3. `worker`执行`map`任务时，会从切分的文件中读取数据，它从文件中读取`key/value`对，在`map()`函数中执行数据处理，生成的`intermediate key`会缓存在`memory`中。
4. `memory`中的`pair`会定期的写入本地磁盘，并将其位置信息返给`master`节点，其负责将`buffered pair`对应的位置信息转发给其它`worker`节点。
5. 当`master`节点将`map`函数产生的中间数据位置告知`reduce worker`时，其会使用`rpc`从`map worker`的本地磁盘中读取数据。当`reduce worker`读完所有数据后，它会对`intermediate key`对数据进行排序，因此，具有相同`key`的中间结果就会被`group`在一起。`sort`是非常必要的，因为通常情况下，许多不同的`key`会映射到相同的`reduce`函数中。当中间数据太大在`memory`中放不下时，会使用外部排序进行处理。
6. `reduce worker`会遍历排序后的中间数据，将`intermediate key`及对应`value`集合传给`reduce`函数，`reduce()`函数的输出结果会`append`到一个最终的输出文件中。当所有`map`和`reduce`任务都执行完成后，它会告知用户程序返回用户代码。

### 容错性考虑
由于`mapreduce`旨在帮助使用成百上千台机器处理处理大量数据，因此该机器必须优雅地容忍机器故障，分别讨论下当`worker`和`master`节点故障时，如何进行容错？

**`worker`节点故障**，`master`节点会周期性的`ping`所有的`worker`节点，若`worker`在给定时间内未响应，则`master`会标记`worker`为`failure`状态。此时，该`worker`节点上已执行完的`map task`会被重新置为`initial idle`状态，然后会等待其它`worker`执行此`task`。类似的，任何此`worker`上正在执行的`map()`或`reduce()`任务也会被重置为`idle`状态，然后等待调度。

为什么已经完成的`map task`还要被重新执行呐？因为`map()`会将`intermediate data`写在本次磁盘上，当`worker`不可访问时，执行`reduce()`时无法从`failure worker`中取数据。而`completed reduce`不需要重新执行，因为`reduce()`函数已将最终结果写到外部存储`HDFS`上。

**`master`节点故障问题**，容错方案较为简单，就是让`master`每隔一段时间将`data structures`写到磁盘上，做`checkpoint `。当`master`节点`die`后，重新启动一个`master`然后读取之前`checkpoint`的数据就可恢复状态。
