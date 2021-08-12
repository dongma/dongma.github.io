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
