---
title: CMU213 学习笔记-Lec6-Machine-Level Programming II：Control
date: 2020-04-09 18:16:00
categories:
	- 操作系统
	- CMU213
tags:	
	- 操作系统
	- CMU213
mathjax: true
typora-root-url: ../../../
---

# Processor State

寄存器包含了当前正在执行程序的信息。

- 暂存数据。
  - `%rax`，...
- 正在运行的栈顶。
  - `%rsp`。
- 当前正在执行指令的地址。
  - `%rip`。
  - instruction pointer。
  - 不能以正常方式访问到它。
  - 有些技巧可以访问到它的值。

- Codition Codes。
  - CF、ZF、SF、OF。

## Codition Codes

### 四种 Codition Codes

被算数/`cmp`/`test `、`setX` 指令隐式设置的 bit 寄存器。（注意，`leaq` 不会设置这四个标志。）

- CF (无符号溢出 Carry Flag for unsigned)。两个无符号数相加且溢出时，被置为 1。
- ZF (零值 Zero Flag)。如果刚刚计算的值为 0，被置为 1。
- SF (负数 Sign Flag for signed)。有符号数最高位为 1（负值），被置为 1。
- OF (有符号溢出 Overflow Flag for signed)。两个有符号数相加且溢出时，被置为 1。
  - 对于 t = a + b。在 `(a > 0 && b > 0 && t < 0) || (a < 0 && b < 0 && t >= 0)` 时溢出。
  - a 和 b 一正一负，不会溢出。

### 相关指令

#### cmp

`cmpq Src2, Src1` 指令。

- 类似于 `subq Src2, Src1`；但是只是单纯的计算 `Src1 - Src2`，不存储结果，而是设置 Condition Codes。

#### test

`testq Src2, Src1` 指令。

- 计算 `Src1 & Src2`，设置 Codition Codes（ZF 和 SF）。
  - ZF 为 1，如果 `Src1 & Src2 == 0`。
  - SF 为 1，如果 `Src1 & Src2 < 0`。

#### setX 系列指令

- 根据 Condition Codes，设置目标位置的做低 byte 为 0 / 1。
- 不会设置其他的 bytes。

| SetX  | Condition    | Description                |
| ----- | ------------ | -------------------------- |
| sete  | ZF           | Equal  / Zero              |
| setne | ~ZF          | Not  Equal / Not Zero      |
| sets  | SF           | Negative                   |
| setns | ~SF          | Nonnegative                |
| setg  | ~(SF^OF)&~ZF | Greater  (Signed)          |
| setge | ~(SF^OF)     | Greater  or Equal (Signed) |
| setl  | (SF^OF)      | Less  (Signed)             |
| setle | (SF^OF)\|ZF  | Less  or Equal (Signed)    |
| seta  | ~CF&~ZF      | Above  (unsigned)          |
| setb  | CF           | Below  (unsigned)          |

对于 `setge`，如下情况会为 1，即 Src1 >= Src2。

- SF == 1，OF == 1。即，溢出了、且 Src1、-Src2 为正数，所以 Src1 > Src2。
- SF == 0，OF == 0。即，没有溢出、且 Src1 - Src2 >= 0，所以 Src1 >= Src2。

### 例子

```c
int gt (long x, long y) {
  return x > y;
}
```

转化为 Assembly Code：

```assembly
 	cmpq   %rsi, %rdi   # Compare x:y。注意 %rsi 是 y，%rdi 是 x。
	setg   %al          # Set when >
	movzbl %al, %eax    # Zero rest of %rax
	ret
```

注意，`%eax` 是 `%rax` 低 32bit。对于运算的记过，如果是 32 位的，会把高 32  位 置为 0。

# Conditional Branches

三种方式实现流程跳转。

- jX 系列指令。
- Conditional Move 指令。
- jump table（`switch case` 对应的汇编指令序列中实现的）。

## jX 系列指令

| jX   | Condition    | Description                |
| ---- | ------------ | -------------------------- |
| jmp  | 1            | Unconditional              |
| je   | ZF           | Equal  / Zero              |
| jne  | ~ZF          | Not  Equal / Not Zero      |
| js   | SF           | Negative                   |
| jns  | ~SF          | Nonnegative                |
| jg   | ~(SF^OF)&~ZF | Greater  (Signed)          |
| jge  | ~(SF^OF)     | Greater  or Equal (Signed) |
| jl   | (SF^OF)      | Less  (Signed)             |
| jle  | (SF^OF)\|ZF  | Less  or Equal (Signed)    |
| ja   | ~CF&~ZF      | Above  (unsigned)          |
| jb   | CF           | Below  (unsigned)          |

### 例子

```C
long absdiff(long x, long y) {
  long result;
  if (x > y)
    result = x-y;
  else
    result = y-x;
  return result;
}
```

对应的 Assembly Code：

```assembly
absdiff:
   cmpq    %rsi, %rdi  # x - y
   jle     .L4				 # 如果 x <= y，跳到 .L4
   movq    %rdi, %rax  
   subq    %rsi, %rax  # x - y
   ret
.L4:       # x <= y
   movq    %rsi, %rax
   subq    %rdi, %rax	# y - x
   ret
```

