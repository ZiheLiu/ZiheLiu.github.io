---
title: Java 线程池拒绝策略
date: 2020-03-17 16:27:18
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../../
---

在提交任务时，如果任务队列满了、线程数超过了最大线程数，以及线程池已经关闭了（状态不是 RUNNING），会拒绝这个任务，共有四种拒绝策略。

# AbortPolicy

默认的策略。直接抛出异常。

```java
public static class AbortPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}
```

# CallerRunsPolicy

当前提交任务的线程去执行这个被拒绝的任务。

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}
```

# DiscardPolicy

直接丢弃任务，什么也不做。

```java
public static class DiscardPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}
```

# DiscardOldestPolicy

抛弃队列中最旧的任务，重新执行被拒绝的任务。

注意，这里只是简单的出队了首节点，如果任务是 `FutureTask`，且有线程调用了 `FutureTask#get()`，会永远阻塞在这里。

```java
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}
```

# 自定义拒绝策略

实现 `RejectedExecutionHandler` 接口即可。