---
title: CMU213 学习笔记-Lec7-Machine-Level Programming II：Procedures
date: 2020-04-13 16:45:00
categories:
	- 操作系统
	- CMU213
tags:	
	- 操作系统
	- CMU213
mathjax: true
typora-root-url: ../../../
---

# ABI

Application Binary Interface (ABI) 应用程序二进制接口。是机器程序级别的接口。

- 从开发 x86-64 机器设计时演化而来。超越了硬件成为软件标准。

- 针对 Linux。

- 系统的不同部分（Linux 程序、编译器、操作系统）对如何管理机器上的资源有一些共同的理解、并遵循这套标准。

# Mechanisms in Procedures

> 这里的 procedures 是统一的术语，指的是 函数（function）、过程（procedure）、类的方法（method）。

## 三方面的机制

### 转移控制权

- 将控制权转移给一个 procedure。
- 确保 procedure 执行到退出点后，能返回到正确的位置。因此需要记录返回位置的信息。

### 传递数据

- procedure 的参数。

- 返回值。因此需要规定如何回传数据的约定。

### 内存分配

- 在 procedure 执行期间分配内存。在哪里分配？如何确保正确分配？
- 在 procedure 返回后，局部内存需要被释放掉。 

## 减少开销

x86-64 会尽量减少 procedure 调用的开销。因为好的编程风格，每个函数应该专注于很小的功能。

因此，原则之一是 只做绝对必要的事情、尽可能规避开销。

- 如果数据不是必要的，就不分配和释放内存给它。
- 如果没有传递任何值，就不传递他们。

# 栈 Stack

使用栈来管理 procedure 调用与返回中涉及的状态。

之所以用栈，是因为栈的`后进先出`与 procedure 的调用与返回的思想一致。

## 基本结构

- 组织形式为内存中某一段连续的区域。

- 栈的开始地址是编号非常高的地址。
- `%rsp` 是栈顶指针，值是当前栈顶的地址。
  - 在栈生长时（即分配更多的数据给栈），通过递减指针来完成的。

![cmu213-stack](/images/cmu213-stack.png)

## Push 指令

把数据写入栈中。

- 格式：`pushq Src`。
- 源操作数可以来自寄存器、内存、立即数。
- 目标的内存地址通过 把 `%rsp` 递减 8 （申请内存）来确定。

由于 x86-64 的地址是 8 byte的，`push`、`pop`、`call`、`ret` 等命令只能适用于 8 byte 的地址。

## Pop 指令

从栈中读取数据，将其存储在**寄存器**中。

- 格式：`popq Dest`。
- 读取的地址由当前的 `%rsp` 给出。
- 然把把 `%rsp` 递增 8（释放内存）。
- 将结果存储在寄存器中。

# 传递控制权

## call 指令

`call label`。

1. 把这个 `call` 指令之后的指令（返回时要继续执行的指令）的地址 `push` 到栈中。
2. 跳到 `lable` 的地址（`%rip` 设置为   `label` 的地址）  。

## ret 指令

`ret`。

1. 从栈中 `pop` 出地址。
2. 跳到这个地址。

可以看到，`call` 和 `ret` 指令并没有完成 procedure 调用和返回的全部任务，只完成了控制权的部分。

# 传递数据

## 参数

procedure 的前 6 个参数依次使用如下的寄存器存储。

- `%rdi`
- `%rsi`
- `%rdx`
- `%rcx`
- `%r8`
- `%r9`

参数要求是整数或指针类型。浮点类型由另外一组单独的寄存器传递的。

超过 6 个的参数放入栈上的内存中。在 A32 中所有参数都是通过栈传递的。

## 返回值

返回值由 `%rax` 存储。

# 管理局部数据

## 栈帧

### 定义

栈上用于特定 `call` 的每个内存块成为栈帧。

在栈中为每个被调用且未返回的过程保留一个栈帧。

为了避免无限循环版本的递归，会把数据不断 `push` 到栈中，大部分系统限制了栈的最大深度。

###两个指针分割

一个栈帧由两个指针分割。

- `%rbp` 基指针（base pointer）。是可选的，一般不会使用。
- `%rsp` 栈指针。

所以大部分情况，甚至都不知道一个栈帧的具体范围。

既然 `%rbp` 是可选的，程序怎么知道如何释放空间，如何将栈重置会正确的位置？

- 编译器知道它分配时，将分配多少 byte（例如 16 byte），那么它知道最后要释放 16 byte。
- 有一些特殊情况，无法提前知道分配多少空间，例如分配一个可变大小的数组或内存缓冲区时，会在这些情况使用 `%rbp` 是实现。

### 栈帧中的内容

- `%rbp`。如果使用了基指针，需要在 procedure 调用时，存储旧的基指针的值，以便在被调用的 procedure 返回时恢复它。
- 其他需要被保存的状态（无法被保存在寄存器中），例如一些寄存器、局部数组。
- 第6个之外的参数。

<img src="/images/cmu213-stackframepng.png" alt="cmu213-stackframepng" width="50%" />

 ## 例子

### incr

```C
long incr(long *p, long val) {
    long x = *p;
    long y = x + val;
    *p = y;
    return x;
}
```

