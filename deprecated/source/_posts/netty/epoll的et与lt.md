---
title: Epoll 边缘触发与水平触发
date: 2020-03-10 16:12:00
categories:
	- netty
tags:
	- netty
typora-root-url: ../../
---

> [epoll LT/ET 深入剖析](https://blog.csdn.net/dongfuye/article/details/50880251)（主要参考）
>
> [java nio使用的是水平触发还是边缘触发](https://www.zhihu.com/question/22524908/answer/69054646)

# 水平触发 Level Triggered (LT) 

- socket 接收缓冲区不为空，有数据可读， 读事件一直触发。
- socket 发送缓冲区不满，可以继续写入数据， 写事件一直触发。

# 边缘触发 Edge Triggered (ET) 

仅在状态变化时触发事件。

- socket 的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时，触发读事件。
- socket 的发送缓冲区状态变化时触发写事件，即满的缓冲区刚空出空间时，触发写事件。

# LT 处理过程

## 监听

select 一个一个连接，监听每个连接的 `EPOLLIN` 事件。

## 读取

当 `EPOLLIN` 事件到达时，read fd 中的数据并处理。

### 数据读取一部分

在 Netty 中，读取 16 次后，会让出 EventLoop 给其他任务执行。

如果还有数据需要读，不需要额外操作。因为下次 select  时，直接被 `EPOLLIN` 事件唤醒，继续处理。

## 写入

当需要写出数据时，把数据 write 到 fd 中。

- 如果数据较大，无法一次性写出，那么在 Epoll 中监听 `EPOLLOUT`事件。

当 `EPOLLOUT` 事件到达时，继续把数据 write 到 fd 中。

- 如果数据写出完毕，那么在 Epoll 中关闭 `EPOLLOUT` 事件。

# ET 处理过程

## 监听

select 一个一个连接，监听每个连接的 `EPOLLIN|EPOLLOUT` 事件。

## 读取

当 `EPOLLIN` 事件到达时，read fd 中的数据并处理。

read 需要一直读，直到返回 `EAGAIN` 为止。

### 数据读取一部分

在 Netty 中，读取 16 次后，会让出 EventLoop 给其他任务执行。

如果还有数据需要读，将继续读的任务提交到 EventLoop 中。

这里比 LT 模式少了一次 `select` 系统调用。

## 写入

当需要写出数据时，把数据 write 到 fd 中，直到数据全部写完，或者 write 返回 `EAGAIN`。

当 `EPOLLOUT` 事件到达时，继续把数据 write 到 fd 中，直到数据全部写完，或者 write 返回 `EAGAIN`。

这里比 LT 模式少了 添加和移除 `EPOLLOUT` 的系统调用。

### netty 中的实现

Netty 在之后版本，会这样实现。

[GitHub issue Never unset EPOLLOUT interest flag](https://github.com/netty/netty/pull/9847)

[GitHub issue Epoll: Avoid modifying EPOLLOUT interest when in ET mode #9347](https://github.com/netty/netty/pull/9347)

# 对比

- ET 的要求是需要一直读写，直到返回 `EAGAIN`，否则就会遗漏事件。

- LT 的处理过程中，直到返回 `EAGAIN` 不是硬性要求，但通常的处理过程都会读写直到返回 `EAGAIN`。
- LT 比 ET 多了一个开关 `EPOLLOUT`、读取部分时 `select`的系统调用。