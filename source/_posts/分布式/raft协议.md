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

选举约束：lastLog的term越大谁越新，如果term相同，谁的lastLog的index越大谁越新。

约束2：当前term的leader不能“直接”提交之前term的entries 必须要等到当前term有entry过半了，才顺便一起将之前term的entries进行提交。

不能覆盖已经提交的请求（因为领导者要半数节点同意才可以，而同意需要领导者的上一条的term和index最大）