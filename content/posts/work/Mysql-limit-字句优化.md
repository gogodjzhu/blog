---
title: "Mysql Limit 字句优化"
date: 2019-11-05T17:45:11+08:00
description: "limit是最常使用的mysql关键字之一，在不同场景下有不同的优化策略，善用这些策略少踩坑"
tags: ["MySQL"]
draft: false
---

`LIMIT`字句常用的语法类似于: `LIMIT m,n`, 针对不同的情况, MySQL会对查询做一些优化.
总的来说性能主要由以下几个条件决定:

- LIMIT遍历数据量少, 性能高
- LIMIT通过索引实现筛选, 性能比较高
- LIMIT找到所需的数据就停止排序, 性能优于先完整排序再截取数据
- 语句整体能被索引覆盖, 不需要回表, 性能比较高

下面分别举例说明:



# 普通SELECT + LIMIT

这是最简单的场景, 按照存储顺序遍历所有数据, 直到遍历到目标位置才停止.

```sql
create tbl6(
    ...
    PRIMARY KEY(id),
    index(col1)
)ENGINE=InnoDB
mysql root@localhost:test_db>  select * from tbl6 limit 8000,10;
-- resultSet...
Time: 0.005s

mysql root@localhost:test_db> explain select * from tbl6 limit 800000,10;
+------+---------------+---------+--------------+--------+-----------------+--------+-----------+--------+--------+------------+---------+
|   id | select_type   | table   |   partitions | type   |   possible_keys |    key |   key_len |    ref |   rows |   filtered |   Extra |
|------+---------------+---------+--------------+--------+-----------------+--------+-----------+--------+--------+------------+---------|
|    1 | SIMPLE        | tbl6    |       <null> | ALL    |          <null> | <null> |    <null> | <null> | 983291 |        100 |  <null> |
+------+---------------+---------+--------------+--------+-----------------+--------+-----------+--------+--------+------------+---------+
mysql root@localhost:test_db> select * from tbl6 limit 800000,10;
-- resultSet...
Time: 0.240s
```

可以看到遍历的数量少时, 速度更快, `LIMIT`操作需要扫描的数据增加, 就需要通过一些方式来有优化.



## 覆盖索引优化

当SELECT的字段被某个索引覆盖, 由于不需要回表查询, 效率少许提升.

```sql
mysql root@localhost:test_db> explain select col1 from tbl6 limit 800000,10;
+------+---------------+---------+--------------+--------+-----------------+----------+-----------+--------+--------+------------+-------------+
|   id | select_type   | table   |   partitions | type   |   possible_keys | key      |   key_len |    ref |   rows |   filtered | Extra       |
|------+---------------+---------+--------------+--------+-----------------+----------+-----------+--------+--------+------------+-------------|
|    1 | SIMPLE        | tbl6    |       <null> | index  |          <null> | idx_col1 |        69 | <null> | 983291 |        100 | Using index |
+------+---------------+---------+--------------+--------+-----------------+----------+-----------+--------+--------+------------+-------------+

mysql root@localhost:test_db> select col1 from tbl6 limit 800000,10;
-- resultSet...
Time: 0.132s
```



## 延迟索引优化

但有的时候我们需要的字段确实无法覆盖索引, 那可以通过延迟索引的方式来实现. 通过`主键索引id`的覆盖索引子查询可以加速遍历数据, 最后通过join操作将索引效果延迟影响到所有非覆盖索引字段.

```sql
mysql root@localhost:test_db> explain select tbl6.* from tbl6 inner join(
                              select id from tbl6 limit 800000, 10
                              ) X on tbl6.id = X.id;
+------+---------------+------------+--------------+--------+-----------------+----------+-----------+--------+--------+------------+-------------+
|   id | select_type   | table      |   partitions | type   | possible_keys   | key      |   key_len | ref    |   rows |   filtered | Extra       |
|------+---------------+------------+--------------+--------+-----------------+----------+-----------+--------+--------+------------+-------------|
|    1 | PRIMARY       | <derived2> |       <null> | ALL    | <null>          | <null>   |    <null> | <null> | 800010 |        100 | <null>      |
|    1 | PRIMARY       | tbl6       |       <null> | eq_ref | PRIMARY         | PRIMARY  |         4 | X.id   |      1 |        100 | <null>      |
|    2 | DERIVED       | tbl6       |       <null> | index  | <null>          | idx_col1 |        69 | <null> | 983291 |        100 | Using index |
+------+---------------+------------+--------------+--------+-----------------+----------+-----------+--------+--------+------------+-------------+

mysql root@localhost:test_db> select tbl6.* from tbl6 inner join(
                              select id from tbl6 limit 800000, 10
                              ) X on tbl6.id = X.id;
-- resultSet...
Time: 0.126s
```



