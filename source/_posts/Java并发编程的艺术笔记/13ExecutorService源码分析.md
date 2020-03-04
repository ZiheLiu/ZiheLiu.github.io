---
title: 13. ExecutorService 源码分析
date: 2020-03-04 14:48:15
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../
---

# 13. ExecutorService 源码分析

## 0. 引用

- [[JUC线程池服务ExecutorService接口实现源码分析](http://www.throwable.club/2019/07/27/java-concurrency-executor-service/)](http://www.throwable.club/2019/07/27/java-concurrency-executor-service/)

## 1. FutureTask

### 主要接口

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
   
    void run();
}

@FunctionalInterface
public interface Runnable {
    
    public abstract void run();
}    

public interface Future<V> {
    
    // 取消任务。
  	//		1. mayInterruptIfRunning 用于控制是否要中断任务。
  	//				注意，任务可能不响应中断。
  	//		2. 会唤醒所有调用 get() 的线程。
  	//		3. 调用 get() 的线程抛出 CancellationException。
    boolean cancel(boolean mayInterruptIfRunning);
    
    // 任务是否被取消了。
  	//	状态是否为 CANCELLED、INTERRUPTING、INTERRUPTED。
    boolean isCancelled();
    
    // 任务是否结束了.
  	//	包括正常和异常的情况。
  	//	状态是否不等于 NEW。
    boolean isDone();
    
    // 永久阻塞获取结果，响应中断
    V get() throws InterruptedException, ExecutionException;
    
    // 带超时的阻塞获取结果，响应中断
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
### 构造函数

`FutureTask` 一般是 `AbstractExecutorService` 在 `submit()` 时候创建的。

```java
public class FutureTask {
  // 适配使用Callable类型任务的场景
  public FutureTask(Callable<V> callable) {
      if (callable == null)
          throw new NullPointerException();
      this.callable = callable;
      this.state = NEW;       
  }

  // 适配使用Runnable类型任务和已经提供了最终计算结果的场景
  public FutureTask(Runnable runnable, V result) {
      this.callable = Executors.callable(runnable, result);
      this.state = NEW;      
  }
}

public interface Callable<V> {
    V call() throws Exception;
}

public class Executors {
  // 将 Runnable 包装为 Callable，返回值是传入的 result。
  public static <T> Callable<T> callable(Runnable task, T result) {
      if (task == null)
          throw new NullPointerException();
      return new RunnableAdapter<T>(task, result);
  }

  // Runnable 到 Callable 的适配器。
  private static final class RunnableAdapter<T> implements Callable<T> {
      private final Runnable task;
      private final T result;
      RunnableAdapter(Runnable task, T result) {
          this.task = task;
          this.result = result;
      }
      public T call() {
          task.run();
          return result;
      }
      public String toString() {
          return super.toString() + "[Wrapped task = " + task + "]";
      }
  }
}
```

### 状态转化

![FutureTask状态转化](/images/FutureTask状态转化.svg)

### 调用函数的执行流程

![FutureTask执行流程](/images/FutureTask执行流程.svg)

- 内部使用链表组成的栈，来记录调用 `get()` 的线程。
- 使用 CAS 出栈和入栈。
- 使用 `LockSupport.park()/unpark()` 来阻塞和唤醒调用 `get()` 的线程。
- `finishCompletion()` 最后会执行 `done()` 钩子函数。

## 2. AbstractExecutorService

### submit()

```java
// ThreadPoolExecutor 的父类。
public abstract class AbstractExecutorService implements ExecutorService {
    // 静态工厂方法，用 Runnable 实例、提前设好的返回结果，创建 FutureTask 实例
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }

    // 静态工厂方法，用 Callable 实例，创建 FutureTask 实例。
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    
    // 提交Runnable类型任务，get()的返回值为null。
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        // 适配任务为FutureTask实例，注意最终计算结果已经提前设置为null
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        // 提交到线程池
        execute(ftask);
        return ftask;
    }
    
    // 提交Runnable类型任务，get()的返回值为传入的result。
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        // 适配任务为FutureTask实例
        RunnableFuture<T> ftask = newTaskFor(task, result);
        // 提交到线程池
        execute(ftask);
        return ftask;
    }
    
    // 提交Callable类型任务，get()的返回值为计算的结果。
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
         // 适配任务为FutureTask实例
        RunnableFuture<T> ftask = newTaskFor(task);
        // 提交到线程池
        execute(ftask);
        return ftask;
    }
}
```

`submit(Callable task)` 中，

1. 任务提交到 `execute(futureTask)` 后，
2. 工作线程领到这个任务，开始执行 `futureTask.run()`。
3. `futureTask.run()` 中会执行 `futureTask.callable.call()`。

这样工作线程就执行到了传入的 `Callable` 的 `task`。

### cancelAll()

```java
// 取消所有的Future实例
private static <T> void cancelAll(ArrayList<Future<T>> futures) {
  cancelAll(futures, 0);
}

// 遍历所有的Future实例调用其cancel方法，因为参数为true，所以会响应中断
// j参数是决定遍历的起点，0表示整个列表遍历
private static <T> void cancelAll(ArrayList<Future<T>> futures, int j) {
  for (int size = futures.size(); j < size; j++)
    futures.get(j).cancel(true);
}
```

循环遍历传入的 FutureTask 的实例列表，调用每一个实例的 `cancel(true)` 方法。

### invokeAny()

#### 语法

`<T> T invokeAny(Collection<? extends Callable<T>> tasks)` 函数会执行列表中的任意一个任务。可能会执行多个任务，但是只会返回第一个执行完的结果。

#### 基本原理

- 使用一个 `BlockingQueue`，在任务执行完后，会放入队列。
- 遍历 `tasks`，如果非阻塞出队失败（即还没有任务执行完），就 `submit()` 当前遍历到的 `task`。
- 如果都遍历完了，就阻塞出队。

#### 实际的实现

- 使用一个内部类 `ExecutorCompletionService`，它内置了一个 `LinkedBlockingQueue completionQueue`。

- 提交的任务，会使用 `FutureTask` 的子类 `QueueingFuture` 包装。
  - 使用钩子函数 `done()`，在任务执行完成后，把它放入 `completionQueue` 中。

```java
public class ExecutorCompletionService<V> implements CompletionService<V> {
    private final Executor executor;
    private final AbstractExecutorService aes;
    private final BlockingQueue<Future<V>> completionQueue;
  
    public ExecutorCompletionService(Executor executor) {
        if (executor == null)
            throw new NullPointerException();
        this.executor = executor;
        this.aes = (executor instanceof AbstractExecutorService) ?
            (AbstractExecutorService) executor : null;
        this.completionQueue = new LinkedBlockingQueue<Future<V>>();
    }
  
    private class QueueingFuture extends FutureTask<Void> {
      QueueingFuture(RunnableFuture<V> task) {
        super(task, null);
        this.task = task;
      }
      protected void done() { completionQueue.add(task); }
      private final Future<V> task;
    }
  
    public Future<V> submit(Callable<V> task) {
      if (task == null) throw new NullPointerException();
      RunnableFuture<V> f = newTaskFor(task);
      executor.execute(new QueueingFuture(f));
      return f;
    }
  
  	public Future<V> take() throws InterruptedException {
        return completionQueue.take();
    }

    public Future<V> poll() {
        return completionQueue.poll();
    }

    public Future<V> poll(long timeout, TimeUnit unit)
            throws InterruptedException {
        return completionQueue.poll(timeout, unit);
    }
}
```

### invokeAll()

`<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)` 函数会执行列表中的所有任务。

- 遍历任务列表，`submit()` 每个任务得到 `futureTask` 列表。
- 遍历  `futureTask` 列表，执行每个 `futureTask` 的 `get(0)` 方法。

