---
title: Netty 接收数据流程
date: 2020-03-09 23:21:00
categories:
	- netty
tags:
	- netty
typora-root-url: ../../
---

# 引用

- [源码剖析：接收数据](https://time.geekbang.org/course/detail/237-158157)

# 触发读操作

当读事件发生时，WorkerEventLoop 调用 `unsafe.read()`，这里是  `NioSocketChannel. NioByteUnsafe#read()`。

# 循环读取

循环读取，根据 `continueReading` 判断是否还要继续读取。

- 每次读取后，调用 `pipeline.fireChannelRead(byteBuf)` 把读到的数据传播到链中。
- 读取结束后，调用 `pipeline.fireChannelReadComplete()` 传播读取完成事件。

```java
// NioByteUnsafe#read()
public final void read() {
    // 省略
    do {
      byteBuf = allocHandle.allocate(allocator);
      allocHandle.lastBytesRead(doReadBytes(byteBuf));
      // 省略 ...
      pipeline.fireChannelRead(byteBuf);
    } while (allocHandle.continueReading());
  	
    allocHandle.readComplete();
    pipeline.fireChannelReadComplete();
    // 省略 ...
}
```

## 分配 ByteBuf

> [Netty 源码解析 ——— AdaptiveRecvByteBufAllocator](https://www.jianshu.com/p/e12c7f3861e1)

会根据之前接收数据的大小，来调整本次分配 ByteBuf 的大小。

- 初始大小 1024 字节。
- 如果连续两次缓冲区没有填满，则减小缓冲区大小。
- 如果上次缓冲区填满了，则增大缓冲区大小。

增大和减小的规则如下：

```java
// 上次缓冲区大小为 SIZE_TABLE[i]，本次为 SIZE_TABLE[i + INDEX_INCREMENT]。
private static final int INDEX_INCREMENT = 4;    

// 上次缓冲区大小为 SIZE_TABLE[i]，本次为 SIZE_TABLE[i - INDEX_DECREMENT]。
private static final int INDEX_DECREMENT = 1;    

// 用于存储缓冲区容量大小的数组。
private static final int[] SIZE_TABLE;   

// [16, 512) 中 16 的倍数。
// [512, MAX_INT) 中 512 * (2^N)。
static {
  List<Integer> sizeTable = new ArrayList<Integer>();
  for (int i = 16; i < 512; i += 16) {
    sizeTable.add(i);
  }

  for (int i = 512; i > 0; i <<= 1) {
    sizeTable.add(i);
  }

  SIZE_TABLE = new int[sizeTable.size()];
  for (int i = 0; i < SIZE_TABLE.length; i ++) {
    SIZE_TABLE[i] = sizeTable.get(i);
  }
}
```

## continueReading()

根据 `continueReading()` 最多只会读取 16 次。那么如果读取 16 次后，还有数据怎么办？分为 LT 和 ET 两种模式。

```java
// DefaultMaxMessagesRecvByteBufAllocator#continueReading()
public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
    return config.isAutoRead() &&
           (!respectMaybeMoreData || maybeMoreDataSupplier.get()) && // 没有数据了
           totalMessages < maxMessagePerRead && // 默认 16
           totalBytesRead > 0; // 本次读到数据
}
```

### Epoll LT 模式

由于只要 socket 还有数据可读，就会触发 EPOLLIN 事件，所以不用额外做事情。

下一次 `select()`，就会被 EPOLLIN 唤醒，继续读取数据。

### Epoll ET 模式

对于 Epoll ET 模式，只有从不可读变为可读才会触发，如果只读取 16 次，剩下的数据将没有办法继续读取。

所以 `EpollSocketChannel` 会在时 Epoll ET 模式时，如果数据没有读取完，会把读取任务重新添加到 `EventLoop` 中。

相比 Epoll LT，少了一次 `select()` 的系统调用。

![EpollRecvByteAllocatorHandle](/images/EpollRecvByteAllocatorHandle-3767276.png)

如下，在 `AbstractEpollChannel#epollInFinally()` 中，如果是 EPOLLET 模式、且还有数据没有读完，会提交一个 调用 `epollInReady()` 的任务到 `EventLoop` 中。

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

## fireChannelRead()

调用 `pipeline.fireChannelRead(byteBuf)` 后，会从 `head` 开始调用它的 `ChannelHandler# channelRead(byteBuf)` 方法。

在 `ChannelHandler#channelRead(byteBuf)` 方法中，可以调用 `ChannelHandler# fireChannelRead(obj)` 方法，把数据传递给下一个 `InBoundHandler`。

这样就形成了，从 `pipeline` 的 `head` 开始到 `tail` 的 `InBoundHandler` 上的 `channelRead(obj)` 的传播。

![netty-pipeline](/images/netty-pipeline.png)