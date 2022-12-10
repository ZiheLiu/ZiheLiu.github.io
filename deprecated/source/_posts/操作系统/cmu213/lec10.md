---
title: CMU213 学习笔记-Lec10-Program Optimization
date: 2020-04-19 20:40:00
categories:
	- 操作系统
	- CMU213
tags:	
	- 操作系统
	- CMU213
mathjax: true
typora-root-url: ../../../
---


# 编译器通用优化方式
如果编译器不确定某些优化的效果，就不会优化它。

## Code Motion

减少计算执行的频率。

- 如果它每次都会产生相同的结果。
- 常见的例子，把代码移出循环。

### 例子

```C
void set_row(double *a, double *b, long i, long n) {
    long j;
    for (j = 0; j < n; j++) {
      a[n*i+j] = b[j];
    }
}
```

会优化为

```C
void set_row(double *a, double *b, long i, long n) {
    long j;
    int ni = n * i; // 预先计算 n*i
    for (j = 0; j < n; j++) {
      a[ni+j] = b[j];
    }
}
```
优化后的 Assembly code：
```assembly
set_row:
	testq	%rcx, %rcx		            # Test n
	jle	.L1			                    # If 0, goto done
	imulq	%rcx, %rdx		            # ni = n*i.这里！
	leaq	(%rdi,%rdx,8), %rdx	      # rowp = A + ni*8
	movl	$0, %eax	               	# j = 0
.L3:				      	              # loop:
	movsd	(%rsi,%rax,8), %xmm0    	# t = b[j]
	movsd	%xmm0, (%rdx,%rax,8)   	  # M[A+ni*8 + j*8] = t
	addq	$1, %rax			            # j++
	cmpq	%rcx, %rax		            # j:n
	jne	.L3			                    # if !=, goto loop
.L1:				      	              # done:
	rep ; ret

```

## Reduction in Strength

使用性能好的指令代替性能差的指令。

- 使用移位运算代替乘法和除法。
  - 例如 `16 * x` --> `x << 4`。
  - 是否优化，依赖于乘法和除法指令的性能。
    - 在 Intel Nehalem，整数乘法需要 3 个 CPU cycles。
- 识别结果序列。

### 例子

```C
for (i = 0; i < n; i++) {
  int ni = n*i;
  for (j = 0; j < n; j++)
    a[ni + j] = b[j];
}
```

会优化为

```C
int ni = 0;
for (i = 0; i < n; i++) {
  for (j = 0; j < n; j++)
    a[ni + j] = b[j];
  ni += n;
}
```

## Share Common Subexpressions

- 复用表达式的共通部分。
- gcc 在开启 `-O1` 后使用此优化。

### 例子

```C
/* Sum neighbors of i,j */
// 需要 3 次乘法。
up =    val[(i-1)*n + j  ];
down =  val[(i+1)*n + j  ];
left =  val[i*n     + j-1]; // left 与 right 只用求一次 i*n
right = val[i*n     + j+1];
sum = up + down + left + right;
```

会优化为

```C
// 需要 1 次乘法。
long inj = i*n + j;
up =    val[inj - n];
down =  val[inj + n];
left =  val[inj - 1];
right = val[inj + 1];
sum = up + down + left + right;
```

# Optimization Blocker

## Procedure Calls

编译器把 procedure 看做是黑盒的。

- procedure 可能有副作用。例如维护了全局状态，每次调用值 +1。
- 每次调用 procedure，代码中可能会对其参数进行改变。

所以，

- 对于 procedure，要自己去做 code motion。

### 例子

```C
void lower(char *s) {
  size_t i;
  for (i = 0; i < strlen(s); i++)
    if (s[i] >= 'A' && s[i] <= 'Z')
      s[i] -= ('A' - 'a');
}
```

`strlen()` 每个循环都会被执行一次。不会用 code emtion 进行优化。

而 `strlen()` 需要遍历直到字符是 `\0`，即时间复杂度是 $O(n)$ 的。

## Memory Aliasing

两个不同的指针可以指向同一块内存，这样会导致编译器有时无法进行优化。

