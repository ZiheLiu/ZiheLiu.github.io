---
title: CMU213 学习笔记-Lec9-Machine-Level Programming V：Advanced Topics
date: 2020-04-16 21:03:00
categories:
	- 操作系统
	- CMU213
tags:	
	- 操作系统
	- CMU213
mathjax: true
typora-root-url: ../../../
---

# 内存布局

![cmu213-lec9-memory-layoutpng](/images/cmu213-lec9-memory-layout.png)

## Stack

- 有 8 MB 大小的限制。
  - 可以使用 `limit` 命令，查看 `stackszie` 字段。
  - 如果访问超过 8 MB 的内存地址，会产生一个 segmentation fault。
- 例如局部变量。
- 从高地址到低地址分配内存。
- 起始地址是 `0x07FF_FFFF_FFFF_FFFF`。
  - 因为 DRAM 很贵，所以 64 位机器限制只能使用地址空间中的 47 位地址，128 TB 的内存限制。

## Heap

- 存储动态申请的变量。
- 调用 `malloc()`、`calloc()`、`new()`。
- 如果尝试引用 Heap 区中还没有通过内存分配程序分配的地址，会产生一个 segmentation fault。

## Data

- 存储静态变量。
  - 存放程序开始时分配的数据。
  - 例如全局变量。

## Text / Shared Libraries

- 存储可执行的机器指令序列。
- 只读的。
- Text 区存储自己的代码。
- Shared Libraries 区存储库的代码。

# 缓冲区溢出 Buffer Overflow

## 危害

如果有途径能让外界使程序的 buffer 溢出，将会产生安全问题。在编写程序时，要考虑变量的值是否值得信任。

## 产生 Buffer Overflow 的情形

- 访问数组范围的索引元素。
- 没有检查输入字符串的长度。
  - 很多存储字符串、但不检查边界情况的库函数。

### String Library Code

`gets()` 库函数与下面类似：

```C
/* Get string from stdin */
char *gets(char *dest){
    int c = getchar();
    char *p = dest;
    while (c != EOF && c != '\n') {
        *p++ = c;
        c = getchar();
    }
    *p = '\0';
    return dest;
}
```

这个函数没有停止读取的条件，可能传入的 Buffer `dest` 不够大。

如下的函数都有类似的问题：

- `strcpy()`、`strcat()`。
- `scanf()`、`fscanf()`、`sscanf()` 在使用 `%s` 读取字符串时。

### 例子

```C
/* Echo Line */
void echo() {
    char buf[4];  /* Way too small! */
    gets(buf);
    puts(buf);
}

void call_echo() {
    echo();
}
```

```shell
unix>./bufdemo-nsp
Type a string: 012345678901234567890123
012345678901234567890123

unix>./bufdemo-nsp
Type a string:0123456789012345678901234
Segmentation Fault
```

注意，虽然 `buf` 只能存储 3 个字符，但是输入 24 个字符也是可以正常执行的。原因请看下面的 Assembly Code：

```assembly
 00000000004006cf <echo>:
  4006cf:	48 83 ec 18          	sub    $0x18,%rsp
  4006d3:	48 89 e7             	mov    %rsp,%rdi
  4006d6:	e8 a5 ff ff ff       	callq  400680 <gets>
  4006db:	48 89 e7             	mov    %rsp,%rdi
  4006de:	e8 3d fe ff ff       	callq  400520 <puts@plt>
  4006e3:	48 83 c4 18          	add    $0x18,%rsp
  4006e7:	c3                   	retq 

<call_echo>:
  4006e8:	48 83 ec 08          	sub    $0x8,%rsp
  4006ec:	b8 00 00 00 00       	mov    $0x0,%eax
  4006f1:	e8 d9 ff ff ff       	callq  4006cf <echo>
  4006f6:	48 83 c4 08          	add    $0x8,%rsp
  4006fa:	c3                   	retq 
```

`4006cf` 处代码，在栈上分配了 24 bytes 的内存。本来的栈顶是 `4006f6` 这个地址（`call_echo()` 调动 `echo()` 前把下一条指令放入栈中）。

