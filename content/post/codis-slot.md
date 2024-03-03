---
author: "weedge"
title: "kv-codis迁移"
date: 2023-07-30T10:26:23+08:00
tags: [
	"kv",
]
categories: [
	"技术",
]


---

{{< neteasemusic id="1890574156" >}}

最近在搞一个kv系统，想接入codis来管理slot，实现数据的迁移，进行scaling水平扩展，从网上找了点资料，然后随笔记录梳理一下，以便相应的代码可以联调接入时优化下~

<!--more-->

### codis slot迁移

#### 同步：

1. 主动调用`SLOTSMGRTTAGSLOT` 进行同步迁移,  对val dump成rdb 格式进行单个key的同步，直到db,slot同步完成

   ![](https://github.com/weedge/mypic/raw/master/codis1)

2. key所属的slot正在迁移，则被动调用`SLOTSMGRTTAGONE`命令将这个key迁移完成再返回给客户端，即必须要迁移这个key完成才返回给客户端。

#### 异步：

（针对redis 处理命令的主线程的优化，因为是单线程，命令通eventloop分发在单线程中依次执行，为了缓解该问题，进行了优化 ）

1. 主动调用`SLOTSMGRTTAGSLOT-ASYNC`命令，整体流程和同步差不多，主要是将耗时命令进行拆解成耗时小的命令进行批量处理。

   1）源Redis对key进行序列化异步发送给目标Redis；

   2）目标Redis通过Restore还原后回复给源Redis； 

   3）源Redis收到目标Redis确认后标记这个key迁移完成，迁移下一个key；

   对于大key，如一个长度为1W的list，Codis会将key分拆成多个命令chunck，因为通过不断的rpush最终的结果一样；

   ![](https://github.com/weedge/mypic/raw/master/codis2)

   Codis会在每一个拆分后的指令中加上一个临时TTL；等全部拆分的指令执行成功才会删除本地的key；因此即使中途迁移失败，已迁移成功的key也会超时自动删除，最终效果就好比迁移没有发生一样。 但是这里不是原子的，如果执行设置真正的过期时间expire key ttl 执行超时了，这个时候目标节点已经设置成功了，但是源节点没有收到ack，本地没有删除，返回迁移失败(其实已经成功了)，则需要重新触发迁移。

   ![](https://github.com/weedge/mypic/raw/master/codis3)

   迁移通过分批pipeline进行了优化

   ![](https://github.com/weedge/mypic/raw/master/codis4)

2. key所属的slot正在迁移，则被动调用`SLOTSMGRT-EXEC-WRAPPER`命令，将key请求操作发给迁移的slot节点，返回错误码(-1 参数错误，0 不存在，1.命令是写操作并且正在迁移，2.命令是读操作并且执行)

   1. 如果正在迁移并且当前命令是写命令则返回错误码1，由Prxoy进行重试。
   2. 如果是读命令则执行返回错误码2和执行结果，由proxy 返回。
   3. 如果key已经迁移走，则返回错误码0，Proxy需要更新路由表。



#### 同步和异步有两个区别：

一是处理请求的不同，如果当前要操作的key所属Slot正在迁移，同步处理会发送命令等待后端迁移完成才往下操作，异步则是将当前请求封装成一次SLOTSMGRT-EXEC-WRAPPER调用，并且将操作命令及参数都发送过去，后者会判断这个key是否在迁移或阻塞，如果是并且当前为写命令则直接返回失败，由Proxy重试。 

二是迁移逻辑不同，同步会调用SLOTSMGRTTAGSLOT迁移，异步则是调用SLOTSMGRTTAGSLOT-ASYNC，前者每次随机迁移一个key，异步的过程则复杂得多，对于小key需要确认才算迁移完成，对于大key还会分拆成多条命令，以保证不阻塞主流程，并且在拆分后的命令都加上TTL，以保证如果中途失败目标Redis的key会及时清掉而不会产生脏数据。

同步,异步迁移通过dashboard配置文件设置 migration_method。

Gist: https://gist.github.com/weedge/8d1d37963a2dfd7ecd47fefc7d8016b1 

#### 其他kv引擎接入：

理论上只要实现了codis的迁移命令就可以对数据实例slot进行管理操作了，还可以进行不同类型实例的数据迁移，不同类型kv存储引擎实例来实现不同的功能slot上， 比如 热数据 可以请求到 内存kv中， 冷数据可以请求到磁盘kv中，有种乾坤大挪移的感觉~，主要是利用codis管理key-> slot 与 node之间的会话关系，其实现在云原生网关中常说的控制面(control)和数据面(data)，这里分别对应dashboard(发送迁移命令)和 proxy (tcp长连,根据session db , key->slot与node映射的路由规则转发命令)。

如果管理迁移slot的kv节点处理命令不是单线程的，比如[dragonfly](https://github.com/dragonflydb/dragonfly) 等处理命令是多线程的sharding 内存(多线程IO模型 [helio](https://github.com/romange/helio) 支持io_uring)，以及 [wedis](https://github.com/weedge/wedis)  处理命令是多协程（一个连接会话对应一个协程，本质调度执行是内核线程, 网络IO模型根据场景使用netpoll可以优化下,以及批量迁移batch put 尽量减少io），则操作只需要优化实现 同步命令 `SLOTSMGRTTAGSLOT` 和被动触发的 `SLOTSMGRTTAGONE` 可以满足需求，当然如果想对接异步操作命令也是可以的。

tips: 

1. 对于slot相关操作如果已插件的方式对接到kv引擎中实现，是比较符合工程实现的，一种是动态链接(c/c++)，通过MODULE LOAD 命令，或者loadmodule 配置项，可以参考[RedisXSlot](https://github.com/weedge/RedisXSlot) module，实现异步非阻塞执行  `SLOTSMGRTTAGSLOT` 和被动触发的 `SLOTSMGRTTAGONE` 等同步命令; 一种是编译时静态链接(plugin比较鸡肋，两种方式：1. golang 依赖注入运行的时候配置区分，2. golang本身是静态编译, 通过go build tag的方式分开编译不同的feature，像rust cargo build with feature crate)
1. 如果使用redis cluster 是去中心化的scaling 方案，slot的迁移会话管理方式嵌入到node节点中，通过gossip协议进行协同，运维相对简单，但是少了些灵活，当然也可以做一层控制面来管理会话元数据，当集群节点比较多的时候。(这个实现下cluster中的slot相关命令可以在思考记录一些~)
1. codis现在没有维护了，但是fork出来进行二次开发，和内部监控系统打通，也可以借鉴下reborndb,但是没有但是了。。。

题外话：

> 基础组件(中间件)如同搭积木(前后端本质上一样，后端需要考虑分布式情况)，稳固牢靠，易扩展，这就考验需求中事件本质的抽象，这个本质的抽象感觉也有点像套路；如果把开源组件进行拼装，给出需求抽象出的安装步骤，画出架构图，交给大模型， 能否给出相关详细设计文档，甚至直接代码模块化，具体细节根据业务需求来写，有套路的东东，应该可以AI化；如果反过来呢，根据业务需求，画出架构图，甚至给出ppt模版，给出模块化的代码，细节也是根据业务来填充；现在有预训练好的开源大模型黑盒，这些数据可以私有安全部署(利用开源，然后商业化闭源，想想都有意思哈~) :d ，但是前提是：提出问题者需要某个垂直领域认知提升上的经验教训(局内人从坑里爬出来过或者旁观者认真思考过前人躺过的坑)，能把控全局~ 

声明：一些资料图片来自互联网，农夫山泉



### 参考：

1. https://github.com/CodisLabs/codis
2. [redis-cluster](https://redis.io/docs/management/scaling/)
3. [dragonfly/high-availability](https://www.dragonflydb.io/docs/managing-dragonfly/high-availability)
4. [RebornDB: the Next Generation Distributed Key-Value Store](https://docs.google.com/document/d/1hJXa0LjMaGYHgL3PeQQu3QMrstxof241OVxbdDKvSbY/edit?usp=sharing)