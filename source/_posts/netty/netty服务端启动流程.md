---
title: Netty 服务端启动流程
date: 2020-03-08 11:18:00
categories:
	- netty
tags:
	- netty
---

# 引用

- [源码剖析：启动服务](https://time.geekbang.org/course/detail/237-158209)

# 主线

- our thread
  - 创建 selector。
  - 创建 server socket channel。
  - 初始化 server socket channel。
  - 给 server socket channel 从 boss group 中选择一个 NioEventLoop。

- boss thread 
  - 将 server socket channel 注册到选择的 NioEventLoop 的 selector。
  - 绑定地址启动 NioEventLoop 的线程来监听事件。
  - 注册接受连接事件（OP_ACCEPT）到 selector 上。

# 创建 Selector

创建 `NioEventLoopGroup` 会创建 `nThreads` 个 `NioEventLoop`。

每个 `NioEventLoop` 会

- 创建、存储一个 `Selector`。
- 在调用 `EventLoop#execute()` 时，如果线程没有启动，就启动线程。

```java
Selector selector = sun.nio.SelectorProviderImpl.opSelector();
```

# 创建和初始化 NioServerSocketChannel

## 创建

用泛型反射工厂方法，创建 `NioServerSocketChannel`。

```java
channel = channelFactory.newChannel();
```

构造函数内，会创建 `pipeline`。`pipeline` 初始会有首尾节点。

```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```

## 初始化

- 初始化 channel 的属性。

- 记录 `NioSocketChannel` 的属性。

- 为 pipeline 在 `initChannel` 被调用时，添加 `ServerBootstrapAcceptor` handler。
  
  - 负责接收客户端连接，创建连接后，对连接的初始化工作。
  
    ```java
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

# 注册 NioServerSocketChannel

注册到 `NioEventLoop` 的 `Selector` 上。

- 提交注册任务到 `NioEventLoop#execute()`。
- 把 `NioServerSocketChannel` 注册到 `Selector` 上，并指定  `attachment` 回调函数是自己（`Selector` 有事件到达时，执行 `attachment`）。

```java
// class AbstractNioChannel
// 注意，这里绑定的是 0 事件，只是为了获取到 selectionKey，在之后多态更新。
selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
```

提交注册任务后，`NioEventLoop` 的线程会启动起来。

```java
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    addTask(task);
    if (!inEventLoop) {
      	// 启动线程！！！
        startThread();
        if (isShutdown() && removeTask(task)) {
            reject();
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
```

# 绑定地址

`channel.bind()` 调用 `pipeline.bind()`调用 `tail.bind()`。即从 `tail` 向前传递。

在 `head` 这个 `ChannelHandlerContext#bind()` 方法中，执行 `unsafe.bind()`。

```java
public void bind(ChannelHandlerContext ctx, 
                 SocketAddress localAddress, 
                 ChannelPromise promise) throws Exception {
    unsafe.bind(localAddress, promise);
}
```

`unsafe.bind()` 最终调用 `javaChannel().bind()` 来完成地址的绑定。

```java
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

# 注册 OP_ACCEPT 事件

绑定地址后，调用 `pipeline.fireChannelActive()`，从 `pipeline.head` 开始执行 `channelActive()`。

其调用 `readIfIsAutoRead()` 调用 `channel.read()`，从 `pipeline.tail` 开始执行 `read()`。

`head.read()` 方法执行 `unsafe.beginRead()`。

```java
public void read(ChannelHandlerContext ctx) {
    unsafe.beginRead();
}
```

最终调用 `AbstractNioChannel#doBeginRead()` 把 `readInterestOp` 注册到 `Selector` 上。

而 `NioServerSocketChannel` 是 `AbstractNioChannel` 的子类，在构造函数中传递了 `readInterestOp` 为  `OP_ACCEPT`。

```java
// AbstractNioChannel#doBeginRead()
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
// class NioServerSocketChannel
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```