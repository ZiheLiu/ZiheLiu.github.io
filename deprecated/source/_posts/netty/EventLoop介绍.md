---
title: EventLoop 主流程
date: 2020-03-10 15:11:00
categories:
	- netty
tags:
	- netty
	- record
typora-root-url: ../../
---

# 引用

- [Netty NioEventLoop源码解读](https://www.jianshu.com/p/3ebd4e997242)
- [Java NIO epoll bug 以及 Netty 的解决之道](http://songkun.me/2019/07/26/2019-07-26-java-nio-epoll-bug-and-netty-solution/)

# 介绍

作为 Worker 的 `EventLoop`，负责多路复用 `select/epoll` 来监听多个 socket 连接的读写事件，并在有读写事件时 进行逻辑的处理、从 socket 读取数据、向socket 写入数据。

而 `EventLoop` 是单线程的，如何同时阻塞监听读写事件、并处理业务逻辑？

#  流程

1. 每次循环，先检测是否有任务要处理。
   - 如果没有任务，就进行 `select(timeout)`。
     - `timeout` 的时间是“下一次要执行任务的时间 + 0.5 ms”，没有要执行的任务，就是 “1s + 0.5 ms”
   - 如果有任务，就不进行 `select()`。
2. 然后，处理到达的读写事件。
3. 最后，执行所有待执行的任务。
4. 随后，重复步骤1。

# 解决 JDK NIO 空轮询 Bug

## Bug 简介

`selector.select()` 应该 **一直阻塞**，直到有就绪事件到达。

但由于 Java NIO 实现上存在 bug，`select()` 可能在 **没有** 任何就绪事件的情况下返回，从而导致 `while(true)` 被不断执行，最后导致某个 CPU 核心的利用率飙升到 100%，这就是臭名昭著的 Java NIO 的 epoll bug。

实际上，这是 Linux 系统下 poll/epoll 实现导致的 bug，但 Java NIO 并未完善处理它，所以也可以说是 Java NIO 的 bug。

## 解决办法

```java
// SELECTOR_AUTO_REBUILD_THRESHOLD 默认 512
if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 && selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
  this.rebuildSelector();
  selector = this.selector;
  selector.selectNow();
  selectCnt = 1;
  break;
}
```

每次阻塞 `select()` 被唤醒后，如果是空转（没有事件被触发），`selectCnt` 加 1。

如果空转次数超过 512，重新 build 一个 `selector`。