### 例子

```C
// a 是 n*n 的矩阵
// b 是 n 维数组
// sum_rows1() 作用是，把 a 的每一行的和存到 b 中。
void sum_rows1(double *a, double *b, long n) {
    long i, j;
    for (i = 0; i < n; i++) {
	      b[i] = 0;
	      for (j = 0; j < n; j++)
	        b[i] += a[i*n + j];
    }
}
```

Assembly Code 如下：

```assembly
# sum_rows1 inner loop
.L4:
        movsd   (%rsi,%rax,8), %xmm0	# FP load
        addsd   (%rdi), %xmm0		# FP add
        movsd   %xmm0, (%rsi,%rax,8)	# FP store
        addq    $8, %rdi
        cmpq    %rcx, %rdi
        jne     .L4
```

通过汇编可以看到，`b[i] += a[i*n + j]` 需要从内存读取 `a` 和 `b` 的对应位置，相加后再写回到内存中 `b` 的对应位置。

由于 `b` 和 `a` 在内存中可能有重叠，所以编译器不能使用 Code Motion 去优化。

例如，可以通过如下方式调用 `sum_rows1()`：

```C
double A[9] = 
  { 0,   1,   2,
    4,   8,  16},
   32,  64, 128};
double *B = A+3;
sum_rows1(A, B, 3);
```

# 利用指令级并行

有些指令之间是独立的，CPU 可以并行执行这些指令。

Superscalar Processor 可以在一个时钟周期执行多条指令。

把指令进行拆分、分组，

## 流水线操作

单个硬件上的操作，可以分解为一系列不同的阶段，例如乘法指令需要多个阶段。

一个阶段的硬件，在计算完一个值把它传给下一个阶段后，可以马上进行下一个值的计算。

### 例子

假设乘法需要 3 个阶段（每个阶段一个时钟周期）。

```C
long mult_eg(long a, long b, long c) {
    long p1 = a*b;
    long p2 = a*c;
    long p3 = p1 * p2;
    return p3;
}
```

|          | **Time** |          |          |          |            |            |            |
| -------- | -------- | -------- | -------- | -------- | ---------- | ---------- | ---------- |
|          | 1        | 2        | 3        | 4        | 5          | 6          | 7          |
| Stage  1 | **a\*b** | **a\*c** |          |          | **p1\*p2** |            |            |
| Stage  2 |          | **a\*b** | **a\*c** |          |            | **p1\*p2** |            |
| Stage  3 |          |          | **a\*b** | **a\*c** |            |            | **p1\*p2** |

总共只需要 7 个时钟周期，而不是 9 个，就可以完成计算。

## **Haswell** CPU

- 每种指令可以并行执行的数量：

  - 2 load, with address computation
  - 1 store, with address computation
  - 4 integer
  - 2 FP multiply
  - 1 FP add
  - 1 FP divide

- 指令的延时、在流水线操作中一个阶段的时间：

| Instruction                 | Latency | Cycles/Issue |
| -------------------------- | ------ | ------------ |
| Load / Store                | 4       | 1            |
| Integer Multiply            | 3       | 1            |
| **Integer/Long Divide** | 3-30    | 3-30         |
| Single/Double FP Multiply   | 5       | 1            |
| Single/Double FP Add        | 3       | 1            |
| **Single/Double FP Divide** | 3-15    | 3-15         |


  可以看到，

  - 大多数指令需要几个时钟周期，但是采用了流水线技术，流水线的一个阶段只需要一个时钟周期。

  - 除法指令非常慢，且没有流水线。

## 分支预测

CPU 会猜测哪个分支会被执行，提前执行对应分支的代码。

执行这部分代码时，要保证没有副作用。

在 CPU 中，有寄存器重命名块，每个寄存器都有几百个副本，计算的结果保存在这些副本中。分支预测的结果存储在寄存器的副本中。

## 例子

### 数据结构

```C
/* data structure for vectors */
typedef struct {
	size_t len;
	data_t *data;
} vec;

/* retrieve vector element
   and store at val */
int get_vec_element(*vec v, size_t idx, data_t *val) {
	if (idx >= v->len)
		return 0;
	*val = v->data[idx];
	return 1;
}
```

