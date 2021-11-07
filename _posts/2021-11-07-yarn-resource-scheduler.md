---
layout: post
title: 大数据三架马车之Yarn
---
在`Hadoop V1.0`版本中，资源调度部分存在扩展性差、可用性差、资源利用率低的改问题，其中，`Job Tracker`既要做资源管理，又要做任务监控，同时`Job`的并发数页存在限制。同时，`JobTracker`存在单点故障问题，任务调度部分不支持调度流式计算、迭代计算、DAG模型。

`2013`年，`Hadoop 2.0`发布，引入了`Yarn`、`HDFS HA`、`Federation`。

### Yarn的设计思路（Yet Another Resource Manager）
`Yarn`由三部分组成：`ResourceManager`、`NodeManager`、`ApplicationMaster`，其中：`RM`掌控全局的资源，负责整个系统的资源管理和分配（处理客户端请求、启动/监控`AM`和`NM`、资源调度和分配），`NM`驻留在一个`YARN`集群的节点上做代理，管理单个节点的资源、处理`RM`、`AM`的命令，`AM`为应用程序管理器，负责系统中所有所有应用程序的管理工作（数据切分、为`APP`申请资源并分配、任务监控和容错）。
