---
title: CMU213 学习笔记-Lec8-Machine-Level Programming VI：Data
date: 2020-04-15 13:48:00
categories:
	- 操作系统
	- CMU213
tags:	
	- 操作系统
	- CMU213
mathjax: true
typora-root-url: ../../../
---

# 标量/聚合数据

标量数据

- 不是任何聚合形式的数据。

聚合数据

- 数组
- 结构体

机器级别代码没有数组/结构体这种高级概念，C 编译器需要生成适当的代码来分配内存、引用结构体/数组的元素时，可以得到正确的值。

机器代码提供了一些指令来实现这些功能。

# 数组

## 一维数组

已知数组的内存区域的起始地址 `x`，使用地址计算，通过给 `x` 加上一个数字的偏移量，来获取数组中特定元素的地址。

对于数组 `T A[L]`。

- 元素的数据类型是 `T`，共有 `L` 个元素。
- `A` 可以用作第 0 个元素的指针，类型是 `T*`，它的值就是 `x`，即数组的首地址。

### 汇编

#### 访问数组

```C
int get_digit(int z[5], int digit) {
  return z[digit];
}
```

```assembly
  # %rdi = z
  # %rsi = digit
movl (%rdi,%rsi,4), %eax  # z[digit]。(%rdi + %rsi * 4)
```

#### 遍历数组

```C
void zincr(int z[5]) {
  size_t i;
  for (i = 0; i < ZLEN; i++)
    z[i]++;
}
```

```assembly
  # %rdi = z
  movl    $0, %eax          #   i = 0
  jmp     .L3               #   goto middle
.L4:                        # loop:
  addl    $1, (%rdi,%rax,4) #   z[i]++。注意这行代码。
  addq    $1, %rax          #   i++
.L3:                        # middle
  cmpq    $4, %rax          #   i:4
  jbe     .L4               #   if <=, goto loop
  rep; ret
```

C 语言起初是为了实现一个操作系统而设计的。所以设计目的是，保持灵活性、以及在汇编代码里玩的技巧。而做到这一点的方式，就是在编程语言里加入指针运算。

## 数组与指针 

- 声明数组，会分配容纳数组的空间。
- 声明指针，只是一个指针。

### 例子 1

<table>
   <tr>
      <th>Decl</th>
      <th colSpan="3">A1, A2</th>
      <th colSpan="3">*A1, *A2</th>
   </tr>
   <tr>
      <th></th>
      <th>Cmp</th>
      <th>Null</th>
      <th>Size</th>
      <th>Cmp</th>
      <th>Null</th>
      <th>Size</th>
   </tr>
   <tr>
      <td>int A1[3]</td>
      <td>Y</td>
      <td>N</td>
      <td>12</td>
      <td>Y</td>
      <td>N</td>
      <td>4</td>
   </tr>
   <tr>
      <td>int *A2</td>
      <td>Y</td>
      <td>N</td>
      <td>8</td>
      <td>Y</td>
      <td>Y</td>
      <td>4</td>
   </tr>
</table>
- Cmp: Compiles
- Null: Possible null pointer reference
- Size: Value returned by `sizeof`

### 例子 2

<table>
   <tr>
      <th>Decl</th>
      <th colSpan="3">An</th>
      <th colSpan="3">*An</th>
      <th colSpan="3">**An</th>
   </tr>
   <tr>
      <th></th>
      <th>Cmp</th>
      <th>Null</th>
      <th>Size</th>
      <th>Cmp</th>
      <th>Null</th>
      <th>Size</th>
      <th>Cmp</th>
      <th>Null</th>
      <th>Size</th>
   </tr>
   <tr>
      <td>int A1[3]</td>
      <td>Y</td>
      <td>N</td>
      <td>12</td>
      <td>Y</td>
      <td>N</td>
      <td>4</td>
      <td>N</td>
      <td>-</td>
      <td>-</td>
   </tr>
   <tr>
      <td>int *A2[3]</td>
      <td>Y</td>
      <td>N</td>
      <td>24</td>
      <td>Y</td>
      <td>N</td>
      <td>8</td>
      <td>Y</td>
      <td>Y</td>
      <td>4</td>
   </tr>
   <tr>
      <td>int (*A3)[3]</td>
      <td>Y</td>
      <td>N</td>
      <td>8</td>
      <td>Y</td>
      <td>Y</td>
      <td>12</td>
      <td>Y</td>
      <td>Y</td>
      <td>4</td>
   </tr>
   <tr>
      <td>int (*A4[3])</td>
    <td>Y</td>
      <td>N</td>
      <td>24</td>
      <td>Y</td>
      <td>N</td>
      <td>8</td>
      <td>Y</td>
      <td>Y</td>
      <td>4</td>
   </tr>
