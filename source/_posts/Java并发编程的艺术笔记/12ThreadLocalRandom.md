---
title: 12. ThreadLocalRandom
date: 2020-03-04 14:48:14
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../
---

# 12. ThreadLocalRandom

## 1. Random

`Random` 类在生成随机数时，所有线程 CAS 竞争同一个 `seed`。

```java
private final AtomicLong seed;

protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
```

## 2. ThreadLocalRandom

```java
package java.util.concurrent;
public class ThreadLocalRandom {
    public static ThreadLocalRandom current() {
        if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
            localInit();
        return instance;
    }

    static final void localInit() {
        int p = probeGenerator.addAndGet(PROBE_INCREMENT);
        int probe = (p == 0) ? 1 : p; // skip 0
        long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
        Thread t = Thread.currentThread();
        UNSAFE.putLong(t, SEED, seed);
        UNSAFE.putInt(t, PROBE, probe);
    }
  
  	// 使用 probe 作为 hash 值，出现竞争时，调整 probe。
    static final int advanceProbe(int probe) {
        probe ^= probe << 13;   // xorshift
        probe ^= probe >>> 17;
        probe ^= probe << 5;
        UNSAFE.putInt(Thread.currentThread(), PROBE, probe);
        return probe;
    }
  
    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long SEED;
    private static final long PROBE;
    private static final long SECONDARY;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> tk = Thread.class;
            SEED = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSeed"));
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
            SECONDARY = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}

package java.util;
public class Thread {
    /** The current seed for a ThreadLocalRandom */
    @sun.misc.Contended("tlr")
    long threadLocalRandomSeed;

    /** Probe hash value; nonzero if threadLocalRandomSeed initialized */
    @sun.misc.Contended("tlr")
    int threadLocalRandomProbe;

    /** Secondary seed isolated from public ThreadLocalRandom sequence */
    @sun.misc.Contended("tlr")
    int threadLocalRandomSecondarySeed;
}
```

- 每个线程第一次调用 `current()` 会分配 `seed` 和 `probe` 给 `Thread`。

  - 注意，`Thread` 中的 `seed` 和 `probe` 是包访问权限的，而 `ThreadLocalRandom` 与其不在一个包，所以需要使用 unsafe 方法更新他们。

- `probe` 的作用。

  - 初始化标识，来表示是否初始化了线程 `ThreadLocalRandom` 相关的域。
  - 作为线程的内部 hash 值，在 ConcurrentHashMap 等类中需要把 Thread 映射为下标的时候使用。避免碰撞。因为 `Thread.hashCode()` 返回的是 `Thread` 对象的地址，没有 `probe` 的值均匀。
  - 使用 probe 作为 hash 值，出现竞争时，使用 `advanceProbe()`，调整 probe。

  