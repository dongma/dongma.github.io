---
layout: post
title: 大数据时代数据仓库Hive
---
在`Hadoop`大数据平台及生态系统中，使用`mapreduce`模型进行编程，对广大用户来说，仍然是具有挑战性的任务。人们希望使用熟悉的`SQL`语言，对`hadoop`平台上的数据进行分析处理，这就是`SQL On Hadoop`系统诞的背景。

`SQL on Hadoop`是一类系统的简称，这类系统利用`Hadoop`实现大量数据的管理，具体是利用`HDFS`实现高度可扩展的数据存储。在`HDFS`之上，实现`SQL`的查询引擎，使得用户可以使用`SQL`语言，对存储在`HDFS`上的数据进行分析。

### `Apache Hive`的产生
`Hive`是基于`Hadoop`的一个数仓工具，可以将结构化的数据文件映射为一张数据库表，并提供简单的类`SQL(HQL)`查询功能，可以将`HQL`语句转换成为`MapReduce`任务进行运行。使用类`SQL`语句就可快速实现简单的`MapReduce`统计，不必开发专门的`MapReduce`应用。`Apache Hive`是由`Facebook`开发并开源，最后贡献给`Apache`基金会。

`Hive`系统整体`3`个部分：用户接口、元数据存储、驱动器(`Driver`)在`Hadoop`上计算与存储。
<!-- more -->
1. 用户接口主要有`3`个，`CLI`、`ThriftServer`和`HWI`。最常用的就是`CLI`，启动`hive`命令回同时启动一个`Hive Driver`。`ThriftServer`是以`Thrift`协议封装的`Hive`服务化接口，可提供跨语言的访问，如`Python`、`C++`等，最后一种是`Hive Web Interface`提供浏览器的访问方式。
2. 表结构的一些`Meta`信息是存储在外部数据库的，如`MySQL`、`Oracle`和`Derby`库。`Hive`中元数据包括表的名字、表的列和分区及其属性、表的属性（是否为外部表等）、表的数据所在目录等。
3. `Driver`部分包括：编译器、优化器和执行器，编译器完成词法分析、语法分析，将`HQL`转换为`AST`。`AST`生成逻辑执行计划，然后物理`MR`执行计划；优化器用来对逻辑计划、物理计划进行优化，生成的物理计划转变为`MR Job`并在`Hadoop`集群上执行。
<img src="../../../../resource/2022/hive/hive_architecture.jpg" width="800" alt="Hive整体设计"/>

### `Hive`数据模型
`Hive`通过以下模型来组织`HDFS`上的数据，包括：数据库`DataBase`、表`Table`、分区`Partition`和桶`Bucket`。
1. `Table`管理表和外表，`Hive`中的表和关系数据库中的表很类似，依据数据是否受`Hive`管理可分为：`Managed Table`（内表）和`External Table`（外表）。对于内表，`HDFS`上存储的数据由`Hive`管理，`Hive`对表的删除影响实际的数据。外表则只是一个数据的映射，`Hive`对表的删除仅仅删除愿数据，实际数据不受影响。
2. `Partition`基于用户指定的列的值对数据表进行分区，每一个分区对应表下的相应目录`${hive.metastore.warehouse.dir}/{database_name}.db/{tablename}/{partition key}={value}`，其优点在于从物理上分目录划分不同列的数据，易于查询的简枝，提升查询的效率。
3. `Bucket`桶作为另一种数据组织方式，弥补`Partition`的短板，通过`Bucket`列的值进行`Hash`散列到相应的文件中，有利于查询优化、对于抽样非常有效。

### `Hive`的数据存储格式，聊聊`Parquet*`
`Parquet*`起源于`Google Dremel`系统，相当于`Dremel`中的数据存储引擎。最初的设计动机是存储嵌套式数据，如`Protocolbuffer`、`thrift`和`json`等，将这些数据存储成列式格式，以便于对其高效压缩和编码，且使用更少的`IO`操作取出需要的数据。并且其存储`metadata`，支持`schema`的变更。
<img src="../../../../resource/2022/hive/parquent-format.jpg" width="760" alt="Parquet格式"/>
`Parquet*`是面向分析型业务的列式存储格式，由`Twitter`和`Cloudera`合作开发。一个`Parquet`文件通常由一个`header`和一个或多个`block`块组成，以一个`footer`结尾。`footer`中的`metadata`包含了格式的版本信息、`schema`信息、`key-value pairs`以及所有`block`中的`metadata`信息。
* `parquent-format`项目定义了`parquent`内部的数据类型、存储格式等。
* `parquent-mr`项目完成外部对象模型与`parquent`内部数据类型的映射，对象模型可以简单理解为内存中的数据表示，`Avro`、`Thrift、Protocol Buffers`等这些都是对象模型。

### `Hive Catalog`介绍
* `HCatalog`是`Hadoop`的元数据和数据表的管理系统，它基于`Hive`中的元数据层，通过类似`SQL`的语言展现`Hadoop`数据的关联关系。
* `Catalog`允许用户通过`Hive`、`Pig`、`MapReduce`共享数据和元数据，用户编写应用程序时，无需关心数据怎样存储、在哪里存储，避免因`schema`和存储格式的改变而受到影响。
* 通过`HCatalog`，用户能通过工具访问`Hadoop`上的`Hive Metastore`。它为`MapReduce`和`Pig`提供了连接器，用户可以使用工具对`Hive`的关联列格式的数据进行读写。
<img src="../../../../resource/2022/hive/hive-catalog.jpg" width="820" alt="HCatalog元数据管理"/>
