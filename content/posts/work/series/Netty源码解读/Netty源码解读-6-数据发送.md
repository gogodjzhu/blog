---
title: "Netty源码解读(5)-数据处理(Handler调用链)"
date: 2020-02-01T18:22:44+08:00
description: "Netty框架将网络处理的场景抽象为一系列责任链模式设计的Handler，基于Netty实现业务逻辑的本质是编排Handler和重写对应的Handler方法。本章我们就来研究下这个Netty设计的精髓。"
tags: ["网络", "中间件", "JAVA", "Netty"]
categories: "Netty源码解读"
draft: false
---

netty发送数据使用的接口方法是`ChannelOutboundInvoker#write()`，它共有三个主要的实现方案：

```java
// 1) 
AbstractChannel#write(Object)
// 2) 
AbstractChannelHandlerContext#write(Object)
// 3) 
DefaultChannelPipeline#write(Object)
```

即可以在Channel, HandlerContext, Pipeline三个级别实行发送。但实际的发送逻辑只有两种类型：

1. 从`Pipeline`的`TailContext`开始发送，即不会触发链上的下游的出站Handler的`write()`方法 (方案1和方案3)
2. 从当前`HandlerContext`开始调用write方法，会依次调用Pipeline链上的下游出站Handler的`write()`方法 (方案2)

大多数业务还以方案2为主，下面我将基于此方案来展开。



# write()方法

write方法支持`ChannelPromise`来实现异步回调：

```java
# ChannelFuture.java
public ChannelFuture write(final Object msg, final ChannelPromise promise) {
    // (省略) msg和promise合法性校验
    write(msg, false, promise);

    return promise;
}

private void write(Object msg, boolean flush, ChannelPromise promise) {
    // 获取下游Context
    AbstractChannelHandlerContext next = findContextOutbound();
    // 引用计数, 用于内存泄露检测
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        // 在当前线程同步调用
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        // 封装成Task在EventLoop线程异步调用
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}

private AbstractChannelHandlerContext findContextOutbound() {
    AbstractChannelHandlerContext ctx = this; // 从当前context开始遍历
    do {
        ctx = ctx.prev;
    } while (!ctx.outbound); // 跳过所有非Outbound的handler
    return ctx;
}
```

关于Flush后文再做介绍，这里先介绍单纯的write操作。可以看到调用`HandlerContext`的write()方法，会通过`findContextOutbound()`寻找Pipeline链上的前一个(prev)出站Handler，即跟read()是正好相反的方向。而非出站Handler会被忽略跳过。

找到Handler之后，下一步就是调用其`write()`方法。对于在Pipeline中间层级的业务Handler，在对输入的Object进行修改和封装后，需要重新通过`HandlerContext#write(Object)`方法交给下级Handler让数据在Pipeline上继续流动。直到到达Pipeline尾部，即HeadContext(因为出站数据是从tail到head流动的)通过`unsafe`写出：

```java
# DefaultChannelPipeline$HeadContext.java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}

# AbstractChannel$AbstractUnsafe.java
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop(); // 必须在EventLoop线程中执行

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        // (省略) 异常处理
    }

    int size;
    try {
        // 将msg进行过滤和修饰, 默认无处理, 由子类Channel实现
        msg = filterOutboundMessage(msg);
        // 获取当前待发送的消息的大小, 对于直接发送文件的返回0(支持零拷贝)
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        // (省略) 异常处理
        return;
    }

    outboundBuffer.addMessage(msg, size, promise);
}
```

`Channel$Unsafe#write()`方法只有`AbstractChannel`实现了，奇怪么？不是说Unsafe是跟具体的实现有关么？原来这里的write方法还未真正触发IO写出，仅仅将`msg:Object`放入`outbundBuffer`，这在所有的Channel实现中都是统一的，所以只在AbstractChannel做了实现。

接下来是`msg = filterOutboundMessage(msg)`方法，留给子类实现，在我们讨论的NioSocketChannel中，将堆内存转化为堆外内存（*准确地说是转化为池化的堆外内存，非池化的堆外内存代价太高，得不偿失*），这样可以利用零拷贝提高性能。

