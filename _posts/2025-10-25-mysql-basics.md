---
layout: post
title: MySQL 基础篇
date: 2025-10-25 +0800
tags: ["MySQL", "大数据"]
---

<!-- more -->
### mysql环境准备
启动一个`mysql`容器，对外端口号是`3316`，`root`用户默认的登陆密码是`root`：
```shell
docker run -p 3316:3306 --name mysql5.7 -v /Users/madong/opt/mysql5_7/conf:/etc/mysql/conf.d -v /Users/madong/opt/mysql5_7/logs:/logs -v /Users/madong/opt/mysql5_7/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
# 进入到mysql容器中，查看当前的mysql的连接信息
docker exec -it mysql5.7 /bin/bash
# 查看数据库的连接情况，当前的mysql实例有一个连接对象：
mysql> show processlist;
#+----+------+-----------+------+---------+------+----------+------------------+
#| Id | User | Host      | db   | Command | Time | State    | Info             |
#+----+------+-----------+------+---------+------+----------+------------------+
#|  2 | root | localhost | NULL | Query   |    0 | starting | show processlist |
#+----+------+-----------+------+---------+------+----------+------------------+
#1 row in set (0.00 sec)
```

### `mysql`组件概览
大体来说，`MySQL`可以分为`Server`层和存储引擎两部分。其中，`Server`层包括连接器、查询缓存、分析器、优化器、执行器等，涵盖`MySQL`的大多数核心服务功能。而存储引擎层负责数据的存储和提取，现常用的存储引擎是`InnoDB`。
- 连接器：连接器负责根客户端建立连接、获取权限、维持和管理连接，连接命令一般为：`mysql -h$ip -P$port -u$user -p`
- 查询缓存：连接建立后，就可以执行`select`语句，若缓存中有此语句的查询缓存，则直接返回结果。不建议开启，`MySQL 8.0`已移除此部分；
- 分析器：包含词法分析、语法分析，执行的`sql`是否符合语法规范，另外，查询的表或字段是否存在？
- 优化器: 当表中有多个索引时，优化器决定使用哪个索引，决定各表的连接顺序；
- 执行器：在执行器阶段，开始执行语句。会判断你对这个表`T`有没有执行查询的权限，若没有，就返回没有权限的错误；

<div>
    <img src="/resource/2025/mysql/mysql_parts_1.jpg" width="620"/>
</div>

`mysql`中的日志模块，`redo log`、`binlog`以及两阶段提交？拿更新语句分析，在专栏中，作者拿"孔乙己"中记账粉板来类比，`MySQL`中的`WAL`也就是先写日志，然后再写磁盘。
```sql
mysql> update T set c=c+1 where ID=2;
```
当有一条记录更新时，`InnoDB`引擎就会先把记录写到`redo log`（粉板）里面，并更新内存，这个时候更新就算完成了。`InnoDB`的`redo log`是固定大小的，可以配置一组`4`个文件，每个文件的大小是`1GB`。

在下图中，`write pos`是当前记录的位置，`checkpoint`是当前要擦除的位置，当`write pos`追上`checkpoint`时，表示"粉板"满了，这时候不能再执行新的更新。
<div>
    <img src="/resource/2025/mysql/redo_log.jpg" width="580"/>
</div>

在`server`层也有自己的日志，称为`binlog`（归档日志），经常问的`redo log`和`bin log`的区别有哪些？
- 1、`redo log`是`InnoDB`引擎特有的，`binlog`是`MySQL`的`Server`层实现的，所有引擎多可以使用。
- 2、`redo log`是物理日志，记录的是"在某个数据页上做了什么修改"；`binlog`是逻辑日志，记录这个语句的原始逻辑，比如"给`ID=2`这一行`c`字段加`1`"。
- 3、`redo log`是循环写，空间固定会用完。`binlog`是追加写入的，当`binlog`文件写到一定大小后会切换下一个，并不会覆盖以前的日志。

`mysql`的两阶段提交，是为了让`redo log`和`binlog`保持一致。具体步骤为，`mysql`中有这一行先写入`redolog`中，处于`prepare`阶段，然后写`binlog`，写完后提交事务，处于`commit`状态。

扩展，`binlog`和`redolog`的写入机制：

`binlog`写入，事务执行过程中，会先把日志写到`binlog cache`，事务提交的时候，再把`binlog cache`写入`binlog`文件中。
每个线程都有自己的`binlog cache`，但是他们共用一份`binlog`文件。

<div>
    <img src="/resource/2025/mysql/redo_log_cache.jpg" width="580"/>
</div>

`redo log`也是先写`redo log buffer`，然后写到`page cache`里面（上图黄色部分），然后持久化到磁盘上。`InnoDB`对`innodb_flush_log_at_trx_commit`提供了`3`种取值：
- `0`，每次提交只把`redo log`留在`redo log buffer`中；
- `1`，每次提交都将`redo log`持久化到磁盘；
- `2`，每次提交都把`redo log`写到`page cache`中；

