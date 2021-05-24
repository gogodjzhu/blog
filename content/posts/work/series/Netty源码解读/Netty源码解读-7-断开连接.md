---
title: "Netty源码解读(7)-断开链接"
date: 2020-02-02T18:33:57+08:00
description: ""
featured_image: "在完成了前一章的数据发送之后，顺理成章地需要引出链接断开——毕竟通常我们的底层都是面向连接的。而链接断开并不是简单的将进程杀死，优雅地断链本质是对一系列资源的回收。"
images: [""]
tags: ["网络", "中间件", "JAVA", "Netty"]
categories: "Netty源码解读"
draft: false
---

Netty连接的断开本质上是channel的断开，更具体的(nio)`ServerSocketChannel`和(nio)`SocketChannel`的断开，这在jdk的api中定义了方法`AbstractInterruptibleChannel#close()`。这是netty对一个连接的处理的结束。在执行关闭之前，还需要保证对资源的有序释放，包括：

1. 释放TCP连接的端口占用
2. 内存回收（尤其是堆外内存的回收）
3. 其他业务代码占用的资源（如数据库连接）

断开链接可以分为两种类型，主动和被动，下面分别说明。



# 主动断开

主动的断开连接是一个出站事件，`close()`方法定义在`ChannelOutboundInvoker`中，因此跟`write()`，`connect()`方法类似，它也有三种实现，而且实现也类似：

- `channel#close()` 调用`pipeline#close()`
- `pipeline#close()`调用pipeline中的`tailContext#close()`
- `context#close()`是实际执行关闭的方法，下面着重讨论

```java
# AbstractChannelHandlerContext.java
public ChannelFuture close(final ChannelPromise promise) {
    if (!validatePromise(promise, false)) {
        // cancelled
        return promise;
    }
    // 以当前context为锚点, 找prev节点
    final AbstractChannelHandlerContext next = findContextOutbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeClose(promise);
    } else {
        safeExecute(executor, new Runnable() {
            @Override
            public void run() {
                next.invokeClose(promise);
            }
        }, promise, null);
    }

    return promise;
}

# AbstractChannelHandlerContext.java
private void invokeClose(ChannelPromise promise) {
    if (invokeHandler()) {
        try {
            /** {@link #handler()}方法返回当前context绑定的handler */
            ((ChannelOutboundHandler) handler()).close(this, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    } else {
        close(promise);
    }
}
```

看过系列文章的你应该非常熟悉这些代码的套路了，当前`context`调用close()方法之后通过`findContextOutbound()`方法寻找Pipeline链表上的下一个出站Context，并调用其绑定的Handler的close方法。直到出站方向的最后一个context，也就是`HeadContext`，它的方法是这样写的：

```java
# DefaultChannelPipeline$HeadContext.java
@Override
public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
    unsafe.close(promise);
}

# AbstractChannel.AbstractUnsafe.java
public final void close(final ChannelPromise promise) {
    assertEventLoop();
    // 两个EXCEPTION参数是为专门为关闭方法提供的特殊异常
    close(promise, CLOSE_CLOSED_CHANNEL_EXCEPTION, CLOSE_CLOSED_CHANNEL_EXCEPTION, false);
}

# AbstractChannel.AbstractUnsafe.java
private void close(final ChannelPromise promise, final Throwable cause,
                   final ClosedChannelException closeCause, final boolean notify) {
    if (!promise.setUncancellable()) {
        return;
    }

    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;

    if (closeFuture.isDone()) {
        // Closed already.
        safeSetSuccess(promise);
        return;
    }

    final boolean wasActive = isActive();
    // 先设置为null，防止更多的写入
    this.outboundBuffer = null; // Disallow adding any messages and flushes to outboundBuffer.
    // 正式关闭前的预备方法，若执行了预备方法，那么这里返回一个Executor，剩下的关闭操作就必须在此Executor中执行
    // 这样可以保证关闭方法不被阻塞，同时又保证了prepare方法正确执行
    // 目前主要的实现由NioSocketChannel提供，对SO_LINGER优雅关闭导致无法关闭的问题，提供先register再关闭的策略
    //
    // SO_LINGER优雅关闭的问题:
    // 如果SO_LINGER配置了，close()方法会阻塞直到
    //   1.没有新的数据需要读写或者
    //   2.超时
    // 无论是哪种情况都会导致当前EventLoop的阻塞
    // 这会导致EventLoop无法处理其他连接事务。所以这里我们判断如果开启了SO_LINGER则把关闭操作放到一个独立的线程中去处理
    Executor closeExecutor = prepareToClose();
    if (closeExecutor != null) {
        closeExecutor.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    // Execute the close.
                    doClose0(promise);
                } finally {
                    // Call invokeLater so closeAndDeregister is executed in the EventLoop again!
                    invokeLater(new Runnable() {
                        @Override
                        public void run() {
                            // Fail all the queued messages
                            outboundBuffer.failFlushed(cause, notify);
                            outboundBuffer.close(closeCause);
                            fireChannelInactiveAndDeregister(wasActive);
                        }
                    });
                }
            }
        });
    } else {
        try {
            // Close the channel and fail the queued messages in all cases.
            doClose0(promise);
        } finally {
            // Fail all the queued messages.
            outboundBuffer.failFlushed(cause, notify); // failed掉outboundBuffer中未发送(flushed)的数据
            outboundBuffer.close(closeCause); // 关闭outboundBuffer, 并移除unflushed数据(不同于flushed，unflused还需要回收资源)
        }
        if (inFlush0) {
            // 正在处理flush操作，需要在当前eventLoop中排队等待
            invokeLater(new Runnable() {
                @Override
                public void run() {
                    // deregister即cancel掉SelectionKey
                    // channel先后触发的状态是: inactive->unregistered
                    fireChannelInactiveAndDeregister(wasActive);
                }
            });
        } else {
            // deregister即cancel掉SelectionKey
            // channel先后触发的状态是: inactive->unregistered
            fireChannelInactiveAndDeregister(wasActive);
        }
    }
}
```

