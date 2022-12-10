---
title: 14. ScheduledThreadPoolExecutor
date: 2020-03-04 14:48:16
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../../
---

# 14. ScheduledThreadPoolExecutor

## 0. 引用

- [JUC源码分析-线程池篇（三）ScheduledThreadPoolExecutor](https://www.cnblogs.com/binarylei/p/10963067.html)

## 1. 基本用法

```java
// 在 delay 时间后执行 command 或 callable。
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);
public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);

// 直接执行任务。
public void execute(Runnable command)；

// 以固定频率执行 command。
// initialDelay 后执行第一次，initialDelay + period 后执行第二次，
// initialDelay + 2*period 后执行第三次。
// 如果一次执行时间超过了 period，后面的任务需要等待这次执行完才能执行，不会并行执行。
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, 
                                              long initialDelay, 
                                              long period, 
                                              TimeUnit unit);

// 每次执行任务结束，过 delay 时间后，再执行下一次。
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
                                                 TimeUnit unit);
```

## 2. 基本思想

`ScheduledThreadPoolExecutor` 继承自 `ThreadPoolExecutor`，使用 `DelayedWorkQueue` + `ThreadPoolExecutor` + `ScheduledFutureTask` 实现。

![ScheduledThreadPoolExecutor执行流程](/images/ScheduledThreadPoolExecutor执行流程.svg)

## 3. DelayedWorkQueue

`DelayedWorkQueue` 是延时阻塞队列，类似于 `DelayQueue`。

与 `DelayQueue` 的算法都相同。

- 使用 `ReentrantLock` 保证线程安全。
- 使用数组作为最小堆的存储结构。

唯一的不同：

- `remove(task)` 的时间复杂度从 $O(n)$ 降低到 $O(logn)$。
- `ScheduledFutureTask` 中记录了在数组中存储的下标，在 `remove()` 的时候，直接删除就可以了。
- 在 `task.cancel()` 任务时，会调用 `task.OuterClass.remove(task)` （`ScheduledFutureTask` 是 `DelayedWorkQueue` 的内部类）。

容量是无限的。

```java
public boolean remove(Object x) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = indexOf(x);
        if (i < 0)
            return false;

        setIndex(queue[i], -1);
        int s = --size;
        RunnableScheduledFuture<?> replacement = queue[s];
        queue[s] = null;
        if (s != i) {
            siftDown(i, replacement);
            if (queue[i] == replacement)
                siftUp(i, replacement);
        }
        return true;
    } finally {
        lock.unlock();
    }
}

// O(1) 找到节点的下标。
private int indexOf(Object x) {
    if (x != null) {
        if (x instanceof ScheduledFutureTask) {
            int i = ((ScheduledFutureTask) x).heapIndex;
            // Sanity check; x could conceivably be a
            // ScheduledFutureTask from some other pool.
            if (i >= 0 && i < size && queue[i] == x)
                return i;
        } else {
            for (int i = 0; i < size; i++)
                if (x.equals(queue[i]))
                    return i;
        }
    }
    return -1;
}
```

## 4. ScheduledFutureTask

`ScheduledFutureTask` 存储了

- 下一次执行的时间 `time`
- 时间间隔 `period`
- 最小堆数组中的下标 `helperIndex`
- 下一次执行的任务（一般为自己）`outerTask`。

每次执行 `ScheduledFutureTask` 结束后，更新下一次的执行时间 `time`，把任务再放到延时队列里。

```java
// 用于ScheduledFutureTask的序列号。
private static final AtomicLong sequencer = new AtomicLong();

private class ScheduledFutureTask<V>
        extends FutureTask<V> implements RunnableScheduledFuture<V> {

    private final long sequenceNumber;

    // 下一次执行的时间戳（nanoTime单位）。
    private long time;

  	// 正数 fixed-rate execution。
  	// 负数 fixed-delay execution。
  	// 0 立即执行一次的任务。
    private final long period;

    /** The actual task to be re-enqueued by reExecutePeriodic */
    RunnableScheduledFuture<V> outerTask = this;

    // 在最小堆数组中的下标，用来 cancel 时从最小堆中删除节点。
    int heapIndex;

  	// 不重复执行的任务。
    ScheduledFutureTask(Runnable r, V result, long ns) {
        super(r, result);
        this.time = ns;
        this.period = 0;
        this.sequenceNumber = sequencer.getAndIncrement();
    }
    ScheduledFutureTask(Callable<V> callable, long ns) {
        super(callable);
        this.time = ns;
        this.period = 0;
        this.sequenceNumber = sequencer.getAndIncrement();
    }

    // fixed-rate or fixed-delay execution 。
    ScheduledFutureTask(Runnable r, V result, long ns, long period) {
        super(r, result);
        this.time = ns;
        this.period = period;
        this.sequenceNumber = sequencer.getAndIncrement();
    }

  	// 还要多少纳秒可以执行。
    public long getDelay(TimeUnit unit) {
        return unit.convert(time - now(), NANOSECONDS);
    }

  	// 如果两个元素 time 相同，使用 sequenceNumber 比较。
    public int compareTo(Delayed other) {
        if (other == this) // compare zero if same object
            return 0;
        if (other instanceof ScheduledFutureTask) {
            ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
            long diff = time - x.time;
            if (diff < 0)
                return -1;
            else if (diff > 0)
                return 1;
            else if (sequenceNumber < x.sequenceNumber)
                return -1;
            else
                return 1;
        }
        long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
        return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
    }
		// 是否重复执行。
    public boolean isPeriodic() {
        return period != 0;
    }

  	// time 记录了上次运行的时间。
  	// 1. fixed-rate 模式，把 time + period。即从上次运行后果 period 时间运行。
  	// 2. fixed-delay 模式，把 当前时间 + period。即从现在开始过 period 时间运行。
    private void setNextRunTime() {
        long p = period;
        if (p > 0)
            time += p;
        else
            time = triggerTime(-p);
    }

  	// 取消任务，并从队列中删除节点。
    public boolean cancel(boolean mayInterruptIfRunning) {
        boolean cancelled = super.cancel(mayInterruptIfRunning);
        if (cancelled && removeOnCancel && heapIndex >= 0)
            remove(this);
        return cancelled;
    }

    public void run() {
        boolean periodic = isPeriodic();
        if (!canRunInCurrentRunState(periodic))
            cancel(false);
      	// 1. 如果时立即执行一次的任务，则直接执行任务。
        else if (!periodic)
            ScheduledFutureTask.super.run();
      	// 2. 是重复执行的任务，
      	//		a. 使用 runAndReset执行任务。
      	//		b. 任务执行结束后，设置 task.time，把 outerTask 再次加入到延时队列中。
        else if (ScheduledFutureTask.super.runAndReset()) {
            setNextRunTime();
            reExecutePeriodic(outerTask);
        }
    }
}

void reExecutePeriodic(RunnableScheduledFuture<?> task) {
    if (canRunInCurrentRunState(true)) {
      	// 把 task 加入到延时队列中。
        super.getQueue().add(task);
        if (!canRunInCurrentRunState(true) && remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
}
```

## 5. schedule()

```java
// 延时执行一次。
public ScheduledFuture<?> schedule(Runnable command,
                                   long delay,
                                   TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
  	// 使用ScheduledFutureTask包装command、time。
    RunnableScheduledFuture<?> t = decorateTask(command,
        new ScheduledFutureTask<Void>(command, null,
                                      triggerTime(delay, unit)));
    // 放入延时队列，创建工作线程。
  	delayedExecute(t);
    return t;
}
// 直接调用。
public void execute(Runnable command) {
    schedule(command, 0, NANOSECONDS);
}

public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                              long initialDelay,
                                              long period,
                                              TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (period <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(period));
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    // 把 outerTask 设为自己。
  	sft.outerTask = t;
    delayedExecute(t);
    return t;
}
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
                                                 TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (delay <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(-delay)); // 注意是 -delay。
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    // 把 outerTask 设为自己。
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}

```

###  delayedExecute()

因为 `ScheduledThreadPoolExecutor` 的 `schedule()` 和 `submit()` 方法都是把任务放入延时队列，没有调用过子类的 `ThreadPoolExecutor.execute()` 方法。所以必须手动 `addWorker()`。

会不断补充核心线程，知道达到核心线程数目。

```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        reject(task);
    else {
        super.getQueue().add(task);
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
}
// 保证执行前至少有1个工作线程。
// 工作线程数目小于corePoolSize，就创建一个核心线程。
// 如果corePoolSize是0，也创建1个非核心线程，来保证至少有1个工作线程。
void ensurePrestart() {
  int wc = workerCountOf(ctl.get());
  if (wc < corePoolSize)
    addWorker(null, true);
  else if (wc == 0)
    addWorker(null, false);
}
```