`mysql`日志的组提交机制，`LSN`是单调递增的，用来对应`redo log`的一个个写入点。如下图，是三个并发事务`(trx1, trx2, trx3)`在`prepare`阶段，都写完`redo log buffer`，持久化到磁盘的过程，对应的`LSN`分别是`50`、`120`和`160`。

`trx1`是第一个到达的，会被选为这组的`leader`，像写`redo log`和`binlog`都会以组为单位，因而，一次组提交里面，组员越多，节约磁盘`IOPS`的效果越好。
<div>
    <img src="/resource/2025/mysql/group_commit.jpg" width="580"/>
</div>

若`MySQL`遇到了性能瓶颈，并且是在`IO`上，则可以考虑调参：
1. 设置`binlog_group_commit_sync_delay`和`binlog_group_commit_sync_no_delay_count`参数，减少`binlog`的写盘次数。
2. 将`sync_binlog`设置为大于`1`的值（比较常见是`100~1000`）。这样做的风险是，主机掉电时会丢`binlog`日志。
3. 将`innodb_flush_log_at_trx_commit`设置为`2`（写`page cache`，避免丢`redo log`）。这样做的风险是，主机掉电的时候会丢数据。

### `mysql`的事务
提到事务，肯定会想到`ACID`（`Atomitity`、`Consistency`、`Isolation`、`Durability`），即原子性、一致性、隔离性、持久性。`MySQL`事务的隔离级别分为：读未提交、
读已提交、可重复读和串行化。

在"可重复读"级别下，视图是在事务启动时创建，整个事务存在期间都用这个视图。在"读提交"隔离级别下，这个视图是在每个`SQL`语句开始执行的时候创建。"读未提交"隔离
级别直接返回记录的最新值，没有视图的概念；而"串行化"隔离级别下直接用加锁的方式来避免并行访问。

在`mysql`中，每条记录在更新时都会同时记录一条回滚操作，记录上的最新值，通过回滚操作，都可以得到钱一个状态的值。一个值从`1`被按顺序改成了`2`、`3`、`4`，在回滚日志里面会有类似下面的记录。

在视图`A`、`B`、`C`里面，同一条记录在系统中可以存在多个版本，就是数据库的多版本并发控制（`MVCC`）。回滚日志，在不需要的时候才删除，也就是说，当没有事务
再需要用到这些回滚日志时，回滚日志会被删除。
<div>
    <img src="/resource/2025/mysql/mysql_mvcc.jpg" width="580"/>
</div>

### 事务到底是隔离的还是不隔离的？
如下启动`3`终端中执行操作，事务`A`和事务`B`查询的值是多少呢？若想要马上启动一个事务，可以使用`start transaction with consistent snapshot`这个命令。
<div>
    <img src="/resource/2025/mysql/tx_exam_1.jpg" width="580"/>
</div>

实际执行下来，事务`A`看到`k`的值为`1`，事务`B`看到`k`的值为`3`。分析原因？
- 第一个有效更新是事务`C`，将数据从`(1,1)`改成了`(1,2)`，这个时候新版本的`row tx_id`是`102`，`90`是历史版本。
- 第二个有效更新是事务`B`，将数据从`(1,2)`改成了`(1,3)`，这时数据的最新版是`101`，而`102`又成为了历史版本。
- 事务`A`查询时，其实事务`B`还没有提交，事务`B`生成的版本对事务`A`必须不可见，否则就变成脏读了。
<div>
    <img src="/resource/2025/mysql/exam_exec_1.jpg" width="720"/>
</div>

**快照在`MVCC`里面是如何工作的？**
`InnoDB`中每个事务都有一个唯一的事务`ID`，叫做`transaction id`。它是在事务开始的时候向`InnoDB`申请的，是按申请顺序严格递增的。

每行数据也都是有多个版本，每次更新事务数据时，都会生成一个新的数据版本，并把transaction_id赋值给这个数据版本的事务`ID`，记为`row trx_id`。

如下图，一个数据版本的`row trx_id`，有以下几种可能：
- 落在绿色部分，表示这个版本是已提交的事务或是当前事务自己生成的，是可见的；
- 落在红色部分，表示这个版本是由将来启动的事务生成的，是不可见的；
- 落在黄色部分，若`row trx_id`在数组中，表示这个版本是由还没提交的事务生成的，不可见；若`row trx_id`不在数组中，表示这个版本是已提交了的事务生成的，可见。
<div>
    <img src="/resource/2025/mysql/tx_visiable.jpg" width="520"/>
</div>

**更新数据规则**: 更新数据都是先读后写的，而这个读，只能读当前的值，称为"当前读"(`current read`)，除了`update`语句外，`select`语句加锁，也是当前读。
```sql
# lock in share mode和for update都是当前读 
select k from t where id = 1 lock in share mode;
select k from t where id = 1 for update;
```
