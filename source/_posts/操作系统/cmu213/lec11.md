---
title: CMU213 学习笔记-Lec11-The Memory Hierarchy
date: 2020-04-21 15:45:00
categories:
	- 操作系统
	- CMU213
tags:	
	- 操作系统
	- CMU213
mathjax: true
typora-root-url: ../../../
---

# 存储技术与趋势

## RAM

Random-Access Memory 随机访问存储器

### 特点

- 一般被打包成芯片。
- 多个 RAM 芯片 组成了主存。
- 最基本的存储单元成为 cell，一个 cell 存储一个 bit。

### 分类

- SRAM（Static）
- DRAM（Dynamic）

根据存储单元实现方式来区分的。

|      | Trans.per bit | Access time | Needs refresh | Needs EDC | Cost | Applications                 |
| ---- | ------------- | ----------- | ------------- | --------- | ---- | ---------------------------- |
| SRAM | 4 or 6        | 1X          | No            | Maybe     | 100X | Cache memories               |
| DRAM | 1             | 10X         | Yes           | Yes       | 1X   | Main memories, frame buffers |

表格解释：

- Trans.per bit：SRAM 实现一个 bit 用的晶体管更多。

- Needs refresh：DRAM 使用电容存储电荷的原理来寄存信息，电容上的电荷一般只能维持1~2ms，因此即使电源不掉电，信息也会自动消失，需要在 2ms 内对其所有存储单元恢复一次原状态。

- Needs EDC：SRAM 更可靠，不太需要进行错误检测和纠正 EDC（Error Detection and Correctoin）。

总结

- SRAM 内存容量小、速度快，用于高速缓存。

- DRAM 被用于主存、图形显卡的帧缓存中。

## 非易失性存储

RAM 是易失性存储，断电后，信息会丢失。

非易失性存储在断电后信息不会丢失。

### 分类

- ROM：Read-only memory。过去只能在芯片生产时被硬编码一次；现在可以重新编码。
- 闪存：Flash memory。Electrically eraseable PROM (EEPROM)。提供擦除功能，在约 10 万次擦除后，会被磨损。 

### 用途

- Firmware programs stored in a ROM (BIOS, controllers for disks, network cards, graphics accelerators, security subsystems,…)

- Solid state disks (replace rotating disks in thumb drives, smart phones, mp3 players, tablets, laptops,…)

- Disk caches

## 总线

### 结构

![cmu213-lec11-bus](/images/cmu213-lec11-bus.png)

数据在内存和 CPU 芯片之间通过一系列并行的电线传输，称为总线（bus）。

- 在指令需要访问内存时（执行 `mov` 指令），由总线接口（Bus interface）处理，它与 I/O 桥通过系统总线（System bus）连接。

- I/O 桥是一个 CPU 之外的芯片集合，它与主存通过内存总线（Memory bus）连接。

上面只是总线系统的抽象，实际很复杂。

### 加载数据过程

使用 `movq A, %rax` 加载数据时，

1. CPU 将 A 地址放到内存总线上。
2. 主存从内存总线读到 A 地址，将主存中 A地址处的 8 bytes 内容放回总线。
3. CPU 从总线读取到这些数据，放到寄存器 `%rax` 中。

### 写入数据过程

使用 `movq %rax, A` 写入数据时，

1. CPU 将 A 地址放到内存总线上。
2. 主存从内存总线读到 A 地址，等待数据到达总线。
3. CPU 将 `%rax` 中的内容放到总线上。
4. 主存读取到这些数据，将其放到 A 地址处。

对于内存的读写，由于要离开 CPU，大约需要 50 或 100 纳秒；寄存器之间的操作小于 1 纳秒。差了两个数量级。

## 磁盘

磁盘中有一个磁盘控制器。

### 几何结构

<img src="/images/cmu213-lec11-diskgeometry.png" alt="cmu213-lec11-diskgeometry" width="50%" />

- 磁盘（Disk）是由盘片（platter）组成的，每个盘片有上下两个表面（surface）。

- 每个盘面包含一系列同心圆，称为磁道（tracks）。

- 每个磁道包含很多扇区（sector），扇区存储着数据（一般一个扇区存储 512 bytes）。扇区之间存在空隙（gaps），空隙不存储数据。

