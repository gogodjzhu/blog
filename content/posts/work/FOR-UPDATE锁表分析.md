---
title: "FOR UPDATE之锁表怪现象"
date: 2019-09-25T02:36:11+08:00
description: "用过MySQL来实现分布式锁的同学一定知道for update这个操作，但是你知道for update一次会锁住多少数据？单条？全表？还是部分？"
tags: ["MySQL"]
draft: false
---

# 非索引字段锁全表

测试表:

```mysql
CREATE DATABASE so1;
USE so1;

CREATE TABLE notification (
    `id` BIGINT(20), 
    `date` DATE, 
    `text` TEXT) 
ENGINE=InnoDB;

INSERT INTO notification(id, `date`, `text`) values (1, '2011-05-01', 'Notification 1');
INSERT INTO notification(id, `date`, `text`) values (2, '2011-05-02', 'Notification 2');
INSERT INTO notification(id, `date`, `text`) values (3, '2011-05-03', 'Notification 3');
INSERT INTO notification(id, `date`, `text`) values (4, '2011-05-04', 'Notification 4');
INSERT INTO notification(id, `date`, `text`) values (5, '2011-05-05', 'Notification 5');
```

执行以下操作会被锁住, *注意date=’2011-05-02’并不在`[2011-05-03,∞)`范围内, 但依然被锁住。*

| Connection1                                                  | Connection2                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| BEGIN;                                                       |                                                              |
| SELECT * FROM notification WHERE `date` >= ‘2011-05-03’ FOR UPDATE; |                                                              |
|                                                              | BEGIN;                                                       |
|                                                              | SELECT * FROM notification WHERE `date` = ‘2011-05-02’ FOR UPDATE; **#BLOCK** |

更诡异的是, 将where条件由区间`>=`变成单条`=`同样会锁住整张表:

| Connection1                                                  | Connection2                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| BEGIN;                                                       |                                                              |
| SELECT * FROM notification WHERE `date` = ‘2011-05-03’ FOR UPDATE; |                                                              |
|                                                              | BEGIN;                                                       |
|                                                              | SELECT * FROM notification WHERE `date` = ‘2010-05-02’ FOR UPDATE; **#BLOCK** |

类似的, insert/delete/update操作也会被锁住。 那么问题来了, `FOR UPDATE`触发的是什么锁? 竟然会锁住整张表?



# 索引字段锁单条记录

当我们将where条件的字段换成索引字段, 情况稍有不同。

将`id`字段设置为主键,并插入同样的数据

```mysql
CREATE TABLE notification (
  id BIGINT(20),
  date DATE,
  text TEXT,
  PRIMARY KEY (id)
)
ENGINE=InnoDB;

-- 插入同样的测试数据
```

执行以下语句, 会发现当where条件仅过滤单条索引记录, 那么只会锁住这一条数据。 当where条件过滤的为非索引记录, 依然会被锁住全表。

| Connection1                                           | Connection2                                                  |
| :---------------------------------------------------- | :----------------------------------------------------------- |
| BEGIN;                                                |                                                              |
| SELECT * FROM notification WHERE `id` = 1 FOR UPDATE; |                                                              |
|                                                       | BEGIN;                                                       |
|                                                       | SELECT * FROM notification WHERE `id` = 2 FOR UPDATE; **#NON-BLOCK** |
|                                                       | SELECT * FROM notification WHERE `id` = 1 FOR UPDATE; **#BLOCK** |
|                                                       | SELECT * FROM notification WHERE `date`=’2011-05-02′ FOR UPDATE; **#BLOCK** |

而当我们对索引字段执行区间查询时, 只有部分区间被锁住:

| Connection1                                                  | Connection2                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| BEGIN;                                                       |                                                              |
| SELECT * FROM notification WHERE `id` between 1 and 3 FOR UPDATE; |                                                              |
|                                                              | BEGIN;                                                       |
|                                                              | SELECT * FROM notification WHERE `id` = 0 FOR UPDATE; **#NON-BLOCK** |
|                                                              | SELECT * FROM notification WHERE `id` = 1 FOR UPDATE; **#BLOCK** |
|                                                              | SELECT * FROM notification WHERE `id` = 2 FOR UPDATE; **#BLOCK** |
|                                                              | SELECT * FROM notification WHERE `id` = 3 FOR UPDATE; **#BLOCK** |
|                                                              | SELECT * FROM notification WHERE `id` = 4 FOR UPDATE; **#BLOCK** |
|                                                              | SELECT * FROM notification WHERE `id` = 5 FOR UPDATE; **#NONBLOCK** |

