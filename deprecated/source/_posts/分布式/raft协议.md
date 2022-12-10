---
title: Raft 协议
date: 2020-03-05 20:24:00
categories:
	- 分布式
tags:
	- 分布式
typora-root-url: ../../
---

# 引用

- [RAFT协议完整笔记](https://www.jianshu.com/p/ddbe4209be0f)
- [Raft协议详解](https://zhuanlan.zhihu.com/p/27207160)
- [请不要再称数据库是CP或者AP](https://blog.the-pans.com/cap/)
- [强一致性与最终一致性与raft](https://lentil1016.cn/consistencies-and-raft/)
- [DDIA-一致性与共识（一致性保证与可线性化）](https://keys961.github.io/2019/03/11/DDIA-一致性与共识-一致性保证与可线性化/)
- [etcd raft cap 理解](https://blog.csdn.net/scylhy/article/details/100173494)

# 介绍

分布式共识算法，通过 Raft 协议让各个节点保持状态一致。

# CAP

## 定义

### 一致性 Consistency。

CAP 中指可线性化。如果B操作在成功完成A操作之后，那么整个系统对B操作来说必须表现为A操作已经完成了或者更新的状态。

强一致性与弱一致性。

- 强一致性集群中，对任何一个节点发起请求都会得到相同的回复，但将产生相对高的延迟。
- 弱一致性具有更低的响应延迟，但可能会回复过期的数据，最终一致性即是经过一段时间后终会到达一致的弱一致性。

ZooKeeper 的读操作默认不是可线性化的、不是强一致性的。单可以开启线性化，在读之前要发一个 SYNC 命令。

Raft 的读操作，使用了 LeaseRead 优化后不是可线性化的，也不是强一致性的。如果只是使用 ReadIndex 是可线性化、强一致性的。

### 可用性 Availability

每一个请求(如果被一个工作中的节点收到，那一定要返回[非错误]的结果。

对于 Raft，在有网络分区时，数量少的那个分区无法完成写操作、读操作。

###分区容错 Partition Tolerance

Raft 满足分区容错。

## Raft 中的 CAP

Raft 满足 P，不满足 A。对于 C 不优化读操作时满足强一致性。

# 节点状态

1. 跟随者
2. 候选者
3. 领导者

# 跟随者 

节点收到心跳/添加日志请求后：

1. 如果当前的term大于请求的term，返回false。
2. 如果当前的term小于请求的term，自己变为跟随者。
3. 如果请求的PrevLogIndex不在自己的log里，返回false。
4. 如果请求的PrevLogTerm不是自己的这个index的term，略过自己所有这个term的log，返回最右边不是这个term的log的index，返回false。
5. 否则可以添加请求中的log entries。
6. 更新commitIndex为请求中的commitIndex（如果更大的话）。
7. 返回true。



# 候选者

跟随者转变为候选者的时机：

- 如果跟随者 一定时间内 没有收到领导者的心跳消息，则转变为候选者。

转变为候选者后，

- term++，
- 投票给自己，
- 给其他所有节点发送投票请求。请求中附带4个信息：
  - 当前的term
  - 候选者id
  - 上一条log的term
  - 上一条log的index

节点在收到投票请求时：

1. 比较自己的term和请求的term，

   - 如果自己的term大，直接返回false。

   - 如果请求的term大，更新自己为跟随者（自己可能是领导者、候选者），投票清空。

2. 如果当前已经投票过了，返回false。

3. 如果没有投过票，并且请求的候选者的log更新（args.LastLogTerm > lastLogTerm || (args.LastLogTerm == lastLogTerm && args.LastLogIndex + 1 >= len(rf.log))，则返回true，更新自己为跟随者。否则返回false。

候选者收到投票请求的回复后：

- 如果收到true的数目多于1半，则成为领导者。
- 否则等待选举时间超时，进行下一个term的选举。或者收到其他候选者的请求（请求的term更大），进行投票，成为跟随者。或者收到其他人的心跳请求，成为跟随者。
  这里为了防止不断的有候选者平分选票重试，候选者和跟随者等待的超时时间要加一个随机数。

# 领导者

领导者在收到添加log请求后，会把log添加到自己的logs里，在下一次心跳时，发送添加log的请求。

领导者在收到添加log请求的回复时：

- 如果返回的是false，把rf.nextIndex[index] = reply.NextIndex，稍后心跳用新的nextIndex重试。
- 如果返回的是true，如果大多数跟随者都返回了true，把这条记录标记为commit。

# 选举约束

选举约束：被选举出来的 leader 必须要包含所有已经提交的 entries。

Raft 的简单实现：lastLog的term越大谁越新，如果term相同，谁的lastLog的index越大谁越新。

但是这样的实现，可能会出现日志覆盖的情况（见[Raft协议详解](https://zhuanlan.zhihu.com/p/27207160)）。

所以 Raft 添加了约束2：当前term的leader不能“直接”提交之前term的entries，必须要等到**当前** term 有entry过半了，才**顺便**一起将之前term的entries进行提交。

不能覆盖已经提交的请求（因为领导者要半数节点同意才可以，而同意需要领导者的上一条的term和index最大）

# 网络分区

> - [PingCAP-线性一致性和 Raft](https://pingcap.com/blog-cn/linearizability-and-raft/)
> - [etcd 中线性一致性读的具体实现](https://zhengyinyong.com/post/etcd-linearizable-read-implementation/#什么是线性一致性读)
> 

A - D 五个节点，A 为 Leader。

某一时刻，网络分区为 AB和CDE。CDE由于心跳超时，假设 C 成为新的 Leader。

这样，CDE 认为 C 是 Leader，而 AB 仍然认为 A 是 Leader。

这个时候，就产生了**网络分区**。

## 写请求

Client 可以把读写请求发送到集群的任意节点，节点会把请求转发给 Leader。

网络分区对于写请求没有影响。

- 因为 Client 如果向 AB 请求写入，没有办法产生过半数的同意，所以没有办法**提交**这个写请求。

- 只有 Client 向 CDE 请求写入，才能被**提交**。

## 读请求

从 AB 读可能读到旧的数据。

### ReadIndex

使用 ReadIndex 解决这个问题，主要是第 2 步。

1. 记录当前的 commit index，称为 ReadIndex
2. 向 Follower 发起一次心跳，如果大多数节点回复了，那就能确定现在仍然是 Leader
3. 等待状态机**至少**应用到 ReadIndex 记录的 Log
4. 执行读请求，将结果返回给 Client

1 和 3 步是为了保证线性一致性读。

- 线性一致性读：当存储系统已将写操作提交成功，那此时读出的数据应是最新的数据、

### LeaseRead

这样每次读请求，都要发起一次心跳，网络消耗很大。使用 LeaseRead 来优化。

基本的思路是 Leader 取一个比 Election Timeout 小的租期，在租期不会发生选举，确保 Leader 不会变，所以可以跳过 ReadIndex 的第二步，也就降低了延时。