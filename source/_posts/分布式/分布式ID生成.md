---
title: 分布式 ID 生成
date: 2020-03-05 20:24:00
categories:
	- 分布式
tags:
	- 分布式
typora-root-url: ../../
---

# 引用

- [Leaf——美团点评分布式ID生成系统](https://tech.meituan.com/2017/04/21/mt-leaf.html)

# UUID

## 优点

- 性能非常高：本地生成，没有网络消耗。

## 缺点

- 不易于存储：UUID太长，16字节128位，通常以36长度的字符串表示，很多场景不适用。
- 信息不安全：基于MAC地址生成UUID的算法可能会造成MAC地址泄露。
- ID 作为数据库主键时，UUID 会引起位置变化频繁。
- 不是递增的。

# 类 snowflake 方案

时间戳 + workerID + 每毫秒的递增序列号。

需要单独部署服务。

每台服务对应一个 workerID。每台服务是单线程的。

![snowflake](/images/snowflake.png)

## 优点

- 序号是递增趋势的。
- 不依赖数据库。

## 缺点

- 时钟回拨，会出现重复的序列号。

## 优化

### workerID

很多台 snowflake 服务的话，手工指定 workerID 不现实。

可以使用 Zookeeper。

每次启动服务，到 Zookeeper 查看 `/leaf_forever` 下是否有自己注册过的节点，有取回，没有生成序列节点。

弱依赖 Zookeeper：只在启动时请求一次。

### 时钟回拨

每隔一段时间，向 Zookeeper 上传自己的时钟。并检查自己的时钟是否回拨了。

回拨了则抛出异常，服务关闭。

# 数据库方案

使用数据库的 `auto_increment_increment` 保证 ID 的自增。

```sql
begin;
REPLACE INTO Tickets64 (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
commit;
```

## 优点

- 实现简单。
- 单调递增。

## 缺点

- 每次生成 ID 都要写入数据库，效率差。瓶颈限制在单台MySQL的读写性能。
- 依赖数据库。

## 优化

### 多台服务 + 分段

多台服务依赖一个数据库，每个服务每次把数据库的值递增 `step`。

服务处理完 `step` 个 ID 后，再去到数据库请求。

趋势上是递增的。

# 微信序列号

每个 user 对应一个 ID，每次一个 user 取到的 ID 比它之前取到的大（不一定递增 1）。

每个 user 设置一个 `maxID`，在 ID 增加到这个值后，`maxID` 增加 `step`，并写入磁盘。

为了重启的时候，不读取大量的 `maxID`。多个 user 公用一个 `regionID`。和 `maxId` 类似，某个 `maxID` 达到 `regionID` 后，`regionID` 增加一定的步长。



