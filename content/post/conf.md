---
author: "weedge"
title: "设计-配置平台"
date: 2021-11-18T10:26:23+08:00
tags: [
	"design","conf",
]
categories: [
	"技术",
]

---

### 介绍

业务服务在启动时加载配置，配置分为静态配置和动态配置，静态配置如服务监听的端口，日志路径，访问依赖服务单元不同云(region / zone) 域名地址等；动态配置包括业务配置，流程管理配置，策略配置等；静态配置服务启动之后不会更改，而动态配置在服务启动运行时可以动态热加载；将配置的内容和版本进行分离，关注配置的管理而无需关注配置的内容，配置内容用户可以自定义配置内容，使用json-schema进行校验，提供自定义配置内容json给后台ui前端进行单个整体配置的交互，配置后台只需加载json-schema进行校验提交内容，json-schema由用户上传提供地址即可。

<!--more-->

### 目标

通过配置后台，业务服务流程可控，可配置，可视化，隔离已有业务场景配置，提高人效；

### 功能

#### 后台基础功能

1. 后台管理发布生产环境的动态配置，包括新增，修改，审核，发布，回滚等功能；
2. 多人发布的时候，检查当前的版本是否已经发布，并进行版本diff;
3. 配置是服务接口维度进行操作管理， 以服务粒度进行整体发布；
4. 提供获取当前已发布的服务配置接口，用于订阅端拉取；
5. 提供RBAC权限管理;

#### 动态配置热加载功能

**业务服务配置获取方案：**

1. 服务配置修改发布，通过发布/订阅模式(Pub-Sub Pattern)通知业务服务进行加载；可用组件为支持pub/sub命令的redis协议kv服务，或者通过get(mget)/set(mset) incr(seq)命令进行模拟实现；
2. 服务配置修改发布，通过观察者模式(Observer Pattern) watch机制通知业务服务进行实时加载；可以使用etcdv3的watch机制实现;

两种方案都可以，看是否有对应的高可用组件服务支持；如果都有，建议使用etcdv3(client 3.4+)的watch机制，通过长链接来通知变更事件。

**加载方式有两种实现方案：**

1. 通过部署单独agent进程来获取配置进行解析写入指定配置路径文件中(重启时)和共享内存中(运行时)，先写文件，成功了，在写共享内存；然后由业务服务读取；

   优点：方便单独维护，无需业务方关心，只需关心加载配置的路径在后台维护，以及共享内存中的配置结构；

   缺点：多了一次进程之间的交互；业务方代码中需要有读取共享内存的逻辑考虑；

2. 提供配置加载lib库，由业务方启动逻辑代码中调用；将获取配置进行解析写入指定配置路径文件中，加载到业务进程配置结构中使用即可；

   优点：业务方直接调用lib库中的方法，无需关心加载逻辑；

   缺点：假如业务方是不同的语言编写，需要提供不同语言版本的lib库方法；

总体来说，2种方案都可行，如果统一技术栈，比如使用golang进行开发，可以选用第二种方案；

