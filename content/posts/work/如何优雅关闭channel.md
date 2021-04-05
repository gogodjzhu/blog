---
title: "golang读取关闭channel遇到的问题/如何优雅关闭channel"
date: 2018-04-12T11:44:22+08:00
tags: ["golang"]
draft: false
---

本文核心内容：

> 1. 已关闭的channel再次读取会出现什么现象？
> 2. 如何判断channel关闭？
> 3. 什么是nil channel有什么用？



先看看出问题的代码片段：

```go
func TestReadFromClosedChan(t *testing.T) {
    asChan := func(vs ...int) <-chan int {
        c := make(chan int)
        go func() {
            for _, v := range vs {
                c <- v
                time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
            }
            close(c)
        }()
        return c
    }
    merge := func(a, b <-chan int) <-chan int {
        c := make(chan int)
        go func() {
            for {
                select {
                case v := <-a:
                    c <- v
                case v := <-b:
                    c <- v
                }
            }
        }()
        return c
    }

    a := asChan(1, 3, 5, 7)
    b := asChan(2, 4, 6, 8)
    c := merge(a, b)
    for v := range c {
        fmt.Println(v)
    }
}
```

目的很简单，就是把a和b两个channel合并到c当中，再通过range遍历把c里的元素全部打印出来。

**BUT!!**看起来如此简单的一段代码得到的结果却出乎意料。像这样：

```
1
2
3
4
5
6
7
8
0
0
0
...
#以及无数的0
...
```

看起来c合并了ab之后还多了一批无意义的0。原因在于：

> closed channel是可以被消费者继续读取的，在读完了有意义的数据之后，将读到一堆空值。比如这里的int类型就是0。

了解这个问题之后，想到第一感觉是对0进行判断，如果发现收到一堆0则将其抛弃。但是有两个问题：

1. 实际上数据还是在管道中流动，会造成空循环，影响性能。
2. 业务上可能存在真实有意义的空值，这个时候0不代表管道关闭。

好在go为`<-chan`操作提供了两个返回值：

```go
item,ok <- chan
```

其中第二个参数就是对channel状态的描述，false表示channnel已经关闭。这就让我们可以通过channel的状态来控制对channel的读取。

可是即便这样再次对channel进行读取还是会读到0，不够优雅。这个时候可以通过nil channel来解决。

思路就是把已关闭的channel置为nil，在读取的时候则优先判断channel是否为nil。
代码就不写了。很简单的实现。

golang对channel的设计，只能说功能强大方便不足吧。很容易就会碰上这样的坑。