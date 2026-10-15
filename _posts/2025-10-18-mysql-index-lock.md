---
layout: post
title: MySQL 索引和锁
date: 2025-10-18 +0800
tags: ["MySQL"]
---

<!-- more -->
### 索引基础
`mysql`索引存储使用`N`叉树，以`InnoDB`为例，当树的高度是`4`时，九可以存`1200`的`3`次方个值，大概`17`亿，查找一个值最多只需要访问`3`次磁盘。

索引类型分为主键索引和非主键索引，在`InnoDB`里，主键索引也被称为聚簇索引，其存储整行数据。非主键索引也被称为二级索引起叶子结点存的是主键的值，在基于二级索引查询时，会进行一次回表。

当`sql`语句只查询`ID`字段时，由于`ID`字段已经在`k`索引树上，不需要回表查询，索引`k`已经覆盖了我们的查询需求，称之为覆盖索引。
```sql
select ID from idx_T where k between 3 and 5;
```
<div>
    <img src="/resource/2025/mysql/mysql_index_1.jpg" width="590"/>
</div>

最左前缀：`mysql`索引的最左前缀匹配原则，建立了`(name, age)`这个联合索引。当查找条件是"查所有姓名是张三"或"姓名的第一个字是张"都会走这个索引。

规则，这个最左前缀可以是联合索引的最左`N`个字段，也可以是字符串索引的最左`M`个字符。

在`mysql 5.6`引入了索引下推（`index condition pushdown`），可以在索引遍历过程中，对索引中包含的字段先做哦安段，过滤不满足条件的记录，减少回表次数。
<div>
    <img src="/resource/2025/mysql/mysql_left_index.jpg" width="590"/>
</div>

### `mysql`的锁
1、**全局锁**，命令是：`flush tables with read lock`，当你需要让整个库处于只读状态时，可以使用这个命令。之后，其它线程的：数据更新语句、数据定义语句和更新类事务的提交语句都会被阻塞。

`mysql`的逻辑备份工具`mysqldump`使用参数`--single-transaction`导数据之前就会启动一个事务，来确保拿到一致性视图。而由于`MVCC`的支持，这个过程中数据是可以正常更新。

2、**表级锁**有两种，一种是表锁，一种是元数据锁。

表锁的语法是`lock tables ...read/write`，可以使用`unlock tables`主动释放锁。需要注意，`lock tables`语法除了会限制别的线程的读写外，也限定了本线程接下来的操作对象。

另一个表级的锁是`MDL`，在访问一个表的时候会被自动加上。`MDL`的作用是，保证读写的正确性。在加上`MDL`锁期间，不允许另一个线程对表结构做变更。

2、**行锁**，`MySQL`的行锁是在引擎层由各个引擎实现的，`MyISAM`不支持行锁，`InnoDB`是支持行锁的。
```sql
# 在第二个窗口中，针对于id = 200这一行的update，事务阻塞住了。
update t_idx set k=k+2 where id = 200;
```
<div>
    <img src="/resource/2025/mysql/row_lock.jpg" width="590"/>
</div>

**死锁和死锁检测**，当出现死锁时，有两种策略：
- 直接进入等待，直到超时，这个超是时间可以通过参数`innodb_lock_wait_timeout`来设置。
- 发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其它事务得以继续执行。将参数`innodb_deadlock_detect`设置为`on`，表示开启这个逻辑。
