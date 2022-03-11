---
title: 【ZooKeeper】The ZooKeeper Data Model
date: 2022-02-25 16:54:06
categories: ZooKeeper
tags: ZooKeeper
---

zookeeper拥有多层的命名空间，就像一个分布式文件系统。唯一的不同就是它的每个节点都可以包含数据，就像文件系统中允许一个文件可以作为目录。它没有相对路径。

<!-- more --> 

### ZNodes

zookeeper中每个节点都都被叫做znode。它包含了数据本身、数据变更的版本和访问控制(acl)的版本

#### Watches

客户端可以在znodes上设置监控，当znode出发了监控，zookeeper就会向客户端发送一个通知，然后清除监控。

#### Data Access

数据在每个节点中的读写操作都是原子的，每个节点都有一个 Access Control List (ACL)来决定谁可以干什么。

#### Ephemeral Nodes

zookeeper可以创建临时性的节点，这些节点只有在session创建和结束期间才是活跃的，可以通过**getEphemerals()** 获取当前session创建的临时节点。

#### Sequence Nodes

有序的节点，拥有唯一的名字。

#### Container Nodes

容器节点是专门用来作为leader或者lock。当容器的最后一个节点被删除，容器就会被服务端在未来的某个时间删除。

#### TTL Nodes

当创建永久或永久有序的节点，你可以选择设置过期时间。

---

### Time in ZooKeeper

