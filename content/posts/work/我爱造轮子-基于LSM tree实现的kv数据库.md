---
title: "我爱造轮子-基于LSM tree实现的kv数据库"
date: 2019-11-24T23:15:21+08:00
description: "手撸LSM tree"
tags: ["算法", "动态规划"]
draft: true
---

> LSM是一种应付大数据量写入磁盘的数据结构模型，在NoSQL系统中非常常见，尤其是应对写多读少的场景非常有效。网上关于LSM的理论文章有很多，但是都仅限于原理，本着`talk is cheap, show me the code`的精神，这次拿github上一个基于LSM实现的key/value文件数据库`keydb` 为例，看一个LSM的代码实现，麻雀小五脏全，有了它定能助你在面试中唬住面试官。另外我fork了这个项目，添加了一些[代码注释](https://github.com/gogodjzhu/keydb)。



# 必也正名乎

LSM Tree (Log-structured merge-tree) ：这个名称挺容易让人困惑的，因为你看任何一个介绍LSM Tree的文章很难直接将之与树对应起来。事实上，它只是一种分层的组织数据的结构，具体到实际实现上，就是一些按照逻辑分层的有序文件[1]。

LSM的优势是能够高效率地写入数据，这种效率来源于充分利用磁盘和内存这两种存储介质的特点。将大量的随机写入落到内存中，再批量地将这些随机写合并成连续写落地磁盘。

实现一个LSM的关键在于 数据结构（包括内存中及磁盘的数据结构） 和 合并算法，以保证LSM的读写性能。

- **写性能** LSM分层组织数据，随机写先落地内存，之后的 *内存刷盘* 和 `合并操作` 都是顺序io，保证了写性能的高效。
- **读性能** 读取的数据可能落在两个部分：内存 和 磁盘。
  - 对于内存，高效读的数据结构选择比较多，针对不同的场景有很多的选择，本项目使用的是 *AVL树* 。
  - 对于磁盘，高效的数据结构往往都来自于 `索引` 和 `并发io`。在本项目中，使用了 *key/data文件*（索引）、*分段segment*（并发）、快照读来（并发）技术。



# Hello keydb

在使用上，keydb是一个go语言编写的高效kv数据库，基于文件做持久化存储，可以作为嵌入应用的简单版数据库。目前仅支持key/value的键值对存储，虽然简单，依然遵循了LSM的基础设计，是一个学习了解LSM的好工具。先看一个简单的使用demo：

```go
// 从指定目录打开一个数据库实例，所有的数据最终会写入到此目录中
db, err := keydb.Open("test/mydb", true)
if err != nil {
    panic(err)
}
// 开启一个操作gogo表的事务，注意在keydb中单个事务只能读写同一个表
// 单个表的数据最终会落地在以表名开头的一系列文件中
tx, err := db.BeginTX("gogo")
if err != nil {
    panic(err)
}
// 往此事务中写入一个数据，此时数据仍未持久化到磁盘，是内存操作，速度快
err = tx.Put([]byte("k1"), []byte("v1"))
if err != nil {
    panic(err)
}
// 按key读取数据，未命中内存的时候才会去磁盘中读取
value, err := tx.Get([]byte("k1"))
if err != nil {
    panic(err)
}
fmt.Printf("value:%v", value)
// 此次事务的所有修改一起落盘
if err = tx.CommitSync(); err != nil {
    panic(err)
}
// 关闭数据库，并显示调用合并方法
if err = db.CloseWithMerge(1); err != nil {
    panic(err)
}
```

由于代码简单，基本上我们可以根据跟踪这个demo来完成源码阅读。首先是通过`keydb.Open()`方法创建了一个`Database`实例，它维护了数据库的一些基本信息(包括开关状态/锁/数据文件路径等等)，这些属于事务性属性，我们不太关心。



# Database结构

- 核心属性

  ```go
  open : bool
  closing : bool
  wg : sync.WaitGroup
  lockfile : lockfile.Lockfile // 文件锁, 单个目录作为一个数据库，
  // 只能单进程访问,可通过goroutine并发
  path : string
  
  tables : map[string]*internalTable
  transactions : map[uint64]*Transaction
  nextSegID : uint46
  
  err : error
  ```

我们重点关注的是`tables`和`transactions`。前者是数据库当前存在的表集合，后者保存当前数据库打开的事务集合。

在keydb中，表在逻辑上是相互独立的map，物理上每个表使用不同的文件（表名作为相同的文件名前缀）保存数据。回到代表表集合的tables属性，它保存的是一个map，key即为表名，value是`internalTable`表示一个表实例。它有且仅有三个核心的属性：

## internalTable

- 核心属性

  ```go
  segments : []segment // 已持久化的数据集合
  transactions : int // 访问此表的事务数量
  name : string // 表名
  ```

`segments`代表的即是LSM-Tree中的`C1..Ck`(也有论文使用SSTable来论述，本质上是同一个东西)。它是保存在当前表中，已持久化入磁盘文件的数据集合。所有的文件都以时间先后顺序排列在数组中。每次调用`db.BeginTX(table string) `创建新事务，会带一个表名作为入参，以指定此事务要访问的表（每个事务只能访问一张表），如果segments中没有则会扫描磁盘文件，得到初始化的表状态。

```go
func (db *Database) BeginTX(table string) (*Transaction, error) {
	// ...
	it, ok := db.tables[table]
	if !ok {
		// 从指定目录读取{table}事务的key/value文件(如果有的话)，并恢复为内存表internalTable
		it = &internalTable{name: table, segments: loadDiskSegments(db.path, table)}
		db.tables[table] = it
	}
       // ...
}
```

注意segment其实是一个接口，共有两个实现。一个是diskSegment，它代表已持久化到硬盘的数据块，这部分数据是不能直接修改的（diskSegment未实现会修改数据的Put,Remove方法），只在创建table的时候通过`loadDiskSegments`方法扫描表文件，每个文件对应会有一个diskSegment对象被创建。



## diskSegment

- 核心属性

  ```go
  id : uint64
  keyBlocks : int64
  keyFile : *os.File
  
  dataFile : *os.File
  
  keyIndex : [][]byte
  ```

前面我们提到，对于落地磁盘的数据来说，创建索引文件是有效的增加读效率的一种方法。本项目采用的正是这种思路，将数据文件分为两类：索引文件(.keys结尾)和数据文件(.data结尾)。在diskSegment中保存了两类文件的信息，利用它们在单个diskSegment中根据key查找数据的步骤是这样的：

1. 通过二分法在key文件中寻找目标key。
2. 找到key之后，可以获取到对应的value被保存在data文件中的位置偏移量。
3. 根据data文件位置偏移量获取并检验value。



### diskSegment的数据结构

LSM在设计上，并未包括segment的具体实现方式，因此各种方案可以有自己的数据结构来支撑segment的功能。这里先看diskSegment包含的两种数据文件：

索引文件。在本项目中，使用类似数组的方式来保存内`索引block`，每个block使用相同的大小，按照key的顺序（划重点！是按顺序的保存）依次保存为block-1, block-2, … block-n。在这个基础上，每次通过key检索的时候就可以使用二分法来找到目标block。

数据文件。相比索引文件，数据文件就比较简单只需将value按照顺序添加即可，因为keyBlock内已保存对应value的offset和长度。



## memorySegment

- 核心属性

  ```go
  tree : *Tree // Tree是一个AVL树
  ```

segment接口的另外一个实现是`memorySegment`，它代表的是未持久化的，尚在内存中的热数据，本项目通过一个AVL树来组织这些数据。那么memorySegment数据跟diskSegment数据有什么不同呢？在本项目中，将未提交的数据保存在memorySegment中，每次创建事务的时候就会创建一个新的memorySegment关联在事务下面，因此这部分数据只在事务内可见。

事务提交的时候，memorySegment会按key的顺序（AVL树前序遍历即可实现单调有序）取出保存到diskSegment中（实现diskSegment中保存的数据都是有序的，这是diskSegment实现二分检索的前提）。在此事务提交之后创建的事务，会根据新的key/data文件来创建自己的segments集合，这就实现了数据可见。



# segment合并

最后一个要解决的问题就是segment的合并。我们已经知道，每次提交事务的时候，都会将一个memorySegment保存为key/data文件，也就是多了一个diskSegment。那么长此以往，diskSegment的文件数量必然越来多；文件数量多了之后，落到diskSegment上面的数据查找性能必然也会下降。因此必须有一套方案来控制diskSegment的数量，LSM中称之为合并。

> **segment不合并导致性能下降的原因？**
>
> 首先是因为不合并导致的垃圾数据冗余问题。假设现在表中有一个键值对`<keyA:valueA-old>`，在新的事务中对此key做修改，变成`<keyA:valueA-new>`。此事务提交的时候，会在磁盘中写下一个新的文件file2，且旧的file1不会删除。由于segments有序，我们查找的时候可以根据优先新数据的方式依次遍历每个file（也即diskSegment），从此file1中保存的keyA数据就属于垃圾数据，空占磁盘空间，也应想查询效率。
>
> 同时，如果不合并segment，挨个扫描文件的也会使得二分查找的性能优势得不到发挥，极端情况下每个事务如果只保存一条数据，那么查找的时间复杂度退化成`o(n)`。

了解合并的必要性之后，我们来看看keydb是如何实现合并操作的。触发keydb合并的时机共有两个：一个是关闭数据库的时候主动触发；另一个是通过一个循环goroutine被动触发。它们调用的方法都是同一个：`func mergeDiskSegments0(db *Database, segmentCount int) error` 将提供的数据库下面的每个表的segment数量都降到`segmentCount`以下。原理也很简单，按照segmets从新到旧的顺序遍历所有segment，每个segment内部按照key遍历并合并重复key。最终保存到一个新的diskSegment中。



# 总结

至此，关于keydb的简单介绍就结束了。总结一下本文内容：

- 首先介绍了什么是LSM树，以及它的基本概念。它不是一个具体的数据结构，它提出的是一个分层合并的模型，通过合并随机写为批量顺序写，拥有很好的写性能。
- 然后给你介绍了一个简单的开源LSM实现方案：`keydb`。并介绍了它的核心逻辑，包括内存 和 磁盘的数据结构设计 以及 如何合并segment。

学习数据结构的最好方法就是自己做一个实现，LSM原理并不难，但是在实现上却有非常多的差异，但是撇开差异，核心思路其实还是一样的，通过剖析keydb，相信能对你理解LSM的设计原理，并进一步学习其他成熟的LSM实现框架带来帮助。



[1] [LSM Tree/MemTable/SSTable基本原理](https://liudanking.com/arch/lsm-tree-summary/)