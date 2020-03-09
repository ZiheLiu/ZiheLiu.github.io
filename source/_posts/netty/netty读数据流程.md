---
title: Netty 读数据流程
date: 2020-03-09 23:21:00
categories:
	- netty
tags:
	- netty
typora-root-url: ../../
---

# 读缓存

当读事件发生时，WorkerEventLoop 调用 `unsafe.read()`（`NioByteUnsafe#read()`）。

根据 `continueReading()` 最多只会读取 16 次。

```java
// NioByteUnsafe#read()
public final void read() {
    // 省略
    do {
      byteBuf = allocHandle.allocate(allocator);
      allocHandle.lastBytesRead(doReadBytes(byteBuf));
      // 省略 ...
    } while (allocHandle.continueReading());
    // 省略 ...
}

// DefaultMaxMessagesRecvByteBufAllocator#continueReading()
public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
    return config.isAutoRead() &&
           (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&
           totalMessages < maxMessagePerRead && // 默认 16
           totalBytesRead > 0;
}
```

## Epoll ET 模式

对于 Epoll ET 模式，只有从不可读变为可读才会触发，如果只读取 16 次，剩下的数据将没有办法继续读取。

所以 `EpollSocketChannel` 会在时 Epoll ET 模式时，如果数据没有读取完，会重新触发一个 EPOLLIN 事件。

![EpollRecvByteAllocatorHandle](/images/EpollRecvByteAllocatorHandle-3767276.png)

如下，在 `AbstractEpollChannel#epollInFinally()` 中，如果是 EPOLLET 模式、且还有数据没有读完，会再触发一次 EPOLLIN 事件。

![image-20200309232504893](/images/image-20200309232504893.png)

```java
final void executeEpollInReadyRunnable(ChannelConfig config) {
    if (epollInReadyRunnablePending || !isActive() || shouldBreakEpollInReady(config)) {
        return;
    }
    epollInReadyRunnablePending = true;
    // 提交任务。任务中触发 epollInReady()。
    eventLoop().execute(epollInReadyRunnable);
}

private final Runnable epollInReadyRunnable = new Runnable() {
    @Override
    public void run() {
        epollInReadyRunnablePending = false;
        epollInReady();
    }
};
```