---
title: "MySQL实战45讲(1)-日志机制"
date: 2021-04-01T17:45:11+08:00
description: ""
tags: ["MySQL", "网课笔记"]
categories: "MySQL实战45讲"
draft: true

---

> 系列文章源自极客时间[《MySQL实战45讲》](http://gk.link/a/10pWf)课程的学习整理。

## 简介

本文主要讨论MySQL提供的两套日志系统，`binlog`和`redo log`。

binlog 全称binary log，是mysql 提供的用于`保存数据修改`的日志系统，可通过`log-bin`开关。binlog的核心思路是WAL，同时它本身也是跨节点数据同步的基础。
redo log是innodb引擎提供的，用于恢复事务数据的日志系统。redo log的设计思路通过二次提交来保证事务读写的安全性。

> binlog vs. redo log 数据恢复功能有什么差异？
>
> binlog保存的是已提交未持久化到最终数据页的修改操作，结合checkpoint可以将数据恢复到binlog保存的任意时刻。它是mysql server提供的功能，所有引擎均可配合使用。
> redo log是则是innodb提供的功能，本质是基于事务设计的二次提交机制。它也支持持久化，但它实现为固定大小的环形结构，因此新数据会覆盖旧数据，无法恢复太长时间前的数据。
> redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”

![img](http://minio.gogodjzhu.com/images/20210418_175013_cccc15a5-40f0-493d-98f6-7cf90c298d7f.png)redo log的环形结构

> undo log又是啥?
>
> undo log 是innodb提供的用于回滚事务的日志系统。事务成功提交需要持久化，这时使用的是redo log；而如果事务失败，则需要回滚事务，这就是undo log提供的，它保存的是事务内每个语句的逆操作。同时，undo log也是MVCC机制的基石。更多内容看

## 工作流程

对于启用了binlog和redo log的数据表，数据写入的流程如下图所示。

![img](http://minio.gogodjzhu.com/images/20210418_175009_34e500ed-48a9-44c4-9384-0bce4d42894c.png)

需要注意的是数据在写入binlog 和redo log时采用的是交叉的二次提交方式，这样的做法是为了保证二者数据的一致性。在数据恢复的时候，会综合二者的记录。举例来说：写入log分三个阶段：1.写prepare阶段，2.写binlog，3.写commit
当在2之前崩溃时
重启恢复：后发现没有commit，回滚。备份恢复：没有binlog 。一致
当在3之前崩溃
重启恢复：虽没有commit，但满足prepare和binlog完整，所以重启后会自动commit。备份恢复：有binlog. 一致

## 原理 & 细节

前面介绍了binlog和redolog的大致工作流程，但还有几个关键问题值得讨论：

- 写入binlog和redo log的时候，数据具体是怎样落盘的？有什么设计来提高QPS？
- 使用binlog和redo log恢复数据是怎样实现的？binlog和redo log如何协同？

### binlog写入机制

binlog提供了binlog cache，用于将日志暂存在内存中，最终刷入磁盘做持久化。对于事务（innodb）控制的数据，为了事务的原子性，单个事务内的日志写入binlog cache后，必须在事务提交的时候整体刷入磁盘（或者整体丢弃）。每个线程都有自己的一块binlog cache，大小由`binlog_cache_size`配置。

![img](http://minio.gogodjzhu.com/images/20210418_175004_5e8db18f-ddfc-4de9-a55b-701ab08e4fa4.png)binlog写入流程，注意每个线程虽然自己维护binlog cache，但最终落盘时还是写到同一个binlog文件

注意到上图写binlog文件涉及到两个步骤：`write`和`fsync`。write是将binlog cache的日志写到目标binlog文件的page cache，fsync则是持久化到磁盘。由于两个操作的效率差别很大，mysql提供了`sync_binlog`参数用于控制write/fsync的操作策略，基本思想还是化零为整批量优化IOPS，具体细节看[文档](https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html#sysvar_sync_binlog)。

> binlog的三种格式
>
> binlog共有三种格式，分别是statement、row和mixed。
>
> statement格式保存的是原样的sql，好处是sql的解释性强，不需要保存每行数据节省空间，坏处则是sql被解释成数据修改时存在变数，在环境发生变化时可能同样的一条sql得到不同的结果。
>
> row是保存每行数据的物理修改，好处是精确无依赖，坏处是保存所有行数据占用空间太多。
>
> mixed模式是两者的综合， mysql会根据执行的每一条具体的sql语句来区分对待记录的日志格式，也就是在Statement和Row之间选择一种。 

### redo log的写入机制

redo log持久化的顺序为：`redo log buffer` -> `page cache` -> `hard disk`这跟binlog的并无二致。

redo log buffer是线程间共享的，由此带来了许多有意思的情况。首先是A线程写入redo log buffer的日志a，哪怕仍是prepare状态，可能因为B线程的commit操作导致日志a也写入磁盘（调用write/fsync？通过`innodb_flush_log_at_trx_commit`配置策略，这也是优化IOPS的一种方法）。也就是说，redo log保存的日志，可能属于未提交的事务。

除了commit操作，redo log buffer还有一种情况会触发写磁盘，redo log buffer 占用的空间即将达到 `innodb_log_buffer_size` 一半的时候，后台线程会主动写盘。此时由于非commit触发，没有事务提交，因此只是write到page cache，并未调用fsync。

> “双1”配置
>
> 指的是将`innodb_flush_log_at_trx_commit`和`sync_binlog`都配置为1，每次事务提交时，都会将binlog和redolog持久化到磁盘

### redo log 和 binlog 协同

redo log和binlog在保存的时候，都保存了一个XID字段（X for transaction，事务ID？），在崩溃回复的时候， 会按顺序扫描 redo log：
如果碰到既有 prepare、又有 commit 的 redo log，就直接提交；
如果碰到只有 parepare、而没有 commit 的 redo log，就拿着 XID 去 binlog 找对应的事务。