```java
# AbstractNioByteChannel.java
protected final Object filterOutboundMessage(Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (buf.isDirect()) {
            return msg;
        }
        // 堆内Buffer转堆外Buffer
        return newDirectBuffer(buf);
    }

    if (msg instanceof FileRegion) {
        // 
        return msg;
    }

    throw new UnsupportedOperationException(
            "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
}
```

接下来就进入发送数据的重头戏，缓存。



## ChannelOutboundBuffer

此Buffer是netty专门提供用于出站数据缓存的容器，内部的数据结构是三个链表。

![img](http://minio.gogodjzhu.com/images/20210418_170047_627be685-b9a1-4f52-ac63-a92b62eb97f4.png)



而填充这些链表的Entry对象，基于可重复使用的对象池来创建，进一步减少了性能消耗。这是netty提供的很有意思的一个设计。

```java
# ChannelOutboundBuffer
public void addMessage(Object msg, int size, ChannelPromise promise) {
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
        tailEntry = entry;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
        tailEntry = entry;
    }
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }
    // 累计写入buffer的数据大小, 超过阈值修改buffer状态为不可写入
    incrementPendingOutboundBytes(size, false);
}

# ChannelOutboundBuffer$Entry.class
static Entry newInstance(Object msg, int size, long total, ChannelPromise promise) {
    // 从线程对象池中获取entry
    Entry entry = RECYCLER.get();
    // (省略) entry赋值初始化
    return entry;
}
```

在`increamentPendingOutboundBytes()`方法中会更新缓存中总的数据量大小，如果超过高水位(通过`WriteBufferWaterMark#high`配置, 默认64k)，会修改`ChannelOutboundBuffer`的状态为不可写。同时还会触发一个Pipeline事件`fireChannelWritabilityChanged`。



# flush()方法

数据通过write方法只是写入到`ChannelOutboundBuffer`做了缓存，真正的数据发送需要通过flush。跟`write()`的调用逻辑类似，经过HandlerContext或者Channel实现的`wtite()`，所有flush方法实现最终会调用AbstractUnsafe的实现：

```java
# AbstractChannel$AbstractUnsafe.java
protected void flush0() {
    if (inFlush0) { return; }

    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null || outboundBuffer.isEmpty()) { return; }

    inFlush0 = true;

    if (!isActive()) {
        // (省略) 如果当前channel已关闭, 所有flushed内的Entry作失败处理
        return;
    }

    try {
        doWrite(outboundBuffer);
    } catch (Throwable t) {
        // (省略) 异常处理
    } finally {
        inFlush0 = false;
    }
}

# NioSocketChannel.java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    for (;;) {
        // flushed链表剩余元素的数量
        int size = in.size();
        if (size == 0) {
            clearOpWrite(); // flushed链表发送结束,清空SelectionKey#OP_WRITE
            break;
        }
        long writtenBytes = 0;
        boolean done = false;
        boolean setOpWrite = false;

        // buffer.flushedEntry链表转为nioByteBuffer数组, 仅仅转换in中类型为ByteBuf的元素
        ByteBuffer[] nioBuffers = in.nioBuffers();
        // 待发送的buffer数, 即nioBuffers数组元素数量
        int nioBufferCnt = in.nioBufferCount();
        // 待发送数据的字节数, 即nioBuffers所有元素总字节数
        long expectedWrittenBytes = in.nioBufferSize();
        SocketChannel ch = javaChannel();

        switch (nioBufferCnt) {
            case 0:
                // 由于in中可能包括非ByteBuf类型的元素(比如直接发送文件的FileRegion类型),nioBuffers()方法
                // 不会将其放入数组, 以致cnt为0. 本方法仅处理nioByteBuffer类型的发送, 交给父类处理
                super.doWrite(in);
                return;
            case 1:// 单个buffer使用
                // 只做有限次(默认16)循环, 防止单个连接占用太多资源
                ByteBuffer nioBuffer = nioBuffers[0];
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                    // nio SocketChannel发送 nio ByteBuffer, 返回实际发送的字节数(大小取决于tcp协议). Socket非阻塞默认(一般都是)下返回0表示无数据发送
                    final int localWrittenBytes = ch.write(nioBuffer);
                    if (localWrittenBytes == 0) {
                        // nioBuffer有数据, 但socket发送出去的字节数为0, 设置setOpWrite=true, 退出发送循环, 且后续不会马上新增task继续发送, 而是注册OP_WRITE等待os可以发送更多数据的时候继续发送
                        setOpWrite = true;
                        break;
                    }
                    expectedWrittenBytes -= localWrittenBytes;
                    writtenBytes += localWrittenBytes;
                    if (expectedWrittenBytes == 0) { // 发送完成
                        done = true;
                        break;
                    }
                }
                break;
            default:// 对于多个Buffer使用
                // 只做有限次(默认16)循环, 防止单个连接占用太多资源
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                    // 使用 gathering writes 发送多个NIO ByteBuffer, 返回实际发送的字节数(大小取决于tcp协议)
                    final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                    if (localWrittenBytes == 0) {
                        setOpWrite = true;
                        break;
                    }
                    expectedWrittenBytes -= localWrittenBytes;
                    writtenBytes += localWrittenBytes;
                    if (expectedWrittenBytes == 0) {
                        done = true;
                        break;
                    }
                }
                break;
        }

        // 更新处理进度
        in.removeBytes(writtenBytes);

        if (!done) {
            // Did not write all buffers completely.
            // 未完成flushed队列中所有的buffer的发送会进入此方法
            // setOpWrite=true,因为socket的问题本次未写入任何数据,是系统原因故注册SelectionKey#OP_WRITE事件,等待系统可写入再写
            // setOpWrite=false,表示此次socket发送了部分数据,认为还可以马上发送更多的数据,这里直接添加task来异步执行新的写入
            incompleteWrite(setOpWrite);
            break;
        }
    }
}
```

`NioSocketChannel#doWrite()`方法值得进行详细分析。

首先是netty有一个`writeSpinCount`配置，控制的是单个连接连续发送数据的最大次数，默认为16。这个配置有效地平衡了单个连接批量发送 和 连接之间的发送平衡。

其次，在16次发送都未完成整个OutboundBuffer的发送时`if(!done)`，方法最后会调用`incompleteWrite`来触发下一次发送，发送的方式有两种，取决于入参`setOpWrite`。若为true，那么本次发送不完全的原因是`socket`发送速率太慢导致，那么先停止发送，注册监听`OP_WRITE`方法等待os通知。若为false，则未完成发送是单纯因为16次的发送限制，此时向当前EventLoop增加一个发送Task排队继续执行即可。

```java
# AbstractNioByteChannel.java
protected final void incompleteWrite(boolean setOpWrite) {
    // Did not write completely.
    if (setOpWrite) {
        // 外层调用认为当前socket不能写入, 停止写入, 注册OP_WRITE事件等待os通知
        setOpWrite();
    } else {
        // Schedule flush again later so other tasks can be picked up in the meantime
        // 外层调用认为当前socket还可以写入更多数据, 新增一个task异步继续发送
        Runnable flushTask = this.flushTask;
        if (flushTask == null) {
            flushTask = this.flushTask = new Runnable() {
                @Override
                public void run() {
                    flush();
                }
            };
        }
        eventLoop().execute(flushTask);
    }
}
```



# 总结

这次我们介绍了netty发送数据的流程。分为两个步骤：write和flush。

write的本质是将数据写入各种IO模式Channel统一的缓存结构`ChannelOutboundBuffer`，由三个链表构成的缓存接口一方面尽可能把待发送的数据批量发出，另一方面也在必要的时候将非堆外内存实现的ByteBuf转化为堆外的ByteBuf以利用零拷贝的优势。

write之后最终的发送动作需要等到flush触发才执行。flush发送数据的过程会兼顾`单连接尽可能一次发送更多的数据` 和 `所有连接相对平衡地拥有发送的机会`。