# 真相只有一个

那就是, `FOR UPDATE`会锁住它扫描过的所有索引记录, 官方文档中是这样描述的:

> A locking read, an UPDATE, or a DELETE generally set record locks on every index record that is scanned in the processing of the SQL statement.  It does not matter whether there are WHERE conditions in the statement that would exclude the row. InnoDB does not remember the exact WHERE condition, but only knows which index ranges were scanned。 [原文](https://dev。mysql。com/doc/refman/5。7/en/innodb-locks-set。html)
>
> 一个加锁读, 更新, 或者删除操作, 都会在扫描的所有记录上加锁。 换个表述方式或许更容易理解：因为where字句的作用是过滤数据，而被过滤的数据是先与where处理之前提取出来的。这也是文档中说的`does not remember the exact WHERE condition`所指的另一层意思。

关键就在于**扫描的所有记录**, 它并不仅仅是`WHERE`条件命中的记录。 如果扫描的列没有索引, 那么需要扫描所有行, 所以就会导致锁全表。 这样做的原因主要为了实现`可重复读`, 在一个事务内, 扫描过的记录可能再次扫描。 如果不加锁, 别的事务过来修改数据, 就会导致数据不一致。

回过头来看前面的诡异现象, 就很好解释了。

在***非索引字段锁全表*** 例子中, 由于`date`字段没有索引。 所以对它进行的WHERE条件过滤都会触发全表扫描, 那么加排他锁读就需要锁全表。
在***索引字段锁单条记录*** 例子中, `id`字段是主键索引, 所以针对它的`=`操作, 只会扫描1条记录, 也就只会锁住一条记录。 但当我们使用`date`字段做查询的时候, 由于它需要锁全表, 所以在尝试获取`id=1`这条记录的锁时, 被阻塞。

比较特殊的是`between`条件。 between条件查询的是一个区间。 为了保证`可重复读`[1,3]区间, 我们可以解释为什么`id=1,2,3`会由于无法获取锁而被阻塞。 但是对于`id=4`这条记录为什么也会阻塞? 它不是在区间之外么?
为了解释这个问题, 让我们回到官方文档的描述:`set record locks on every index record that is scanned`。 它的表述是, 会锁住`所有扫描过的索引记录`。 而且强调`InnoDB does not remember the exact WHERE condition`, 它不会帮你去记住`WHERE`条件, 它只知道它扫描过哪些索引记录, 在这些记录上加锁。

> 扣细节的朋友会发现官方说的`index record`, 锁住索引记录, 如果表没有索引呢? 在另一篇[文档](https://dev。mysql。com/doc/refman/5。7/en/innodb-locking。html#innodb-record-locks)有写: `For such cases, InnoDB creates a hidden clustered index and uses this index for record locking。` 没有索引, 它会帮你创建一个隐藏的`clustered index`(clustered index被翻译成[聚簇索引](https://dev.mysql.com/doc/refman/5.7/en/innodb-index-types.html))

具体到我们这个例子, 处理`between 1 and 3`的右边界3时, 会再往下扫描(为什么要往下扫描?留给你自己想想), 看看下一个键(术语叫临键`Next-Key`)`id=4`是否也满足区间条件。 虽然它不满足条件, 在最终的结果集中不会包含这个键。 但是由于它被扫描过, 所以依然会被加锁。

# 总结

至此, 我们就完美地解释了`FOR UPDATE`引发的诡异问题。 总结一下, 当我们使用`FOR UPDATE`(当然也包括UPDATE, DELETE)时, InnoDB会按照我们SQL语句解释出来的规则去扫描索引树。 为了保证数据一致性, 所有扫描过的索引记录, 均会被加锁。而不同场景下显示出来的怪现象的本质，则是各种锁的加锁范围上的区别而已。

