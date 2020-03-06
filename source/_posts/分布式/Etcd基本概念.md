---
title: Etcd 基本概念
date: 2020-03-06 21:11:00
categories:
	- 分布式
tags:
	- 分布式
typora-root-url: ../../
---

# 引用

- [分布式锁的最佳实践之：基于 Etcd 的分布式锁](https://youyou-tech.com/2019/07/01/%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E7%9A%84%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5%E4%B9%8B%EF%BC%9A%E5%9F%BA%E4%BA%8EEtcd%E7%9A%84%E5%88%86%E5%B8%83/)
- [分布式锁第五篇 基于etcd](https://www.jianshu.com/p/7abfb83cfa08)
- [Etcd存储的实现](https://www.codedump.info/post/20181125-etcd-server/#revision概念)

# Lease 机制

租约机制（TTL，Time To Live）。Etcd 可以为存储的 key-value 对设置租约，当租约到期，key-value 将失效删除。

支持续约，通过客户端可以在**租约到期之前**续约，以避免 key-value 对过期失效。

支持解约，一旦解约，与该租约绑定的 key-value 将失效删除。

# Prefix 机制

前缀机制，也称目录机制。

如两个 key 命名如下：key1=`/mykey/key1` 、key2=`/mykey/key2`。

可以通过前缀-`/mykey` 查询，返回包含两个 key-value 对的列表。

# Watch 机制

监听机制。

Watch 机制支持 Watch 某个固定的key，也支持 Watch 一个范围（前缀机制），当被 Watch 的 key 或范围发生变化，客户端将收到通知。

# Revision 机制

每个 key 带有一个 Revision 属性值，etcd 每进行一次事务对应的全局 Revision 值都会加一，因此每个 key 对应的 Revision 属性值都是全局唯一的。通过比较 Revision 的大小就可以知道进行写操作的顺序。

# 实现分布式锁

和 {%post_link 分布式/Zookeeper基本概念 Zookeeper基本概念 %} 类似。

在目录 `/lock` 下， `put` key 为 `/lock/UUID` 的元素。通过比较 `/lock` 下元素的 RevsionID 进行操作。