## Conditional Move

### 流水线技术

流水线技术，在一个指令结束之前执行下一条指令。

选择分支进行执行，会打断流水线过程，较差情况下，可能需要40个指令（时钟周期）。

#### 分支预测技术

猜测分支结果，分支预测技术。

一般 98% 的情况会猜对。

###Conditional Move

有时候很难进行猜测，先执行两个分支来提高效率，最后取其中一个分支的结果，不用暂停处理器的执行，重新选择分支执行。

#### 例子

上述 `absdiff()` 转化为 Assembly Code 的优化版本：

```assembly
absdiff:
   movq    %rdi, %rax  # x
   subq    %rsi, %rax  # result = x-y
   movq    %rsi, %rdx
   subq    %rdi, %rdx  # eval = y-x
   cmpq    %rsi, %rdi  # x : y
   cmovle  %rdx, %rax  # if x <= y, result = eval
   ret
```

### 可以使用 Conditional Move 场景

- 如果两个分支都是简单的计算，会使用 conditional move。

### 不能使用 Conditional Move 的场景

- 无法同时计算两个分支时。例如先判断是否空指针，再进行解引用。

- 每个分支可能会改变程序其他部分的状态。

# Loops

## Do-While 循环

### 转化为 Assembly Code

C Code：

```C
do 
  // Body
  while (Test);
```

Goto Version（可以直接翻译成汇编的版本）：

```C
loop:
  // Body
  if (Test)
    goto loop
```

### 例子

C Code：

```C
long pcount_do(unsigned long x) {
  long result = 0;
  do {
    result += x & 0x1;
    x >>= 1;
  } while (x);
  return result;
}
```

Goto Version：

```C
long pcount_goto(unsigned long x) {
  long result = 0;
  
 loop:
  result += x & 0x1;
  x >>= 1;
  
  if(x) goto loop;
  
  return result;
}
```

对应的 Assembly Code：

```assembly
	 movl    $0, %eax		#  result = 0
.L2:			# loop:
   movq    %rdi, %rdx	
   andl    $1, %edx		#  t = x & 0x1
   addq    %rdx, %rax	#  result += t
   shrq    %rdi		#  x >>= 1
   jne     .L2		#  if (x) goto loop
   rep; ret
```

## While 循环

### 转化为 Assembly Code

C Code：

```C
while (Test)
  // Body
```

对于不同的优化级别，会翻译为不同的 Assembly Code。

#### -Og

Goto Version：

```C
  goto test;
loop:
  // Body
test:
  if (Test)
    goto loop;
done:
```

代码分为` loop` 的主体、`test` 条件判断。

循环开始前，先跳到 `test` 代码处进行测试。

#### -O1

```C
  if (!Test)
    goto done;
loop:
  // Body
  if (Test)
    goto loop;
done:

```

转化为 `Do-While` 循环的形式。

在 `Do-While` 循环前，先进行一次条件测试。

## For 循环

C Code：

```C
for (Init; Test; Update )
    // Body
```

对于不同的优化级别，会翻译为不同的 Assembly Code。

### -Og

```C
// Init;
while (Test ) {
    // Body
    // Update;
}

```

把 `For` 循环翻译为 `While` 循环。

### -O1

开启 `-O1` 优化后，`While` 循环会转化为 `Do-While` 循环 + 开始的一次测试。

由于 `For` 循环的初始值是确定的，因此可以将“开始的一次测试”优化删除掉。

因此，`For` 循环会转化为 `Do-While` 循环。

# Switch Statements

对于 `Switch` 表达式，编译器会找到 `case` 中中最小和最大的值，得到 `case` 的范围。

- 对于不在范围内的数值，跳转到默认代码块。
- 对于范围内的值，建立索引表。由于用了 array indexing，要求范围值要为非负数。

需要注意的地方：

- 如果范围的数值有负数，进行偏置处理（加上某个数），使得范围的值都是非负数。
- 如果 `case` 的范围过大，会变为 `if else` 语句。通过二分的形式，形成 `if else` 树，使得时间复杂度为 O(logn)、而不是 O(n)。

## 例子

C Code：

```C
long switch_eg(long x, long y, long z) {
    long w = 1;
    switch(x) {
    case 1:
        w = y*z;
        break;
    case 2:
        w = y/z;
        /* Fall Through */
    case 3:
        w += z;
        break;
    case 5:
    case 6:
        w -= z;
        break;
    default:
        w = 2;
    }
    return w;
}

```

Assembly Code：

```assembly
switch_eg:
    movq    %rdx, %rcx
    cmpq    $6, %rdi      # x:6
    ja      .L8           # Use default
    jmp     *.L4(,%rdi,8) # goto *JTab[x]. *(.L4 + %rdi * 8)
.section	.rodata
	.align 8
.L4:
	.quad	.L8	# x = 0
	.quad	.L3	# x = 1
	.quad	.L5	# x = 2
	.quad	.L9	# x = 3
	.quad	.L8	# x = 4
	.quad	.L7	# x = 5
	.quad	.L7	# x = 6
```

