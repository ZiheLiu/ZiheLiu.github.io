---
title: Netty 写数据流程
date: 2020-03-10 18:32:00
categories:
	- netty
tags:
	- netty
typora-root-url: ../../
---

# 总线

- Write操作：写数据到 buffer。
  - 写数据到缓存 `ChannelOutboundBuffer#addMessage()`。
    - 把数据缓存在链表 `unflushedEntry` 上。

- Flush 操作：发送 buffer 里面的数据。
  - `AbstractChannel.AbstractUnsafe#flush()`。
  - 准备数据 `ChannelOutboundBuffer#addFlush()` 
    - 把链表 `unflushedEntry` 上的数据转移到链表 `flushedEntry` 上。
  - 发送数据到 socket `NioSocketChannel#doWrite()`。
    - 循环写入到 socket 中。
      - 如果没有数据要写了，清除 OP_WRITE 事件。
      - 每次写入时，把链表缓存的一部分转化为数组。转化的长度会在每次写入后进行调整。
      - 如果发现写不进去 socket 了，注册一个 OP_WRITE 事件。
      - 如果写了 16 次还有数据，先结束写操作。清除 OP_WRITE，把继续写的任务放到 `EventLoop` 里。

![netty写数据流程](/images/netty写数据流程.png)

# 写数据到 buffer

## 写入方法的调用位置

- `ChannelHandlerContext#write(msg)` 把 `msg` 写入到前一个 `OutboundHandler` 中。
- `ChannelHandlerContext#handler().write(msg)` 把 `msg` 写入到 `pipeline.tail` 中。

最后实际写数据到 buffer，都是在 `pipeline.head.write(msg)` 中 完成。

最终在 `unsafe.write()`中调用了 `ChannelOutboundBuffer#addMessage(msg, size, promise)`。

这也意味着，每个 Channel 会有一个 `outboundBuffer` 缓存。

```java
// pipeline.head.write()
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    unsafe.write(msg, promise);
}
// AbstractChannel.AbstractUnsafe#write()
public final void write(Object msg, ChannelPromise promise) {
    // 省略...
    outboundBuffer.addMessage(msg, size, promise);
}
```

##addMessage()

`addMessage()` 方法把传入的 `msg` 数据存储在缓存中。

缓存是一个链表组成的栈。头结点是 `unflushedEntry`。

注意在 `total()` 方法中，顺便检查了，传入的 `msg` 必须是可写入的类型。

```java
// 写入缓存，是链表组成的栈。
public void addMessage(Object msg, int size, ChannelPromise promise) {
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
    }
    tailEntry = entry;
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }

    // increment pending bytes after adding message to the unflushed arrays.
    // See https://github.com/netty/netty/issues/1619
    incrementPendingOutboundBytes(entry.pendingSize, false);
}
// 传入的 msg 必须是可写入的类型。
private static long total(Object msg) {
    if (msg instanceof ByteBuf) {
        return ((ByteBuf) msg).readableBytes();
    }
    if (msg instanceof FileRegion) {
        return ((FileRegion) msg).count();
    }
    if (msg instanceof ByteBufHolder) {
        return ((ByteBufHolder) msg).content().readableBytes();
    }
    return -1;
}
```

# 发送 buffer 中的数据

## 写入方法的调用位置

与 `write()` 类似，最终调用 `pipeline.head.flush()`。

其调用了 `unsafe.flush()` 来完成写入。

```java
// pipeline.head.write()
public void flush(ChannelHandlerContext ctx) {
    unsafe.flush();
}
// AbstractChannel.AbstractUnsafe#flush()
public final void flush() {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }

    outboundBuffer.addFlush();
    flush0();
}
```

## addFlush()

`outboundBuffer.addFlush()` 把 `unflushedEntry` 上的链表传递给 `flushedEntry`。

```java
public void addFlush() {
    // There is no need to process all entries if there was already a flush before and no new messages
    // where added in the meantime.
    //
    // See https://github.com/netty/netty/issues/2577
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            // there is no flushedEntry yet, so start with the entry
            flushedEntry = entry;
        }
        do {
            flushed ++;
            if (!entry.promise.setUncancellable()) {
                // Was cancelled so make sure we free up memory and notify about the freed bytes
                int pending = entry.cancel();
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);

        // All flushed so reset unflushedEntry
        unflushedEntry = null;
    }
}
```

## flush0()

`flush0()` 最终调用 `doWrite()` 来完成向 socket 写入数据。

循环写入到 socket 中。

- 如果没有数据要写了，清除 OP_WRITE 事件。
- 每次写入时，把链表缓存的一部分转化为数组。转化的长度会在每次写入后进行调整。
- 如果发现写不进去 socket 了，注册一个 OP_WRITE 事件。
- 如果写了 16 次还有数据，先结束写操作。清除 OP_WRITE，把继续写的任务放到 `EventLoop` 里。

对于 EPOLL_ET 模式，Netty 在之后的实现中，不用操作 OP_WRITE 标志。[GitHub issue Never unset EPOLLOUT interest flag](https://github.com/netty/netty/pull/9847)

```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    SocketChannel ch = javaChannel();
    int writeSpinCount = config().getWriteSpinCount();
    do {
        // 如果没有数据要写了，确保清除了 OP_WRITE 事件。
        if (in.isEmpty()) {
            clearOpWrite();
            return;
        }

        int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
        // 把链表缓存的一部分转化为数组。
        ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
        int nioBufferCnt = in.nioBufferCount();

        switch (nioBufferCnt) {
            case 0:
                writeSpinCount -= doWrite0(in);
                break;
            // 只有一块数据要写。
            case 1: {
                ByteBuffer buffer = nioBuffers[0];
                int attemptedBytes = buffer.remaining();
              	// 调用 JDK Nio 的方法。
                final int localWrittenBytes = ch.write(buffer);
                // 如果写不进 socket 了，注册一个 OP_WRITE 事件。在能写进去的时候再写入。
                if (localWrittenBytes <= 0) {
                    incompleteWrite(true);
                    return;
                }
                // 根据写入量调整下次要写入多少个缓存块。
                adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
                in.removeBytes(localWrittenBytes);
                --writeSpinCount;
                break;
            }
            // 有多块数据要写。
            default: {
                long attemptedBytes = in.nioBufferSize();
                // 写入多个 buffer。
                final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                if (localWrittenBytes <= 0) {
                    incompleteWrite(true);
                    return;
                }
                adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes,
                        maxBytesPerGatheringWrite);
                in.removeBytes(localWrittenBytes);
                --writeSpinCount;
                break;
            }
        }
    } while (writeSpinCount > 0);

    
    incompleteWrite(writeSpinCount < 0);
}

protected final void incompleteWrite(boolean setOpWrite) {
    // Did not write completely.
    if (setOpWrite) {
        setOpWrite();
    } else {
        // It is possible that we have set the write OP, woken up by NIO because the socket is writable, and then
        // use our write quantum. In this case we no longer want to set the write OP because the socket is still
        // writable (as far as we know). We will find out next time we attempt to write if the socket is writable
        // and set the write OP if necessary.
        clearOpWrite();

        // Schedule flush again later so other tasks can be picked up in the meantime
        eventLoop().execute(flushTask);
    }
}
```