### 评价指标

- CPE（Cycles Per Element）：每个元素计算所需要的时钟周期。CPE 是下图直线的斜率。

- $T = CPE*n + Overhead$。总的时间。

<img src="/images/cmu213-lec10-exmaple-pce.png" alt="cmu213-lec10-exmaple-pce" width="75%" />

### baseline

```C
void combine1(vec_ptr v, data_t *dest) {
    long int i;
    *dest = IDENT;
    for (i = 0; i < vec_length(v); i++) {
	      data_t val;
	      get_vec_element(v, i, &val);
	      *dest = *dest OP val;
    }
}
```

- `data_t` 通过 `typedef` 设置为 `int` 或 `double`。

- `OP` 通过 `typedef` 设置为 `+` 或 `*`。

| Method                   | Integer | Integer | Double FP | Double FP |
| ---------------------------- | ----------- | ------------- | --------- | --------- |
| Operation                | Add     | Mult      | Add   | Mult  |
| Combine1 unoptimized | 22.68   | 20.02     | 19.98 | 20.18 |
| Combine1 –O1             | 10.12   | 10.12     | 10.17 | 11.14 |

### 基础优化

```C
void combine4(vec_ptr v, data_t *dest) {
  long i;
  long length = vec_length(v);
  data_t *d = get_vec_start(v);
  data_t t = IDENT;
  for (i = 0; i < length; i++)
    t = t OP d[i];
  *dest = t;
}
```

进行了如下的优化：

- `vec_length()` 调用移出循环。
- 避免每次调用 `get_vec_element()` 的边界检查。
- 用临时变量累加。（使用寄存器，而不是内存累加）。

| Method            | Integer | Integer | Double FP | Double FP |
| ----------------- | ------- | ------- | --------- | --------- |
| Operation         | Add     | Mult    | Add       | Mult      |
| Combine1 –O1      | 10.12   | 10.12   | 10.17     | 11.14     |
| **Combine4  -O1** | 1.27    | 3.01    | 3.01      | 5.01      |

可以看到 整数加法的 耗时略高于 1，因为在循环的计数上开销比较大。

### 循环展开

```C
void unroll2a_combine(vec_ptr v, data_t *dest) {
    long length = vec_length(v);
    long limit = length-1;
    data_t *d = get_vec_start(v);
    data_t x = IDENT;
    long i;
    /* Combine 2 elements at a time */
    for (i = 0; i < limit; i+=2) {
	      x = (x OP d[i]) OP d[i+1];
    }
    /* Finish any remaining elements */
    for (; i < length; i++) {
	      x = x OP d[i];
    }
    *dest = x;
}
```

每个迭代，计算两个数字的累加。

| Method          | Integer | Integer | Double FP | Double FP |
| --------------- | ------- | ------- | --------- | --------- |
| Operation       | Add     | Mult    | Add       | Mult      |
| Combine4        | 1.27    | 3.01    | 3.01      | 5.01      |
| **Unroll 2x1a** | 1.01    | 3.01    | 3.01      | 5.01      |
| Latency Bound   | 1.00    | 3.00    | 3.00      | 5.00      |

### 循环展开 + 结合律

```c
void unroll2aa_combine(vec_ptr v, data_t *dest) {
    long length = vec_length(v);
    long limit = length-1;
    data_t *d = get_vec_start(v);
    data_t x = IDENT;
    long i;
    /* Combine 2 elements at a time */
    for (i = 0; i < limit; i+=2) {
	       x = x OP (d[i] OP d[i+1]); // 这里与 unroll2a_combine() 不同。
    }
    /* Finish any remaining elements */
    for (; i < length; i++) {
	      x = x OP d[i];
    }
    *dest = x;
}
```

使用结合律后，在计算 `x OP tmp` 时，可以提前计算下一个迭代的 `d[i] OP d[i+1]`，时钟周期缩减一半。