```assembly
incr:
  movq    (%rdi), %rax # x = *p。
  addq    %rax, %rsi   # val += x。 
  movq    %rsi, (%rdi) # *p = val。
  ret                  # return x。
```

### Calling incr

```C
long call_incr() {
    long v1 = 15213;            // 1
    long v2 = incr(&v1, 3000);  // 2
    return v1+v2;               // 3
}
```

由于要使用 `v1` 的地址，所以 `v1` 不能存储在寄存器中，而要在栈中。

```assembly
call_incr:
  subq    $16, %rsp        # 1.1 %rsp递减16，申请16byte空间。
  movq    $15213, 8(%rsp)  # 1.1 存储15213在%rsp+8地址处。
  movl    $3000, %esi      # 2.1 把 3000 放入第二个参数。
  leaq    8(%rsp), %rdi    # 2.2 把 %rsp+8，即&v1放入第一个参数。
  call    incr             # 2.3 调用 incr()。
  addq    8(%rsp), %rax    # 3.1 %rax += v1。
  addq    $16, %rsp        # 0   %rsp递增16，释放16byte空间。
  ret                      # 3.2 跳到%rsp存储的地址对应的指令。
```

需要注意的点：

- 1.1 在栈中申请了 16 byte 内存，但是只是在高 8 byte 作为 long 来使用了，低 8 byte 没有使用。用来对齐。

- 2.1 使用 `movl` 和 `%esi`、而不是 `movq` 和 `%rsi`，因为 3000 足够小，使用 `movl` 足够了，前面 4 位会自动置为 0。

- 1.1 和 0 处申请和释放了栈汇总的内存。显式使用栈帧来进行数据管理、栈管理。

## 保存寄存器状态的约定

因为调用者和被调用者 procedure 都会使用寄存器，为了保证被调用者使用寄存器时，调用者的数据不会被覆盖掉，需要保存寄存器状态。

可以有两种方法保存寄存器的状态：

- 在调用者处保存使用过的寄存器的状态。
  - ABI 中，`%r10` 和 `%r11` 用来在**调用者处**，调用其他函数前，保存临时数据。
- 在被调用者处，在使用寄存器前保存寄存器的状态。
  - `%rbx`、`%r12`、`%r13`、`%r14` 用来在**被调用者处**保存临时数据。
  - 如果不使用栈指针，`%rbp` 也可以用来在**被调用者处**保存临时数据。
  - 这些寄存器在使用前需要把数据 `push` 到栈中，在 procedure 返回前再 `pop` 回寄存器中。

### 例子

```c
long call_incr2(long x) {       // 0
    long v1 = 15213;            // 1
    long v2 = incr(&v1, 3000);  // 2
    return x+v2;                // 3
}
```

```assembly
call_incr2:
  pushq   %rbx              # 把 %rbx 的旧值 push 到栈上。
  subq    $16, %rsp         
  movq    %rdi, %rbx        # 把 %rdi 的值暂存到 %rbx 中。
  movq    $15213, 8(%rsp)
  movl    $3000, %esi
  leaq    8(%rsp), %rdi
  call    incr
  addq    %rbx, %rax
  addq    $16, %rsp
  popq    %rbx              # 把栈中%rbx的旧值pop出恢复给%rbx。
  ret
```

以上例子，使用 `%rbx`，在**被调用者**处暂存数据 `x`。

- 由于在 0 处 `x` 需要使用 `%rdi`，在 2 处 `pcount_r()` 也需要使用 `%rdi`，因此要把 `x` 的值从 `%rdi` 暂存到 `%rbx` 中。

# 递归

## 例子

```C
/* Recursive popcount */
long pcount_r(unsigned long x) {       // 0
  if (x == 0)                          // 1
    return 0;
  else                                 // 2
    return (x & 1) + pcount_r(x >> 1);
}
```

```assembly
pcount_r:
  movl    $0, %eax      # 1.1 return_val = 0。
  testq   %rdi, %rdi    # 1.2 测试x是否为0。
  je      .L6           # 1.3 如果x为0，跳到.L6指令处。
  pushq   %rbx          # 2.0 把%rbx旧值存储到栈中。
  movq    %rdi, %rbx    # 2.1 暂存x到%rbx。
  andl    $1, %ebx      # 2.2 tmp = x & 1。
  shrq    %rdi          # 2.3 x >>= 1。
  call    pcount_r      # 2.4 调用pcount_r。递归调用。
  addq    %rbx, %rax    # 2.5 %rax += tmp。 
  popq    %rbx          # 2.6 恢复%rbx旧值。
.L6:
  rep; ret
```

2.1 在**被调用者**处使用 `%rbx` 暂存数据 `x`。

## 发现

无须特殊考虑递归的情况，原因如下：

- 栈帧意味着每个函数调用都有各自私有的存储空间。
  - 用来保存寄存器状态、本地变量。
  - 用来 保存返回的指针。
- 保存寄存器状态的约定 避免一个函数调用 弄坏其他函数调用的数据。
- 栈的规则与 call/return 的模式相一致。
  - 如果 `P` 调用了 `Q`，则 `Q` 会在 `P` 之前结束。
  - 后进先出。