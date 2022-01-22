---
layout: post
title: 大数据时代数据仓库Hive
---
在`Hadoop`大数据平台及生态系统中，使用`mapreduce`模型进行编程，对广大用户来说，仍然是具有挑战性的任务。人们希望使用熟悉的`SQL`语言，对`hadoop`平台上的数据进行分析处理，这就是`SQL On Hadoop`系统诞的背景。

`SQL on Hadoop`是一类系统的简称，这类系统利用`Hadoop`实现大量数据的管理，具体是利用`HDFS`实现高度可扩展的数据存储。在`HDFS`之上，实现`SQL`的查询引擎，使得用户可以使用`SQL`语言，对存储在`HDFS`上的数据进行分析。

### `Apache Hive`的产生
`Hive`是基于`Hadoop`的一个数仓工具，可以将结构化的数据文件映射为一张数据库表，并提供简单的类`SQL(HQL)`查询功能，可以将`HQL`语句转换成为`MapReduce`任务进行运行。使用类`SQL`语句就可快速实现简单的`MapReduce`统计，不必开发专门的`MapReduce`应用。`Apache Hive`是由`Facebook`开发并开源，最后贡献给`Apache`基金会。

`Hive`系统整体`4`个部分：用户接口、元数据存储、驱动器(`Driver`)、`Hadoop`计算与存储。
<!-- more -->
1. 用户接口主要有`3`个，`CLI`、`ThriftServer`和`HWI`。最常用的就是`CLI`，启动`hive`命令回同时启动一个`Hive Driver`。`ThriftServer`是以`Thrift`协议封装的`Hive`服务化接口，可提供跨语言的访问，如`Python`、`C++`等，最后一种是`Hive Web Interface`提供浏览器的访问方式。
2. 表结构的一些`Meta`信息是存储在外部数据库的，如`MySQL`、`Oracle`和`Derby`库。`Hive`中元数据包括表的名字、表的列和分区及其属性、表的属性（是否为外部表等）、表的数据所在目录等。

<img src="../../../../resource/2022/hive/hive_architecture.jpg" width="800" alt="Hive整体设计"/>
