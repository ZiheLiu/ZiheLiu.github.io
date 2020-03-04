---
title: 10. CyclicBarrier
date: 2020-03-04 14:48:12
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../../
---

# 10. CyclicBarrier

## 1. 用法

1. 构造函数传入线程数目 `threads`。

2. 线程调用 `barrier.await()` 后，会阻塞线程。

3. 线程调用 `barrier.await(long timeout, TimeUnit unit)` 后，如果超时，抛出`InterruptedException`。

4. 有线程被中断后，抛出 `InterruptedException`。

5. 正常结束一代后

   - 有 `threads` 个线程调用 `await()`。
   - 返回是第一个调用的 `await()`。

   - `barrier` 进入下一代，可以循环使用。

6. 非正常结束一代时，所有线程

   - 调用了`reset()`、某个线程被中断、某个线程超时了。
   - 线程抛出 `BrokenBarrierException`。
   - `reset()` 进入下一代。
   - 其他两种情况，barrier 坏掉了，再调用 `BrokenBarrierException` 会抛出异常。

## 2. 原理

### 数据结构

```java
private static class Generation {
    boolean broken = false;
}

private final ReentrantLock lock = new ReentrantLock();
private final Condition trip = lock.newCondition();

private Generation generation = new Generation();
private int count;
private final int parties;
```

- 使用可重入锁，调用 `await()` 的线程等待在 `Condition` 上。
- 使用 `generation` 区分不同的调用批次。每次 所有线程苏醒/`reset()` 后进入下一代。
- `count` 记录还要调用多少次 `await()`。

### 构造函数

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}

public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

### await()

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
public int await(long timeout, TimeUnit unit)
    throws InterruptedException,
           BrokenBarrierException,
           TimeoutException {
    return dowait(true, unit.toNanos(timeout));
}

private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    // 1. 加锁。
    lock.lock();
    try {
        final Generation g = generation;

      	// 2.1 如果已经坏了，抛出坏掉异常。
        if (g.broken)
            throw new BrokenBarrierException();

      	// 2.2 如果线程被中断，唤醒所有线程，让他们都抛出异常。
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count;
      	// 2.3.1 如果凑够了线程，唤醒所有线程。
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
          	// 2.3.2 还没有凑够线程，等待在 Condition上。
            try {
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
              	// 2.3.3 线程等待中被中断了，抛出中断异常。唤醒所有线程。
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

          	// 2.3.4 非正常被唤醒的，抛出坏掉异常。
          	// 		reset、某一个线程被中断了、某一个线程超时了。
            if (g.broken)
                throw new BrokenBarrierException();

          	// 2.3.5 凑够数目后被唤醒的。正常返回。
            if (g != generation)
                return index;

          	// 2.3.6 超时了，抛出超时异常。
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
      	// 2.4 释放锁
        lock.unlock();
    }
}

public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
      	// 破坏这一代的 barrier，唤醒这些线程。
        breakBarrier();   // break the current generation
        // 进入下一代。
      	nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}

private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
private void nextGeneration() {
    // signal completion of last generation
    trip.signalAll();
    // set up next generation
    count = parties;
    generation = new Generation();
}
```