但是这样就可以了么？还有业务集群配置一致性需要考虑到，比如业务集群某台机器突然抽风，未加载最新的配置，打到这台机器的请求和其他正常加载配置的机器的请求逻辑会有所不同，导致整体服务接口不是幂等的，会出现不一致的情况；可以借鉴一下 [gaea配置热加载设计与实现](https://github.com/XiaoMi/Gaea/blob/master/docs/config-reloading.md)；

**配置一致性方案：**

通过两阶段提交的方式，由业务服务侧提供prepare接口和commit接口；

准备阶段：conf-dashboard模块发布的时候获取业务服务的ip列表并发调用每个业务服务ip的prepare接口，开始触发业务服务本地配置的更新流程，首先通过conf-dashboard模块获取最新版本和当前版本比较，如果相同就无需比较了，否则从获取配置接口获取配置进行更新；更新流程不是直接替换线上正在使用的文件，而是生成一个准备文件进行提交时替换；每个prepare接口同步调用返回结果都ok了，进入下一步提交阶段，如果其中一个返回失败，发布失败，展现失败详情；

提交阶段：prepare成功后；conf-dashboard模块继续并发调用的commit接口，开始文件替换，加载至缓存配置结构中；这个操作比较快，都提交成功之后，发布成功，否则发布失败，展现失败详情；

失败处理：如果某台机器发布失败，可以排查下具体原因之后(原因应尽量在prepare和commit接口中详细给出)，看是否跳过继续发布还是会滚，重复发布不影响，有md5和版本比较；

tips: 如果发布机器很多，并发力度根据conf-dashboard模块部署机器接口负载，可以自适应调整降低；

**兜底检查更新方案：**

配置发布成功后，调用业务服务机器的meta接口获取最新更新文件信息，进行兜底检查确认；

业务服务开启兜底定时任务获取从conf-dashboard获取配置版本，配置信息进行兜底更新, 每隔10～30分钟执行一次；

tips: 请求conf-dashboard获取配置信息的接口随机打散，防止并发流量对conf-dashboard模块的影响；

定时任务流程图如下：

![cron-confmg](https://raw.githubusercontent.com/weedge/mypic/master/confmg.drawio.png)

### 整体交互框架

![config-arch](https://raw.githubusercontent.com/weedge/mypic/master/config-arch.drawio.png)



### 数据库核心动态配置表设计

当前服务版本和当前conf版本是1对多的场景，而历史服务快照版本和历史conf快照版本是多对多的场景；

如图所示：

![config-version](https://raw.githubusercontent.com/weedge/mypic/master/config-version.png)

服务类型枚举表 service_type

| 枚举值 | 描述 |
| ------ | ---- |
| 0      | 无   |

配置类型枚举表 conf_type

| 枚举值 | 描述 |
| ------ | ---- |
| 0      | 无   |

枚举值由配置平台统一分配收敛管理



当前服务版本表 tb_service_cur_ver

| 字段         | 类型    | 描述                                       |
| ------------ | ------- | ------------------------------------------ |
| id           | bigint  | 主键                                       |
| service_type | tinyint | 服务类型 比如：0.无 1.liveme 2.livestation |
| cluster_name | string  | 服务集群名称                               |
| cur_ver      | bigint  | 服务当前版本                               |
| max_ver      | bigint  | 最大版本                                   |
| pub_time     | uint    | 发布时间                                   |
| create_time  | uint    | 创建时间                                   |
| update_time  | uint    | 更新时间                                   |


当前conf版本表 tb_conf_cur_ver

| 字段               | 类型          | 描述                                                         |
| ------------------ | ------------- | ------------------------------------------------------------ |
| id                 | bigint        | 主键                                                         |
| cur_service_ver_id | bigint        | 当前服务版本 id                                              |
| conf_type          | tinyint       | conf类型，比如：<br />0.无 <br />1.livemeDSL <br />101.livestationAggrDSL 102.livestationSceneDSL |
| cur_ver            | bigint        | conf当前版本                                                 |
| max_ver            | bigint        | 最大版本                                                     |
| conf_name          | varchar(128)  | conf名称                                                     |
| conf_target_path   | varchar(128)  | conf生成的目标路径                                           |
| conf_val           | varchar(8192) | conf当前内容                                                 |
| pub_time           | uint          | 发布时间                                                     |
| create_time        | uint          | 创建时间                                                     |
| update_time        | uint          | 更新时间                                                     |


服务版本用户操作记录表 tb_service_op_record

| 字段         | 类型    | 描述                                               |
| ------------ | ------- | -------------------------------------------------- |
| id           | bigint  | 主键                                               |
| service_type | tinyint | 服务类型                                           |
| cluster_name | string  | 服务集群名称                                       |
| check_status | tinyint | 服务配置审核状态: 0.待审核 1.审核不通过 2.审核通过 |
| op_uid       | bigint  | 操作者uid                                          |
| pub_time     | uint    | 发布时间                                           |
| create_time  | uint    | 创建时间                                           |
| update_time  | uint    | 更新时间                                           |


conf版本用户操作记录 tb_conf_op_record

| 字段             | 类型          | 描述                       |
| ---------------- | ------------- | -------------------------- |
| id               | bigint        | 主键                       |
| service_op_id    | bigint        | 服务版本操作 id            |
| conf_type        | tinyint       | conf类型                   |
| conf_name        | varchar(128)  | conf名称                   |
| conf_target_path | varchar(128)  | conf生成的目标路径         |
| conf_val         | varchar(8192) | conf当前内容               |
| is_del           | tinyint       | 是否删除，0.可用，1.不可用 |
| pub_time         | uint          | 发布时间                   |
| create_time      | uint          | 创建时间                   |
| update_time      | uint          | 更新时间                   |


服务历史版本快照表 tb_service_ver_snapshot

| 字段         | 类型    | 描述         |
| ------------ | ------- | ------------ |
| id           | bigint  | 主键         |
| service_type | tinyint | 服务类型     |
| cluster_name | string  | 服务集群名称 |
| history_ver  | bigint  | 服务历史版本 |
| pub_time     | uint    | 发布时间     |
| create_time  | uint    | 创建时间     |
| update_time  | uint    | 更新时间     |


conf历史版本快照表 tb_conf_version_snapshot

| 字段             | 类型          | 描述               |
| ---------------- | ------------- | ------------------ |
| id               | bigint        | 主键               |
| conf_type        | tinyint       | conf类型           |
| conf_name        | varchar(128)  | conf名称           |
| history_ver      | bigint        | conf历史版本       |
| conf_target_path | varchar(128)  | conf生成的目标路径 |
| conf_val         | varchar(8192) | conf历史快照内容   |
| pub_time         | uint          | 发布时间           |
| create_time      | uint          | 创建时间           |
| update_time      | uint          | 更新时间           |


服务conf历史版本关联表 tb_service_conf_ver_snapshot

| 字段                    | 类型   | 描述                |
| ----------------------- | ------ | ------------------- |
| id                      | bigint | 主键                |
| service_ver_snapshot_id | bigint | 服务历史版本快照 id |
| conf_ver_snapshot_id    | bigint | conf历史版本快照 id |
| create_time             | uint   | 创建时间            |
| update_time             | uint   | 更新时间            |

### 场景模拟

1. 新增版本保存更新删除

   

2. 审核

   

3. 发布

   

4. 回滚

   

5. 获取当前服务conf版本配置

   服务粒度整体版本文件一次拉取下来，这里是接口形式提供， 后续可以提供将可用版本以服务粒度打个包提交；如果后续有多服务一起捆绑打包，需要考虑上线依赖顺序，暂时不提供



### 接口

线下内网域名访问需要配置router模块中的ngx proxy配置，新增一个location /confdashboard 进行路由跳转；或者指定ip端口访问路径；

线上通过通过服务发现模块来获取confdashboard服务单元ip列表负载均衡进行访问；可以多机房部署，根据业务服务部署场景定；

#### conf-dashboard 接口 

提供给后台前端的CRUD接口不在这里描述，通过[smallnest/gen](https://github.com/smallnest/gen)生成RESTful接口提供给前端使用就行，修改操作，根据不同的conf_type加载不同的json_schema进行校验，以及审核，发布接口；主要是提供给业务服务调用的接口实现，如下：

请求调用方式：http1.1 POST 或者 GRPC

公共返回参数：

| 字段   | 类型   | 描述            |
| :----- | :----- | :-------------- |
| errNo  | int    | 错误号，默认为0 |
| errStr | string | 错误描述        |
| data   | json   | 返回响应数据    |

1. 获取当前服务配置文件版本   /confdashboard/v1/getcurversion

   请求：

   | 字段        | 参数是否必须 | 类型    | 描述                                 |
   | ----------- | ------------ | ------- | ------------------------------------ |
   | serviceType | 必须         | tinyint | 服务类型:0.无 1.liveme 2.livestation |

   响应：

   | 字段   | 类型 | 描述     |
   | ------ | ---- | -------- |
   | curVer | int  | 当前版本 |
   | maxVer | int  | 最大版本 |

   

2. 获取当前服务文件配置内容   /confdashboard/v1/getcurserverconf

   (服务粒度整体版本文件一次拉取下来，这里是接口形式给出对应配置数据，兜底定时轮训获取）

   请求：

   | 字段           | 参数是否必须 | 类型    | 描述                                 |
   | :------------- | ------------ | ------- | ------------------------------------ |
   | serviceType    | 必须         | tinyint | 服务类型:0.无 1.liveme 2.livestation |
   | confName       | 可选         | string  | 服务本地配置文件名称                 |
   | confTargetPath | 可选         | string  | 服务本地配置路径                     |

   响应data：

   | 字段        | 类型                                                         | 描述                 |
   | ----------- | ------------------------------------------------------------ | -------------------- |
   | downloadUrl | string                                                       | 服务配置整体打包地址 |
   | confData    | map<string>confData<br />(map<{confTargetPath}/{confName}>{confData}) | 服务配置列表         |

   confData

   | 字段    | 类型   | 描述             |
   | ------- | ------ | ---------------- |
   | version | int64  | 文件当前版本     |
   | data    | []byte | 文件当前版本内容 |

   

3. 业务服务配置更新结果报告 /confdashboard/v1/reportBizServConf

   请求：

   | 字段        | 参数是否必须 | 类型                  | 描述                                 |
   | ----------- | ------------ | --------------------- | ------------------------------------ |
   | serviceType | 必须         | tinyint               | 服务类型:0.无 1.liveme 2.livestation |
   | reportInfos | 必须         | map<string>reportInfo | 业务服务配置更新结果报告列表         |
   | clusterName | 必须         | string                | 集群名称                             |

   reportInfo

   | 字段             | 类型   | 描述                         |
   | ---------------- | ------ | ---------------------------- |
   | version          | int64  | 版本                         |
   | confFilePath     | string | 业务配置文件路径（绝对路径） |
   | md5sum           | string | 文件MD5值                    |
   | confUpdateTime   | string | 业务配置文件更新时间         |
   | updateResultCode | int    | 0, 更新成功，1，更新失败     |
   | updateErrStr     | string | 更新失败原因                 |

   响应data：

   | 字段 | 类型 | 描述 |
   | ---- | ---- | ---- |
   |      |      |      |




#### conf-agent/lib 业务服务接口 

通过在业务服务机部署agent, 或者使用lib库封装函数的方式，最终都是需要提供获取配置的服务提供对应的接口，来保证整体服务配置的一致性，这里采用lib库封装函数提供给业务服务启动时调用这个函数的方式，是否启动单独启动端口是可选的，如果是在业务服务中启动，就用业务服务的端口，如果是单独agent方式启动，使用单独的端口；接口定义如下：

1. 获取配置开始准备替换工作 prepare接口 /{serviceType}/localconf/prepare

   

2. 提交本地替换工作 commit接口 /{serviceType}/localconf/commit

   

3. 获取本地配置文件元数据，比如md5,路径,版本等，meta接口 /{serviceType}/localconf/meta

### 施工

可以进行模块划分，按人力进行分工，可以用Project或者trello这些工具来管理整个项目需求，开发，测试，上线周期；也可以内部整合jira 和 wiki 等平台工具进行管理;

这个有点像建筑施工队，有了设计稿，推演几遍，满足整体需求和目标；剩下的是去实施了，实施的话需要，整体系统架构设计的建筑工程师👷，懂这行工具的专工👷，以及领队包工头👷，大项目可能还有项目监工👷‍♀️，监控进度，以及交互后的质量把控工程师👷；一起配合把事干好，让用户和老板满意。

### 总结

前期业务为了满足快速业务迭代，业务动态配置可能都是随着业务服务一起上线，或者通过单独的代码库进行管理分开单独上线；这个可能会因为代码上线了，但是配置忘记上线的情况；或者只需要修改配置上线，而不需要修改代码，也需要单独部署一次业务服务，如果涉及多个服务模块，不能及时响应了，不够KISS；为了满足业务集群配置化管理，配置平台需要管理业务动态配置来提高人效，保证业务集群上线配置的整体一致，通过后台页面来管理配置和历史配置，追踪配置的修改情况；需要注意的是，第一次上线的配置，可以先在测试环境配置平台上配置好，测试好之后，在到线上平台配置发布上线，然后在部署业务服务代码；发布配置也需要在线上未接入流量的机器上，测试之后才能上。

### Q&A

1. 为什么不直接用etcd来管理配置呢？去掉mysql, 配置直接存etcd不行么？

   主要是因为mysql用来提供管理配置实体的关系，用于后台页面修改使用；如果是k8s中的场景，可以直接使用etcd来存放(100G以下的数据, 大厂多机房大集群会魔改etcd)；

2. 如果业务服务部署在Pod容器中, 怎么更新配置呢？

   1. agent通过DaemonSet容器化部署,单独升级, agent与业务Pod之间通过共享内存进行通信更新配置；

   2. 以SideCar Container方式将agent与业务Container部署在同一Pod中，利用Pod的共享IPC特性及Memory Medium EmptyDir Volume方式共享内存进行通信更新配置，随业务容器化部署上线；
   3. agent通过[OpenKruise](https://github.com/openkruise/kruise) SidecarSet部署在SideCar容器中；



### references

1. [QConf](https://github.com/Qihoo360/QConf/wiki)
2. [apollo](https://www.apolloconfig.com/#/zh/design/apollo-design)
3. [凤凰架构-service-discovery](http://icyfenix.cn/distribution/connect/service-discovery.html)
4. [etcd-watch](https://time.geekbang.org/column/article/341060)
5. [redis-pub/sub](https://redisbook.readthedocs.io/en/latest/feature/pubsub.html)
6. [Service Mesh 在超大规模场景下的落地挑战](https://mp.weixin.qq.com/s?__biz=MzU4NzU0MDIzOQ==&mid=2247490475&idx=2&sn=83e79449b409c363239de1b37b96f8c8)

