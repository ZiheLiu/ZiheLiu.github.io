---
title: Reactor 模式
date: 2020-03-07 20:19:00
categories:
	- netty
tags:
	- netty
---

# 引用

- [IO多路复用之Reactor](https://www.cnblogs.com/amei0/p/8117660.html)

- [Netty中的三种Reactor](https://www.cnblogs.com/duanxz/p/3696849.html)

# 简介

Reactor（反应器）模式常用来实现 I/O 多路复用机制。

Reactor 管理器调用事件分离器等待事件的发生。事件发生后，Reactor 管理器被唤醒，调度事件处理器链进行事件的处理。

核心流程：注册感兴趣的事件 -> 扫描是否有感兴趣的事件发生 -> 事件发生后做出相应的处理。

# Reactor模式组成

## 描述符 Handles 

由操作系统提供的资源，用于识别每一个事件，如 Socket 描述符、文件描述符、信号的值等。

在Linux中，它用一个整数来表示。

在网络编程中，一般是一个 Socket 描述符。

## 同步多路事件分离器 Event Demultiplexer

循环等待。一次同时等待多个事件的到来。

在 Linux 系统上使用 select、poll 等系统调用来实现。

调用者会被阻塞，直到分离器的分离的描述符集上有事件发生。

## 事件处理器 Event Hanlder

定义事件的处理程序，具有统一的若干种接口（出站、入站）。

## Reactor 管理器

- 管理描述符

- 注册/移除事件处理器

- 调用事件分离器来等待事件

- 事件到来时，把时间分发给特定的事件处理器进行处理。

# 模式演进

## 单线程模式

系统只使用一个线程。

Reactor 管理器来分离连接/读写事件、事件处理器来处理事件 都在一个线程。

## 单 Reactor 多线程模式

- Reactor 管理器使用**线程池**来分离连接/读写事件。

- 事件处理器交给另外的**线程池**来执行。

### 缺点

在客户端连接特别多的时候、或客户端与服务端需要进行安全认证时，Reactor 在建立连接上可能耗费大量资源，使得读写事件受到影响。

所以有了下面的主从 Reactor 模式。

## 主从 Reactor 多线程模式

- MainReactor 管理器使用**线程池**来分离连接事件。

- MainReactor 将建立的连接分派给 SubReactor，SubReactor 使用另外的**线程池**来分离读写事件。

- 事件处理器交给另外的**线程池**执行。

在 Netty 中推荐主从 Reactor 模式，使用例子如下

```java
// MainReactor
EventLoopGroup bossGroup = new NioEventLoopGroup(2);
// SubReactor
// 默认值 CPUS * 2
EventLoopGroup workerGroup = new NioEventLoopGroup();
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup, workerGroup);
```

# Preactor 模式

Reactor 采用同步 I/O 网络模式。

Preactor 采用异步 I/O 网络模式。