</table>
- `A3` 是一个指针，它指向一个具有三个 `int` 元素的数组。

- `A4`/`A2` 是一个数组，每个元素是一个 `int` 的指针。

## 多维数组

### 定义

`T A[R][C]`

- `A` 是一个由 `R` 个元素的数组，每个元素是一个有 `C` 个元素的数组（其元素类型是 `T`)。
- 行优先表示法。
- 共有 `R * C * K` bytes（`T` 具有 `K` bytes）。

### 存储方式

在内存中，以行优先的方式存储在一块连续的内存中。

```C
  int pgh[4][5] = {{1, 5, 2, 0, 6},
                   {1, 5, 2, 1, 3 },
                   {1, 5, 2, 1, 7 },
                   {1, 5, 2, 2, 1 }};
```

`pgh` 在内存中的分布如下图所示：

![cmu213-lec8-multiarray-example1](/images/cmu213-lec8-multiarray-example1.png)

### 访问元素

#### 访问行向量

- `A[i]` 是一个具有 `C` 个元素的数组。
- 起始地址是 $A + i * (C * K)$。

```C
int pgh[4][5];
int *get_pgh_zip(int index) {
  return pgh[index];
}
```

```assembly
  # %rdi = index
	leaq (%rdi,%rdi,4),%rax	# 5 * index
	leaq pgh(,%rax,4),%rax	# pgh + 5 * (index * 4)
```

#### 访问元素

- `A[i][j]` 是第 `i` 行的第 `j` 个元素，类型是 `T`。
- 地址是 $ A + i * (C * K) + j * K = A + (i * C + j) * K$。

```C
int get_pgh_digit(int index, int dig) {
  return pgh[index][dig];
}
```

```assembly
	leaq	(%rdi,%rdi,4), %rax	# 5 * index
	addl	%rax, %rsi	        # 5 * index + dig
	movl	pgh(,%rsi,4), %eax	# M[pgh + 4*(5*index+dig)]
```

#### 访问数组指针的元素 

注意，第一维大小必须是固定的。

```C
int[5] cmu = { 1, 5, 2, 1, 3 };
int[5] mit = { 0, 2, 1, 3, 9 };
int[5] ucb = { 9, 4, 7, 2, 0 };
int *univ[3] = {mit, cmu, ucb};

int get_univ_digit (size_t index, size_t digit) {
  return univ[index][digit];
}
```

其内存空间如下图所示：

![cmu213-lec8-dynamicmultiarray-exmaple1](/images/cmu213-lec8-dynamicmultiarray-exmaple1.png)

Assembly Code：


```assembly
  salq    $2, %rsi            # 4*digit
  addq    univ(,%rdi,8), %rsi # p = univ[index] + 4*digit。
  movl    (%rsi), %eax        # return *p
  ret	
```

经过了两次的内存引用。

- 第一次拿到列所表示数组的首地址。
- 第二次拿到元素。

#### 访问动态数组的元素

```c
/* Get element A[i][j] */
int var_ele(size_t n, int A[n][n], size_t i, size_t j) {
  return A[i][j];
}
```

```assembly
  # n in %rdi, A in %rsi, i in %rdx, j in %rcx
  imulq   %rdx, %rdi           # n*i
  leaq    (%rsi,%rdi,4), %rax  # A + 4*n*i
  movl    (%rax,%rcx,4), %eax  # A + 4*n*i + 4*j
  ret
```

因为在编译时不知道数组的大小，所以必须用乘法指令去计算 `n * i`，而不像之前的可以用移位计算，开销更大。

# 结构体

## 基础结构

### 内存分布

```C
struct rec {
    int a[4];
    size_t i;
    struct rec *next;
};
```

![cmu213-lec8-struct-basic](/images/cmu213-lec8-struct-basic.png)

### 例子 1

```C
struct rec {
    int a[4];
    size_t i;
    struct rec *next;
};

int *get_ap(struct rec *r, size_t idx) {
  return &r->a[idx];
}
```

```assembly
  # r in %rdi, idx in %rsi  
  leaq  (%rdi,%rsi,4), %rax # %rdi + idx * 4
  ret
```

