---
title: 5. concurrenct 包中的锁
date: 2020-03-04 14:48:05
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../../../
---

# 5. concurrenct 包中的锁

## 1. Lock 接口

### 相比 synchronized 的优势

`concurrent` 包中的 Lock 接口，比 `synchronized` 灵活，有如下的优势：

- 可以非阻塞的获取锁
- 获取锁过程中可以被中断。
- 超时获取锁。
- 使用 `AbstractQueuedSynchronizer` (`AQS`) 自定义锁。

### 主要的方法

- `void lock()` 
- `void lockInterruptibly() throws InterruptedException` 可以被中断。
- `boolean tryLock()` 非阻塞获取锁，能够获得返回 true，否则 false。
- `boolean tryLock(long time, TimeUnit unit) throws InterruptedException` 非阻塞获取锁、可以被中断。 
- `void unlock()` 解锁。
- `Condition newCondition()` 获取等待通知组件，只有获取了锁后，才可以调用 `condition.await()` 和 `condition.signal()`。

### 使用方式

```java
Lock lock = new ReentrantLock();
// 获取锁不能在 try 里，否则获取失败、抛出异常时，还会去解锁。
lock.lock();
try {
    // 更新对象
    //捕获异常
} finally {
    lock.unlock();
}
```

## 2. 队列同步器 AbstractQueuedSynchronizer

### 简介

AQS 用来构建锁和其他同步组件。`ReentrantLock`、`ReentrantReadWriteLock` 和 `CountDownLatch`。

使用 `volatile int state` 表示同步的状态。

内置了 同步队列，我们只需要实现如何更新 `state`，就可以实现自定义的锁。

### 使用方式

1. 类 `MyLock` 实现 `Lock` 接口。
2. 类 `MyLock` 的内部类 `Sync` 继承 `AQS`，重写响应的方法。
3. 类 `MyLock` 中的方法调用 `Sync` 的方法。

#### AQS 可重写的方法

- `protected boolean tryAcquire(int arg)`
- `protected boolean tryRelease(int arg)`
- `protected int tryAcquireShared(int arg)`
  - 返回值小于0，获取锁失败。
  - 返回值等于0，获取锁失败，下一个线程获取锁应该会失败。
  - 返回值大于0，获取锁失败，下一个线程获取锁应该会成功。
- `protected boolean tryReleaseShared(int arg)`
- `protected boolean isHeldExclusively()` 是否在独占模式下被线程占用。

#### AQS 可以使用的方法

![image-20200226152753493](/Users/liuzihe/Library/Application Support/typora-user-images/image-20200226152753493.png)

- 独占模式获取锁
  - `public void acquire(int arg)`
  - `public void acquireInterruptibly(int arg)`
  - `public boolean tryAcquireNanos(int arg, long nanos)`
- 共享模式获取锁
  - `public void acquireShared(int arg)`
  - `public void acquireSharedInterruptibly(int arg)`
  - `public boolean tryAcquireSharedNanos(int arg, long nanos)`
- 释放锁
  - `public boolean release(int arg)`
  - `public boolean releaseShared(int arg)`
- 获取等待在同步队列上的线程集合
  - `public Collection<Thread> getQueuedThreads()`

### 三个要点

- 状态
  - 使用 `volatile int state` 来描述同步资源的状态
- 队列
  - 等待获取锁的线程都进入同步队列，每次只有头结点的后继节点会被唤醒，进行获取锁的尝试。
- CAS
  - 更新 `state`、头结点、尾节点，都是用的 CAS 完成。
  - 这里没有使用 AtomicXXX 相关类，而是直接用了 unsafe 方法，来提供性能。

## 3. AQS 独占锁的获取源码分析

### 引用

