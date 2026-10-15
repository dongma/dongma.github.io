---
layout: post
title: MySQL实践-锁和高可用
date: 2025-10-28 +0800
tags: ["MySQL"]
---

<!-- more -->
## 锁和高可用

### `order by`是怎么工作的?
创建向表`t`插入数据的存储过程，删除存储过程`drop procedure city_date;`。
```sql
delimiter ;;
create procedure city_data()
begin
  declare i int;
  set i=1; 
  while(i<=6000) do
       if i <= 4000 then
         insert into t values(i, 'hangzhou', 'tony', 12, 'hangzhou2');
       else
         insert into t values(i, 'beijing', 'lili', 13, 'beijing3');
       end if;
       set i=i+1; 
  end while;
end;;
delimiter ;
call city_data();
```

全字段排序，`explain`语句后会有`Using filesort`，`sort_buffer_size`就是`MySQL`为排序开辟的内存`（sort_buffer）`的大小。
```sql
select city,name,age from t where city='杭州' order by name limit 1000;

/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 
/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';
/* 执行语句 */
select city, name,age from t where city='hangzhou' order by name limit 1000; 
/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';
/* 计算Innodb_rows_read差值 */
select @b-@a;
```
`rowid`排序和全部字段排序，查看`trace`后，当`sort_buffer`比较大时，`sort_mode`为`additional_fields`，比较小时，`sort_mode`为`row_key`。
<div>
    <img src="/resource/2025/mysql/full_fields_sort.jpg" width="680"/>
</div>
<div>
    <img src="/resource/2025/mysql/row_id_sort.jpg" width="680"/>
</div>

### 谈几个索引不生效的场景
1、`tradelog`表中`t_modified`字段有索引，但使用`month`函数后，在`sql`查询时，索引会失效。原因是，对索引字段做函数操作，可能会破坏索引值的
有序性，因此优化器就决定放弃走树搜索功能。
```sql
select count(*) from tradelog where month(t_modified)=7;
```
2、第二种是隐式类型转换，在如下的`sql`中，在字符串与数值比较时，实际会使用`cast`函数将字符串转换为整数，由于加了函数，也会导致索引失效。
```sql
/* tradeid 的字段类型是 varchar(32)，而输入的参数却是整型 */
select * from tradelog where tradeid = 110717;
/* 等价于如下的sql内容，从本质看，也是使用了CAST函数 */
mysql> select * from tradelog where CAST(tradid AS signed int) = 110717;
```
3、关联的字段字符集不统一，导致的索引失效，在如下表中`tradelog`表的字符集是`utf8mb4`，`trade_detail`表的字符集是`utf8`。
```sql
select d.* from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2; /*Q1*/
```
查看`sql`的执行计划，其中`tradelog`为`utf8mb4`，与`detail`表字符集不统一，导致在第二行中`key`列为`NULL`，导致走的全表扫描。明明`detail`表的`tradeid`有索引，但是使用不上。
<div>
    <img src="/resource/2025/mysql/charset_mismatch.jpg" width="680"/>
</div>

在统一`tradeid`的字符集为`utf8`后，在执行计划的第`2`行的`key`列，其值为`traceid`，使用上了索引。
<div>
    <img src="/resource/2025/mysql/charset_uniform.jpg" width="680"/>
</div>

### 执行单条`sql`语句但很慢的原因？
1、当`session A`持有表`t`的`MDL`写锁，而`session B`查询需要获取`MDL`读锁，所以，`session B`的查询进入等待状态。这类问题处理，就是找到谁持有
`MDL`写锁，然后把它`kill`掉。
```sql
/* 在`session A`中执行`unlock tables;`会释放表上的锁，解除死锁 */
lock table t write;    
```
<div>
    <img src="/resource/2025/mysql/lock_table_read.jpg" width="680"/>
</div>

2、等待表`flush`，下图中的`session C`就处于阻塞的情况，等待`flush tables`完成操作。
<div>
    <img src="/resource/2025/mysql/wait_flush_tables.jpg" width="680"/>
</div>

