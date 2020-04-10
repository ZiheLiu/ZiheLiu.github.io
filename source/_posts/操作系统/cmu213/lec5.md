---
title: CMU213 学习笔记-Lec5-Machine-Level Programming I：Basics
date: 2020-04-09 16:16:00
categories:
	- 操作系统
	- CMU213
tags:	
	- 操作系统
	- CMU213
mathjax: true
typora-root-url: ../../../
---

# 引用

- [【精校中英字幕】2015 CMU 15-213 CSAPP 深入理解计算机系统 课程视频](https://www.bilibili.com/video/BV1iW411d7hd?t=2580&p=5)

# 处理器

Intel 

- 常用处理器：x86 x86-64
- 使用 MISC（复杂指令集）。

ARM

- Acorn RISC Machine
- 使用 RISC （简单指令集）。

# 术语

Architecutre: instruction set architecutre 指令集架构。

Microarchitecture: architecutre的实现。

Machine Code: 处理器去执行的二进制程序。

Assembly Code：Machine Code的text表示。

# 汇编代码的相关结构

## 程序计数器

- 下一条要执行的指令地址。

## 寄存器

汇编中使用名字来指定它们。由 `%名称` 给出。

x86-64 共有 16 个寄存器。如下图：

![x86-64-registers](/images/x86-64-registers.png)

- 使用 `%r` 得到寄存器的 64 bits，使用 `%e` 得到寄存器的低 32 bits。
- 前 8 个寄存器的名字从 IA32 继承来，命名方式是为了标识每个寄存器的用途。但在 x86-64 中一般不区分它们的用途。
  - `%esp` stack pointer，栈指针。

## Condition codes

- 状态寄存器。
- 存储最近指令的运行结果。
- 用于实现条件分支。

# 将 C 代码转变为可执行程序

1. C 代码**编译**到 Assembly Code。
2. Assembly Code **Assembler** 到字节形式 Machine Code。
3. 把项目的 Machine Code 与静态库的 Machine Code **Linker** 得到最终的可执行程序。

## 例子

### C 代码

```c
#include <stdio.h>
#include <stdlib.h>

long plus(long x, long y) {
  return x + y;
}

void sumstore(long x, long y, long *dest) {
  long t = plus(x, y);
  *dest = t;
}

int main(int argc, char *argv[]) {
  long x = atoi(argv[1]);
  long y = atoi(argv[2]);
  long z;
  sumstore(x, y, &z);
  printf("%ld + %ld --> %ld\n", x, y, z);

  return 0;
}
```

### Assembly Code

使用如下命令把 C 代码编译为 Assembly Code：

```shell
// -O：优化级别。
// -Og：优化级别为调试，便于阅读。
// -S：Only run preprocess and compilation steps。
$ gcc -Og -S main.c
```

结果 Assembly Code 如下：

```assembly
_sumstore:
LFB5:
	pushq	%rbx
LCFI0:
	movq	%rdx, %rbx
	call	_plus
	movq	%rax, (%rbx) # *dest = t
	popq	%rbx
LCFI1:
	ret
```

### Machine Code

使用如下命令把 C 代码（经过编译、汇编两步骤）转化为 Machine Code：

```shell
$ gcc -Og main.c -o main
```

#### 反汇编

使用 `objdump` 可以对 Machine Code 反汇编为 Assembly Code：

```shell
$ objdump -d main
_sumstore:
100000ea5:      53      pushq   %rbx
100000ea6:      48 89 d3        movq    %rdx, %rbx
100000ea9:      e8 f2 ff ff ff  callq   -14 <_plus>
100000eae:      48 89 03        movq    %rax, (%rbx)
100000eb1:      5b      popq    %rbx
100000eb2:      c3      retq
```

使用 `gdb` 也可以进行反汇编：

```shell
$ gdb main
(gdb)$ disassemble sumstore
Dump of assembler code for function sumstore:
   0x0000000100000ea5 <+0>:     push   %rbx
   0x0000000100000ea6 <+1>:     mov    %rdx,%rbx
   0x0000000100000ea9 <+4>:     callq  0x100000ea0 <plus>
   0x0000000100000eae <+9>:     mov    %rax,(%rbx)
   0x0000000100000eb1 <+12>:    pop    %rbx
   0x0000000100000eb2 <+13>:    retq   
End of assembler dump.
```

# 汇编的数据类型

Integer

- 1、2、4、8 bytes。
- 不区分无符号和有符号数字。
- 地址也是用它表示的。

Float

- 4、8、10 bytes。

Code

- 二进制序列表示一系列的指令。

# 汇编的操作

## 代数运算

在寄存器/内存上进行代数运算。

### leaq

load effective address。

将一个内存地址直接存储在目的寄存器中。

#### 例子

```c
long m12(long x) {
  return x*12;
}
```

对应的 Assembly Code：

```Assembly
# 与move类似，但是只是把计算后的地址存进寄存器，而不是地址对应的内存中的内容。
leaq (%rdi,%rdi,2), %rax # t <- x+x*2。
salq $2, %rax            # return t<<2
```

### 二元指令

| Format            | Computation                 |
| ----------------- | --------------------------- |
| `addq Src, Dest`  | Dest = Dest + Src           |
| `subq Src, Dest`  | Dest = Dest - Src           |
| `imulq Src, Dest` | Dest = Dest * Src           |
| `salq Src, Dest`  | Dest = Dest << Src          |
| `sarq Src, Dest`  | Dest = Dest >> Src 算数右移 |
| `shrq Src, Dest`  | Dest = Dest >> Src 逻辑右移 |
| `xorq Src, Dest`  | Dest = Dest ^ Src           |
| `andq Src, Dest`  | Dest = Dest & Src           |
| `orq Src, Dest`   | Dest = Dest                 |

 注意，`Dest` 是第一个操作数。

### 一元指令

| Format      | Computation     |
| ----------- | --------------- |
| `incq Dest` | Dest = Dest + 1 |
| `decq Dest` | Dest = Dest - 1 |
| `negq Dest` | Dest = -Dest    |
| `notq Dest` | Dest = ~Dest    |

### 例子

```c
long arith(long x, long y, long z) {
  long t1 = x+y;
  long t2 = z+t1;
  long t3 = x+4;
  long t4 = y * 48;
  long t5 = t3 + t4;
  long rval = t2 * t5;
  return rval;
}
```

对应的 Assembly Code：

```assembly
arith:
  leaq    (%rdi,%rsi), %rax   # t1. rax = rdi + rsi.
  addq    %rdx, %rax          # t2. rax = rax + rdx.
  leaq    (%rsi,%rsi,2), %rdx #     rdx = rsi + rsi * 2 = 3 * rsi.
  salq    $4, %rdx            # t4. rdx = rdx << 4 = (3 * rsi) * 16 = rsi * 48.
  leaq    4(%rdi,%rdx), %rcx  # t5. rcx = (rdi + 4) + rdx.
  imulq   %rcx, %rax          # rval. rax = rcx * rax.
  ret
```

## 传递数据

在寄存器/内存间传递数据。

### movq

用于在 寄存器与寄存器、寄存器与内存、寄存器与常数、常数与内存之间的数据传输。

注意，内存与内存不能直接进行数据传输。

![movq](/images/movq.png)

### 如何使用某一位置的内存

- `(rb)`：`Mem[Reg[rb]]`。
- `D(rb)`：`Mem[Reg[rb] + D]`。
- `D(rb,ri,s)`：`Mem[Reg[rb] + s * Reg[ri] + D]`。
  - 可以用来实现取数组元素的内存。

### 例子

交换两个数字的 C 代码：

```c
void swap(long *xp, long *yp) {
  long t0 = *xp;
  long t1 = *yp;
  *xp = t1;
  *yp = t0;
}
```

Assembly Code：

```assembly
swap:
   movq    (%rdi), %rax
   movq    (%rsi), %rdx
   movq    %rdx, (%rdi)
   movq    %rax, (%rsi)
   ret
```

寄存器 `%rdx`、`%rax` 用于存储临时变量 `t0`、`t1`，而没有存在内存里。

注意，固定的，`%rdi` 是第一个参数寄存器，`%rsi` 是第二个参数寄存器。

## 流程控制

- Unconditional jumps to/from procedures
- Conditional branches