`HeadContext`关闭连接的过程是：

1. 将当前`channel`绑定的`ChannelOutboundBuffer`置为null，阻止新的write和flush数据
2. 关闭`nioChannel`
3. failed掉`ChannelOutboundBuffer`中的所有flushed/unflused数据（会触发对应的`ChannelPromise`的listener回调）
4. 关闭channel绑定的`SelectionKey`
5. 调用`Pipeline#fireChannelInactive()`方法，`inactive`是入站事件。从`head`开始链式调用Pipeline上的InboundHandler的`channelInactive`方法。
6. 调用`Pipeline#fireChannelUnregistered()`方法，跟inactive一样也是入站事件。不再赘述。

以上是主动关闭连接的调用过程。下面讨论被动断开连接的情况。



# 被动断开



## 对端显式关闭

先讲本质。连接被动断开跟主动断开不同，是通过`OP_READ`事件触发消费的字节数是否为-1进行判断的，直接看javadoc原文：

```java
#ReadableByteChannel.java

/**
* @Return The number of bytes read, possibly zero, or -1 if the channel has reached end-of-stream
*/
public int read(ByteBuffer dst) throws IOException;
```

那么回到Channel对`OP_READ`事件的数据读取逻辑代码，

```java
# AbstractNioByteChannel.NioByteUnsafe#read()

do {
    /** {@link io.netty.channel.AdaptiveRecvByteBufAllocator} */
    byteBuf = allocHandle.allocate(allocator);
    // 执行消费
    allocHandle.lastBytesRead(doReadBytes(byteBuf));
    // 无新消息, 释放buffer
    if (allocHandle.lastBytesRead() <= 0) {
        // nothing was read. release the buffer.
        byteBuf.release();
        byteBuf = null;
        close = allocHandle.lastBytesRead() < 0; // 此次读事件消费的字节数为负, 即连接断开
        break;
    }

    // 增加读入消息数量
    allocHandle.incMessagesRead(1);
    readPending = false;
    // 触发pipeline 消费消息 事件
    pipeline.fireChannelRead(byteBuf);
    byteBuf = null;
} while (allocHandle.continueReading());

// 读取结束后动作, 对于自适应的Allocator会根据此次读取的字节数调整
allocHandle.readComplete();
// 触发pipeline 读取消息完成 事件
pipeline.fireChannelReadComplete();

if (close) {
    // 关闭pipeline
    closeOnRead(pipeline);
}
```

然后关键在`closeOnRead(pipeline)`，正如方法名，这是在read方法中发起的close：

```java
private void closeOnRead(ChannelPipeline pipeline) {
  if (isOpen()) {
    if (Boolean.TRUE.equals(config().getOption(ChannelOption.ALLOW_HALF_CLOSURE))) {
      // 半关闭，这里不做展开解释. 大意就是不再接收读事件，但是还可以给对端写数据
      shutdownInput();
      SelectionKey key = selectionKey();
      key.interestOps(key.interestOps() & ~readInterestOp);
      pipeline.fireUserEventTriggered(ChannelInputShutdownEvent.INSTANCE);
    } else {
      // 调用close方法关闭channel, 此方法开始跟主动关闭连接一致
      close(voidPromise());
    }
  }
}
```

如果不考虑半关闭的条件分支，那么从此方法之后就进入了跟主动关闭相同的方法。



# 总结

本文我给你介绍了Netty断开连接相关的一些信息。可以分为`主动断开`和`被动断开`两种场景，前者是通过`close()`方法主动发起；后者则依靠读取数据的字节数是否为负以判断连接状态。另外我还给你详细分析了两种关闭方式的代码逻辑，大部分是相同的，都涉及到`SelectionKey`，`java.nio.Channel`的关闭，`OutboundHandlerBuffer`的资源回收和关闭，以及`inactive`和`unregistered`事件的触发。