但是浮点数除法不满足结合律，要保证不会出现舍入的情况下使用。

- 因此，编译器不会对浮点数采用结合律进行优化。

| Method           | Integer | Integer | Double FP | Double FP |
| ---------------- | ------- | ------- | --------- | --------- |
| Operation        | Add     | Mult    | Add       | Mult      |
| Combine4         | 1.27    | 3.01    | 3.01      | 5.01      |
| Unroll 2x1a      | 1.01    | 3.01    | 3.01      | 5.01      |
| **Unroll 2x1aa** | 1.01    | 1.51    | 1.51      | 2.51      |
| Latency Bound    | 1.00    | 3.00    | 3.00      | 5.00      |

### 循环展开 + 多个累加器

可以使用两个累加器来进行计算，每次迭代，分别累加到不同的累加器上。

```c
void unroll2a_combine(vec_ptr v, data_t *dest) {
    long length = vec_length(v);
    long limit = length-1;
    data_t *d = get_vec_start(v);
    data_t x0 = IDENT;
    data_t x1 = IDENT;
    long i;
    /* Combine 2 elements at a time */
    for (i = 0; i < limit; i+=2) {
       x0 = x0 OP d[i];
       x1 = x1 OP d[i+1];
    }
    /* Finish any remaining elements */
    for (; i < length; i++) {
	x0 = x0 OP d[i];
    }
    *dest = x0 OP x1;
}
```

也相当于使用了结合律，改变了元素结合的顺序。

| Method         | Integer | Double FP |      |      |
| -------------- | ------- | --------- | ---- | ---- |
| Operation      | Add     | Mult      | Add  | Mult |
| Combine4       | 1.27    | 3.01      | 3.01 | 5.01 |
| Unroll 2x1a    | 1.01    | 3.01      | 3.01 | 5.01 |
| Unroll 2x1aa   | 1.01    | 1.51      | 1.51 | 2.51 |
| **Unroll 2x2** | 0.81    | 1.51      | 1.51 | 2.51 |
| Latency Bound  | 1.00    | 3.00      | 3.00 | 5.00 |

### 循环展开 + 多个累加器 + 调参

- 循环展开。每个迭代计算 `L` 个元素。
- 多个累加器。每次迭代使用 `K` 个累加器，要求 `L` 是 `K` 的倍数。

机器

- Intel Haswell 

- Double FP Multiplication

- Latency bound: 5.00. Throughput bound: 0.50 

| **FP  \*** | **Unrolling  Factor L** |      |      |      |      |      |          |      |
| ---------- | ----------------------- | ---- | ---- | ---- | ---- | ---- | -------- | ---- |
| **K**      | 1                       | 2    | 3    | 4    | 6    | 8    | 10       | 12   |
| 1          | 5.01                    | 5.01 | 5.01 | 5.01 | 5.01 | 5.01 | 5.01     |      |
| 2          |                         | 2.51 |      | 2.51 |      | 2.51 |          |      |
| 3          |                         |      | 1.67 |      |      |      |          |      |
| 4          |                         |      |      | 1.25 |      | 1.26 |          |      |
| 6          |                         |      |      |      | 0.84 |      |          | 0.88 |
| 8          |                         |      |      |      |      | 0.63 |          |      |
| 10         |                         |      |      |      |      |      | **0.51** |      |
| 12         |                         |      |      |      |      |      |          | 0.52 |

在 $L = 10$、$K=10$ 时，CPE 达到了吞吐率的边界值 0.5。

最优 CPE 与 Latency 无关，只与吞吐率有关。

### 向量化

向量化浮点数计算，可以同时进行 4 个 double 的加法计算。

gcc 对这个优化得不太好，需要用 gcc 的插件。

# 总结

- 使用编译器的优化级别。
- 不要编写愚蠢的代码。
  - 编写编译器友好的代码。
    - Procedure Calls & Memory Aliasing    
  - 关注循环中的代码（大部分工作在循环中完成）。
- 针对特定机器优化代码。
  - 利用指令级并行。
  - 避免不可预测的分支。
  - 缓存友好（还没有讲）。