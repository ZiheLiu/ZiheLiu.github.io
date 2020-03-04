---
title: 6. ConcurrentHashMap 源码分析
date: 2020-03-04 14:48:06
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../../../
---

# 6. ConcurrentHashMap 源码分析

## 0. 引用

- [JUC源码分析-集合篇（一）ConcurrentHashMap](https://www.cnblogs.com/binarylei/p/10921214.html)
- [probable bug in logic of ConcurrentHashMap.tryPresize()](https://bugs.openjdk.java.net/browse/JDK-8215409)
- [value of 'sizeCtl' in ConcurrentHashMap varies with the constructor called](https://bugs.openjdk.java.net/browse/JDK-8202422)

## 1. 介绍

JDK 1.7 中，`ConcurrentHashMap` 使用分段锁 `ReentrantLock` 实现；

而在 JDK 1.8 中，使用 CAS + `synchronized` 锁每个桶的第一个节点实现。

### 为什么换了实现方式

1. JDK 1.8 中的 `synchronized` 已经足够优化。
2. 锁细化到每个桶一个，并发度更高。
3. 锁细化到每个桶一个，出现锁竞争的概率很小，`synchronized` 很小概率会膨胀为重量级锁。
4. `ReentrantLock` 尝试加锁失败就会加入等待队列，`synchronized` 首次进入重量级锁也会尝试自旋若干次。

###  volatile + CAS

#### table

```java

transient volatile Node<K,V>[] table;

private transient volatile Node<K,V>[] nextTable;

static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```

虽然 `table` 声明为 `volatile` 的数组，但是只是这个数组的引用为 `volatile` 的。

- 即只有 `table = XXX` 时生效。
- 而为数组的元素赋值并不保证 `volatile` 的效果。
- 所以使用了` Unsafe` 的 `volatile` 的相关函数来从 `table` 赋值和取值。

#### Node

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
  
    Node(int hash, K key, V val, Node<K,V> next) {
      this.hash = hash;
      this.key = key;
      this.val = val;
      this.next = next;
    }
}
```

- `key` 和 `hash` 是不会改变的，使用 `final` 就可以保证可见性、避免重排序。

- `val` 和 `next` 是会改变的，所以需要使用 `volatile` 保证 可见性、避免重排序。

### key 和 value 不能是 null

> - [Why does ConcurrentHashMap prevent null keys and values?](https://stackoverflow.com/questions/698638/why-does-concurrenthashmap-prevent-null-keys-and-values)

如果 `value` 可以是 null，在 `map.get(key)` 返回 null 的时候，不知道 map 中是否包含这个元素。

在 `HashMap` 中可以用 `map.containsKey(key)` 看这个 key 是否存在。

但是在多线程版本中，用 `map.containsKey(key)` 和 `map.get(key)` 不是原子的。

## 2. 构造函数

```java
public ConcurrentHashMap() {
}

// 2
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

// 3
public ConcurrentHashMap(int initialCapacity) {
  this(initialCapacity, LOAD_FACTOR, 1);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
  this(initialCapacity, loadFactor, 1);
}

public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
  if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
    throw new IllegalArgumentException();
  if (initialCapacity < concurrencyLevel)   // Use at least as many bins
    initialCapacity = concurrencyLevel;   // as estimated threads
  long size = (long)(1.0 + (long)initialCapacity / loadFactor);
  int cap = (size >= (long)MAXIMUM_CAPACITY) ?
    MAXIMUM_CAPACITY : tableSizeFor((int)size);
  this.sizeCtl = cap;
}
```

- `sizeCtl` 与 HashMap 中的 `threshold` 含义类似。
  - 数组未初始化时表示需要初始化数组的大小，
  - 为 -1 时，表示正在初始化。
  - 为 整数时，为扩容的阈值。
  - 为极小的负数时，为正在 transfer 的线程数目。
- 第 2 个构造函数中，`sizeCtl` 这里设置为了 `initialCapacity * 1.5`，应该设置为 `initialCapacity / 0.75` 才对，是一个 bug。https://bugs.openjdk.java.net/browse/JDK-8202422。JSR 166 中，更新为第 3 个构造函数的版本。
- 没有初始化 `table`。

## 3. put()

```java
public V put(K key, V value) {
  	return putVal(key, value, false);
}

static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
// 与 HashMap 中有些不一样，去除了最高位的1。
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
  	// 1. 自旋保证元素一定会添加到 map 中。
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
      	// 1.1 如果没有初始化 table，则初始化。
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
      	// 1.2 如果桶空的、且 CAS 添加节点成功，退出循环。
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
      	// 1.3 如果这个桶正在扩容，该线程先帮助扩容、再添加节点。
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
      	// 1.4 锁住这个桶，进行插入/更新操作，与 HashMap 类似。
        else {
            V oldVal = null;
          	// 为这个桶加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
              			// 1.4.1 链表
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                          	// 1.4.1.1找到了对应 key 的节点，更新值。
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                          	// 1.4.1.2 没找到，创建新的节点。
                            if ((e = e.next) == null) { 
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                  	// 1.4.2 红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
          
          	// 1.5 判断链表长度
            if (binCount != 0) {
              	// 如果链表长度大于 8，进行扩容或变为红黑树。
              	// 如果本来就是红黑树，binCount 固定为2，不会进到这个分支。
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
  
 		// 2. 元素个数加1，并判断是否扩容
    addCount(1L, binCount);
    return null;
}
```

主要的流程：

- 在第一次 put 时，初始化 table。
- 桶空的时候，CAS 尝试添加节点。
- 否则说明桶有节点，对桶首节点加锁，进行链表/红黑树的插入/更新节点的操作。
- 如果桶在进行扩容，要先帮助扩容，再添加节点。
- 如果是新增节点，元素个数加 1，判断是否扩容。

还需要注意以下操作的线程安全性：

- `initTable()` 初始化 table。
- `helpTransfer()` 帮助扩容。
- `addCount()` 增加元素。

### initTable()

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // 初始化前，CAS 把 sizeCtl 设为 -1，使得只有一个线程可以进入这个分支。
      	else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                 		// 数组大小为 sizeCtl
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n]; 
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
              	// sizeCtl 设为数组大小的 3/4
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

- CAS 把 sizeCtl 设为 -1，使得只有一个线程在初始化 table。
- 初始化后，数组长度为构造函数传入的初始容量、或默认值16.
- 初始化后，阈值 sizeCtl 为数组长度的 3/4。

###  treeifyBin()

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
      	// 1 如果数组大小小于 64，进行扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
      	// 2 否则锁住桶，把链表转为红黑树。
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                  	// 节点的 hash 值为 -2
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

- 把链表转为红黑树时，要把桶加锁。
- 节点的 hash 值
  - -1 表示正在 MOVE（扩容，拆分链表）
  - -2 表示是红黑树
  - 大于等于 0，存储的是链表的节点。

### tryPresize()

```java
// putVal() 调用时，传入的 size 是数组长度的两倍。
// putAll() 调用时，传入的是要放入元素的个数。
private final void tryPresize(int size) {
  	// size 的 1.5 倍，再变为 2 的次幂。
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
      	// 1. 数组没有初始化先初始化，和数组 initTable 类似
        //    putAll 时可能数组还未初始化
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
      	// 2. 数组足够大了，不需要扩容，直接退出。
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            int rs = resizeStamp(n);
          	// 这个分支与上层 while 条件互斥，不可能进入，是 bug。JDK 11 去掉了这个分支。
          	// https://bugs.openjdk.java.net/browse/JDK-8215409
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
          	// 第一次进行 transfer 时 + 2，
           	// 之后进行 transfer 时 + 1，
          	// 完成 transfer 时 - 1。因此，所有线程都完成时，sc 为 (rs << RESIZE_STAMP_SHIFT) + 1。
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}

static final int resizeStamp(int n) {
    // Integer.numberOfLeadingZeros(n) 表示 n 转二进制后最高位 1 之前的 0 个数，这个数一定小于 32
    // 1 << (RESIZE_STAMP_BITS - 1) 表示 0x7fff，也就是将低 16 位的第一位改成 1
    // 最终结果为 0000 0000 1000 0000 000x xxxx 
  	// 这样在 tryPresize() 中，resizeStamp(n) << 16，最高位为1，一定是一个负数，并且是极小的负数。
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
private static final int RESIZE_STAMP_SHIFT = 16;
```

- 如果数组没有初始化先进行初始化。
- 只有第一个 CAS `sizeCtl` 为 (rs << RESIZE_STAMP_SHIFT) + 2  的线程可以进行  transfer 扩容。
- 帮助扩容的线程是通过 `helpTransfer()` 进入的（在 `putVal()` 时，发现对应的桶在扩容触发），每个帮助的线程会把 `sizeCtl `+ 1。

### helpTransfer()

```java
// putVal() 时，发现桶在 transfer，调用 helpTransfer() 来帮助扩容。
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
     
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
          	// 1 如果 sc 的高 16 位不等于标识符(说明sizeCtl已经改变，扩容已经结束)
         		// 2 如果 sc == 标识符 + 1 （扩容结束了，不再有线程进行扩容）
            // 3 如果 sc == 标识符 + 65535（辅助扩容线程数已经达到最大）
            // 4 如果 transferIndex <= 0 (转移状态变化了)
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
          	// sizeCtl 加 1，来使该线程参与到扩容中。
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

### transfer()

第一个 `transfer()` 的线程传入的是 `transfer(tab, null)`。

之后传入的是 `transfer(tab, nextTab)`。根据第二个参数判断是不是第一次 `transfer()`。

`transferIndex` 记录了扩容任务进行到的下标，从右向左处理。

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 1. stride 可以理解为”步长“，有 n 个位置是需要进行迁移的
    //    将这 n 个任务分为多个任务包，每个任务包有 stride 个任务
    //    stride 在单核下直接等于 n，多核模式下为 (n>>>3)/NCPU，最小值是 16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range

    // 2. 如果 nextTab 为 null，是第一次进入。先进行一次初始化。
    if (nextTab == null) {            // initiating
        try {
          	// 数组长度加倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
      	//     transferIndex初始值为table.length
        transferIndex = n;
    }
    int nextn = nextTab.length;

    // 3. ForwardingNode 是占位用的，标记该节点已经处理过了
    //    这个构造方法会生成一个 Node，hash 为 MOVED (-1)，
  	//			nextTab 缓存如Node。以便之后来帮忙的线程使用。
    //    后面我们会看到，原数组中位置 i 处的节点完成迁移工作后，
    //    就会将位置 i 处设置为这个 ForwardingNode，用来告诉其他线程该位置已经处理过了
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
  	// 下边的while循环用来找到第一个没有处理的节点，
  	// advance为false时，表示找到了。
  	// advance为true时，说明当前节点已经处理完了，需要处理下一个节点。
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab

    // i 每次任务的上边界，bound 是下边界，注意是从后往前
    // --i < bound 时就领取一次的任务，直到任务处理完毕
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 4. advance 为 true 表示可以进行下一个位置的迁移了
        //    第一次while循环：当前线程领取任务，走第三个else(一直自旋尝试领取任务)
        //          最终的结果为：i 指向了 transferIndex，bound 指向了 transferIndex-stride
        //    之后每处理完一个节点：走第一个if，处理的下一个槽位的节点，
      	//		直到当前线程领取的任务处理完毕， 再次走第三个else，领取步长stride的任务直到transferIndex<=0
        while (advance) {
            int nextIndex, nextBound;
            // 4.1 --i表示处理一下槽位的节点
            if (--i >= bound || finishing)
                advance = false;
            // 4.2 当前任务包处理完了，看是否还有节点没处理。
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 4.3 自旋尝试领取步长stride的任务包。
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                // nextBound 是这次迁移任务的下边界，注意，是从后往前
                bound = nextBound;
                // nextIndex 是这次迁移任务的上边界
              	// 因为找到了一个要处理的节点，会先跳出while处理
              	// 下次要处理的是 i = nextIndex - 1。
                i = nextIndex - 1;
                advance = false;
            }
        }
        // 5. 如果扩容结束了。
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 5.1 完成扩容
            if (finishing) {                    // 完成扩容
                nextTable = null;
                table = nextTab;                // 更新 table
                sizeCtl = (n << 1) - (n >>> 1); // 更新阈值为新数组长度的 3/4
                return;
            }
            // 5.2 扩容结束，把 sizeCtl - 1。
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 不相等说明有其它线程在辅助扩容，当前线程直接返回。
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                // 相等说明没有线程在帮助他们扩容了。也就是说，扩容结束了。
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 6. 如果位置 i 处是空的，没有任何节点，那么放入刚刚初始化的 ForwardingNode ”空节点“
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 7. 该位置处是一个 ForwardingNode，代表该位置已经迁移过了
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        // 8. 数组迁移，原理和 HashMap 一样
        else {
            // 对数组该位置处的结点加锁，开始处理数组该位置处的迁移工作
            synchronized (f) {
                ...
                advance = true;
            }
        }
    }
}
```

## size()

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int)n);
}

// sumCount 用于统计当前的元素个数
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

计数并没有使用一个 `AtomicLong`，这样会使每次操作都有竞争，降低并发。

### addCount()

```java
//基础值，没有竞争时会使用这个值
private transient volatile long baseCount;
//存放Cell的hash表，大小为2的幂
private transient volatile CounterCell[] counterCells;
// 通过cas实现的锁，0无锁，1获得锁
private transient volatile int cellsBusy;

private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 1. 统计元素个数
    // 1.1 baseCount 只有 counterCells 还没有初始化时才使用
  	//		如果baseCount CAS 成功，则统计元素个数结束。
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        // 1.1 uncontended表示是否存在锁竞争
        boolean uncontended = true;
        // 1.2 将线程分流到 CounterCell[] 数组中，使用更小粒度的 cas 操作
        //     (1) CounterCell[]未初始化
        //     (2) CounterCell[]中的节点未初始化
        //     (3) 对 as[random] 进行 cas 操作，失败则存在锁竞争问题uncontended=false，成功直接返回
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
        }
        if (check <= 1)
            return;
        // 1.3 统计当前元素的个数
        s = sumCount();
    }
    // 2. 扩容操作，与 tryPresize()、helpTransfer() 类似。
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            // sc<0 表示有其它线程在扩容
            if (sc < 0) {
                // 扩容已经结束
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 加入到扩容中，扩容的线程数+1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 当前线程发起扩容，注意线程数是直接+2
            else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

使用 `CounterCell[]` 数组来分散 元素数量变化时候的竞争。

1. 每次计数，根据 `Thread.probe` 映射到一个  counter  中去处理，使用 CAS 尝试计数。

2. 如果存在竞争，且 counter 数组长度小于 CPU 核数，申请volatile锁，加倍 counter 数组长度。
3. 如果在 counter CAS 尝试失败、长度小且申请volatile锁失败，对 baseCount 尝试 CAS 计数。

## 5. 扩容总结

### 触发实际

- 在添加节点 `addCount()` 后，如果元素数目超过阈值，会进行 `tansfer` 扩容。

- 在一个桶的元素数目超过8个、且table总长度小于64时，会进行 `tryPresize` 扩容。
- 在添加节点时，发现对应的桶在进行扩容，会进行帮助扩容 `helpTransfer`。

其中 `tryPresize()` 和 `helpTransfer` 最后都会调用 `transfer()`。

### 扩容过程

- 每个线程，每次申请扩容 `stride` 个桶，从右到左进行扩容。

- 使用 `transferIndex` 记录还没有线程认领进行扩容的最右边的位置。

- 会把处理结束的桶标记为 `ForwardingNode` 类型的节点，其中存储了 `nextTable`，即扩容后新的数组。

- 在第一次进行扩容（调用 `transfer()` 时），
  - 设置`transferIndex` 为旧数组长度。
  - 创建 `nextTable`，长度为旧数组长度两倍。
- 在所有扩容结束后
  - 设置 `sizeCtl` 阈值 为新数组长度的 3/4。

## 6. get()

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

和 HashMap 类似，

1. 先看桶的首节点的 key
2. 如果桶中是红黑树，在红黑树里找。
3. 否则在链表中找。