## 索引跳表优化

但上面两种优化都需要遍历非必要数据, 假设我们这里的id主键是有序递增的, 我们可以通过`id > n`的方式来跳过数据扫描, 这对性能有显著提升.

```sql
mysql root@localhost:test_db> explain select col1 from tbl6 where id > 800000 limit 10;
+------+---------------+---------+--------------+--------+-----------------+---------+-----------+--------+--------+------------+-------------+
|   id | select_type   | table   |   partitions | type   | possible_keys   | key     |   key_len |    ref |   rows |   filtered | Extra       |
|------+---------------+---------+--------------+--------+-----------------+---------+-----------+--------+--------+------------+-------------|
|    1 | SIMPLE        | tbl6    |       <null> | range  | PRIMARY         | PRIMARY |         4 | <null> | 450462 |        100 | Using where |
+------+---------------+---------+--------------+--------+-----------------+---------+-----------+--------+--------+------------+-------------+

mysql root@localhost:test_db> select col1 from tbl6 where id > 800000 limit 10;
-- resultSet...
Time: 0.002s
```



# ORDER BY + LIMIT

在和`ORDER BY`组合使用的场景下, 如果`SELECT字段`+`ORDER BY字段`满足覆盖索引条件, MySQL会使用索引进行LIMIT. 但如果无法满足覆盖索引的条件, MySQL会做不同的处理:

1. 当LIMIT筛选的数据量小, 依然使用`ORDER BY字段`对应的索引(如果有的话)做LIMIT筛选.
2. 如果LIMIT需要的数据量大, 则先将符合条件的所有数据检索出来, 之后再根据`ORDER BY字段`做文本排序, 直到找到LIMIT所需的数据则停止排序.

```sql
create tbl6(
    ...
    index(col1)
)ENGINE=InnoDB

mysql root@localhost:test_db> explain select col1 from tbl6 order by col1 limit 800000,10\G;
-- 由于满足覆盖搜索引条件, 数据量再大也会使用索引进行排序后再LIMIT
***************************[ 1. row ]***************************
id            | 1
select_type   | SIMPLE
table         | tbl6
partitions    | None
type          | index
possible_keys | None
key           | idx_col1
key_len       | 69
ref           | None
rows          | 800010
filtered      | 100.0
Extra         | Using index

mysql root@localhost:test_db> explain select * from tbl6 order by col1 limit 800000,10\G;
-- 数据量大, 且不满足覆盖索引条件, 使用文本排序, 效率低
***************************[ 1. row ]***************************
id            | 1
select_type   | SIMPLE
table         | tbl6
partitions    | None
type          | ALL
possible_keys | None
key           | None
key_len       | None
ref           | None
rows          | 983291
filtered      | 100.0
Extra         | Using filesort

mysql root@localhost:test_db> explain select * from tbl6 order by col1 limit 100,10\G;
-- 数据量小, 虽然不满足覆盖索引条件, 但是还会使用索引进行排序
***************************[ 1. row ]***************************
id            | 1
select_type   | SIMPLE
table         | tbl6
partitions    | None
type          | index
possible_keys | None
key           | idx_col1
key_len       | 69
ref           | None
rows          | 110
filtered      | 100.0
Extra         | None
```

> 需要注意的是, 如果`order by`字段不唯一, 实际返回的顺序由执行计划不同可能会有变化. 而`LIMIT`字句是会影响执行计划的, 所以如果使用中对返回结果的顺序有严格要求, 请显式使用唯一建做`order by`操作。



# LIMIT 0

`LIMIT 0`会返回一个空集, 这在校验SQL正确性的时候很适用. 也可以作为获取应用中通过`ResultSet`来判断返回数据类型的一种方法。



# 写在最后

SQL优化跟应用场景密不可分, 通常一个查询的优化总是有极限的, 往往在业务上和架构上作出修改才是最优的解法, 比如:

- 对于直接面向用户的查询, 控制LIMIT分页的最大数量
- 限制SELECT字段的组合
- 分库分表
- 使用更适合大数据的分布式数据库