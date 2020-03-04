---
title: 16. Executors 工具类
date: 2020-03-04 14:48:18
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../
---

# 16. Executors 工具类

`Executors` 工具类提供了创建各类线程池的静态方法，这些方法都是基于 `ThreadPoolExecutor` 和 `ScheduledThreadPoolExecutor` 实现的。

## 1. 基于 ThreadPoolExecutor

### 1.1 newSingleThreadExecutor()

- 唯一一个核心工作线程不断从任务队列中取任务去执行。
  - 只有一个核心工作线程。
  - 使用 `LinkedBlockingQueue`  **无界**模式 作为任务队列，所以任务总是可以放到队列里。

- 使用 `FinalizableDelegatedExecutorService` 代理了 `ThreadPoolExecutor`。
  - 在被 GC 的时候，会调用 `shutdown()`。

```java
public class ThreadPoolExecutor {
  public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue);
}

public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS, // poll(0) 等同于 poll()
                                new LinkedBlockingQueue<Runnable>()));
}


static class FinalizableDelegatedExecutorService
    extends DelegatedExecutorService {
    FinalizableDelegatedExecutorService(ExecutorService executor) {
        super(executor);
    }
    protected void finalize() {
        super.shutdown();
    }
}
```

### 1.2 newFixedThreadPool()

- `nThreads` 个核心工作线程不断从任务队列中取任务去执行。
  - 有 `nThreads` 个核心工作线程。
  - 使用 `LinkedBlockingQueue` **无界**模式 作为任务队列，所以任务总是可以放到队列里。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

### 1.3 newCachedThreadPool()

- 没有核心工作线程。
- 可以有 `Integer.MAX_VALUE` 个工作线程。
- 每个工作线程如果 60s 没有领到任务，就结束。
- 使用 `SynchronousQueue`。
  - `poll(e)` 时，必须存在出队阻塞的操作，才会插入成功；否则返回false。
  - 可以实现**没有空闲的工作线程，就创建一个新的工作线程去执行任务**。
  - 因为没有空闲的工作线程 说明没有线程阻塞在 `SynchronousQueue` 的出队操作上，在 `poll(task)` 的时候会返回 false，继而会创建一个新的非工作线程。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

## 2. 基于 ScheduledThreadPoolExecutor

### 2.1 newSingleThreadScheduledExecutor()

- 唯一工作线程不断从延时队列中取任务去执行。

```java
// 任务总是放到延时队列中。因为 DelayedWorkQueue 是无上界的阻塞队列。
public class ScheduledThreadPoolExecutor {
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
}

// corePoolSize = 1
public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
    return new DelegatedScheduledExecutorService
        (new ScheduledThreadPoolExecutor(1));
}
```

### 2.2 newScheduledThreadPool()

- `corePoolSize` 个工作线程不断从延时队列中取任务去执行。

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```