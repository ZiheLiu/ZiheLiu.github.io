---
title: 11. ThreadPoolExecutor 源码分析
date: 2020-03-04 14:48:13
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../
---

# 11. ThreadPoolExecutor 源码分析

## 0. 引用

- [JUC源码分析-线程池篇（一）：ThreadPoolExecutor](https://www.cnblogs.com/binarylei/p/10952055.html)

## 1. 数据结构

### 线程池表示

```java
// ctl 高 3 位：线程池状态，低 29 位：工作线程数
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// 从 ctl 获取线程池状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 从 ctl 获取工作线程数
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 用线程池状态、工作线程数 构造ctl
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

## 2. 构造函数



## 3. execute()

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    
    int c = ctl.get();
  	// 1. 工作线程数目<核心线程数目，创建新的核心线程执行任务。
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
  	// 2. 工作线程数目>=核心线程数目、任务队列未满，将任务提交到任务队列中。
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
      	// 2.1 如果这时线程池关闭了、且任务还没有开始执行（开始执行会从队列中取出，所以删除会失败），
      	//		把任务从队列删除、执行拒绝策略。
        if (! isRunning(recheck) && remove(command))
            reject(command);
      	// 2.2 没有工作线程（可能corePoolSize为0），
      	//		创建新的非核心线程。这个线程会去队列里拿任务执行。
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
  	// 3. 工作线程数目>=核心线程数目、任务队列已满，创建新的非核心线程执行任务。
    else if (!addWorker(command, false))
      	// 如果创建失败（线程数目达到最大值、或线程池关闭了），执行拒绝策略。
        reject(command);
}

final void reject(Runnable command) {
    handler.rejectedExecution(command, this);
}
```

调用线程池的域 `RejectedExecutionHandler` 实例的回调任务函数。

## 4. addWorker()

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
  	// 1. 检查是否可以创建新的线程，多个线程使用 workerCount 串行化。
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 1.1. 检查线程池状态。
      	// 如果a. 线程池为SHUTDOWN状态，并且传入的任务不为空（execute传过来 要执行新任务的）、或任务队列为空；
      	//		b. 线程池过了SHUTDOWN状态了。
      	// 则不能再创建新的线程。
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
          	// 1.2. 检查线程池容量。
          	// 如果a. 要创建核心线程、且工作线程数目超过核心线程数目了
          	//		b. 要创建非核心线程、且工作线程数目超过最大线程数目了
          	// 不能再创建新的线程。
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
          	// 1.3. 把工作线程数目 CAS 加1。
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
          	// 1.4. 如果 1.3步骤 中 CAS 失败了。
          	// 		a. 线程池状态发生变化了，返回1.1步骤。
          	//		b. 有其他线程改变了工作线程数目，返回1.2步骤。
            if (runStateOf(c) != rs)
                continue retry;
        }
    }

  	// 2. 开始创建工作线程、并运行。
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
      	// 2.1. 创建工作线程。
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
          	// 2.2 加全局锁。
          	//		因为要操作HashMap、线程内、全局计数、线程池可能正在terminate。
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

              	// 2.3 检查线程池状态。
              	//	如果 a. 线程池是SHUTDOWN状态、且传入任务不为空
              	//			b. 线程池度过SHUTDOWN状态了。
              	//	则不能使用这个线程，添加线程失败。
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 2.4 把新的工作线程加入 HashSet 中。
                  	workers.add(w);
                    int s = workers.size();
                  	// 2.5 更新线程池最大线程数目。
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
              	// 2.6 解锁。
                mainLock.unlock();
            }
            if (workerAdded) {
              	// 2.7 启动新的工作线程。
                t.start();
                workerStarted = true;
            }
        }
    } finally {
      	// 2.8 如果添加或启动线程失败了，进行收尾处理。
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}