- [逐行分析AQS源码(1)——独占锁的获取](https://segmentfault.com/a/1190000015739343)

### acquire()

```java
public final void acquire(int arg) {
  if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}

static void selfInterrupt() {
  Thread.currentThread().interrupt();
}
```

1. 使用 `tryAcquire(arg)` 获取同步状态。
2. 如果失败了，构造独占式节点（Node.EXCLUSIVE），使用 `addWaiter()` 加入到同步队列的尾部。
3. 使用 `acquireQueued()` 在被唤醒时，死循环获取同步状态。
4. 因为是不可中断的，整个过程记录中断状态，在获取到同步状态后，如果被中断了，则调用 `Thread.currentThread().interrupt();` 中断线程。

### addWaiter()

```java
private Node addWaiter(Node mode) {
  Node node = new Node(Thread.currentThread(), mode);
  // Try the fast path of enq; backup to full enq on failure
  Node pred = tail;
  if (pred != null) { // 队列不为空
    node.prev = pred;
    if (compareAndSetTail(pred, node)) { // 添加节点到尾部
      pred.next = node;
      return node;
    }
  }
  enq(node);
  return node;
}

private Node enq(final Node node) {
  for (;;) {
    Node t = tail;
    if (t == null) { // Must initialize 添加一个空的头结点
      if (compareAndSetHead(new Node()))
        tail = head;
    } else {
      node.prev = t;
      if (compareAndSetTail(t, node)) {
        t.next = node;
        return t;
      }
    }
  }
}
```

1. 如果队列不为空、且 CAS 添加节点到尾部成功，则直接返回。
2. 否则，调用 `enq` 来把节点加入到队列尾部。
   1. 如果队列为空，添加一个空的头结点，**continue**。
   2. 如果队列不为空、且 CAS 添加节点到尾部成功，则返回。

###  acquireQueued()

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      final Node p = node.predecessor();
      if (p == head && tryAcquire(arg)) {	// 获取到锁了
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
      }
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
} 
```

1. 如果前驱节点为头结点、且获取同步状态成功，说明获取到锁了。直接设置节点为头结点，返回。注意不用 CAS，因为独占模式下，只有一个节点能获取到锁。
2. 否则，使用 `LockSupport.park(this);` 阻塞线程，并记录中断状态。
3. 在被唤醒的时候，从步骤 1 开始重复。

### shouldParkAfterFailedAcquire()

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
    // 如果前驱节点的状态已经设为 SIGNAL，自己可以直接睡了。
    return true;
  if (ws > 0) {
    // 如果前驱节点状态是 CANCELLED，向前找节点，舍弃 CANCELLED 节点。
    do {
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
  } else {
    // 前驱结点状态是 0，说明没有设置过，CAS 更新为 SIGNAL。
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}

// 线程所处的等待锁的状态，初始化时，该值为0
volatile int waitStatus;
static final int CANCELLED =  1;
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;
```

节点的 `waitStatus`，SIGNAL 表示后继节点被挂起了（马上要被挂起了）。

- 一个节点在陷入阻塞状态前，需要把前驱结点的状态设为 SIGNAL。以便自己之后可以被唤醒。

- 在节点释放锁时，如果节点状态是 SIGNAL，需要把后继节点唤醒。

函数 `shouldParkAfterFailedAcquire()` 就是用来做这件事的。

- 如果前驱节点的状态已经设为 SIGNAL，自己可以直接睡了。返回 true。

- 如果前驱节点状态是 CANCELLED，说明前驱节点被取消了，向前找节点，舍弃 CANCELLED 节点。返回 false。

- 前驱结点状态是 0，说明没有设置过，CAS 更新为 SIGNAL。返回 false。

## 4. AQS 独占锁的释放源码分析

### 引用

