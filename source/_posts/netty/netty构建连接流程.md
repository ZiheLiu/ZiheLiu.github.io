---
title: Netty 构建连接
date: 2020-03-09 20:21:00
categories:
	- netty
tags:
	- netty
---

# 引用

- [源码剖析：构建连接](https://time.geekbang.org/course/detail/237-158156)

# 主线

- boss thread
  - NioEventLoop 中的 selector 轮询创建连接事件（OP_ACCEPT）。
  - 创建 socket channel。
  - 初始化 socket channel 并从 worker group 中选择一个 NioEventLoop。

- worker thread
  - 将 socket channel 注册到选择的 NioEventLoop 的 selector。
  - 注册读事件（OP_READ）到 selector 上。

# 创建 SocketChannel

NioEventLoop 的**死循环**中，接收到 `OP_ACCEPT` 事件后，调用 `unsafe.read()`，最终调用 `serverSocketChannel.accept()` 得到 `socketChannel`。

- `unsafe.read()` 有两种
  - `NioMessageUnsafe#read()` 处理 OP_ACCEPT 事件。这里是调用这个。
  - `NioByteUnsafe#read()` 处理 OP_READ 事件。

# 注册 SocketChannel

## 传播 SocketChannel 到 handler 链中

把创建的 `socketChannel` 放到 `readBuf` 中。通过 `pipeline.fireChannelRead(readBuf)` 把 `socketChannel` 传播到 handler 链中。

```java
readBuf.add(new NioSocketChannel(this, ch));
pipeline.fireChannelRead(readBuf.get(i));
```

## ServerBootstrapAcceptor

```java
// ServerBootstrapAcceptor 在 ServerSocketChannel 初始化时，
// 通过 ChannelInitializer 加入到 pipeline 中的。
p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(final Channel ch) throws Exception {
        final ChannelPipeline pipeline = ch.pipeline();
        ChannelHandler handler = config.handler();
        if (handler != null) {
            pipeline.addLast(handler);
        }

        ch.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }
});
```

其中 `ServerBootstrapAcceptor` 完成主要工作。

把刚刚创建的 `SocketChannel` 注册到 `childGroup` 中的一个 `EventLoop` 上。

注册好之后，调用 `pipeline.fireChannelActive()` 传播事件。

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

# 注册 OP_READ 事件

和 {%post_link netty/netty服务端启动 %} 类似，最终调用 `AbstractNioChannel#doBeginRead()` 把 `readInterestOp` 注册到 `Selector` 上。

而 `NioSocketChannel` 在构造函数中传递了 `readInterestOp` 为  `OP_READ`。

```java
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    super(parent, ch, SelectionKey.OP_READ);
}
```

# 传播的事件

注意，这里的 `pipeline` 是 `ServerSocketChannel.pipeline`。

1. 由 `unsafe.read() ` accept 得到 `socketChannel`。`pipeline.fireChannelRead()` 传播创建的 `socketChannel`。
2. 由 `ServerBootstrapAcceptor#channelRead()` 把 `socketChannel` 注册到一个 `WorkerEventLoop` 中。`pipeline.fireChannelActive()` 传播出去。
3. 由 `head.channelActive()`。调用 `channel.read()` 从tail 开始传播。
4. 由 `head.read()` 注册 `OP_READ` 。

# 接受连接的本质

- selector.select()/selectNow()/select(timeoutMillis) 发现 OP_ACCEPT 事件，处理：
  - SocketChannel socketChannel = serverSocketChannel.accept()
  - selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this)
  - selectionKey.interestOps(OP_READ)

# 总结

- 创建连接的初始化和注册是通过 pipeline.fireChannelRead 在 ServerBootstrapAcceptor 中完成的。

- 第一次 Register 并不是监听 OP_READ ，而是 0 ：

  ```java
  selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this)
  ```

- 最终监听 OP_READ 是通过注册完成后的 `fireChannelActive()`，在 pipeline 的 head 中完成注册 OP_READ 事件的。

- Worker’s NioEventLoop 是通过 Register 操作执行来启动。

- 接受连接的读操作，不会尝试读取更多次（16次）。