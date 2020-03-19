---
title: Linux 命令记录
date: 2020-03-19 16:30:00
categories:
	- 操作系统
tags:
	- 操作系统
	- Linux
	- record
---

# 引用

- [Linux grep 命令](https://www.runoob.com/linux/linux-comm-grep.html)
- [linux文件名匹配（通配符使用）](https://blog.csdn.net/BlueCY/article/details/6545517)
- [linux shell 管道命令(pipe)使用及与shell重定向区别](https://www.cnblogs.com/chengmo/archive/2010/10/21/1856577.html)
- [ps aux指令詳解](https://blog.csdn.net/hanner_cheung/article/details/6081440)
- [SIGINT、SIGQUIT、 SIGTERM、SIGSTOP区别](https://blog.csdn.net/pmt123456/article/details/53544295)
- [Linux后台进程管理以及ctrl+z（挂起）、ctrl+c（中断）、ctrl+\（退出）和ctrl+d（EOF）的区别](https://blog.csdn.net/u012787436/article/details/39722583)
- [3. lsof 一切皆文件](https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/lsof.html)
- [Linux 工具参考篇](https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/index.html)

# grep 查找字符串

用于查找文件/目录里符合条件的字符串。

```shell
# 查找符合 <filename-pattern> 的文件、且包含 <str> 的行。
$ grep <str> <filename-pattern>
# 查找符合 <dir-pattern> 的目录下、且包含 <str> 的行。
$ grep -r <str> <dir-pattern>
```

## 参数

- `-n --line-number` 在每一行开头显示行数。只在有且只有一个文件匹配的时候，才会有效。

  ```shell
  $ grep -n 'te' parent/child/file1
  1:test
  ```

## 匹配规则

* `*` 匹配文件名中的任何字符串，包括空字符串。

-  `?` 匹配文件名中的任何单个字符。

- `[...]`  匹配 `[ ]` 中所包含的任何字符。

- `[!...]`  匹配 `[ ]` 中非感叹号 `!` 之后的字符。

# 管道命令

管道符号 `|` 左边的命令的输出作为右边命令的输入。

```shell
$ cmd1 | cmd2 | cmd3
```

例如

```shell
$ cat file1 | grep -n 'test'
1:test
```

# ps 查看进程信息

显示进程的信息。

## 参数

- `a` 显示当前终端所有进程，包括其他用户的进程。
- `x` 显示所有进程，包括其他终端的进程。
- `u` 以用户为主的格式来显示程序状况。多显示如下的列：
  - USER
  - `%CPU` 进程占用的 CPU 百分比。
  - `%MEM` 占用内存的百分比。
  - `VSZ` 该进程使用的虚拟內存量（KB）。
  - `RSS` 该进程占用的固定內存量（KB）（驻留中页的数量）。
- `e` 显示每个进程所使用的环境变量。

# ctrl+z ctrl+c ctrl+\ ctrl+d 发送信号

- `ctrl + c` 发送 `SIGINT(2)` 信号给前台进程组中的所有进程。常用于终止正在运行的程序。
- `ctrl + z` 发送 `SIGSTOP(19)` 信号给前台进程组中的所有进程，常用于挂起一个进程。可以发送 `SIGCONT` 信号恢复。
- `ctrl + \` 发送 `SIGQUIT(3)` 信号给前台进程组中的所有进程，终止前台进程并生成 core 文件。在这个意义上类似于一个程序错误信号。
- `ctrl + d` 不是发送信号，而是表示一个特殊的二进制值，表示 EOF。

## 其他信号

- `SIGTERM(15)` 正常退出。`kill <pid>` 默认发的信号。
- `SIGKILL(9)` 强制退出，不能被捕获、阻塞、忽略，只能执行缺省操作。
- `SIGSTOP(19)` 也不能被捕获、阻塞、忽略，只能执行缺省操作。

# netstat 查看端口监听信息

查看监听了各个端口的进程信息。

可以与 `grep` 命令用管道命令查询指定端口的进程。

```shell
$ netstat -ntulp | grep 3306
```

## 参数

- `-t` 显示 TCP 监听的端口。
- `-u` 显示 UDP 监听的端口。
- `-l` 仅显示监听套接字。
- `-p` 显示进程标识符和程序名称，每一个套接字/端口都属于一个程序。
- `-n` 不进行DNS轮询，显示IP（可以加速操作）。

# lsof 查看文件描述符信息

list open files。列出**当前用户**打开的文件描述符。netstat 会列出所有端口监听信息。

## 参数

- `-i <条件>` 列出符合条件的文件描述符。
  - `lsof -i tcp` 列出所有tcp 网络连接信息。
  - `lsof -i :3306` 列出谁在使用某个端口。

# top 任务管理器

Linux 下的任务管理器。实时显示系统中各个进程的资源占用状况。

> [top linux下的任务管理器](https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/top.html)
>
> [linux top 命令](https://www.cnblogs.com/sparkdev/p/8176778.html)

可以按各项指标（%MEM、PID、%CPU、TIME+）排序。

# free 查询可用内存

> [linux free 命令](https://www.cnblogs.com/sparkdev/p/7994666.html) 说的比较清楚

```shell
              total        used        free      shared  buff/cache   available
Mem:       16425452      746144     8364652       75604     7314656    15522836
Swap:        969964           0      969964
```

1. total：物理内存实际总量。
2. used：已经被使用的物理内存。
3. free：未被分配的内存。
4. shared：共享内存。
5. buffers：系统分配的，但未被使用的 buffer 剩余量。
6. cached：系统分配的，但未被使用的 cache 剩余量。
7. available：还可以被应用程序使用的物理内存大小。
   - 当应用程序需要内存时，如果没有足够的 free 内存可以用，内核就会从 buffer 和 cache 中回收内存来满足应用程序的请求。
   - 所以从应用程序的角度来说，`available = free + buffer + cache`。这只是一个很理想的计算方式，实际中的数据往往有较大的误差。

## buffer 与 cache

- buffer : 作为 buffer cache 的内存，是块设备的读写缓冲区。在不用文件系统的时候使用。

- cache: 作为 page cache 的内存， 文件系统的 cache。

# vmstat 监视内存使用情况

Virtual Meomory Statistics（虚拟内存统计），可实时动态监视操作系统的虚拟内存、进程、CPU活动。

# uptime/w

最后的三个数字分别表示最后一分钟、最后五分钟、最后十五分钟的系统平均负载（CPU使用、内存使用、IO消耗三部分构成）。

```shell
$ uptime
 20:43:27 up 38 days,  5:53,  2 users,  load average: 2.36, 1.91, 1.90
$ w
 20:43:30 up 38 days,  5:53,  2 users,  load average: 2.41, 1.93, 1.90
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root     tty1     -                10Feb20 38days  0.03s  0.02s -bash
root     pts/0    119.54.11.242    19:37    0.00s  0.03s  0.00s w
```

