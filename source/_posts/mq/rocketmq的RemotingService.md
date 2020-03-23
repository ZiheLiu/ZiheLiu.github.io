---
title: RocketMQ 的 RemotingService
date: 2020-03-23 15:37:00
categories:
	- mq
tags:
	- mq
	- rocketmq
typora-root-url: ../../
---

# 介绍

![RemotingService类图](/images/RemotingService类图.svg)

RocketMQ 不同节点之间的交流都是使用 `RemotingService` 相关类实现的。

`NettyRemotingServer` 为使用 Netty 实现的 TCP 服务端。

`NettyRemotingClient` 为使用 Netty 实现的 TCP 客户端。

# NettyRemotingAbstract

## 信号量控制并发写入

使用信号量，在 ASYNC 和 ONEWAY 写入时，控制调用 `channel.writeAndFlush()` 的并发量。

- 在写入前获取信号量。
- 写入完成后，在 `writeAndFlush.addListener()` 的回调函数中释放信号量。

```java
// 默认 256
protected final Semaphore semaphoreOneway;
// 默认 64
protected final Semaphore semaphoreAsync;	
```

## invokeOnewayImpl()

1. 获取信号量。
2. 如果获取成功，进行写入。
   1. 调用 `channel.writeAndFlush()` 进行写入。
   2. 在写入监听器的回调函数中释放信号量。
3. 获取失败，抛出异常。

## invokeSyncImpl()

1. 实例化 `responseFuture` ，并缓存 `request.opaque` --> `responseFuture` 的映射到 `responseTable`。
2. 写入到 `channel`。
3. 调用 `responseFuture.waitResponse(timeoutMillis)` 阻塞等待回复。
4. 收到对方的回复则返回结果、或者超时了/写入失败了则抛出异常。

## invokeAsyncImpl()

1. 获取信号量。
2. 如果获取成功，进行写入。
   1. 实例化 `responseFuture` ，并缓存 `request.opaque` --> `responseFuture` 的映射到 `responseTable`。
   2. `responseFuture` 存储了信号量。
   3. `responseFuture` 设置了 `invokeCallback`，在收到回复时会先调用这个回调函数、再释放信号量。
   4. 写入到 `channel`。
3. 获取失败，抛出异常。

## 处理读请求

读请求到来后，经过解码后，最终调用 `processMessageReceived(ctx, msg)`。

-  到来的数据是 Request。
   1. 根据 Request 的类型 `code`，从 `processorTable` 中找到处理方法和executor进行执行。

   2. 如果没有找到处理方法，使用默认的处理方法处理。

      - Server 需要使用者主动提供默认的处理方法。

      - Client 没有默认的处理方法。这里会抛出异常，说没有找到对应的处理方法。

- 到来的数据是 Response。
  1. 从 responseTable 根据 `respone.opaque` 找到 `responseFuture`。
  2. 把回复结果放入 `responseFuture`。
  3. 如果有 `InvokeCallback`，则执行回调函数、释放限号量。
  4. 否则，唤醒调用 `responseFuture.await()` 的线程、释放信号量。

#NettyRemotingServer

## 主要功能

### 注册请求处理器

使用 `registerProcessor` 来注册请求类别 `requestCode` 对应的处理方法和执行处理方法的线程池。

```java
void registerProcessor(final int requestCode, final NettyRequestProcessor processor,
    final ExecutorService executor);
```

### 三种写数据方法

都是调用 `NettyRemotingAbstract#invokeXXXX` 实现的。

- `invokeOneway()`
- `invokeSync()`
- `invokeAsync()`

## 使用的线程池

1. `publicExecutor`。默认为 4。

   在注册请求的处理方法时，如果没有传入线程池，分配这个线程池给这个处理方法。

2. `eventLoopGroupBoss`。大小为 1。

   监听连接事件。

3. `eventLoopGroupSelector`。默认为 3。

   监听读写事件。

4. `defaultEventExecutorGroup`。默认为 8。

   读写事件到来后，进行逻辑处理的 handler 在这个线程池执行。

5. `nettyEventExecutor`。单线程，队列长度 10000。

   在有Netty事件发生时，执行相应的 Netty 事件监听器。

6. `Timer` 定时任务。

   每3秒扫描一次等待Response列表，删除超时的。

# NettyRemotingClient

功能和 `NettyRemotingServer` 都是类似的，只不过它是 Netty 的客户端，向 Netty 的服务端主动发起 TCP 连接。

## 主要功能

### 注册请求处理器

使用 `registerProcessor` 来注册请求类别 `requestCode` 对应的处理方法和执行处理方法的线程池。

```java
void registerProcessor(final int requestCode, final NettyRequestProcessor processor,
    final ExecutorService executor);
```

### 三种写数据方法

都是调用 `NettyRemotingAbstract#invokeXXXX` 实现的。

- `invokeOneway()`
- `invokeSync()`
- `invokeAsync()`

## 使用的线程池。

1. `publicExecutor`。默认为 4。

   在注册请求的处理方法时，如果没有传入线程池，分配这个线程池给这个处理方法。

2. `eventLoopGroupWorker`。大小为 1。

   监听读写事件。

3. `defaultEventExecutorGroup`。默认为 4。

   读写事件到来后，进行逻辑处理。
   
4. `nettyEventExecutor`。单线程，队列长度 10000。

   在有Netty事件发生时，执行相应的 Netty 事件监听器。

5. `Timer` 定时任务。

   每3秒扫描一次等待Response列表，删除超时的。

