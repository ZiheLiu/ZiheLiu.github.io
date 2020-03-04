---
title: 8. BlockingQueue 简介
date: 2020-03-04 14:48:08
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../../
---

# 8. BlockingQueue 简介

## 0. 引用

- [SynchronousQueue原理解析](https://www.jianshu.com/p/af6f83c78506)

## 1. 介绍

阻塞队列 `BlockingQueue` 继承自 `Queue`，添加了阻塞入队、阻塞出队的功能。

- `put(e)`：入队时，如果队列已满，则阻塞直到队列不满。
- `take()`：出队时，如果队列为空，则阻塞直到队列不空。

`BlockingQueue` 提供的方法：

| 操作     | 返回特殊值 | 超时退出             | 抛出异常  | 阻塞   |
| -------- | ---------- | -------------------- | --------- | ------ |
| 插入     | offer(e)   | offer(e, time, unit) | add(e)    | put(e) |
| 移除     | poll()     | poll(e, time, unit)  | remove()  | take() |
| 获取队首 | peek()     | pe(e, time, unit)    | element() | 无     |

## 2. 7 个阻塞队列

1. ArrayBlockingQueue：数组组成、有界、一把公平/非公平锁。
2. LinkedBlockingQueue：链表组成、有界、两把非公平锁。
3. LinkedBlockingDeque：链表组成、有界、一把非公平锁、双向。
4. PriorityBlockingQueue：基于数组的最小堆、无界、一把非公平锁 + 扩容自旋锁、优先级排序。
5. DelayQueue：基于 `PriorityQueue`、无界、一把非公平锁、优先级排序。
6. SynchronousQueue：链表组成的队列（公平）/栈（非公平）、无界、CAS 更新、每一个出队和入队操作必须匹配，否则阻塞。
7. LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。TODO

这些阻塞队列一般都是由 `ReentrantLock` 锁 + `Condition` 实现的。

| 类名                     | 底层结构                                     | 是否有界 | 锁的形式                         | 备注                           |
| ------------------------ | -------------------------------------------- | -------- | -------------------------------- | ------------------------------ |
| ArrayBlockingQueue       | 数组                                         | 有界     | 一把公平/非公平锁                |                                |
| LinkedBlockingQueue      | 链表                                         | 有界     | 两把非公平锁                     |                                |
| LinkedBlockingDeque      | 链表                                         | 有界     | 一把非公平锁                     | 双向队列                       |
| PriorityBlockingQueue    | 基于数组的最小堆                             | 无界     | 一把非公平锁 + 扩容自旋锁        | 优先级排序                     |
| DelayQueue               | `PriorityQueue`(基于数组的最小堆)                    | 无界     | 一把非公平锁                     | 优先级排序                     |
| SynchronousQueue TODO    | 链表组成的队列（公平）、<br />或栈（非公平） | 无界     | CAS 更新，LockSupport 等待和唤醒 | put 入队操作需要等待出队操作；<br />offer 入队操作需要有出队阻塞存在才可以，否则返回false。 |
| LinkedTransferQueue TODO | 松弛型双重队列                               | 无界     | CAS 更新，LockSupport 等待和唤醒 | transfer 入队操作必须等待出队操作。 |

### 1. ArrayBlockingQueue

- 由固定长度数组组成。构造函数传入容量。
- 使用 `ReentrantLock lock` 一把锁控制入队、出队操作，并发度低于 `LinkedBlockingQueue`。构造函数可以传入是否使用公平锁。
  - 入队/出队前，都需要持有锁。
  - 因为只用一把锁，`count` 使用 `int` 就可以。
- 使用 `lock` 的两个 `Condition` 等待队列：`notEmpty`、`notFull`。
  - 入队后，唤醒一个 `notEmpty` 上的线程。
  - 出队后，唤醒一个 `notFull` 上的线程。
- 入队/出队移动数组元素来实现。

### 3. LinkedBlockingDeque

- 由链表组成，构造函数传入容量，默认 `Integer.MAX_VALUE`。
- 和 `ArrayBlockingQueue` 类似，使用一把`ReentrantLock lock` 一把锁控制入队、出队操作。
- 使用 `lock` 的两个 `Condition` 等待队列。

### 7. LinkedTransferQueue

与 `SynchronousQueue` 的异同点：

| 类名                | `offer(e)`                                | `poll()`                                  | `put(e)`                       | `take()`                 |
| ------------------- | ----------------------------------------- | ----------------------------------------- | ------------------------------ | ------------------------ |
| LinkedTransferQueue | 总是可以入队                              | 队列为空，返回false                       | 没有阻塞的take()线程，则阻塞。 | 需要阻塞等待一个put(e)。 |
| SynchronousQueue    | 如果没有需要阻塞等待的take()，返回false。 | 如果没有需要阻塞等待的put(e)，返回false。 | 需要阻塞等待一个take()。       | 需要阻塞等待一个put(e)。 |

`LinkedTransferQueue` 的

-  `offer(e)` 、 `poll()` 与 `LinkedBlockingQueue` 功能相同。

- `put(e)`、`take()` 与 `SynchronousQueue` 的功能相同。