---
author: "weedge"
title: "CRDT"
date: 2021-12-28T10:26:23+08:00
tags: [
	"CRDT",
]
categories: [
	"技术",
]

---

## 介绍

**无冲突复制数据类型**(CRDT: **conflict-free replicated data type**) 是一种简化分布式数据存储系统和多用户应用程序的数据结构。

在许多系统中，某些数据的副本需要存储在多台计算机上。此类系统的示例包括：

- 在本地设备上存储数据，并且需要将该数据同步到属于同一用户的其他设备（同一用户多端设备同步，例如日历、笔记、联系人或提醒）的移动应用程序；
- 分布式数据库，维护数据的多个副本（在同一数据中心或不同位置,一般是不同数据中心的多活场景），以便在某些副本离线时系统继续正常工作；
- 协作软件，例如 Google Docs、Trello、Figma 或许多其他软件，其中多个用户可以同时更改同一文件或数据；
- [边缘计算场景](https://docs.microsoft.com/en-us/shows/tech-exceptions/concordant-always-know-what-to-expect-from-your-data)，比如多个手机/车载app 在无信号的森林中，产生的本地离线协同数据，多个设备同步到云上处理;
- 大规模数据存储和处理系统，复制数据以实现全球可扩展性。

<!--more-->

所有此类系统都需要处理数据可能在不同副本上同时修改的事实。从广义上讲，有两种可能的方法来处理此类数据修改：

- 强一致性复制：

  在这个模型中，副本相互协调以决定何时以及如何应用修改。这种方法支持强一致性模型，例如可序列化事务和可线性化。然而，等待这种协调会降低这些系统的性能；此外，CAP 定理告诉我们，当副本与系统的其余部分断开连接时（例如，由于网络分区，或者因为它是具有间歇性连接的移动设备），不可能对副本进行任何数据更改。

- 乐观复制：

  在此模型中，用户可以独立于任何其他副本修改任何副本上的数据，即使该副本离线或与其他副本断开连接。这种方法可实现最大的性能和可用性，但当多个客户端或用户同时修改同一条数据时，它可能会导致冲突。当副本相互通信时，需要解决这些冲突。

无冲突复制数据类型 (CRDT) 用于具有乐观复制的系统，它们负责解决冲突。CRDT 确保，无论在不同的副本上进行什么数据修改，数据始终可以合并到一致的状态。此合并由 CRDT 自动执行，无需任何特殊的冲突解决代码或用户干预。

## 原理



## 实现

### [automerge](https://github.com/automerge/automerge)

### [Yjs](https://github.com/yjs/yjs)

### [redis (CRDB)](https://redis.com/redis-enterprise/technology/active-active-geo-distribution/)



## references

1. [wiki: Conflict-free replicated data type](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)
2. [awesome-crdt#know-before-you-go](https://github.com/alangibson/awesome-crdt#know-before-you-go)
3. **[crdt.tech-resources](https://crdt.tech/resources)** **[crdt.tech-implementations](https://crdt.tech/implementations)**
4. [Conflict-free Replicated Data Types.pdf](https://hal.inria.fr/hal-00932836/file/CRDTs_SSS-2011.pdf)
5. **[A comprehensive study of Convergent and Commutative Replicated Data Types.pdf](https://hal.inria.fr/file/index/docid/555588/filename/techreport.pdf?spm=a2c6h.12873639.0.0.a25177aaoVMJMH&file=techreport.pdf)**
6. **[Conflict-free Replicated Data Types: An Overview.pdf](https://arxiv.org/pdf/1806.10254.pdf)**
7. [A framework for establishing Strong Eventual Consistency for Conflict-free Replicated Data types.pdf](https://www.isa-afp.org/browser_info/current/AFP/CRDT/document.pdf)
8. **[A Conflict-Free Replicated JSON Datatype.pdf](https://arxiv.org/pdf/1608.03960.pdf)** [code](https://cs.paperswithcode.com/paper/a-conflict-free-replicated-json-datatype) [vedio introduce](https://www.youtube.com/watch?v=TRvQzwDyVro)
9. **[Automerge: Making Servers Optional for Real-Time Collaboration](https://www.youtube.com/watch?v=GXJ0D2tfZCM)** [slide](https://speakerdeck.com/ept/automerge-making-servers-optional-for-real-time-collaboration) [github.com/automerge](https://github.com/automerge)
10. [Near Real-Time Peer-to-Peer Shared Editing on Extensible Data Types.pdf](https://www.researchgate.net/publication/310212186_Near_Real-Time_Peer-to-Peer_Shared_Editing_on_Extensible_Data_Types)
11. [Toward Fast and Reliable Active-Active Geo-Replication for a Distributed Data Caching Service in the Mobile Cloud](https://www.sciencedirect.com/science/article/pii/S1877050921014101)
12. [redis: active-active-geo-distribution](https://redis.com/redis-enterprise/technology/active-active-geo-distribution/) [redis: Developing applications with Active-Active databases](https://docs.redis.com/latest/rs/references/developing-for-active-active/)
13. [riak-kv:Data Types](https://docs.riak.com/riak/kv/2.0.0/developing/data-types/?spm=a2c6h.12873639.0.0.a25177aaoVMJMH)
14. **[阿里云redis：CRDT——解决最终一致问题的利器](https://developer.aliyun.com/article/635632)**  [多中心容灾实践：如何实现真正的异地多活？](https://developer.aliyun.com/article/781709?utm_content=g_1000239229)
15. [基于CRDT的数据最终一致性](https://zhuanlan.51cto.com/art/202107/674513.htm)
16. [多主复制](http://ddia.vonng.com/#/ch5?id=多主复制)

