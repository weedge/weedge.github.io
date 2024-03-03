---
author: "weedge"
title: "译：Uber是如何解决数据一致性问题的？"
date: 2023-08-16T10:26:23+08:00
tags: [
	"oneday",
]
categories: [
	"技术",
]
---

# 介绍

![](https://github.com/weedge/mypic/raw/master/oneday/how-did-uber-solv-data-consistency-problem/1.png)

Uber 的请求流程相当复杂，从上图可以看出，他们使用 Spanner 来存储大量数据。Spanner 是一种完全托管的关键任务关系数据库服务，可提供全球范围内的事务一致性、自动同步复制以实现高可用性。

<!--more-->

但在此之前，最初的架构有本地数据库。更具体地说，他们使用 Cassandra 来存储实时数据。另外，在 Cassandra 之上，他们还使用了 Ringpop。有关更多详细信息，您可以查看[此博客文章](https://www.uber.com/blog/ringpop-open-source-nodejs-library/)。

但当扩展到数百万个并发请求时，Cassandra 很难保证低延迟写入。另一个问题是工程团队开始注意到需要多行和多表写入的复杂存储交互（我不知道这意味着什么）。但无论如何，本地 Cassandra DB 开始变得非常具有挑战性。

对于 Uber 来说，数据不一致可能会导致两名司机接送同一位顾客。

解决方案是构建一个应用层框架，引入中间层(间), 使用**Saga 模式**来编排数据库操作。

**Saga设计模式**

![](https://github.com/weedge/mypic/raw/master/oneday/how-did-uber-solv-data-consistency-problem/2.png)

Saga设计模式是一种在分布式事务场景中管理跨微服务的数据一致性的方法。saga是更新每个服务并发布消息或事件以触发下一个事务步骤的事务序列。如果某个步骤失败，saga 会执行补偿事务来抵消前面的事务。

简而言之，如果事务期间出现问题（*事务*是单个逻辑或工作单元，有时由多个操作组成），则应恢复之前的更改。

**什么是事务？**

事务是单个逻辑或工作单元，有时由多个操作组成。在事务中，*事件*是实体发生的状态更改，*命令*封装了执行操作或触发后续事件所需的所有信息。

事务必须是*原子的、一致的、隔离的和持久的（ACID）*。单个服务内的事务是ACID的，但是跨服务的数据一致性需要跨服务的事务管理策略。

在多服务架构中：

- *原子性* 是一组不可分割且不可简化的操作，要么全部发生，要么不发生。
- *一致性* 是指事务仅将数据从一种有效状态带到另一种有效状态。
- *隔离性* 保证并发事务产生与顺序执行的事务产生的数据状态相同的数据状态。
- *持久性* 确保即使在系统故障或断电的情况下，已提交的事务仍保持提交状态。

**解决方案**

*Saga 模式使用本地事务*序列提供事务管理。本地事务是 saga 参与者执行的原子工作。每个本地事务都会更新数据库并发布消息或事件以触发Sagas中的下一个本地事务。如果本地事务失败，saga 会执行一系列*补偿事务*，以撤消先前本地事务所做的更改。

有两种常见的 saga 实现方法：choreography 和 orchestration。每种方法都有自己的一套挑战和技术来协调工作流程。

**choreography**是一种协调sagas的方法，参与者可以在没有集中控制点的情况下交换事件。通过编排，每个本地事务都会发布触发其他服务中的本地事务的域事件。rocketMQ中的消息事务采用的这种编排模式，满足RC隔离(*Read Committed*)。

![](https://github.com/weedge/mypic/raw/master/oneday/how-did-uber-solv-data-consistency-problem/3.png)

**orchestration**是一种协调sagas的方法，其中集中控制器告诉saga参与者要执行哪些本地事务。saga 协调器处理所有事务并告诉参与者根据事件执行哪个操作。编排器执行 saga 请求，存储和解释每个任务的状态，并通过补偿事务处理故障恢复，本质是2PC。

![](https://github.com/weedge/mypic/raw/master/oneday/how-did-uber-solv-data-consistency-problem/4.png)

观看视频[How does Uber scale to millions of concurrent requests?](https://www.youtube.com/watch?v=DY2AR8Wzg3Y)了解有关 Uber 迁移及其挑战的更多信息。如果您想了解有关 Saga 模式的更多详细信息参考[此链接](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga)。

**spanner：**

![](https://github.com/weedge/mypic/raw/master/oneday/how-did-uber-solv-data-consistency-problem/5.png)

### 总结

主要是为了保证数据一致性， 不出现司机乘客对同一个事务操作异常，从原来的nosql 转向 newsql， 引入了google cloud spanner (new SQL) 提供全球范围内的事务一致性、自动同步复制以实现高可用性。结合uber这个场景，然后看下spanner的论文去寻求解决方案，了解下细节结合场景进行设计。视频中提到了大规模数据迁移的挑战，特别是全球跨数据中心的迁移同步，有相应的网络优化技术（这些技术解决方案刚出来的时候国内有跟进，当然技术文章介绍肯定少不了）

题外话：

> 需求场景驱动技术方案去落地，技术方案不一定最优解， 能满足当前需求场景，过度设计优化产生无意义的消耗，不过可以留下接口可以去扩展，满足不同场景的进一步优化，软件中没有通用的解决方案银弹。

### reference

1. https://medium.com/@dmosyan/how-did-uber-solve-data-consistency-problem-dcdd39bd3ed6
2. https://www.uber.com/blog/ringpop-open-source-nodejs-library/
3. https://www.uber.com/blog/fulfillment-platform-rearchitecture/
4. https://www.uber.com/blog/building-ubers-fulfillment-platform/
5. https://www.youtube.com/watch?v=DY2AR8Wzg3Y
6. https://research.google/pubs/pub39966/
