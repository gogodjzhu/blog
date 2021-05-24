---
title: "Netty源码解读(8)-关闭服务"
date: 2020-02-03T18:44:33+08:00
description: "直接将进程kill掉？no no no"
featured_image: ""
images: [""]
tags: ["网络", "中间件", "JAVA", "Netty"]
categories: "Netty源码解读"
draft: false
---

Netty服务的关闭涉及以下几种资源:

1. 线程
   1. `EventLoopGroup`线程池
   2. `EventLoop`线程
2. 连接
   1. `EventLoop`管理下的所有`Channel`
   2. Selector
3. 内存
   1. Channel下的各种缓存资源

# 线程的关闭

## EventLoopGroup

关闭是从EventLoopGroup调用shutdown相关方法开始的，优雅关闭的方法为：

```java
# EventExecutorGroup.java EventLoopGroup 实现了 此接口

Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit);
```

之所以称之为优雅，是此方要求关闭前必须满足在静默时间`quietPeriod`(单位:秒)内无新的task提交到`EventLoopGroup`。为了防止无限期等待，还设置了超时时间`timeout`。若不提供这两个值，则使用默认值`quietPeriod=2, timeout=15, unit=TimeUnit.Second`。

下面开始上代码，首先是`NioEventLoopGroup`在父类`MultithreadEventExecutorGroup`实现了`shutdownGracefully()`方法:

```java
# MultithreadEventExecutorGroup.java

public Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit) {
    for (EventExecutor l: children) {
        l.shutdownGracefully(quietPeriod, timeout, unit);
    }
    return terminationFuture();
}
```

作为线程池，本身不维护什么信息，任务都是分给具体的线程`EventExecutor`去执行，这里的`EventExecutor`(接口)实际就是`NioEventLoop`(类)实例，后者实现前者。

这里不贴`NioEventLoop`的`shutdownGracefully()`的方法，简单概括就是将`NioEventLoop`的状态变量修改为`ST_SHUTTING_DOWN`，等待线程方法自己判断状态变化后进行关闭。

## EventLoop

```java
# NioEventLoop.java

protected void run() {
    for (;;) {
        // 业务逻辑，包括处理SelectionKey事件
        try {...} catch (Throwable t) {...}
        try {
            // 判断正在结束此workerThread, 跳出循环
            if (isShuttingDown()) {
                // 关闭
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}

private void closeAll() {
    // 触发一次selector#selectNow方法，获取已接收但未处理的事件
    selectAgain();
    Set<SelectionKey> keys = selector.keys();
    Collection<AbstractNioChannel> channels = new ArrayList<AbstractNioChannel>(keys.size());
    
    // 当前是NioEventLoop, 仅处理NioChannel的关闭
    for (SelectionKey k: keys) {
        Object a = k.attachment();
        if (a instanceof AbstractNioChannel) {
            channels.add((AbstractNioChannel) a);
        } else {
            k.cancel();
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            invokeChannelUnregistered(task, k, null);
        }
    }

    for (AbstractNioChannel ch: channels) {
        // 实际执行channel关闭
        ch.unsafe().close(ch.unsafe().voidPromise());
    }
}
```

在`NioEventLoop`的run()方法内，每次循环都会判断状态，若为关闭，

- 先执行`closeAll()`方法，获取Selector注册的所有SelectionKey，从后者中提取出Channel，执行关闭。关闭方法跟[断开连接](https://www.gogodjzhu.com/202001/netty-8-disconnect-channel)所用的方法一致，不再赘述。但在线程中可能还有别的任务在执行，所以需要优雅关闭。
- `confirmShutdown()`判断是否完成关闭，内部是优雅关闭的逻辑。超时或者完成关闭，返回true，NioEventLoop关闭完成，退出循环。否则会再次进入run方法内的循环。

### 优雅关闭

```java
protected boolean confirmShutdown() {
    // (省略) 状态判断

    // 取消所有定时任务的执行
    cancelScheduledTasks();

    if (gracefulShutdownStartTime == 0) {
        // 记录当前相对时间戳, 整个netty进程启动到今的相对时间
        gracefulShutdownStartTime = ScheduledFutureTask.nanoTime();
    }

    // 尝试执行队列中的task和shutdownHooks，
    // 如果执行了新的任务会更新this#lastExecutionTime的值，并返回true
    if (runAllTasks() || runShutdownHooks()) {
        if (isShutdown()) {
            // Executor shut down - no new tasks anymore.
            return true;
        }
        if (gracefulShutdownQuietPeriod == 0) {
            return true;
        }
        wakeup(true);
        return false;
    }

    final long nanoTime = ScheduledFutureTask.nanoTime();

    // 已经关闭 或者 超时
    if (isShutdown() || nanoTime - gracefulShutdownStartTime > gracefulShutdownTimeout) {
        return true;
    }

    // 在静默时间内有任务被执行，故返回false，本轮优雅关闭失败
    if (nanoTime - lastExecutionTime <= gracefulShutdownQuietPeriod) {
        wakeup(true);
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            // Ignore
        }

        return false;
    }

    // 现在可以判断在静默时间段内，无新的任务执行过，返回true，现在可以放心关闭服务了
    return true;
}
```

![img](http://minio.gogodjzhu.com/images/20210418_173934_d7fb8f0e-246a-4baf-a376-b2418016d2cf.png)



# 总结

本章给你介绍了Netty关闭服务的一些内容，由于Netty工作在React线程模型中，所以关闭服务也就是把`worker`和`boss`线程池关闭。

每个线程池的关闭，都落地为线程本身，也即`NioEventLoop`在判断线程状态为`ST_SHUTTING_DOWN`之后主动地优雅关闭。

优雅关闭的第一步是关闭连接，其次是反复地尝试`优雅关闭`直到超时。