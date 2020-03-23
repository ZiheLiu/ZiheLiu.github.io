---
title: RocketMQ 发送消息流程
date: 2020-03-22 09:59:00
categories:
	- mq
tags:
	- mq
	- rocketmq
typora-root-url: ../../
---

# 异步发送流程

1. `send()` 提交任务给executor执行。
   - 调用 `sendDefaultImpl()`。
   - 并用 `try catch{}` 包起来，把异常传给 `sendCallback.onException(e)`。
2. `sendDefaultImpl()` 选择队列，交给下一步执行。SYNC模式会在这里重试。
   - 检查 Producer 运行状态、消息的格式。
   - 获取topic路由信息。
   - SYNC模式，以下最多重复RetryTimes次。其他模式，只执行一次。
     - 选择一个queue。
     - 调用 `sendKernelImpl()`。
3. `sendKernelImpl()` 设置要发往的地址、请求头部信息。
   - 找到queue对应的broker的Addr。
   - 设置uniqID。
   - 设置Request的Header信息。
   - 调用 `MQClientAPIImpl#sendMessage()`。
4. `MQClientAPIImpl#sendMessage()`
   - 设置Request的Body信息。
   - 调用 `MQClientAPIImpl#sendMessageAsync()`。
5. `MQClientAPIImpl#sendMessageAsync()`
   - 调用 `NettyRemoteClient#invokeAsync()`。
   - 传入了一个回调函数 `invokeCallback`。
     - 如果response不为空，即Broker在timeout前给了回复。
       - 如果用户传入的`sendCallback`是null，则执行钩子函数后返回。
       - 否则，调用 `sendCallback.onSuccess(sendResult)`。
     - 如果response为空，重新调用 `MQClientAPIImpl#sendMessageAsync()`。
       - 这样每次失败，都在回调函数里再次调用，直到达到 RetryTimes后把异常传给`sendCallback.onException(e)`。
6. `NettyRemoteClient#invokeAsync()`
   - 检查 channel
     - 如果没有创建过，则连接这个broker。
     - 如果channel inactive了，关闭连接，抛出异常。
   - 调用 `NettyRemoteClient#invokeAsyncImpl()`。
7. `NettyRemoteClient#invokeAsyncImpl()`
   - 创建ResponseFuture，设置回调为第5步传入的回调函数`invokeCallback`。
     - 在Broker返回消息的时候，会调用这个回调函数。
   - 用 `channel.write()` 写入request，这里是异步的。

## 问题

- 超时时间是否需要包含在 executor 的排队的时间？

- 为什么 netty的EventLoopGroup线程数目只有一个，连接所有的broker、NameServer都用一个。