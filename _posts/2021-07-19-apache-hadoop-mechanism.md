---
layout: post
title: 大数据的三架马车之HDFS
---
> 主要介绍HDFS的基本组成和原理、Hadoop 2.0对HDFS的改进、HADOOP命令和基本API、通过读Google File System论文来理解HDFS设计理念。

### 大数据的三架马车之HDFS
Hadoop是Apache一个开源的分布式计算平台，核心是以HDFS分布式文件系统和MapReduce分布式计算框架组成，为用户提供了一套底层透明的分布式基础设施。
HDFS是Hadoop分布式文件系统，具有高容错性、高伸缩性，允许用户基于廉价精简部署，构件分布式文件系统，为分布式计算存储提供底层支持。MapReduce提供简单的API，允许用户在不了解底层细节的情况下，开发分布式并行程序，利用大规模集群资源，解决传统单机无法解决的大数据处理问题，其设计思想起源Google GFS、MapReduce Paper。

<!-- more -->
