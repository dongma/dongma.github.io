---
layout: post
title: 大数据的三架马车之HDFS
---
> 主要介绍HDFS的基本组成和原理、Hadoop 2.0对HDFS的改进、HADOOP命令和基本API、通过读Google File System论文来理解HDFS设计理念。

### 大数据的三架马车之HDFS
Hadoop是Apache一个开源的分布式计算平台，核心是以HDFS分布式文件系统和MapReduce分布式计算框架组成，为用户提供了一套底层透明的分布式基础设施。

HDFS是Hadoop分布式文件系统，具有高容错性、高伸缩性，允许用户基于廉价精简部署，构件分布式文件系统，为分布式计算存储提供底层支持。MapReduce提供简单的API，允许用户在不了解底层细节的情况下，开发分布式并行程序，利用大规模集群资源，解决传统单机无法解决的大数据处理问题，其设计思想起源Google GFS、MapReduce Paper。

<!-- more -->
#### 在Mac上搭建Hadoop单机版环境
从 https://hadoop.apache.org 下载二进制的安装包，具体配置可进行Google。配置完成后，在执行HDFS命令时会提示 `Unable to load native-hadoop library for your platform...using buildin-java classes..`，运行`Hadoop`的二进制包与当前平台不兼容。

为解决该问题，需在机器上编译Hadoop的源码包，用编译生成的`native library`替换二进制包中的相同文件。编译`Hadoop`源码需安装`cmake`、`protobuf`、`maven`、`openssl`组件。
```shell
$ mvn package -Pdist,native -DskipTests -Dtar
```
在编译hadoop-2.10.1的`hadoop-pipes`模块时出现错误，原因是由于`openssl`的版本不兼容，机器上的是`32`位，而实际需要`64`位。最后从github下载`openssl-1.0.2q.tar.gz`安装包，通过源码安装，并在/etc/profile中配置环境变量：
```shell
export OPENSSL_ROOT_DIR=/usr/local/Cellar/openssl@1.0.2q
export OPENSSL_INCLUDE_DIR=/usr/local/Cellar/openssl@1.0.2q/include/
```
然后重新执行`maven`命令，`hadoop`源码编译通过了。最后将`hadoop-dist`目录下的`native`包拷贝到`hadoop`二进制的源码包下就可以了。