因为 `a` 是 `rec` 结构体的第一个字段，所以 `r` 既是 `rec` 的首地址、也是 `rec.a` 的首地址。 

### 例子 2

```C
void set_val(struct rec *r, int val) {
  while (r) {
    int i = r->i;
    r->a[i] = val;
    r = r->next;
  }
}
```

```assembly
.L11:                         # loop:
  movslq  16(%rdi), %rax      #   1. i = M[r+16] = r->i	  
  movl    %esi, (%rdi,%rax,4) #   2. M[r+4*i] = val
  movq    24(%rdi), %rdi      #   3. r = M[r+24] = r->next
  testq   %rdi, %rdi          #   4. Test r
  jne     .L11                #   if !=0 goto loop
```

- 虽然 `i` 是 `int`，但是由于把它用作了数组的索引（地址是 8 bytes），所以要进行缩放。因此，`1` 处的汇编使用了 `movslq`（符号扩展，从 双字 4 bytes 传递给 四字 8 bytes）。
- 1、2、3 处的代码对内存共进行了 3 次引用。

## 内存对齐

对于一个由 `k` 字节的类型，机器更喜欢起始地址是 `k` 的倍数。

- 因为硬件一次会从内存取 64 个字节。
- 如果没有对齐，一个数据可能会跨越两个块。这样需要硬件、操作系统采取额外的步骤来处理。且效率更低。

所以在结构体的内存中，

- 编译器会填充一些 unused 字节，以保证每个字段的起始地址都是 `k` 的倍数。
- 在末尾填充字节，确保结构体的总大小满足下一个变量的对齐的要求。

### 调整字段顺序来节省空间

把字节数目大的字段放在前面。

```C
struct S4 {
  char c;
  // 填充 3 bytes
  int i;
  char d;
  // 填充 3 bytes
} *p;

struct S5 {
  int i;
  char c;
  char d;
  // 填充 2 bytes
} *p;
```

### 例子 1

```C
struct S1 {
  char c;
  int i[2];
  double v;
} *p;
```

![cmu213-lec8-struct-aligned](/images/cmu213-lec8-struct-aligned.png)

- `i` 的起始地址需要是 4 的倍数。
- `v` 的起始地址需要使 8 的倍数。

### 例子 2

```C
struct S2 {
  double v;
  int i[2];
  char c;
} *p;

double num;
```

![cmu213-lec8-struct-aligned2](/images/cmu213-lec8-struct-aligned2.png)

## 结构体数组

数组的每个元素要进行对齐，以满足每个元素的起始地址都是 8 的倍数。

```C
struct S2 {
  double v;
  int i[2];
  char c;
} a[10];
```

![cmu213-lec8-struct-arrayaligned](/images/cmu213-lec8-struct-arrayaligned.png)

# 浮点数

## 寄存器

XMM 寄存器

- 共有 16 个。每个有 16 bytes。

## 指令

SIMD 指令（single instruction multiple data 单指令多数据）。

`addss %xmm0, %xmm1`

单精度标量（single precision scalar）做加法运算。

`addps %xmm0, %xmm1`

`p` 指 `packet`，可以同时进行四个不同数的加法。

`addsd %mm0, %mm1`

双精度做加法。

## 基础

- 参数使用 `%xmm0`、`%xmm1`、...  传递。
- 返回值放在 `%xmm0` 中。
- 所有寄存器都是**被调用者**保存的。

### 汇编

#### 单精度

```C
float fadd(float x, float y) {
    return x + y;
}
```

```assembly
  # x in %xmm0, y in %xmm1
  addss   %xmm1, %xmm0
  ret
```

#### 双精度

```c
double dadd(double x, double y) {
    return x + y;
}
```

```assembly
 # x in %xmm0, y in %xmm1   
  addsd   %xmm1, %xmm0
  ret
```

## 与常规寄存器混合使用

- 整数、指针类型的参数使用常规寄存器传递。
- 浮点数类型的参数使用 XMM 寄存器传递，两者可以交替。
- 使用针对浮点数的 `mov` 指令。

```C
double dincr(double *p, double v) {
    double x = *p;
    *p = x + v;
    return x;
}
```

```Assembly
  # p in %rdi, v in %xmm0
  movapd  %xmm0, %xmm1   # Copy v
  movsd   (%rdi), %xmm0  # x = *p
  addsd   %xmm0, %xmm1   # t = x + v
  movsd   %xmm1, (%rdi)  # *p = t
  ret
```

