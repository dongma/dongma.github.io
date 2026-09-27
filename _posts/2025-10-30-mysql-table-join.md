---
layout: post
title: 数据安全和表 join
date: 2025-10-30 +0800
tags: ["MySQL", "大数据"]
---

<!-- more -->
### 数据被误删后，应怎么办？
我们不止要说误删数据的事后处理办法，更重要是要做到事前预防。我有以下两个建议：
1. 把`sql_safe_updates`参数设置为`on`。这样一来，如果我们忘记在`delete`或者`update`语句中写`where`条件，或者`where`条件里面没有包含索引字段的话，这条语句的执行就会报错。
2. 代码上线前，必须经过`SQL`审计。

`mysql`中的`kill`命令，一个是`kill query` + 线程`id`，表示终止这个线程中正在执行的语句；一个是`kill connection` + 线程`id`，表示断开这个线程的连接。

`sql kill`不掉的原因，其实是因为发送`kill`命令的客户端，并没有强行停止目标线程的执行，而只是设置了个状态，并唤醒对应的线程。而被`kill`的线程，需要执行到判断状态的"埋点"，才会开始进入终止逻辑阶段。并且，终止逻辑本身也是需要耗费时间的。

在`mysql`中，对一个`200G`的大表做全表扫描，会不会把内存用光？ 如下图所示，`mysql`是**边读边发的**：
1. 从`mysql`读出的行会写到`net_buffer`中，此内存大小由`net_buffer_length`定义，默认`16k`；
2. `net_buffer`写满后，会调用网络接口发出去，然后继续取下一行；
3. `socket send buffer`本地网络栈写满后，会进入等待，直到网络栈重新可写，再继续发送；
<div>
    <img src="/resource/2025/mysql/server_send_data.jpg" width="620"/>
</div>

`InnoDB`内存管理使用`LRU`算法，这个算法的核心就是淘汰最久未使用的数据。按`5:3`的比例将整个`LRU`链分成了`young`区域和`old`区域，改进后的算法，不会导致有大量的页被淘汰。
<div>
    <img src="/resource/2025/mysql/lru_young_old.jpg" width="620"/>
</div>

### 表之间的`join`
**`index nested-loop join`**：`join`的两张表都能使用上索引，其`explain`执行计划如下所示：
```sql
select * from t1 straight_join t2 on (t1.a=t2.a);
```
计算逻辑：先遍历`t1`表，从数据行中取出字段`a`的值，去表`t2`中查找满足条件的记录，并且在`join`的过程中，可以使用上被驱动表的索引。

思考，`join`为什么要选择小表？驱动表的行数是`N`，然后对于每一行，到被驱动表上匹配一次，在被驱动表上查一行的时间复杂度是`2*log2M`，因此整个执行过程，近似复杂度是`N + N*2*log2M`，显然主表`N`的影响更大。
<div>
    <img src="/resource/2025/mysql/index_loop_join.jpg" width="620"/>
</div>

**`block nested-loop join`**：被`join`的表使用不上索引，其`explain`执行计划如下所示，在`Extra`中会展示`Block Nested Loop`：
```sql
select * from t1 straight_join t2 on (t1.a=t2.b);
```
计算逻辑：会将驱动表加载到内存`join buffer`（由`join_buffer_size`设定的，默认值是`256k`，超过就要分段）中，然后扫描表`t2`，将`t2`中的每一行与`join buffer`中的数据做对比，满足`join`条件的，做为结果集的一部分返回。

思考，`join`为什么要选择小表？尤其是在大表上的`join`操作，这样可能要扫描被驱动表很多次，会占用大量的系统资源，因此也建议使用小表做为`join`表。
<div>
    <img src="/resource/2025/mysql/block_nested_loop.jpg" width="620"/>
</div>

**MRR优化（`Multi-Range Read`）**

此优化的目的是尽量使用顺序读盘，大多数的数据都是按照主键递增顺序插入的，所以，按照主键的递增顺序查询能够提升读性能。
```sql
set optimizer_switch="mrr_cost_based=off";
```
<div>
    <img src="/resource/2025/mysql/use_mrr_optimize.jpg" width="620"/>
</div>

**`BNL`转`BKA`，添加索引**

有些时候，会碰到一些不适合在被驱动表建索引的情况，看执行计划和实际执行，查询速度是比较慢的（`sql`执行接近`38.37s`）。
```sql
select * from t1 join t2 on (t1.b=t2.b) where t2.b>=1 and t2.b<=2000;
```
<div>
    <img src="/resource/2025/mysql/slow_query_nlj.jpg" width="620"/>
</div>

优化方式，使用临时表，然后再临时表上创建索引，最终执行时间不到`0.1s`。
<div>
    <img src="/resource/2025/mysql/temp_table_for_query.jpg" width="620"/>
</div>

### 临时表和内部临时表
1. 建表语法是`create temporary table...`，一个临表只能被创建它的`session`访问，对其它线程不可见；
2. 临时表可以与普通表重名，同名时`show create`语句展示的是临时表；
3. `show tables`命令不显示临时表，临时表前缀是：#sql{进程 id}_{线程 id}_ 序列号；

执行`union`语句时会用到临时表，其执行计划如下，在`Extra`中含有`Using temporary`，使用临时表主要是为了去重，向临时表写入已存在的`id`值时，
会拒绝写入。在`union all`中则不会使用临时表。
```sql
(select 1000 as f) union (select id from t1 order by id desc limit 2);
```
<div>
    <img src="/resource/2025/mysql/union_and_unionall.jpg" width="620"/>
</div>

**优化`group by`** 查询，在`MySQL 5.7`版本支持了`generated column`机制，用来实现列数据的关联更新。
```sql
alter table t1 add column z int generated always as(id % 100), add index(z);
/* 重新按group by查询 */
select z, count(*) as c from t1 group by z;
```
<div>
    <img src="/resource/2025/mysql/group_by_tmp.jpg" width="620"/>
</div>

优化后，看`sql`的执行计划，在`Extra`中已经没有`Using temporary`了，走的索引。
<div>
    <img src="/resource/2025/mysql/group_by_generated_row.jpg" width="620"/>
</div>

当`group by`的表不好加索引，并且临时表数据量特别大的，可以使用`SQL_BIG_RESULT`这个提示，告诉优化器，请直接用磁盘临时表。
```sql
select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by m;
```
<div>
    <img src="/resource/2025/mysql/sql_big_result.jpg" width="620"/>
</div>