<img src="/images/cmu213-lec11-diskgeometry2.png" alt="cmu213-lec11-diskgeometry2" width="50%" />

- 盘片在主轴（Spindle）上是彼此对齐的，因此在不同表面上，磁道也是对齐的。

- 同一条磁道的集合，形成了一个圆柱形，成为柱面（Cylinder）。

### 容量

磁盘的容量是它可以存储的 bit 数。

由记录密度和磁道密度决定：

- 记录密度：一个扇区可以存储多少 bit。
- 磁道密度：可以将相邻的磁道放置得多近。
- 面密度：记录密度 * 磁道密度。决定了整个磁盘的存储容量。

### 记录区

以前，每个磁道包含的扇区数量是相等的。这样会导致越靠近边缘的扇区间空隙会越大。

所以现代磁盘，会把磁道划分为不同的子集，称为记录区。

- 一个记录区里的每一条磁道都有相同数量的扇区。

- 不同记录区的每条磁道拥有的扇区数目不同。靠近外侧的记录区中每条磁道拥有更多的扇区数目。

在计算容量时，使用平均的每条磁道拥有的扇区数目。

### 磁臂

<img src="/images/cmu213-lec11-diskarm.png" alt="cmu213-lec11-diskarm" width="50%" />

盘面以一个固定的频率在逆时针旋转（典型的速率是 7200转/分钟）。

磁臂（arm）沿着半径轴前后移动，可以定位到任意一个磁道上。

每个盘面有一个磁臂，注意一个盘片有两个磁臂。所有磁臂只能一起移动。

### 数据读取

![cmu213-lec11-diskread](/images/cmu213-lec11-diskread.png)

1. 将磁臂移动到红色扇区所在磁道。
   - **寻道时间**。一般是 3 - 9 ms。
2. 扇区旋转到读写头的下方。

   - **旋转延迟**。平均情况下就是磁盘旋转一圈所花费时间的一半。
- 1/2 * (60s / 7200 RPM) * 1000 ms/sec = 4ms。
3. 当前读写头处于红色扇区的开始位置，红色扇区在读写头下旋转。它会检测这些 bit，将它们发送到控制器，控制器将它们传递给 CPU。
   - **传输时间**。该磁道在读写头下通过的时间。
   - 60/7200 RPM * 1/400 secs/track * 1000 ms/sec = 0.02ms。

访问时间主要是寻道时间和旋转延迟。 

#### 对比

- 从 SRAM 读取一个 double，约 4ns。
- 从 DRAM 读取一个 double，约 60ns。
- 从磁盘读取一个 double，约 13 ms。

### 磁盘逻辑块

- 现代磁盘控制器将磁盘作为一系列逻辑块提供给 CPU。
  - 每个逻辑块是扇区大小的整数倍。
  - 块号从 0 编号，是一系列增长的数字。

- 磁盘控制器维护物理扇区与逻辑块之间的映射。

- 允许磁盘控制器将一些柱面保留为备用柱面。

  - 它们没有被映射为逻辑块。

  - 如果有个柱面的一个扇区坏了，磁盘控制器可以将数据复制到备用柱面。磁盘可以继续正常工作。

    因此，磁盘的 fomatted 容量比实际容量小。

## I/O 总线

### 结构

![cmu213-lec11-iobus](/images/cmu213-lec11-iobus.png)

之前的实现方式使 PCI 总线是广播总线，单一线路。

- 如果总线上的任何设备更改了某个值，该总线上的每个设备都可以看到这些值。

现代系统使用 PCI Express 总线，点对点的。

- 设备通过一组点对点连接、开关仲裁进行连接。

### 加载数据过程

读取磁盘扇区。假设一个逻辑块由一个扇区组成。

CPU通过编写三元组来启动此读取行为。`read 逻辑块号 目的内存地址`。

1. 将读请求发送给磁盘控制器，CPU 可以执行其他指令。
2. 磁盘控制器获得总线的控制权。
3. 通过 I/O 桥将数据复制到 I/O 总线。
4. 直接复制到主存，不用通知 CPU。
5. 传输完成后，磁盘控制器通过中断机制通知 CPU。
   - 实际上，CPU 芯片上有一个引脚，将引脚的值从 0 改为 1。