3、等行锁，`session A`启动事务，并且不进行提交，`session B`查询时使用`lock in share mode`方式，则就会被阻塞。
<div>
    <img src="/resource/2025/mysql/row_lock_share_mode.jpg" width="680"/>
</div>

### `mysql`中的幻读是怎么解决的？
`session A`用`beigin;`开启事务，然后用`sql select * from t where d=5 for update;`，此时`session B`和`session C`的语句都处于阻塞状态。

产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。为了解决幻读问题，InnoDB 只好引入新的锁，也就是间隙锁 (`Gap Lock`)。
<div>
    <img src="/resource/2025/mysql/gap_lock.jpg" width="680"/>
</div>

如下图所示，库中有`6`条记录，但生成了`7`个间隙锁，间隙锁会阻止在间隙中插入记录行。
<div>
    <img src="/resource/2025/mysql/gap_lock2.jpg" width="680"/>
</div>

### `mysql`中加锁的规则
`mysql`的加锁规则中，包含了两个原则、两个优化和一个`bug`:
1. 原则`1`，加锁的基本单位是`next-key lock`，其对应前开后闭区间。
2. 原则`2`，查找过程中访问的对象才会加锁。
3. 优化`1`，索引上的等值查询，给唯一索引加锁的时候，`next-key lock`退化为行锁。
4. 优化`2`，索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，`next-key lock`退化为间隙锁。
5. 一个`bug`，唯一索引上的范围查询会访问到不满足条件的第一个值为止。

等值查询，在最左侧的`Session A`中，会加上`(5,10)`这个范围的间隙锁，因而`Session B`会阻塞，`Session C`可正常执行。
<div>
    <img src="/resource/2025/mysql/equal_query_lock.jpg" width="680"/>
</div>

非唯一索引等值锁，`Session A`会在`(0,5)`和`(5,10)`上加间隙锁，此外，`Session A`的查询语句使用覆盖索引，并不需要访问主键，因而，主键索引上没有任何锁，`Session B`的语句可正常执行。
<div>
    <img src="/resource/2025/mysql/not_unique_lock.jpg" width="680"/>
</div>

此外，`limit`场景，在删除数据的时候尽量加`limit`，不仅可以控制删除数据的条数，还可以减小锁的范围。

### `mysql`高可用和主备延迟
主备延迟，有哪些可能的原因？
1. 有些部署条件下，备库所在机器的性能要比主库所在的机器性能差。
2. 备库的压力大，备库上执行一些运营后台需要的分析语句，耗费了大量的`CPU`资源，影响了同步速度，造成主备延迟。
3. 大事务，如果一个主库上的语句执行`10`分钟，那这个事务很可能就会导致从库延迟`10`分钟，一次性地用`delete`语句删除太多数据。

备库和主库延迟可以用`seconds_behind_master`这个指标判断，其单位为`s`，需根据实际情况，看满足"可用性优先策略"还是"可靠性优先策略"。

`mysql`在`5.6`版本引入了`GTID`（`Global Transaction Identifier`，也称全局事务`ID`）来改善主备同步麻烦的问题。
```sql
GTID=source_id:transaction_id
```
实例`A’`的`GTID`集合记为`set_a`，实例`B`的`GTID`集合记为`set_b`，在实例`B`上执行`start slave`命令，取`binlog`的逻辑是这样的：
1. 实例`B`指定主库`A’`，基于主备协议建立连接。
2. 实例`B`把`set_b`发给主库`A’`。
3. 实例`A’`算出`set_a`与`set_b`的差集，也就是所有存在于`set_a`，但是不存在于`set_b`的`GTID`的集合，判断`A’`本地是否包含了这个差集需要的所有`binlog`事务。
 - a. 如果不包含，表示`A’`已经把实例`B`需要的`binlog`给删掉了，直接返回错误；
 - b. 如果确认全部包含，`A’`从自己的`binlog`文件里面，找出第一个不在`set_b`的事务，发给`B`；
4. 之后就从这个事务开始，往后读文件，按顺序取`binlog`发给`B`去执行。
