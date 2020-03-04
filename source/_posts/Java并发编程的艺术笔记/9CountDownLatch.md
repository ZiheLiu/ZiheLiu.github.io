---
title: 9. CountDownLatch
date: 2020-03-04 14:48:11
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../
---

# 9. CountDownLatch

## 1. 使用

### 构造函数

```java
CountDownLatch counter = new CountDownLatch(10);
```

传入一个要计数的数量。

### await()

```java
counter.await()
```

调用 `await()` 的线程会阻塞住，直到 `counter` 内部的计数器为 0。

### countDown()

```
counter.countDown()
```

一个线程调用 `countDown()` 后，会使其内部的计数器减 1。

## 2. 原理

```java
public class CountDownLatch {
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
  
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
  
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
  
    public void countDown() {
        sync.releaseShared(1);
    }
}
```

内部使用 AQS 实现，相当于一个共享锁。

- 构造函数初始化 `state`。

- 每次调用 `countDown()` 方法，会把 `state` 减 1。
- 调用 `await()` 方法，只有 `state` 为 0 时，才可以获取锁成功。
  - 而且获取锁只会检查 `state`，不会更新它，所以可以有无数个线程调用 `await`，都会被唤醒。