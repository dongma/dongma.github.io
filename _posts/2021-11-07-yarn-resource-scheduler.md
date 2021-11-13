---
layout: post
title: 大数据三架马车之Yarn、BigTable
---
在`Hadoop V1.0`版本中，资源调度部分存在扩展性差、可用性差、资源利用率低的改问题，其中，`Job Tracker`既要做资源管理，又要做任务监控，同时`Job`的并发数页存在限制。同时，`JobTracker`存在单点故障问题，任务调度部分不支持调度流式计算、迭代计算、DAG模型。

`2013`年，`Hadoop 2.0`发布，引入了`Yarn`、`HDFS HA`、`Federation`。

### Yarn的设计思路（Yet Another Resource Manager）
`Yarn`由三部分组成：`ResourceManager`、`NodeManager`、`ApplicationMaster`，其中：`RM`掌控全局的资源，负责整个系统的资源管理和分配（处理客户端请求、启动/监控`AM`和`NM`、资源调度和分配），`NM`驻留在一个`YARN`集群的节点上做代理，管理单个节点的资源、处理`RM`、`AM`的命令，`AM`为应用程序管理器，负责系统中所有所有应用程序的管理工作（数据切分、为`APP`申请资源并分配、任务监控和容错）。

`Yarn`主要解决数据集群资源利用率低、数据无法共享、维护成本高的问题，常见的应用场景有：`MapReduce`实现离线批处理、`Impala`实现交互式查询分析、用`Strom`实现流式计算、在`Spark`下来完成迭代计算。
<!-- more -->

### Yarn Container及资源调度流程
`Container`是`Yarn`资源的抽象，它封装了某个节点上一定量的资源（`CPU`和内存两类资源），它跟`Linux Container`没有任何关系，仅仅是`Yarn`提出的一个概念（可序列、反序列的对象）。
```scala
message ContainerProto {
  optional ContainerIdProto id = 1; // container id
  optional NodeIdProto nodeId = 2;  // 资源所在节点
  optional string node_http_address = 3;
  optional ResourceProto resource = 4; // container资源量
  optional PriorityProto priority = 5; // container优先级
  optional hadoop.common.TokenProto container_token = 6;
}
```
`Container`由`ApplicationMaster`向`ResourceManager`申请的，由`ResourceManager`中的资源调度器异步分配给`ApplicationMaster`。`Container`的运行是由`ApplicationMaster`向资源所在的`NodeManager`发起的，`Container`运行时需提供内部执行的任何命令（比如`Java`、`Python`、`C++`进程启动命令均可）及该命令执行所需的环境变量和外部资源。

### 资源调度算法及调度器
调度算法是整个资源管理系统中一个重要的部分，简单地说，调度算法的作用是决定一个计算任务需要放在集群中的哪台机器上面。待调度的任务需考虑资源需求（`CPU`、`Memory`、`Disk`），应用亲和及反亲和性等。
* `FIFO`调度，先来的先被调用、分配`CPU`、内存等资源，后来的在队列等待。适用于平均计算时间、耗时资源差不多的作业，通常还可匹配优先级，不足在于用户将`Job`作业优先级设置的最高时，会导致排在后面的短任务等待。
* `SJF（Shortest Job First）`调度，为了改善`FIFO`算法，减少平均周转时间，提出了短作业优先算法。任务执行前预先计算好其执行时间，调度器从中选择用时较短的任务优先执行，但优先级无法保证。
* 时间片轮转调度`(Round Robin，RR)`，核心思想是`CPU`时间分片（`time slice`）轮转就绪任务，当时间片结束时，任务未执行完时发生时钟中断，调度器会暂停当前任务的执行，并将其置于就绪队列的末尾。此调度优点在于跟任务大小无关，都可获得公平的资源分配。但实现较为复杂，计算框架需支持中断。
* 最大最小公平调度（`Min-Max Fair`），将资源平分为`n`份（每份`S/n`），把每份分给相应的用户。若超过了用户的需求，就回收超过的部分，然后将总体回收的资源平均分给上一轮分配中尚未得到满足的用户，直到没有回收的资源为止。
* 容量调度（`Capacity`）,首先划分多个队列，队列资源采用容量占比的方式进行分配。每个队列设置资源最低保证和使用上限。如果队列中的资源有剩余或空闲，可暂时共享给那些需要资源的队列，一旦该队列有新的应用程序需要运行资源，则其它队列释放的资源会归还给该队列。

`Yarn`的三种调度器实现为：`Fair Scheduler`(公平调度器)、`FIFO Scheduler`(先进先出调度器)、`Fair Scheduler`(公平调度器)，`FIFO`先进先出调度器，同一时间队列中只有一个任务在执行，可以充分利用所有的集群资源。`Fair Scheduler`和`Capacity Scheduler`有区别的一些地方，`Fair`队列内部支持多种调度策略，包括`FIFO`、`Fair`、`DRF（Dominant Resource Fairness）`多种资源类型（`e.g.CPU`、内存的公平资源分配策略）。

### Job提交流程
在`yarn`上提交`job`的流程如下方的步骤图所示，`yarnRunner`向`rm`申请一个`Application`，`rm`返回一个资源提交路径和`application_id`，客户端提交`job`所需要的资源(切片+配置信息+`jar`包)到资源提交路径。
<img src="../../../../resource/2021/yarn/yarn-submit-job.jpg" width="700" alt="yarn上job提交流程"/>
`Capacity Scheduler`参数调整是在`yarn-site.xml`中，`yarn.resourcemanager.scheduler.class`用于配置调度策略`org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler`。

`Yarn`的高级特性`Node Label`，`HDFS`异构存储只能设置让某些数据（以目录为单位）分别在不同的存储介质上，但是计算调度时无法保障作业运行的环境。在`Nodel Label`出现之前，资源申请方无法指定资源类型、软件运行的环境（`JDK`、`python`）等，目前只有`Capacity Scheduler`支持此功能，`Fair Scheduler`正在开发，`yarn.node-labels.enable`用于开启`Node Label`的配置。
