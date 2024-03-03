---
author: "weedge"
title: "用户行为分析方案设计"
date: 2022-11-02T10:26:23+08:00
tags: [
	"aws","方案","全栈"
]
categories: [
	"技术",
]
---

## 背景

用户在手机和pc端使用客户业务产品，比如浏览网页，购买商品，查看文档，观看视频，直播，IOT场景；会产生大量的用户行为数据，主要包括：

1. 非结构化数据：日志(前端事件埋点日志，服务端处理事件日志)，还有些非结构化的图片，音视频数据等等，主要存放在文件存储系统中；
2. 结构化和半结构化数据： 用户操作产品写入的结构化数据存放于数据库表中，将文档型半结构化的数据放入文档数据库中；

需要分析用户的行为数据，进行决策；分为实时流式处理和离线批处理：

1. 实时流处理，主要用于实时展现客户端看板，后台BI实时分析，实时风控/推荐，异常报警等场景；
2. 离线批处理，分析用户历史数据，进行推荐算法等机器学习算法模型训练使用，数据仓库中根据不同维度对数据过滤聚合，进行上卷下钻分析，比如计算DAU,WAU,MAU，转化率(购买率，注册率)分析等，通常对数据建设投入多的话， 会把用户产生的结构化非结构化的数据都存下，放在一个大的池子里待使用时进行分析，即所谓的数据湖，围湖而建挖掘数据价值；而数仓相对精细化的分析，前置建模建表分析；

对此进行方案分析，本文将介绍一种实时离线处理分析用户行为数据方案，即能帮助企业低成本地使用海量数据，又能更快速地响应业务需求，同时借助亚马逊云科技的托管服务，能够快速实施和轻松运维。

<!--more-->

### 操作概括

1. 权限设置：根据公司业务组织，分配对应服务资源权限，比如系统管理员OP, 开发人员DEV, 还有业务管理员OP；组织架构权限建设；
2. 服务基础架构：首先需要搭好基础服务框架，结合云厂商服务，进行可水平垂直自动扩展，高容错性，低成本，可观测监控，易于维护，持续集成发布的稳定性架构建设；
3. 业务迭代数据建设开发：架子搭好之后，需要对特定的业务场景进行数据建模，AI模型训练，挖掘出数据的价值，进行决策；服务代码质量，框架建设；

## 解决方案

使用aws 现有产品服务组件进行搭建，主要分为三个阶段，数据采集，数据处理存储，数据分析，整体架构如下：

