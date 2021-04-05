---
title: "Mysql in/exist/join 策略分析"
date: 2019-11-05T17:45:11+08:00
description: "in/exist/join都是常用的关联查询语法，本文将关注大小表关联场景下各自使用的优化策略"
tags: ["MySQL"]
draft: false
---

# 测试数据

创建两个表t1和t2，它们格式完全相同，区别在与数据量t1是1k，t2是100k。
另外需要注意的是三个字段的区别，id是主键，a是索引字段，b则无索引，下面我们会比较不同类型字段对查询的影响。

```mysql
CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;
create table t1 like t2;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

insert into t1 (select * from t2 where id<=1000)
```



# in

#### 按主键id关联

![img](http://minio.gogodjzhu.com/images/20210404_163205_dc8cfb0e-f851-4c6c-a200-8b52d60ea38a.png)in关联查询的是主键字段，位置无影响，优化器会主动优化

#### 按索引字段a关联

![img](http://minio.gogodjzhu.com/images/20210404_163218_f7c9d47a-8516-4260-9393-e85758c354d5.png)如果in关联查询的是索引字段，表位置会影响优化策略

首先mysql会选择外层表为驱动表，in包含的表则为被驱动表，这不会随着数据量的变化而变化。确定驱动表后，如果驱动表是小表，则使用[FirstMatch](http://www.javacoder.cn/?p=15)策略匹配被驱动的大表；如果驱动表是大表，那就使用[LooseScan](http://www.javacoder.cn/?p=39)策略匹配被驱动的小表。

#### 按普通字段b关联

![img](http://minio.gogodjzhu.com/images/20210404_163230_1e71b5a9-ad03-4943-9096-d69349a665c2.png)in关联查询对非索引字段，使用物化(Materialization)优化策略

[Materialization](http://www.javacoder.cn/?p=45)（直译是物化，可以理解成将子查询‘物化’成表）是一种通过创建临时表来间接关联的优化策略。在执行的过程中，第一次需要子查询结果时，将整个子查询的结果保存到一个内存表（过大时也会落地磁盘），以后需要子查询时直接从表中取值，甚至将表做成hash映射进一步提高效率。

> FirstMatch，LooseScan，Materialization都被称为半连接(Semi Join)策略



# exists

比起in查询，exists提供的优化手段就相对单薄，同样的查询我们换成exists的形式：

![img](http://minio.gogodjzhu.com/images/20210404_163244_87bb4aa1-e361-4fc9-bc1f-5b479b306519.jpeg)exists默认的优化手段段仅停留在对主键/索引的使用上，没有in关联提供的半连接优化策略

# join

直接使用join语句的关键在于驱动表的选择，从下面的测试情况看来，left join会固定使用左表作为驱动表。如果使用主键/索引字段关联被驱动表，那么会触发树搜索。如果使用非索引字段关联被驱动表，那么就需要全表扫描。

![img](http://minio.gogodjzhu.com/images/20210404_163301_7fa2123d-b54a-4b78-8836-d5b0faad462c.jpeg)

> **join_buffer**
>
> 值得注意的是，在使用非索引字段来做关联join的时候，对被驱动表的扫描使用的`Block Nested Loop`方法。
> join关联查询中，第一步是从驱动表取出一条数据，第二步是去被驱动表中找出关联上的数据，第三部则是将关联上的数据拼成结果返回。对于第二步，如果对被驱动表做逐条判断性能太低，批量将被驱动表中的数据放入内存做计算效率显然更高。但由于全表扫描，无法预知被驱动表的数据量是否会撑满内存，因此使用分块的方式批量读取（这也是Block的来源）。临时存被驱动表分块数据的空间叫做`join_buffer`，由`join_buffer_size`参数控制。当join操作太慢时，可以通过适当增加此参数来优化。