因此，输出 24 个字符后，会把返回地址变为 `400600`，但是程序并没有崩溃，而是可能跳到了奇怪的地方。

## 代码注入攻击

类似于 `gets()` 没有边界检查的输入函数，

1. 可以把想要执行的代码作为字符串输入。
2. 并把栈中的调用函数结束后的下一条指令地址替换为自己的地址。
3. 在这个函数执行到 `ret` 后，会跳到我们注入的代码中。

## 如何防止代码注入攻击

### 在代码中防止溢出缺陷

使用 `fgets()` 代替 `gets()`。

使用 `strncpy()` 代替 `strcpy()`。

在 `scanf()` 中使用 `%ns` 代替 `%s`。

### 系统级保护

#### ASLR

随机化栈偏移量（ASLR，Address Space Layout Randomization），每次运行时地址是变化的。

在主程序被调用之前，会在栈上分配随机大小（几MB）的空间。

#### Nonexecutale code segments 

在 x86 中，每个内存区域会标记是 `read-only` 还是 `writeable` 的。

x86-64 增加了第三权限 `execute`。

所以，标记栈为 `non-executable` 来防止代码注入攻击。

### Stack Canaries

#### 思想

检车代码中潜在的缓冲区溢出风险。

- 在栈 buffer 中存入一个特殊的值 `canary`。
- 在退出函数之前，检查 `cannary` 值是否发生了变化。

#### Assembly Code

在 gcc 编译时，添加 `-fstack-protector` 参数来启用，新的版本是默认启用的。

```C
/* Echo Line */
void echo() {
    char buf[4];  /* Way too small! */
    gets(buf);
    puts(buf);
}
```

```assembly
  40072f:	sub    $0x18,%rsp
  400733:	mov    %fs:0x28,%rax   # 获取 canary
  40073c:	mov    %rax,0x8(%rsp)  # 把 canary 存入栈中，buffer 只能使用 8 bytes 了。
  400741:	xor    %eax,%eax       # 清空 %rax
  400743:	mov    %rsp,%rdi
  400746:	callq  4006e0 <gets>
  40074b:	mov    %rsp,%rdi
  40074e:	callq  400570 <puts@plt>
  400753:	mov    0x8(%rsp),%rax  # 从栈中获取值
  400758:	xor    %fs:0x28,%rax   # 检查与canary 是否相同，不相同，则执行__stack_chk_fail。
  400761:	je     400768 <echo+0x39>
  400763:	callq  400580 <__stack_chk_fail@plt>
  400768:	add    $0x18,%rsp
  40076c:	retq 

```

`%fs` 是为 8086 设计的寄存器，是某块内存中的值，每次运行程序会变。

## 面向返回编程攻击

不使用指令来填充栈，而是使用 gadget 的地址来填充栈。

gadget：在代码区域中本来存在的代码中，`ret`  指令 + 前一条指令 称为 gadget。

可以把想要执行的指令序列，拆分为一个个指令，找到对应的 gadget，用它们的地址填充栈。

![cum213-lec9-面向返回编程攻击](/images/cum213-lec9-面向返回编程攻击.png)

### Gadget 例子 1

```c
long ab_plus_c(long a, long b, long c) {                                                             
   return a*b + c;                                                                           
}
```

![cum213-lec9-面向返回编程攻击-example1](/images/cum213-lec9-面向返回编程攻击-example1.png)

### Gadget 例子 2

有时候，可以不是本来代码区中存在的代码，而是只要代码区中的字节可以组成 gadget 就可以。

```C
void setval(unsigned *p) {                                                                         
    *p = 3347663060u;                                                                              
}
```

![cum213-lec9-面向返回编程攻击-example2](/images/cum213-lec9-面向返回编程攻击-example2.png)

# Unions

## union

- 根据最大的字段分配内存。
- 所有字段共用一块内存，因此只能使用一个字段。

注意，在 IA32 与 x86-64 中，是小端表示法（对于一个数据类型，地址小的字节是数据的低位）。

而在 Sun 中，是大端表示法。使用 union 可能会写出只适用于一种机器的代码。