![](https://github.com/weedge/user-behavior-analytics-cdk/blob/master/docs/aws-user-behavior-analytics.drawio.png?raw=true)

### 数据源采集

1. 访问日志和请求事件：用户在手机端通访CloudFront 内容分发服务, 会生成用户行为[访问日志](https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/AccessLogs.html)，这些[实时日志](https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/real-time-logs.html)中会有产品中定义的行为分析埋点记录事件，存放在S3中，通过Lambda无服务函数写入kinesis data streams中；如果需要实时处理，需要开启[CloudFront实时日志功能](https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/real-time-logs.html)，将数据写入Kinesis Data Streams中，会有几秒的处理延时；还有一种方式是服务端在线实时通过使用aws SDK方式直接写入记录事件数据，可通过无服务部署的lambda函数写入或者API gateway配置写入(参考:[在 API Gateway 中创建 REST API 作为 Amazon Kinesis 代理](https://docs.aws.amazon.com/zh_cn/apigateway/latest/developerguide/integrating-api-with-aws-services-kinesis.html))，常用于实时异常报警和统计；
2. 数据库数据： 存放在数据库中的数据，需要将数据同步存放在数仓和数据湖中，进行离线分析; 数据库的数据可以通过[aws DMS](https://docs.aws.amazon.com/zh_cn/dms/latest/userguide/CHAP_Introduction.html)服务来支持存量增量数据同步至Kinesis Data Streams中，具体方案可以参考**[使用 AWS DMS 将更改数据流式传输到 Amazon Kinesis Data Streams](https://aws.amazon.com/cn/blogs/big-data/stream-change-data-to-amazon-kinesis-data-streams-with-aws-dms/)**；也可以使用[flink CDC connector](https://ververica.github.io/flink-cdc-connectors/master/)  [on aws EMR](https://docs.aws.amazon.com/zh_cn/emr/latest/ReleaseGuide/emr-flink.html)支持同步(需要理解反压机制，以便触发时看是否增大下游消费能力提高吞吐), 可参考[多库多表场景下使用Amazon EMR CDC实时入湖最佳实践](https://aws.amazon.com/cn/blogs/china/best-practice-of-using-amazon-emr-cdc-to-enter-the-lake-in-real-time-in-a-multi-database-multi-table-scenario/)，注意数据库表中的数据字段需要规范，需要一个更新时间字段方便数据顺序同步，比如mysql `update_time timestamp DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`；

您可以参阅 AWS 白皮书[AWS Cloud Data Ingestion Patterns and Practices](https://d1.awsstatic.com/whitepapers/aws-cloud-data-ingestion-patterns-practices.pdf)，了解有关数据采集模式的更多详细信息；

将数据采集写入消息队列中主要是方便多个消费方来处理流，以及服务之间的整体解耦，可作为数据流缓存层，数据流量突增时，增加分片数，提高吞吐，当然整体吞吐量也取决于下游数据的消费能力，引入消息队列，以pull方式进行消费(主动消费)，不至于将下游服务打挂，而且方便下游异常消费重启时，继续在未消费点开始消费；这里提供两种方案，一种将数据写入Kinesis Data Streams中，一种将数据写入[MSK](https://docs.aws.amazon.com/zh_cn/msk)(aws 托管的kafka集群服务，如果直接自己搭建维护，成本比较高，对接其他aws服务相对复杂些) 中；

1. 数据写入Kinesis Data Streams中主要是方便多个消费方来处理流，以及服务之间的整体解耦，数据流缓存层，更重要的是方便使用aws Kinesis方案，减少运维成本，对接也丰富，主要是方便对接内部Kinesis相关服务组件;
2. 数据写入MSK中，使用的开源解决方案对接；下游对接kinesis Data Analytics 服务需要通过flink 计算引擎加载对应kafka connector包，从kafka中获取数据源进行分析；如果下游对接其他服务，比如：Kinesis Data Firehose，将数据传输转化写入数据湖S3中存储，写入Redshift数仓中分析；使用SNS 进行报警等； 需要引入无服务框架lambda函数，通过使用[kafka client SDK库](https://docs.confluent.io/platform/current/clients/index.html)来进行生产/消费处理；

具体使用结合公司实际场景而定，如果想想通过kafka对接更多的开源大数据服务框架， 可以选择MSK方案，额外需要开发维护lambda函数服务；使用Kinesis Data streams 很方便接入aws相关服务，减少额外的开发维护成本，不过如果对接其他开源大数据服务框架，同样需要引入lambda函数服务；本文采用将数据写入Kinesis Data streams 服务，提供给下游对接。

### 数据处理存储

数据处理分析，分为实时处理的[流数据](https://aws.amazon.com/cn/streaming-data/)和离线处理批数据处理存放；

实时处理使用AWS Kinesisi Data Analytics(KDA)进行分析，支持三种分析方式：

1. 使用老的方式[SQL 应用程序](https://docs.aws.amazon.com/zh_cn/kinesisanalytics/latest/dev/what-is.html)进行分析处理，不能使用编程语言python/Scala来调用api来进行精细化操作(需要定义UDF包提供使用),以及即席查询；
2. 使用基于 [Apache Flink](https://flink.apache.org/) 的开源库在 Kinesis Data Analytics 中构建 Java ,Scala 和Python 应用程序,Apache Flink 是处理数据流的常用框架和引擎，Kinesis Data Analytics现使用flink支持最高版本是[1.13](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/learn-flink/overview/)；在开发应用时，需要打印日志，以便使用CloudWatch 日志监控应用程序的性能和错误状况；如果应用程序出现bug或者服务异常中断可以使用检查点([CheckPoints](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/ops/state/checkpoints/))和保存点([SavePoints](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/ops/state/savepoints/) 生成[快照](https://docs.aws.amazon.com/zh_cn/kinesisanalytics/latest/java/how-fault-snapshot.html))在 Kinesis Data Analytics 应用程序中实现容错功能; 同时Kinesis Data Analytics [**可弹性扩缩**](https://docs.aws.amazon.com/zh_cn/kinesisanalytics/latest/java/how-scaling.html)应用程序的并行度，以适应大多数场景下的源数据吞吐量和操作复杂性，Kinesis Data Analytics 监控应用程序的资源 (CPU) 使用情况，并相应地弹性地向上或向下扩展应用程序的并行度; 使用算子([operators](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/datastream/operators/overview/#operators))进行数据流拓扑计算；同时通过[connector](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/connectors/datastream/overview/)可直接引入jar包接入数据源或者sink到下游服务，可以在mvn库中找到，比如[kinesis stream connector](https://mvnrepository.com/artifact/org.apache.flink/flink-sql-connector-kinesis);
3. 在基于 [Apache Flink](https://flink.apache.org/) 的开源库构建应用程序的基础上，结合[Glue](https://docs.aws.amazon.com/zh_cn/glue/latest/dg/how-it-works.html)定义数据库表存放数据catalog元数据，通过[zeppelin](https://zeppelin.apache.org/)增加了可视化即席查询, 直接可以在notebook上编写flink 流/批**[SQL](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/sql/overview/)**(notice: **flinkSQL和KDA SQL有所不同，特别是在window上有些区别**), 以及编写调用flink API的Scala,Python程序, 具体见[zeppelin flink解释器](https://zeppelin.apache.org/docs/0.9.0/interpreter/flink.html); 这种方式是相对于第二种方式，在方便运维管理的基础上更加容易上手，直接在notebook上就可以进行即席查询, 查询会话还可以保留或存放本地，还可以作为测试开发调试的平台，构建好处理程序，可直接转化成[持久化的应用程序部署](https://docs.aws.amazon.com/zh_cn/kinesisanalytics/latest/java/how-notebook-durable.html)；当然这部分费用相比前面的分析方式相对多些，启动的studio notebook 费用，Kinesis 处理单元(KPU)将按小时收费；可参考[实例教程](https://docs.aws.amazon.com/kinesisanalytics/latest/java/how-zeppelin-examples.html)，[使用 Kinesis Data Analytics Studio 和 Python 以交互方式查询数据流](https://aws.amazon.com/cn/blogs/big-data/query-your-data-streams-interactively-using-kinesis-data-analytics-studio-and-python/)

整体来说： [Amazon Kinesis Data Analytics](https://aws.amazon.com/kinesis/data-analytics/) 可以轻松地实时分析流数据，并使用标准 SQL、Python 和 Scala 构建由 Apache Flink 提供支持的流处理应用程序。特别是notebook功能只需在AWS 管理控制台中单击几下，写下分析SQL，就可以启动无服务器笔记本来查询数据流并在几秒钟内获得结果。Kinesis Data Analytics 降低了构建和管理 Apache Flink 应用程序的复杂性。

离线处理的数据主要是通过[AWS Kinesis Data Firehose]() 传输流写入下游服务存储，将数据写入湖仓系统中, 用于后续的数据分析，以及前期规划好的数据分析；选用firehose的原因是有原始备份机制存放于S3中，即使数据传输错误时，数据传输中不会丢失数据，同时内置lambd函数在传输之前将数据转化处理，同时也支持用Glue定义表来转换大数据相关的记录格式;

原始分析数据直接存放于[S3](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/Welcome.html)中，11个9的保障可以非常可靠保证数据不丢失，通过[Glacier](https://aws.amazon.com/cn/s3/storage-classes/glacier/)持久冷热存放，降低成本，方便后面追查数据，以及通过Athena查询引擎结合Glue来定义表从S3中挖掘出更有价值的数据；数据存放下来之后，结合大数据相关平台，[EMR](https://docs.aws.amazon.com/zh_cn/emr/latest/ManagementGuide/emr-what-is-emr.html)进行海量数据处理(PB级别)；同时结合[Glue](https://docs.aws.amazon.com/zh_cn/glue/latest/dg/what-is-glue.html) 进行ETL 数据处理，集成编排成DAG工作流，可视化管理这些ETL任务作业，也可以迁移调度平台比如Azkaban的工作流迁移至Glue ETL工作流,参考[Amazon Glue ETL作业调度工具选型初探](https://aws.amazon.com/cn/blogs/china/preliminary-study-on-selection-of-aws-glue-scheduling-tool/)；

提前业务场景数据分析建模，使用[Redshift](https://docs.aws.amazon.com/zh_cn/redshift/latest/gsg/concepts-diagrams.html)来存放不同维度(DIM)的表，分层(ODS->**DWD->DWM->DWS**)建设数据仓库； 选用redshift性价比比较高，开箱即用，存放结构化，半结构化数据，数据列式存储，分片存放，计算和存储分离，很方便无服务化，在数秒内轻松运行和扩展分析，而无需调配和管理数据仓库; 和数据湖打通，可以使用 [Redshift Spectrum](https://docs.aws.amazon.com/zh_cn/redshift/latest/gsg/data-lake.html) 在 Amazon S3 文件中查询数据，而不必将数据加载到 Amazon Redshift 表中，提高关联查询；还可以对接机器学习[ML](https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/machine_learning.html)，通过CREATE MODEL DDL语句下推到Amazon [SageMaker](https://docs.aws.amazon.com/zh_cn/sagemaker/latest/dg/whatis.html)，从S3中加载数据进行训练；

### 数据分析

主要是通过分析引擎从数据存储获取数据，根据维度展现看板，进行实时数据查看，离线分析，以及可视化即席分析：

1. 实时结果数据查看：通过[DynamoDB](https://docs.aws.amazon.com/zh_cn/amazondynamodb/latest/developerguide/Introduction.html)获取实时结果数据，主要是Key/Value数据，同时查询速度很快，利用[DAX](https://docs.aws.amazon.com/zh_cn/amazondynamodb/latest/developerguide/DAX.html)实现内存中加速；通过[lambda](https://docs.aws.amazon.com/zh_cn/lambda/latest/dg/welcome.html)对接[API Gateway](https://docs.aws.amazon.com/zh_cn/apigateway/latest/developerguide/welcome.html)提供数据接口，方便业务场景实时展现，比如监控查看异常数据，访问计数等结果展现；这些结果数据可以直接通过[SNS](https://docs.aws.amazon.com/zh_cn/sns/latest/dg/welcome.html)发送邮件或者短信进行通知；
2. 通过[OpenSearch](https://docs.aws.amazon.com/zh_cn/opensearch-service/latest/developerguide/what-is.html)提供实时搜索服务，比如日志搜索实时定位追查问题，通过KDA实时分析写入；在线检索商品，这些数据主要来源于数据库，异构成OpenSearch索引数据进行检索，可通过[flink CDC](https://ververica.github.io/flink-cdc-connectors/master/) on [EMR](https://docs.aws.amazon.com/zh_cn/emr/latest/ReleaseGuide/emr-flink.html)来同步数据到OpenSearch中；
3. S3中的数据通过[Athena](https://docs.aws.amazon.com/zh_cn/athena/latest/ug/what-is.html)在Glue/Hive上建表元数据，使用[SQL](https://docs.aws.amazon.com/zh_cn/athena/latest/ug/ddl-sql-reference.html)查询，几分钟内可以查询到结果;
4. 存放在[Redshift](https://docs.aws.amazon.com/zh_cn/redshift/latest/gsg/concepts-diagrams.html)数据仓库中的数据，通过Amazon [QuickSight](https://docs.aws.amazon.com/zh_cn/quicksight/latest/user/welcome.html)接入进行BI分析，响应时间在秒级别，只需要在界面上选择图表组合成一个仪表盘展现即可，方便快速决策；[支持AWS区域接入IP范围](https://docs.aws.amazon.com/zh_cn/quicksight/latest/user/regions.html);

### 组织用户角色权限管理

以上这些谈到的服务需要进行安全访问，通过[IAM](https://docs.aws.amazon.com/zh_cn/iam/)定义策略角色分配权限进行[服务授权](https://docs.aws.amazon.com/zh_cn/service-authorization/latest/reference/reference.html)，来进行安全访问，有两种情况：

1. 服务资源之间的访问，需要分配读写权限，需要把这些策略赋予莫个角色，然后资源通过这个赋予资源权限的角色来访问对应资源；

2. 还有就是操作者访问服务资源，需要分配不同资源的读写权限，可以根据公司组织架构来管理每个员工的权限使用范围，非常方便；

   角色权限设置规则如下：[IAM身份](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id.html)设置用户组 dev, op, biz user，后续 细分在按组织部门进行建组；

   1. op: 理论上构建完一组资源可以分配对应权限策略角色Role，给予服务资源的运维管理操作；
   2. dev: 只有使用开发资源权限，比如lambda编辑发布权限，数据库读写权限，而非管理删除权限策略Role；
   3. bizUser: 业务操作者，大部分只有读权限策略Role，没有写操作权限；
   4. admin: 管理员，Administrator权限，可以访问任何资源

还有是创建的应用是给外部服务用户使用，比如Web,移动端用户，通过Amazon [Cognito](https://docs.aws.amazon.com/zh_cn/cognito/)来注册登录管理用户，也可以通过第三方登录进行身份验证；两个主要组件是用户池和身份池：用户池是为应用程序提供注册和登录选项的用户目录，身份池授予用户访问其他 AWS 服务的权限。请参考[Amazon Cognito常见场景](https://docs.aws.amazon.com/zh_cn/cognito/latest/developerguide/cognito-scenarios.html)。

## 场景

### CASE 实时异常事件报警展现

打点事件数据：（实体数据，通过同步实体表数据流进行关联jion操作）

| field     | type   | desc                                       |
| --------- | ------ | ------------------------------------------ |
| eventId   | string | 用户行为事件id                             |
| action    | string | 统一定义的事件动作，比如 浏览文档：viewDoc |
| userId    | string | 触发事件的用户id                           |
| createdAt | string | 触发事件时间                               |
| objectId  | string | 操作对象id, 比如文档id                     |
| bizId     | string | 所属业务id                                 |
| errorMsg  | string | 错误信息                                   |

实时过滤出异常事件KDA SQL

```sql
-- ** Continuous Filter ** 
    -- Performs a continuous filter based on a WHERE condition.
    --          .----------.   .----------.   .----------.              
    --          |  SOURCE  |   |  INSERT  |   |  DESTIN. |              
    -- Source-->|  STREAM  |-->| & SELECT |-->|  STREAM  |-->Destination
    --          |          |   |  (PUMP)  |   |          |              
    --          '----------'   '----------'   '----------'               
    -- STREAM (in-application): a continuously updated entity that you can SELECT from and INSERT into like a TABLE
    -- PUMP: an entity used to continuously 'SELECT ... FROM' a source STREAM, and INSERT SQL results into an output STREAM
    -- Create output stream, which can be used to send to a destination
-- reference: 
-- https://docs.aws.amazon.com/zh_cn/kinesisanalytics/latest/sqlref/analytics-sql-reference.html
-- https://docs.aws.amazon.com/zh_cn/kinesisanalytics/latest/dev/streaming-sql-concepts.html
-- https://docs.aws.amazon.com/zh_cn/kinesisanalytics/latest/sqlref/kinesis-analytics-sqlref.pdf

-- abnormality event stream
CREATE OR REPLACE STREAM "DESTINATION_SQL_STREAM" 
(
    "eventId"       varchar(64),
    "action"        varchar(256),
    "userId"        varchar(64),
    "objectId"      varchar(64),
    "bizId"         varchar(64),
    "errorMsg"      varchar(1024),
    "createdAt"      varchar(32)
);

-- Filter errorMsg like panic/error pump
CREATE OR REPLACE PUMP "ERROR_PANIC_STREAM_PUMP" AS
    INSERT INTO "DESTINATION_SQL_STREAM"
    SELECT STREAM "eventId", "action", "userId", "objectId", "bizId", "errorMsg","createdAt"
    FROM "SOURCE_SQL_STREAM_001"
    WHERE "errorMsg" LIKE '%[PANIC]%'
        or "errorMsg" LIKE '%[panic]%' 
        or "errorMsg" LIKE '%[ERROR]%' 
        or "errorMsg" LIKE '%[error]%';
```

每1分钟warn数目超过10次的SQL(滚动窗口)

```sql
CREATE OR REPLACE STREAM "DESTINATION_SQL_STREAM" 
(
    "eventId"       varchar(64),
    "action"        varchar(256),
    "userId"        varchar(64),
    "objectId"      varchar(64),
    "bizId"         varchar(64),
    "errorMsg"      varchar(1024),
    "createAt"      varchar(32),
    "INGREST_ROW_TIME"      varchar(32),
    "APPROXIMATE_ARRIVAL_TIME"      varchar(32)
);

-- Filter errorMsg like warning pump
-- Aggregation with time window(u can use stagger windows,tumbling windows, sliding windows)
-- use tumbling windows for this case
CREATE OR REPLACE PUMP "STREAM_PUMP" AS
    INSERT INTO "DESTINATION_SQL_STREAM"
    SELECT STREAM "eventId", "userId", "objectId", "bizId", "createAt"
        "errorMsg","action",
        STEP("SOURCE_SQL_STREAM_001".ROWTIME BY INTERVAL '60' SECOND) AS "INGREST_ROW_TIME",
        STEP("SOURCE_SQL_STREAM_001".APPROXIMATE_ARRIVAL_TIME BY INTERVAL '60' SECOND) AS "APPROXIMATE_ARRIVAL_TIME",
        -- STEP("SOURCE_SQL_STREAM_001".EVENT_TIME BY INTERVAL '60' SECOND) AS "EVENT_TIME",
        COUNT(*) AS "action_warn_count"
    FROM "SOURCE_SQL_STREAM_001"
    WHERE "errorMsg" LIKE "% WARNNING %" or "errorMsg" LIKE "% warnning %"
    GROUP BY "action",
        STEP("SOURCE_SQL_STREAM_001".ROWTIME BY INTERVAL '60' SECOND),
        STEP("SOURCE_SQL_STREAM_001".APPROXIMATE_ARRIVAL_TIME BY INTERVAL '60' SECOND)
        -- STEP("SOURCE_SQL_STREAM_001".EVENT_TIME BY INTERVAL '60' SECOND) 
    Having "action_warn_count" >= 10;
```

## 服务构建

基于以上服务搭建，需要用户去aws云平台上点击，配置使用， 特别是构建表，以及数据传输，属性和安全访问权限角色的管理，随着业务复杂度变高，基础服务也随之增多，可配置化的东西越来越多，会带来灾难性的后果，最终变得不可控，人力管理运维成本增加；aws平台提供了AWS CloudFormation 允许您通过将基础设施视为代码来建模、预置和管理 AWS 和第三方资源。即IaC，也是Devops经常所做的事情，可以使用相关IaC工具来自动化构建一个系统，常用的场景是CI/CD支持快速变化的业务需求开发；AWS一直提供基础服务的配置API，有相关的SDK可供使用，也有`aws` 工具来操作这些基础服务资源； 后面提供了一种开源软件开发框架AWS Cloud Development Kit (AWS [CDK](https://docs.aws.amazon.com/zh_cn/cdk/v2/guide/home.html))，可使用熟悉的编程语言来定义云应用程序资源；将云上硬件资源可编程化，可以很方便的实现自动化运维管理，充分利用云上的资源来组装construct成一个模块栈stack, 各个模块栈最终合成一个落地解决方案，方便开箱即用(安装软件一样)，降低云构建的复杂性；cdk construct分3个层次的封装，L1是最低级别初始封装，直接对应CloudFormation配置模版文件映射，称之为CFN 资源，必须配置每一项属性，需要深入了解资源模型的详细信息；L2是是对了L1的组合封装，使用资源属性默认值，比如new VPC；L3则是一种模式(pattern)，通过多种资源组合成一个常见资源架构，比如 new Fargate无服务化容器集群；具体参考见：[constructs](https://docs.aws.amazon.com/zh_cn/cdk/v2/guide/constructs.html)

通过[cdk workshop](https://cdkworkshop.com) 大概花半天时间学习这里的demo就可以试着搭建相关的服务， 也有更多的[awesome workshop]( https://awesome-aws-workshops.com/)可供参考和学习的，并且 [aws blog](https://aws.amazon.com/blogs)  [builders' lib](https://aws.amazon.com/cn/builders-library)也会有很多相关的解决方案提供学习；当然需要很好的架构aws服务，需要多落地实践，深入服务细节(查看帮助文档[docs](https://docs.aws.amazon.com/))，结合需求，才能构建一个相对完美的解决方案；当然前期生产落地需要使用CDK来进行架构推理。

这里结合上述需求，使用CDK来搭建一些stack, 方便数据流分析管道的组装，后续也方便与其他contstucts进行组装；主要是分为以下stack:

1. **CDK-Workshop-Lambda-KDS-stack**：

   APIGateway->lambda(put KDS record & hit counter in dynamodb)->lambda(hello)  dynamotableviewer

2. DMS-KDS-stack

3. **KDS-KDA-sql-Lambda-Dynamodb-stack**: KDS->KDA->lambda->dynamodb dynamotableviewer

4. **KDS-KDF-S3-stack**: KDS->KDF->S3

5. KDS-KDA-flink-OpenSearch-stack

6. KDS-KDA-flink-KDS-stack: eg: for near-realtime-warehouse, make a pipeline (ODS->**DWD->DWM->DWS**) sink to Redshift; like this [tencent news Pipeline pattern](https://cloud.tencent.com/developer/article/1919594)

7. KDS-KDF-Redshift-stack: 

8. Redshift-QuickSight-stack

9. S3-Glue-Athena-stack 

这里主要部署CDK-Workshop-Lambda-KDS-stack, KDS-KDA-sql-Lambda-Dynamodb-stack 和KDS-KDF-S3-stack 搭建SQL流式处理用户行为数据,具体见[代码](https://github.com/weedge/user-behavior-analytics-cdk)；其他stack可以后续进行扩展进行构建，推理整体基础设施架构。

### 构建

首先需要安装CDK, 具体查看[入门教程](https://aws.amazon.com/cn/getting-started/guides/setup-cdk/)，执行如下命令：

```shell
# download code to start
git clone https://github.com/weedge/user-behavior-analytics-cdk.git && cd user-behavior-analytics-cdk
# tips: cdk load cdk.json + cdk.context.json to run by js on node
# list stacks
cdk ls
# deploy KDS-KDA-sql-Lambda-DynamoDB-stack with KDS-KDF-S3-stack(need created kinesis data stream)
cdk deploy KDS-KDA-sql-Lambda-DynamoDB-stack
# deploy CDK-Workshop-Lambda-KDS-stack for hit event stream; lambda func put record to KDS, dependcy KDS
cdk deploy CDK-Workshop-Lambda-KDS-stack
```

部署完之后，会输出kinesis数据流的名称以及用于查看数据结果地址；

开始写入测试数据，需要使用python3, 使用pip3 安装依赖包

```shell
# init python virtural env
python3 -m venv .venv && source .venv/bin/activate 
# install boto3  faker
pip3 install boto3 faker
# run test script, wait KDA run, put record to KDS
python3 src/scripts/producer-kds-test.py
```

运行测试脚本，输入region地域名称,比如us-east-1, 等待启动后，输入数据流名称，开始发送数据；

每1秒发一次数据写入KDS中，发了10次含有错误事件，发了10次随机事件，总共20条；

也可以使用aws提供KDS数据生成器：https://github.com/awslabs/amazon-kinesis-data-generator

从刚才部署输出结果地址查看含有错误的数据已经有10条展现出来(数据获取式前端每隔几秒轮训获取api数据，实时展现可通过[API Gateway WebSocket](https://docs.aws.amazon.com/zh_cn/apigateway/latest/developerguide/apigateway-websocket-api.html) 来支持)；历史数据可以在KDF配置的S3目标中点击查看，原始数据可以下载gz包进行解压查看;

通过使用提供给前端访问的事件api(api地址在部署CDK-Workshop-Lambda-KDS-stack后输出的访问地址)来写入异常`[error]` 数据

```shell
curl -XPOST -iv https://{APIGateway}.execute-api.{region}.amazonaws.com/prod/event -d '{"eventId":"1-1-1","bizId":"123123","objectId":"123","action":"test","errorMsg":"[error]","userId":"1231321"}'
```

通过刚才部署KDS-KDA-sql-Lambda-DynamoDB-stack的结果地址可以 查看异常数据已经实时写入。

最后将部署资源清除，依赖删除(和卸载软件一样)。

```shell
# destroy resources with dependent resources
cdk destroy KDS-KDF-S3-stack
cdk destroy KDS-KDA-sql-Lambda-DynamoDB-stack
cdk destroy CDK-Workshop-Lambda-KDS-stack
# or cdk destroy --all
cdk destroy --all
```

### 代码结构

```
├── cmd                              -- golang cmd bin dir
├── docs                             -- help doc
├── infra                            -- infrastructures stack
│   └── lib                          -- cdk constuct in stack
├── src                              -- source code to run
│   ├── kinesis-analytics-pyflink    -- KDA python scripts use flink python api 
│   ├── kinesis-analytics-sql        -- KDA sql
│   ├── lambda                       -- js,python,golang lambda func 
│   ├── redshift-sql                 -- redshift sql 
│   └── scripts                      -- local run test code by use aws sdk
├── test                             -- test cdk logic
```

## 参考

sam cli [local debug lambda](https://docs.aws.amazon.com/zh_cn/serverless-application-model/latest/developerguide/serverless-sam-cli-using-invoke.html), **[The Complete AWS SAM workshop](https://catalog.workshops.aws/complete-aws-sam/en-US)**,  **[AWS re:Invent 2022 - Best practices for advanced serverless developers](https://www.youtube.com/watch?v=PiQ_eZFO2GU&list=PL2yQDdvlhXf_lYR5Ntvr9V5iVYv5rcbNc&index=6)**

streaming cdk: https://github.com/aws-samples/streaming-solution-aws-cdk

kds and msk cdk: https://github.com/aws-solutions/streaming-data-solution-for-amazon-kinesis-and-amazon-msk

Redshift cdk : https://github.com/miztiik/redshift-demo

Glue cdk: https://github.com/aws-samples/glue-workflow-aws-cdk

opensearch cdk: https://www.luminis.eu/blog/cloud-en/deploying-a-secure-aws-elasticsearch-cluster-using-cdk/

mysql cdk: https://aws.amazon.com/cn/blogs/infrastructure-and-automation/use-aws-cdk-to-initialize-amazon-rds-instances/

mysql dms cdk: https://aws.amazon.com/cn/blogs/database/accelerate-data-migration-using-aws-dms-and-aws-cdk/

aurora mysql dms kds opensearch cdk: https://github.com/aws-samples/aws-dms-cdc-data-pipeline.git

kda-flink-py: https://aws.amazon.com/cn/blogs/china/python-stream-data-processing-and-analysis-using-pyflink-in-amazon-kinesis-data-analytics/

kda-zeppelin-flink-py: https://aws.amazon.com/cn/blogs/big-data/query-your-data-streams-interactively-using-kinesis-data-analytics-studio-and-python/

**[Implementing Microservices on AWS](https://docs.aws.amazon.com/pdfs/whitepapers/latest/microservices-on-aws/microservices-on-aws.pdf)**

**[AWS Cloud Data Ingestion Patterns and Practices](https://d1.awsstatic.com/whitepapers/aws-cloud-data-ingestion-patterns-practices.pdf)**

[Flink-on-KDS Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/c342c6d1-2baf-4827-ba42-52ef9eb173f6/en-US/flink-on-kda)

[Real time streaming with kinesis Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/2300137e-f2ac-4eb9-a4ac-3d25026b235f/en-US)