private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
  	// 1. 加全局锁
    mainLock.lock();
    try {
      	// 2. 从 HashMap 中删除这个工作线程。
        if (w != null)
            workers.remove(w);
      	// 3. 工作线程数目 CAS 减1
      	//	（因为这是在addWorker()中CAS加1成功后、创建线程失败时，调用的此函数）
        decrementWorkerCount();
      	// 4. 因为减少了工作线程数目，尝试终结。
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

## 5. tryTerminate()

```java
// 在可能会使变为 TERMINATED 状态可能的地方，都会调用这个函数。
//
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
      	// 1. 检查线程池状态。
      	// 如果 a. 线程池是RUNNING状态。
      	//		 b. 线程池是TIDYING或TERMINATED状态，说明已经终结过了。
      	//		 c. 线程池是SHUTDOWN状态、且任务队列非空。
      	// 不满足终结的条件，返回。
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
      	// 2. 如果工作线程数目还存在，中断第一个空闲线程。
      	//		可能调用shutDown()后，有些核心线程在工作，随后会陷入无限等待
      	//		所以每个工作线程退出时，会中断一个空闲线程。
        if (workerCountOf(c) != 0) { // Eligible to terminate
          	// 因为要遍历 HashMap，所以要加全局锁。
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
      	// 3. 加全局锁。
        mainLock.lock();
        try {
          	// 4. CAS 设置状态为 TIDYING。
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                  	// 5. 调用钩子方法。
                    terminated();
                } finally {
                  	// 6. 状态变为TERMINATED。
                    ctl.set(ctlOf(TERMINATED, 0));
                  	// 7. 唤醒阻塞在termination条件的所有线程，
                  	//		这个变量的await()方法在awaitTermination()中调用
                    termination.signalAll();
                }
                return;
            }
        } finally {
          	// 8. 解锁。
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

## 6. Worker

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;
    Runnable firstTask;
  	// 统计本工作线程完成的任务数目
    volatile long completedTasks;

  	// firstTask 可以为 null。
    Worker(Runnable firstTask) {
      	// 禁止中断，直到 runWorker() 调用。
        setState(-1);
        this.firstTask = firstTask;
      	// 使用线程池的 ThreadFactory 创建线程。
        this.thread = getThreadFactory().newThread(this);
    }

    // 代理到线程池的 runWorker() 函数。
    public void run() {
        runWorker(this);
    }

    
  
		// 1持有锁、说明线程在执行任务；0不持有锁、说明线程没有在执行任务。
  	// 通过 tryAcquire() 是否成功，判断线程是否处于空闲状态。
    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

  	// 只有调用过runWorker()、线程没有被中断，才会去中断。
    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

## 7. runWorker()

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
  	// 1. 使得线程可以被中断。
    w.unlock(); 
    boolean completedAbruptly = true;
    try {
      	// 2. 循环从任务队列取任务。
        while (task != null || (task = getTask()) != null) {
          	// 2.1 工作线程加锁，说明正在执行任务。
            w.lock();
            // 2.2 如果线程池正在进入STOP状态，检查本工作线程的中断状态，
          	//		没有中断的话，就去中断它。
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
              	// 2.3 钩子函数。
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                  	// 2.4 执行任务。
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                  	// 2.5 执行钩子函数。
                  	// 如果发生了异常，会在2.8后直接跳到2.9。
                    afterExecute(task, thrown);
                }
            } finally {
              	// 2.6 清空task，准备下次从队列取task。
                task = null;
              	// 2.7 统计数目+1。
                w.completedTasks++;
              	// 2.8 工作线程解锁。
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
      	// 2.9 为要终结的工作线程进行清理和统计。
        processWorkerExit(w, completedAbruptly);
    }
}
```

## 8.  getTask()

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

      	// 1. 检查线程池状态。
      	// 如果 a. 状态为SHUTDOWN、并且任务队列为空
      	//		b. 状态度过了SHUTDOWN
      	// 则把工作线程数目减1，返回null。
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 2. 检查是是使用核心线程还是临时线程模式。
      	// 如果是临时线程模式，在阻塞出队poll时，会使用超时等待。
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

      	// 3. 检查数目状态。
      	// 		如果
      	//			a. 当前工作线程数目超过了maximumPoolSize（说明用setMaximumPoolSize()减少了）
      	// 			b. 出队超时了
      	//			并且工作线程数目大于1（线程池至少有一个线程）、或任务队列是空的（没有还在等待执行的任务）
      	//		则返回null，关闭这个工作线程。
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

      	// 4. 从任务队列出队任务。
        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

## 9. processWorkerExit()

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
  	// 1. 检查是否把工作线程数目减1
  	// 		a. 如果是getTask()为null而退出的工作线程，在getTask()中已经减1了。
  	//		b. 如果是用户任务抛出异常，需要在这里减1。
    if (completedAbruptly) 
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
  	// 2. 加全局锁。
    mainLock.lock();
    try {
      	// 3. 把退出的线程线程完成的任务数统计到全局的计数上。
        completedTaskCount += w.completedTasks;
      	// 4. 从 HashMap 中移除。
        workers.remove(w);
    } finally {
      	// 5. 解锁。
        mainLock.unlock();
    }

  	// 6. 因为减少了工作线程数目，尝试终结。
    tryTerminate();

    int c = ctl.get();
  	// 7. 看是否要创建一个非核心线程。
    if (runStateLessThan(c, STOP)) {
      	// 7.1 不是用户任务抛出异常、且工作线程数目少于核心线程数目个（至少为1）。
      	// 7.2 用户任务抛出异常。
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

退出时的清理工作

1. 把退出的线程线程完成的任务数统计到全局的计数上。

2. 从HashMap移除。
3. 1、2 需要加锁。

4. 确保 workerCount 减1。
5. 如果 SHUTDOWN 了，尝试中断、唤醒一个空闲工作线程。
   - 可能 SHUTDOWN 后，多个工作线程 poll 阻塞在任务队列，没有足够的任务，这些线程会永远阻塞在队里上。所以要一个线程退出了，去尝试唤醒一个空闲工作线程。
6. 如果是因为用户任务抛出异常、或工作线程数目少于核心线程数目个，创建一个工作线程。

## 10. shutdown()

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
  	// 1. 加全局锁。
    mainLock.lock();
    try {
        checkShutdownAccess();
      	// 2. 把RUNNING状态改为SHUTDOWN，其他状态不变。
        advanceRunState(SHUTDOWN);
      	// 3. 中断所有空闲的工作线程。
        interruptIdleWorkers();
      	// 4. 钩子函数。
        onShutdown(); 
    } finally {
      	// 5. 解锁。
        mainLock.unlock();
    }
  	// 6. 中断了线程，尝试终结线程池。
    tryTerminate();
}

private void advanceRunState(int targetState) {
    for (;;) {
        int c = ctl.get();
        if (runStateAtLeast(c, targetState) ||
            ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
            break;
    }
}

public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 1. 设置为STOP状态
        advanceRunState(STOP);
        // 2. 中断所有的工作线程（不管是否空闲）。
        interruptWorkers();
        // 3. 把任务队列中的任务都转移到 ArrayList 中。
        tasks = drainQueue();
    } finally {
      	// 4. 解锁。
        mainLock.unlock();
    }
    // 5. 中断了线程，尝试终结线程池。
    tryTerminate();
    return tasks;
}
```

## 11. awaitTermination

```java 
public boolean awaitTermination(long timeout, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 循环等待直到线程池状态更变为TERMINATED，每轮循环等待 timeout 时间
        while (runStateLessThan(ctl.get(), TERMINATED)) {
            if (nanos <= 0L)
                return false;
            nanos = termination.awaitNanos(nanos);
        }
        return true;
    } finally {
        mainLock.unlock();
    }
}
```

## 12. 总结

### 执行任务

![线程池execute](/images/线程池execute.png)

在新建线程的时候需要加锁，代价较大。

### 工作线程的生命历程

1. 工作线程循环从任务队列取任务(Runnable 实例）去执行。

   - 如果线程池状态为STOP、或SHUTDONW且任务队列空了，就退出。

   - 如果当前工作线程数目多于核心线程数目，如果 KeepAlive 时间内没有领到任务，就退出。

   - 如果用户任务抛出了异常，就退出。

2. 在退出的时候，工作线程数目少于核心线程数目、或是因为用户任务抛出异常，再创建一个工作线程。

通过这种做法，多于核心线程数目的工作线程，只会存活 KeepAlive 的空闲时间。

### 关闭线程池

#### shutdown()

1. 调用 `shutdown()` 后，线程池状态变为 SHUTDOWN。

2. 拒绝新的 `execute()` 任务。
3. 此时工作线程和任务队列有两种情况。
   - 任务队列是空的，说明有空闲的工作线程在阻塞。
     - 这些空闲线程被中断、唤醒，都进行退出工作。
   - 任务队列非空的，说明所有线程都在工作。
     - 他们会不断的认领任务去执行，任务队列空了之后，都进行退出工作。
     - 在任务队列非空之前，都会

4. 所有工作线程退出后，进入 TIDYING 状态。
5. 执行 `onShutdown()` 钩子函数后，进入 TERMINATED 状态。唤醒所有调用 `awaitTermination()` 的线程。

#### shutdownNow()

1. 调用 `shutdownNow()` 后，线程池状态变为 STOP。
2. 所有工作线程被中断、唤醒，不再领任务，都进行退出工作。
3. 所有工作线程退出后，进入 TIDYING 状态。
4. 执行 `onShutdown()` 钩子函数后，进入 TERMINATED 状态。唤醒所有调用 `awaitTermination()` 的线程。