- [逐行分析AQS源码(2)——独占锁的释放](https://segmentfault.com/a/1190000015752512)

### release()

```java
public final boolean release(int arg) {
  if (tryRelease(arg)) {
    Node h = head;
    if (h != null && h.waitStatus != 0)
      unparkSuccessor(h);
    return true;
  }
  return false;
}
```

1. 释放同步资源状态。
2. 如果同步队列不为空、且头结点的 `waitStatus` 不为 0（头结点的后继节点在等待被唤醒），则使用 `unparkSuccessor()` 唤醒头结点的后继节点。

### unparkSuccessor()

```java
private void unparkSuccessor(Node node) {
  int ws = node.waitStatus;
  // 把节点的 waitStatus 清 0。
  if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);

  Node s = node.next;
  // 如果使用 node.next 查看，没有后继节点、或后继节点为 CANCELLED 状态，
  // 从尾节点反向遍历，找到距离 node 最近的没有被 CANCELLED 的节点。
  if (s == null || s.waitStatus > 0) {
    s = null;
    for (Node t = tail; t != null && t != node; t = t.prev)
      if (t.waitStatus <= 0)
        s = t;
  }
  // 如果找到后面需要唤醒的节点，唤醒它。
  if (s != null)
    LockSupport.unpark(s.thread);
}
```

1. 把节点的 waitStatus 清 0。
2. 如果使用 node.next 查看，没有后继节点、或后继节点为 CANCELLED 状态。从尾节点反向遍历，找到距离 node 最近的没有被 CANCELLED 的节点。
3. 如果找到后面需要唤醒的节点，唤醒它。

**之所以从尾部向前找**，是因为节点入队并不是原子操作。在 CAS 入队后，只保证了节点是尾节点、指向前一个尾节点，但是可能这个时候可能还没有设置前一个尾节点的 `next` 值。

```java
private Node enq(final Node node) {
  for (;;) {
    Node t = tail;
    if (t == null) { // Must initialize 添加一个空的头结点
      if (compareAndSetHead(new Node()))
        tail = head;
    } else {
      node.prev = t;	// 1							
      if (compareAndSetTail(t, node)) {	// 2
        t.next = node;	// 3
        return t;
      }
    }
  }
}
```

## 5. AQS 共享锁的获取源码分析

### 引用
- [逐行分析AQS源码(3)——共享锁的获取与释放](https://segmentfault.com/a/1190000016447307)

### acquireShared()

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

1. 尝试获取同步资源，成功则返回。
2. 失败调用 `doAcquireShared(arg);`。

### doAcquireShared()

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    /*boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();*/
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            /*if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }*/
}

// Node#isShard()
final boolean isShared() {
    return nextWaiter == SHARED;
}
```

相同的部分注释掉了。

可以看到和独占锁的获取只有两处不同：

1. 新建的节点类型为 `SHARED`，用来之后判断同步队列中的节点是否是 共享的。
2. 在获取同步资源成功后，使用 `setHeadAndPropagate()`，而不是 `setHead()`。
   - 除了更新自己为头结点，还要唤醒后继节点。

### setHeadAndPropagate()

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);

    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

1. 调用 `setHead()` 更新自己为头结点。
   - 这里不用 CAS 设置，因为在获取共享锁的时候，也只有自己是头结点的后继节点才能进入这里，也就是说获取锁是“串行”调用 `setHead()` 的。
2. 如果 `tryAcquireShared()` 大于0，自己的 `waitStatus` 不是 0 和 CANCELLED，自己的后继节点存在且为共享模式，则调用 `doReleaseShared()` 唤醒后继节点。

## 5. AQS 共享锁的释放源码分析

### 引用

- [逐行分析AQS源码(3)——共享锁的获取与释放](https://segmentfault.com/a/1190000016447307)

### releaseShared()

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

1. 释放同步资源。
2. 调用 `doReleaseShared()` 来唤醒后继节点。

### doReleaseShared()

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) { // 队列至少有两个节点
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

如果队列至少有两个节点，并且头结点的 `waitStatus` 为 SIGNAL，则唤醒头结点的后继节点。

注意，因为可能有多个线程在执行这个函数，在唤醒前，要先 CAS 使头结点的状态清0成功，才可以唤醒头结点的后继节点。

## 6. ReentrantLock 重入锁

### 状态

`state` 值表示锁被获取的次数。

### 非公平锁获取

```java
final boolean nonfairTryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {
    if (compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0) // overflow
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
```

1. 如果 `state` 为 0、且 CAS 把 `state` 设为 1 成功，则获取锁成功，把当前线程设为 `exclusiveOwnerThread`。
2. 如果 `state` 不为0、但是 `exclusiveOwnerThread` 为当前线程，则获取锁成功，把 `state` 加 1。

翻译一下：

1. 如果没有人持有锁、且当前线程获取锁成功，把当前线程设为 `exclusiveOwnerThread`。
2. 如果是自己线程持有锁，`state` 加 1。

### 公平锁获取

```java
// FairSync
protected final boolean tryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {
    if (!hasQueuedPredecessors() &&
        compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0)
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
```

和非公平锁的获取相比，需要满足同步队列为空、或头节点是当前线程，才可以去 CAS 尝试获取锁。

这样每个线程都必须先入同步队列（除了队列为空时），才有可能获取锁。

#### 与非公平锁对比

- 对于非公平锁，假设线程 A 持有锁、线程 B 在同步队列中，在 A 释放锁时，线程 C 请求获取锁，可能会比同步队列的头结点 B 更早获取到锁。

- 对于公平锁，线程 C 无法在队列为空时获取到锁。

## 7. ReentrantReadWriteLock 读写锁

### 状态

`state` 前 16 位表示获取读锁的次数。

`state` 后 16 位表示获取写锁的次数。

### 写锁的获取

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        if (w == 0 || current != getExclusiveOwnerThread()) // 有人持有读锁、或持有写锁线程不为自己
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
  	// writerShouldBlock() 
  	//		非公平锁 返回 false
    //		公平锁，返回 hasQueuedPredecessors()
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

1. 如果有人持有读锁，返回 false。
2. 如果有人持有写锁，
   1. 不是自己线程持有，返回 false。
   2. 是自己持有，`state` 加 1，返回 true。
3. 否则，没有人持有锁（`state == 0`）
   1. 如果是非公平锁，尝试 CAS 加锁。
   2.  如果是公平锁，同步队列为空、或头结点是自己线程才可以尝试 CAS 加锁。

### 读锁的获取

可以使用 `readHoldCount()` 来获取本线程持有读锁的次数，使用 `ThreadLocal` 来计数。

### 锁降级

持有写锁的线程，可以在释放写锁之前获取读锁。

这样使整个读写过程都是本线程持有锁的。

否则先释放写锁再获取读锁的话，在释放后可能其他线程获取了读锁，使得线程读到的值不是本线程写的值。

## 8. LockSupport

- `LockSupport.park()` 阻塞当前线程。
  - 可以被中断，中断后 `Thread.interrupted()` 返回 true。
- `LockSupport.unpark(thread)` 
  - 如果 thread 被 `park` 阻塞了，唤醒它。
  - 如果 thread 没有被 `park`，下次 `park` 不会阻塞线程。

## 9. Condition

### 数据结构

```java
public class ConditionObject implements Condition, java.io.Serializable {
    /** First node of condition queue. */
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    private transient Node lastWaiter;
}
```

每个 `Conditoin` 对象都存储了自己的等待队列的链表。

### await()

```java
public final void await() throws InterruptedException {
  if (Thread.interrupted())
    throw new InterruptedException();
  Node node = addConditionWaiter(); 	// 把当前线程加入到等待队列中。
  int savedState = fullyRelease(node);	// 释放锁资源，唤醒后继节点。
  int interruptMode = 0;
  while (!isOnSyncQueue(node)) {
    LockSupport.park(this);
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
  if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters();
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```

1. 把当前线程加入到等待队列中。
2. 释放锁资源，唤醒后继节点。
3. 使用 `LockSupport.park()` 陷入阻塞状态。
4. 在被唤醒后，调用 `acquireQueued` 方法，尝试获取锁资源。
5. 被中断，会抛出 InterruptedException。

### signal()

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

private void doSignal(Node first) {
  do {
    if ( (firstWaiter = first.nextWaiter) == null)
      lastWaiter = null;
    first.nextWaiter = null;
  } while (!transferForSignal(first) && (first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) {
  /*
   * If cannot change waitStatus, the node has been cancelled.
   */
  if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
    return false;

  /*
   * Splice onto queue and try to set waitStatus of predecessor to
   * indicate that thread is (probably) waiting. If cancelled or
   * attempt to set waitStatus fails, wake up to resync (in which
   * case the waitStatus can be transiently and harmlessly wrong).
   */
  Node p = enq(node);
  int ws = p.waitStatus;
  if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
    LockSupport.unpark(node.thread);
  return true;
}
```

1. 当前线程必须持有锁。
2. 把等待队列的头节点移除、加入到同步队列中。
3. 使用 `LockSupport.unpark()` 唤醒节点中的线程。

### signalAll()

1. 会把等待队列的所有节点移除、加入到同步队列中。
2. 唤醒所有节点中的线程。

