---
title: Recycler 对象池
date: 2020-03-12 13:41:00
categories:
	- netty
tags:
	- netty
typora-root-url: ../../
---

# SO_REUSEADDR

1. SO_REUSEADDR 允许启动一个监听服务器并捆绑一个端口，即使以前建立的将此端口用做他们的本地端口的连接仍存在（TIME_WAIT）。这通常是**重启监听服务器**时出现，若不设置此选项，则 bind 时将出错。
2. SO_REUSEADDR 允许在同一端口上启动同一服务器的**多个实例**，只要每个实例捆绑一个**不同的本地 IP 地址**即可。
3. SO_REUSEADDR 允许**单个进程**捆绑同一端口到多个套接口上，只要每个捆绑指定**不同的本地 IP 地址**即可。这一般不用于TCP服务器。
4. SO_REUSEADDR 允许完全重复的捆绑：当一个IP地址和端口绑定到某个套接口上时，还允许此IP地址和端口捆绑到另一个套接口上。一般来说，这个特性仅在支持多播的系统上才有，而且只对UDP套接口而言（TCP不支持多播）。

# SO_REUSEPORT

## 介绍

使用 `select`、`epoll` 多路复用技术，也只能在一个线程监听连接的到来，无法利用多核 CPU。

在 Linux 3.9 以后增加了 OP_REUSEPORT 参数，可以多次绑定同一个端口。

> The new socket option allows multiple sockets on the same host to bind to the same port, and is intended to improve the performance of multithreaded network server applications running on top of multicore systems.

## 作用

- 允许多个套接字 bind()/listen() 同一个 TCP/UDP 端口。
  - 每一个线程拥有自己的服务器套接字。
  - 在服务器套接字上没有了锁的竞争。
- 内核层面实现负载均衡。
- 安全层面，监听同一个端口的套接字只能位于同一个用户下面。

## Netty 中使用

在 `ChannelOption` 没有 `SO_RESUEPORT` 选项。

而在 `EpollChannelOption` 中有 `SO_RESUEPORT` 选项。

因此，使用这个参数，需要进行如下的替换：

- `NioEventLoopGroup` -> `EpollEventLoopGroup`。
- `NioServerSocketChannel.class` -> `EpollServerSockerChannel.class`。