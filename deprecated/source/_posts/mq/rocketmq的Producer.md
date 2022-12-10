---
title: RocketMQ 的 Producer
date: 2020-03-22 09:59:00
categories:
	- mq
tags:
	- mq
	- rocketmq
typora-root-url: ../../
---

# 启动

1. 设置 namespace
2. 设置 producerGroup
3. 实例化 defaultMQProducerImpl
  
   - 创建为ASYNC send所用的线程池，大小CPUS，60秒 keepalive。
4. start()
   1. 检查producerGroup命名。

   2. 如果instanceName是DEFAULT，把 instanceName 改为 PID。

   3. 根据clientId（`<ip>@<instanceName>(@unitName)`）获得或创建MQClientInstance。

   4. 在MQClientInstance注册producer，在producerTable中存入group->producer的映射。

   5. 创建AUTO_CREATE_TOPIC_KEY_TOPIC主题。

   6. 启动MQClientInstance。`MQClientInstance#start()`。

      - 只有没有启动过才会真正启动。否则直接返回。

      1. 如果配置中没有指明NameServer的地址，从指定的URL用HTTP GET获取。
      2. 启动remotingClient。
      3. 启动一些定时任务。都是在同一个单线程的newSingleThreadScheduledExecutor中。
         1. 如果配置中没有指明NameServer的地址，每120秒从指定的URL用HTTP GET获取。
         2. 默认每30秒从NameServer更新路由。
         3. 默认每30秒向所有Broker发送心跳。
         4. 默认每5秒持久化Consumer的offset。
      4. 启动 pull message 线程（Consumer使用）。
      5. 启动rebalance consumer线程（Consumer使用）。默认每20000秒rebalance一次。

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
- encoder，没用netty的对象池

# 选择消息队列

## 默认机制

sendLatencyFaultEnable=false，没有开启故障延迟。

- 如果lastBrokerName为null，轮询下一个queue。
- 否则，轮询时，避开lastBrokerName的queue。

## 故障延迟机制

sendLatencyFaultEnable=true时开启。

有没有故障，是在发送消息后，加入了发送这个消息的queue的故障延迟

- 如果成功了，用处理时间来评估延迟时间。
- 如果失败了，延迟10分钟。

选择机制。

- 如果lastBrokerName为null，直接选择一个没有故障的queue。

- 否则，
  1. 首先看lastBrokerName是否恢复了，如果恢复了选择它的queue。
  2. 否则，随机选择一个坏掉的broker，为它分配一个queue，返回这个queue。

