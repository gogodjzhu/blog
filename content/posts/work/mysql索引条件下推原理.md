---
title: "mysql索引条件下推原理"
date: 2020-02-17T17:45:11+08:00
description: "mysql5.6增加了一个针对索引性能的优化方案，可以利用索引把本来由服务层做的过滤操作下推到引擎层完成，较少了数据的传输，提升索引效率。"
featured_image: "http://minio.gogodjzhu.com/images/20210403_120010_e8dcfb2a-bb4f-45c4-9fd3-8f0ce50a65c5.png"
images: ["http://minio.gogodjzhu.com/images/20210403_120010_e8dcfb2a-bb4f-45c4-9fd3-8f0ce50a65c5.png"]
tags: ["MySQL"]
draft: false
---

# 何为「索引条件下推」？

假设存在一张表结构如下：

```mysql
create table people
(
 zipcode varchar(11) null,
 lastname varchar(20) null,
 address varchar(20) null
);

create index people_zipcode_lastname_address_index
 on people (zipcode, lastname, address);
```

以下面这条SQL为例，当没有索引条件下推优化的时候，由于联合索引只能命中`zipcode`(最左原则+模糊匹配字符串不支持索引过滤)，导致引擎层只能将`zipcode='95054'`的数据全部取出来交给服务器层再做两个`LIKE`过滤。而有了联合索引过滤，这个条件过滤操作被下推到引擎层直接执行，优化性能。具体来说，就是引擎层会通过索引找到`zipcode='95054'`的数据，并进一步遍历判断所有数据，判断满足两个LIKE条件的数据才进行返回。

```mysql
SELECT * FROM people
  WHERE zipcode='95054'
  AND lastname LIKE '%etrunia%'
  AND address LIKE '%Main Street%';
```

> 在使用`EXPLAIN`命令进行执行分析的时候，如果查询使用了ICP特性，Extra会显示`Using index condition`，而不是普通索引时的`Using index`。



# 使用条件

要使用ICP特性，必须满足几个条件：

- 引擎支持，毕竟是ICP本身是引擎特性
- 过滤条件必须是通过`LIKE '%str%'`，`BETWEEN`，`OR NULL`等方法且需要获取所有字段时方可触发。
- Innodb只用于二级索引，因为聚簇索引会直接将完整的数据读入InnDB Buffer，这时候无法通过减少从引擎从读入服务层的IO。
- 子查询无法优化
- 存储过程也无法优化，因为在引擎层无法处理存储过程。