之所以分成两步（先传输、再通知），是因为从磁盘读取数据太慢了。

## 固态磁盘

Solid State Disks（SSDs）。

- 没有机械部件，完全由闪存构成。

- 与磁盘具有相同的物理接口。SSD 内部有一组固件，成为**闪存翻译层**，作用与磁盘的磁盘控制器相同。

- 可以以**页**为单位，从闪存读取和写入数据。页的大小取决于技术的不同，从 512KB 到 4 KB。

- 一系列的页组成**块**。从 32 页到 128 页。

- 一个页只能再所属的整个块都被擦除后，才能进行**写入**。在写入时，找到一个被擦除的块，将目标块的所有其他页面复制到新块中，再进行写入。
- 一个块被擦除 10 万次后会磨损，现代系统的闪存翻译层实现了很多**算法**，来延长 SSD 的寿命，例如缓存。所以在实践中，这不是一个问题。

### 性能

机械磁盘的读写速度在 40 或 50 MB/s。

- 读取
  - 顺序读取：550 MB/s
  - 随机读取：365 MB/s
- 写入
  - 顺序写入：470 MB/s
  - 随机写入：303 MB/s

擦除一个块需要约 1ms。在早期的 SSD 中，读写操作的时间差距是巨大的。通过对闪存翻译层的优化，现在写入稍慢于读取。

# 引用的局部性

CPU 与 内存/磁盘之间的性能差距越来越大，这样 CPU 性能的提升，并不能使程序变快。

**程序的局部性**是跨越 CPU 与内存之间差距的关键。

## 局部性原理

程序倾向于使用其地址接近或等于最近使用过的数据/指令的那些数据/指令。

### 两种局部性

时间局部性。

- 最近引用的存储器位置可能在不久的将来再次被引用。

空间局部性。

- 引用临近存储器位置的倾向。

### 两种引用

- 数据引用。

- 指令引用。

### 例子

```C
sum = 0;
for (i = 0; i < n; i++)
	sum += a[i];
return sum;
```

- 数据引用。
  - `a[i]` 是空间局部性。
  - `sum` 是时间局部性。
- 指令引用。
  - 空间局部性。循环中的每次迭代，引用的都是一系列相同的指令。

# 存储器层次结构

## 结构

![cum213-lec11-memory-hierarchy](/images/cum213-lec11-memory-hierarchy.png)

容量数量级。

- 寄存器有 16 个。

- SRAM cache 大小有几 MB。

- DRAM 主存大小有十几 GB。

每层都包含从下一层锁获取的数据。

## 缓存

### 缓存

- 一个更小、更快的存储设备（第 K 层），充当更大、更慢的设备（第 K+1 

  层）中数据的暂存区域。

- 例如，可以把主存看做磁盘的缓存。

### 为什么缓存是有效的

根据程序局部性原理，相比于第 K + 1 层数据，程序倾向于访问第 K 层的数据。在访问第 K + 1 层数据时，把它拷贝到第 K 层。

因为我们不经常访问第 K + 1 层，可以使用更慢、更便宜的设备，以得到更大的容量。

### 主要思想

存储器层次结构创建了一个大型存储池，其存储容量等于最底层存储设备的大小，却可以以最高层存储设备的速度来访问。

## General Cache Concepts

### 缓存命中

- 要访问的块正位于缓存中。

### 缓存未命中

- 冷未命中（强制未命中）。
  - 缓存中没有任何数据。
- 容量未命中。
  - 程序局部性需要用到的容量大于缓存容量。
- 冲突未命中。
  - 缓存一般是硬件实现的，所以功能简单，限制了块可以被放置的位置。
  - 例如，第 `i` 块只能放在 `i % 缓存大小` 处。
  - 例如，缓存大小是 4，依次访问第 0、8、0、8 块，每次都会发生冲突未命中。

## 存储器层次结构中的缓存

![cmu213-lec11-mem-hierarchy-cache](/images/cmu213-lec11-mem-hierarchy-cache.png)

TLB（Translation Lookaside Buffer 后背缓冲），是在虚拟内存中使用的缓存。

Buffer cache。操作系统保留一部分内存来存储已加载的文件。