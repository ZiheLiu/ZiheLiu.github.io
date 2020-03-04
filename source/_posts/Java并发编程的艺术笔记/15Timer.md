---
title: 15. Timer
date: 2020-03-04 14:48:17
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../
---

# 15. Timer

`Timer` 是 `java.util` 包的定时任务。

## 原理

1. 使用一个数组实现的最小堆 `TaskQueue queue`。
2. 在操作 `queue` 时，使用 `synchronized` 对加锁。
3. 启动一个线程 TimerThread，循环去查看 `queue` 的首节点的时间是否到了，
   - 没有到则在 `queue` 上沉睡对应的时间。
   - 如果首节点可以执行了，在执行前检查是否需要重复执行。
     - 如果需要，更新节点的 `nextExecutionTime`，重新放入堆中。
     - 否则，从堆中删掉。
4. 在有新的任务加到 `timer` 中后，如果新加的任务成为了首节点，因为需要睡眠的时间变小了，所以唤醒 TimerThread。

## 缺陷

由于是只用一个线程去循环执行多个定时任务，会有两个缺陷：

1. 当前正在执行的任务耗时太久，导致后边的任务即使时间已经到了，也无法执行。
2. 当前正在执行的任务抛出异常（除了 `InterruptedException`)，会使这个线程结束，后边的任务都不执行了。

- 此外，`Timer` 不能像 `ScheduleThreadPoolExecutor` 那样 `cancel(taks)` 掉一个任务。
  只能用 `cancel()` 取消掉所有任务。
