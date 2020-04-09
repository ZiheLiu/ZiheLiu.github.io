---
title: RocketMQ 的 Namesrv
date: 2020-03-23 17:49:00
categories:
	- mq
tags:
	- mq
	- rocketmq
typora-root-url: ../../
---

# 启动流程

入口：`NamesrvStartup.main()`。

1. 读取命令行参数、配置文件，填充配置对象。
   - 命令行参数的优先级高于配置文件。因为是先用配置文件填充配置对象，再用命令行参数填充配置对象的。
2. 实例化 `NamesrvController`。
3. 初始化 `controller`。
   - 从磁盘加载 KV 配置。
   - 实例化 `NettyRemotingServer`。
   - 创建 `remotingExecutor` 线程池，默认大小为 8。向 `remotingServer` 注册默认处理函数、executor为这个线程池。
   - 注册定时任务1。 每10秒扫描所有Broker，如果已经120秒没有收到Broker的消息，从缓存中移除这个Broker。
   - 注册定时任务2。每10秒打印KV配置。
4. 添加JVM关闭时钩子函数。JVM 关闭时，关闭各个线程池。
5. 启动 `remotingServer`。

# 路由注册

## Broker 端

Broker 每 30 秒向每个 Namesrv 发起 REGISTER_BROKER 请求，带上如下信息：

- brokerAddr：broker 地址。 
- brokerId：brokerld，0为Master，大于 0为Slave。
- brokerName：broker 名称。
- clusterName：集群名称 。
- haServerAddr：master 地址，初次请求时该值为空，slave 向 Nameserver 注册后返回。

## Namesrv 端

收到 REGISTER_BROKER 请求后，由 `namesrv.processor.DefaultRequestProcessor` 处理函数进行处理。

更新响应缓存配置即可。

# 路由删除

两种方法

- Namesrv 定时扫描。
- Broker 关闭时，主动发起 UNREGISTER_BROKER 请求。

# 客户端索取路由

客户端每 30 秒从 Namesrv 获取一次路由信息。