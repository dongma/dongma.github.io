---
layout: post
title: 大数据的三架马车之HDFS
---
> 主要介绍HDFS的基本组成和原理、Hadoop 2.0对HDFS的改进、HADOOP命令和基本API、通过读Google File System论文来理解HDFS设计理念。

`Hadoop`是`Apache`一个开源的分布式计算平台，核心是以`HDFS`分布式文件系统和`MapReduce`分布式计算框架组成，为用户提供了一套底层透明的分布式基础设施。

`HDFS`是`Hadoop`分布式文件系统，具有高容错性、高伸缩性，允许用户基于廉价精简部署，构件分布式文件系统，为分布式计算存储提供底层支持。`MapReduce`提供简单的`API`，允许用户在不了解底层细节的情况下，开发分布式并行程序，利用大规模集群资源，解决传统单机无法解决的大数据处理问题，其设计思想起源`Google GFS`、`MapReduce Paper`。

<!-- more -->
#### 在Mac上搭建Hadoop单机版环境
从 https://hadoop.apache.org 下载二进制的安装包，具体配置可进行`Google`。配置完成后，在执行`HDFS`命令时会提示 `Unable to load native-hadoop library for your platform...using buildin-java classes..`，运行`Hadoop`的二进制包与当前平台不兼容。

为解决该问题，需在机器上编译`Hadoop`的源码包，用编译生成的`native library`替换二进制包中的相同文件。编译`Hadoop`源码需安装`cmake`、`protobuf`、`maven`、`openssl`组件。
```shell
$ mvn package -Pdist,native -DskipTests -Dtar
```
在编译hadoop-2.10.1的`hadoop-pipes`模块时出现错误，原因是由于`openssl`的版本不兼容，机器上的是`32`位，而实际需要`64`位。最后从github下载`openssl-1.0.2q.tar.gz`安装包，通过源码安装，并在/etc/profile中配置环境变量：
```shell
export OPENSSL_ROOT_DIR=/usr/local/Cellar/openssl@1.0.2q
export OPENSSL_INCLUDE_DIR=/usr/local/Cellar/openssl@1.0.2q/include/
```
然后重新执行`maven`命令，`hadoop`源码编译通过了。最后将`hadoop-dist`目录下的`native`包拷贝到`hadoop`二进制的源码包下就可以了。

#### Hadoop 1.0架构
`GFS cluster`由一个`master`节点和多个`chunkserver`节点组成，多个`GFS client`可以对其进行访问，其中每一个通常都是运行用户级服务器进程的商用`linux`机器。大文件会被分为大小固定为`64MB`的块。

<img src="../../../../resource/2021/hadoop/hadoop-architecture.jpg" width="700" alt="Hadoop 1.0架构图"/>

`HDFS 1.0`中的角色划分：

  * `NameNode`：对应论文中的`GFS master`，`NN`维护整个文件系统的文件目录树，文件目录的元信息和文件数据块索引；元数据镜像`FsImage`和操作日志`EditLog`存储在本地，但整个系统存在单点问题，存在`SPOF（Simple Point Of Filure）`。
  * `SecondNameNode`：又名`CheckPoint Node`用于定期合并`FsImage`和`EditLog`文件，其不接收客户端的请求，作为`NameNode`的冷备份。
  * `DataNode`：对应`GFS`中的`chunkserver`，实际的数据存储单元（以`Block`为单位），数据以普通文件形式保存在本地文件系统。
  * `Client`：与`HDFS`交互，进行读写、创建目录、创建文件、复制、删除等操作。`HDFS`提供了多种客户端，命令行`shell`、`Java api`、`Thrift`接口、`C library`、`WebHDFS`等。

`HDFS`的`chunk size`大小为`64MB`，这比大多数文件系统的`block`大小要大。较大的`block size`优势在于，在获取块位置信息时候，减少了`client`与`NameNode`交互的次数。其次，由于在大的`block`上，客户端更有可能在给定块上执行许多操作，可以与`NameNode`保持一个长时间的`TCP`连接来减少网络开销。第三，减少了存储在`NameNode`上的元数据的大小，这就可以使得`NameNode`将元数据信息保存在`Memory`中。
