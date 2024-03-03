---
author: "weedge"
title: "设计-直播间赠送礼物功能"
date: 2022-11-21T10:26:23+08:00
tags: [
	"直播",
]
categories: [
	"技术",
]

---

{{< neteasemusic id="1933940116" >}}

## 背景

直播线上互动，已成为当下生活的一部分，特别是受疫情影响，成为互联网的主要流量入口；研究了下各个平台直播间送礼物的功能，发现大同小异，在礼物分组，一些定制化的礼物有区分，整体交互流程大致相同，主要是直播间主播上播，用户通过礼物打赏给主播们(一个直播间可能有多名主播在互动)，礼物是通过虚拟币(**币) 换算，早期互联网用户线上互动礼物，玩家最多的应该是QQ币了，只不过以前的打赏赠送场景是在web2.0刚开始的时候，交互的大多是文字和图片，相关产品场景，虚拟空间(个人空间，博客，种菜等娱乐互动场景)；随着底层网络基建的发展，4G之后出现了大量的视频网站，用户可以录一些视频内容来互动；到后来音视频流媒体的发展，相关的在线直播间开始涌现，用户之间享受一波直播红利带来的互动，当然影响相对于前面的形式更加实时和直接；现在的5G和未来6G，以及物联网都会给直播形式带来新的互动场景,比如：虚拟会场，人机互动；其中早期培养起来的打赏送礼行为功能经常用于有主播的娱乐互动直播中，也是增值盈利的一部分；

tips: 除了送礼功能，根据不同直播场景，还有语音视频连麦，电商带货商品，没有主播，节目直播/转播，会议直播，自习室，基本的点赞，计数，聊天基础服务功能；还有些抽奖功能，答题功能(教育类直播居多)，投票，红包这些功能服务可在开播时设置，是否启用；有用户基础的流量平台可能还会以竞价排名的方式推荐一波；这些功能可以作为一个可管理的插件，通过组合的方式应用于直播中，方便管理，后续可以添加新功能满足某类型直播场景。

<!--more-->

## 需求设计

设计一个简单礼物打赏功能: A用户(观众)赠送礼物给B用户(主播)，可以给多个主播赠送礼物；

### 分析

礼物：直播平台通过虚拟币来统一等价交换的虚拟物品，而虚拟币是需要通过用户充值购买；发送礼物可以获得一些积分；

A用户发送礼物给B用户，A的虚拟币扣除，B的虚拟币相应增加；需要保证赠送和收到的数量一致；

需要考虑并发场景，对于热门主播高峰期的流量值100w观众用户在线互动，每秒送礼的请求数量也可能高于这个值，用户可能连击送出多次(连击行为客户端可以缓冲一次发送给后端)， 而且为了保证数据一致和吞吐，尽量减少锁的使用，或者说采用[OCC](https://en.wikipedia.org/wiki/Optimistic_concurrency_control)，乐观锁的方式来处理事务，交由业务服务程序来处理，尽量减少或减短数据库中心存储服务上的锁操作，原理就是：夯主后，就呆萌了，资源未释放，请求资源一增多，导致整个服务吞吐下降，一直持续下去随时都会down掉，而且锁是建立在索引数据之上的， 如果没有相关降级处理，弄不好整体服务就[galigeigei](https://www.youtube.com/watch?v=4m48GqaOz90)；

100W 观众A向同一个主播B 赠送礼物，对于主播B的虚拟币累计写操作比较大，同一时间可能100W+的w 操作 主播B 虚拟币这条记录，如果直接同步操作在数据库层面会出现行锁，会等待夯主整个赠送流程，所以需要把这些集中写，通过消息队列异步去更新虚拟币数目，进而提高观众送礼接口的吞吐量，

观众虽然操作自己的虚拟币数目行记录，但是直接操作数据库，即使对用户资产表进行分库分表操作，也会有大量的磁盘i/o，所以直播间的互动数据直接在缓存中操作，把观众的操作记录消息队列的方式异步落库；

缓存的方式操作，需要把观众的用户资产信息前置预热至缓存中，直播中直接操作缓存， 操作缓存需要保证并发操作事务的原子性，保证观众的虚拟币不能多扣或者少扣；

并发场景下，为了减少大量用户冲击底层数据库，减少磁盘io, 送礼物这些互动直接读写用户缓存数据，这些缓存数据的操作类型分为是/否更新频繁；

更新不频繁数据：用户，直播间，物料详情等信息，这些缓存数据在进入直播间的时候直接从数据库以cacheAside pattern获取填充，填充的时候采用singleflight方式；物料信息还可以服务本地缓存一份；变更数据时，远端缓存数据可以通过CDC订阅数据库操作日志(比如：binlog)来主动异步更新缓存数据，或者使用延时双删来被动更新，本地缓存可根据通过控制平面配置下发来触发从远端缓存更新数据；

更新频繁数据： 这个主要发生在用户直播互动，赠送礼物场景，多次并发操作，变更的实体数据 观众/主播虚拟资产扣除/增加，这些数据以writeThrough/Behind parttern方式直接更新缓存，队列异步落库；因为直播场景用户的数据都在缓存中，数据实时更新查看，不影响用户体验；直播监控后台从数据库里订阅近实时查看用户的虚拟资产，会有一个批量窗口的处理延迟；

缓存数据初始化，可以在用户刚打开app的时候初始化，也可以在进入/创建好直播间的时候初始化好用户直播间缓存数据；

以上可以将操作分2个关键步骤： 观众赠送礼物和异步更新观众和主播的资产信息；通过消息队列来解耦，提高送礼接口吞吐和请求响应延迟，以及以pull方式消费，缓解数据库实例的读写压力；

观众赠送礼物： 礼物是通过虚拟货币进行等价交换的，通过礼物id获取到对应消费的虚拟资产，对中心远端缓存分片中的观众虚拟资产进行事务扣除处理，事务提交之后，发送事务消息；这里采用cas方式处理，一种是watch key(string 读写io是O(1), 不用hash是因为读取全部资产信息io是O(n))+事务方式，一种是lua的形式直接写业务提交脚本给redis核心主线程去处理；这里可能会想到直接用 hash incr原子操作，但是这里不行，因为需要读出key对应的虚拟资产，用于判断虚拟资源是否充足，读出来在扣除写入，需要一起执行，保证事务原子性；除了核心流程，还有发送礼物成功后，需要推送消息到直播间，根据产品礼品策略判断是否展示特效; 以及增加用户活动积分，增加互动积极性；

异步更新观众和主播数据库落地资产信息：为了减少对数据库的行锁的并发压力，可采用CAS的方式来更新数据库的数据，前提是单个用户资产操作，如果想单个事务单个事务处理，可以通过消息队列事务消息方式串起来(pipeline)；如果是多个用户资产操作在同一服务事物里操作的话，则不能使用CAS的方式处理了，只能以整体事务方式处理(默认RR级别)；

消息队列：涉及到金钱，为了提高吞吐，需要保证数据准确，数据最终一致([**BASE**](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html))，采用支持事务消息的分布式消息队列，比如：[rocketMQ**事务消息**](https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage/)，这里可能有个疑问如果刚开始发送事务消息就失败了，可能是网络抖动,或者服务负载高等原因，一般是启用failover权重[指数退避](https://en.wikipedia.org/wiki/Exponential_backoff)策略重试到不同机房的rocketMQ集群，可以查看[消息发送重试机制](https://rocketmq.apache.org/zh/docs/featureBehavior/05sendretrypolicy#%E6%B6%88%E6%81%AF%E5%8F%91%E9%80%81%E9%87%8D%E8%AF%95%E6%9C%BA%E5%88%B6) 和 [最佳实践](https://rocketmq.apache.org/zh/docs/bestPractice/15bestpractice), [timeouts-retries-and-backoff-with-jitter](https://aws.amazon.com/cn/builders-library/timeouts-retries-and-backoff-with-jitter/)；如果重试还是失败打印错误日志记录发送详情，通过实时数据流将异常行为写入db中，以便后续补发；

Tips: 如果使用阿里云rocketMQ, 需要注意支持版本提供的SDK，5.0 版本 [client SDK](https://github.com/apache/rocketmq-clients) ; 4.0相关版本有些开发语言不支持tcp方式，仅提供http的方式(会少了一些功能，比如批量发送普通消息)；选用新版的5.0版本的SDK开发(golang版client SDK可以用来生产消息，对应消息类型都已支持); 如果有自建运维能力，直接使用开源方案来搭建一套，比如用[rocketMQ on aws](https://www.amazonaws.cn/solutions/apache-rocketmq-on-aws/)，然后可以基于OTEL的metric标准采集到Promethues中，通过Grafana加上监控报警(监控系统也可以自建，或者用云服务比如aliyun ARMS，同样分布式db/cache也有相应监控解决方案,数据指标采集上报以pull方式居多,相对于业务服务常见以push方式)；并且使用5.0可以实现一层mq-proxy(本地代理和中心代理)，计算存储分离;

题外话: 现在分布式消息队列[kafka-streams](https://kafka.apache.org/33/documentation/streams/architecture), [rocketmq-streams](https://github.com/apache/rocketmq-streams) 支持数据流(stream)处理大数据实时场景，支持一些简单算子操作和SQL([ksql](https://github.com/confluentinc/ksql), [rsql](https://github.com/alibaba/rsqldb))；这个和flink对应功能是重合了，flink也在往table store上发力满足数据堆积的能力; 一波流～

### 调研

以pc端抓http包为例，手机端接口一样，传输格式可能不同，pb/json

#### 抖音接口

```shell
curl 'https://live.douyin.com/webcast/gift/send/?aid=6383&live_id=1&device_platform=web&language=zh-CN&enter_from=web_live&cookie_enabled=true&screen_width=2048&screen_height=1152&browser_language=zh-CN&browser_platform=MacIntel&browser_name=Chrome&browser_version=107.0.0.0&browser_online=true&engine_name=Blink&engine_version=107.0.0.0&os_name=Mac+OS&os_version=10.15.7&cpu_core_num=8&device_memory=8&platform=PC&downlink=10&effective_type=4g&round_trip_time=50&channel=channel_pc_web&app_name=douyin_web&webid=7167235205950047744&user_agent=Mozilla%2F5.0+(Macintosh%3B+Intel+Mac+OS+X+10_15_7)+AppleWebKit%2F537.36+(KHTML,+like+Gecko)+Chrome%2F107.0.0.0+Safari%2F537.36&fp=verify_lam4f5i1_plZJSYeB_iUgU_4z2o_9JHF_d7Z0Z60sLg33&did=0&referer=https:%2F%2Flive.douyin.com%2F444452144000%3Fcover_type%3D0%26enter_from_merge%3Dweb_live%26enter_method%3Dweb_card%26game_name%3D%26is_recommend%3D1%26live_type%3Dgame%26more_detail%3D%26request_id%3D2022111814283801020916816201004237%26room_id%3D7167220081737468683%26stream_type%3Dvertical%26title_type%3D1%26web_live_page%3Dhot_live%26web_live_tab%3Dall&target=&device_id=7167235205950047744&msToken=aOEoMNHAEI98H45Z0n-zUTffiNgv7HNkGU0lwFptk-JBg00tEs0I74G4sYXgG670cAdhSmXNcKlRU3-QaxW7Pflt-p8YAmyU5eC3EGJQfp7Mk7JpmP_P&X-Bogus=DFSzswVL0qCmAcoQS8MuBN7TlqS8&_signature=_02B4Z6wo00001Heu8JQAAIDD43irmgubdKx3rvQAAH6ktTtRdH7gtQJWp3SjldUkeBB51lpkXUQ700UnFEXODUicQ0r1ccxSxq4OwWIjvxuNe8Acl-gJzChe99W43SojHkPp9aa82Qzi9VoTd2' \
  -H 'authority: live.douyin.com' \
  -H 'accept: application/json, text/plain, */*' \
  -H 'accept-language: zh-CN,zh;q=0.9,en;q=0.8,zh-TW;q=0.7' \
  -H 'content-type: application/x-www-form-urlencoded' \
  -H 'cookie: xgplayer_user_id=502915240928; csrf_session_id=f5161482adac52884f25a91fa758d7b0; ttcid=5cd5ff73751f4ab39911c85e57f9ca7479; passport_csrf_token=46e48d1e7b4df245f8caab8a34b64935; passport_csrf_token_default=46e48d1e7b4df245f8caab8a34b64935; home_can_add_dy_2_desktop=%220%22; n_mh=Qo43cptpX41bfwDmWyVyRUGMZnsucxhgcWJqKjjWkvI; sso_uid_tt=a84f8e3ae1f0c8301f5a882b05abe3f0; sso_uid_tt_ss=a84f8e3ae1f0c8301f5a882b05abe3f0; toutiao_sso_user=a28a6e92e7dc7d690b4f6b9a59077692; toutiao_sso_user_ss=a28a6e92e7dc7d690b4f6b9a59077692; passport_assist_user=Cjyr0IY04IJK-7Hxgl87vzPKZIfpFS29oAG2arITfMftUANM_d6SDxwAksiJEuVqZgZdE_BYyvcjLLGAQaYaSAo8zVwGiPnNM3S7eRNlXwMkuS7yNVm17pbNfKqXsXj3Rgk9Nm08DWh_PY0JRbH8FQeLxabRAoqM4bPJTzQ8EM_DoQ0Yia_WVCIBAxToQPk%3D; sid_ucp_sso_v1=1.0.0-KGIxM2FlNmE2MDdiNWE5MDA1YmIwZjg5OWVkNTUxMTBhMzdhYWE2OTIKHQjhrMGYrgIQt8vcmwYY7zEgDDDHuNHRBTgGQPQHGgJobCIgYTI4YTZlOTJlN2RjN2Q2OTBiNGY2YjlhNTkwNzc2OTI; ssid_ucp_sso_v1=1.0.0-KGIxM2FlNmE2MDdiNWE5MDA1YmIwZjg5OWVkNTUxMTBhMzdhYWE2OTIKHQjhrMGYrgIQt8vcmwYY7zEgDDDHuNHRBTgGQPQHGgJobCIgYTI4YTZlOTJlN2RjN2Q2OTBiNGY2YjlhNTkwNzc2OTI; passport_auth_status=ba24f5319c0f02c6232d484fd51c2187%2C; passport_auth_status_ss=ba24f5319c0f02c6232d484fd51c2187%2C; sid_guard=ea2e466c926c317f4f552f0d3982a458%7C1668752823%7C5184000%7CTue%2C+17-Jan-2023+06%3A27%3A03+GMT; uid_tt=23c596f546d448da13c0098152ef5d17; uid_tt_ss=23c596f546d448da13c0098152ef5d17; sid_tt=ea2e466c926c317f4f552f0d3982a458; sessionid=ea2e466c926c317f4f552f0d3982a458; sessionid_ss=ea2e466c926c317f4f552f0d3982a458; sid_ucp_v1=1.0.0-KDRiMzcyMDVkZmZhMWIxNjhjNDM4YjBiZDA4Y2E5ZTBmN2IxNTE1NjIKFwjhrMGYrgIQt8vcmwYY7zEgDDgGQPQHGgJsZiIgZWEyZTQ2NmM5MjZjMzE3ZjRmNTUyZjBkMzk4MmE0NTg; ssid_ucp_v1=1.0.0-KDRiMzcyMDVkZmZhMWIxNjhjNDM4YjBiZDA4Y2E5ZTBmN2IxNTE1NjIKFwjhrMGYrgIQt8vcmwYY7zEgDDgGQPQHGgJsZiIgZWEyZTQ2NmM5MjZjMzE3ZjRmNTUyZjBkMzk4MmE0NTg; FOLLOW_NUMBER_YELLOW_POINT_INFO=%22MS4wLjABAAAAt_v5oVMmcxuNnLLRzi6Ey1GKVQr_2XVFt2jPbkhZPI8%2F1668787200000%2F0%2F1668752826051%2F0%22; strategyABtestKey=%221668752909.101%22; __ac_nonce=06377261b0070042789c9; __ac_signature=_02B4Z6wo00f01JfP74gAAIDDAxm0hJMbmiiX7-sAAEaJTk9YXvYiEKlcYbhZnUTqyIVISJ6Z9uR3UEye00i7MfEP1dAywrmbnaQAIP3m1.7zOA8BmpqIWhyYJal-u6k6LMxVpsCtyakYsz3325; live_can_add_dy_2_desktop=%221%22; tt_scid=PpJHVbeNWnhY54EQ6vWraqXl5A8SZDtl3dn9JTaQQNLPo37ztPaJz.xoJHxyhNYScf8e; s_v_web_id=verify_lam4f5i1_plZJSYeB_iUgU_4z2o_9JHF_d7Z0Z60sLg33; ttwid=1%7C-XDacSDIgDIHmJqBxLj6Op91O91Ww4nvf96AveJpNeE%7C1668752951%7C0275ac24a410dd06b5f87dc4d84188f6eedaf20ed69cc9d0e979559df7df461e; download_guide=%223%2F20221118%22; odin_tt=3a02e4ad7a6c9042e1c42ece6d0d4a1aeeb37ed334f3450eb4adf57a6d0e09938523f8954816d90d557b27f5d3cbea85; msToken=1GTirdFSckP7H_txAPOKIMLWleKhlwccm-ts_3OviXegeQ2cr0B56jMAqfB3SqEGnxPEBjXRWsmg-sxVW3okb1s-acOAnBkIVDA_47g5aZFOqMKEzI-N; msToken=aOEoMNHAEI98H45Z0n-zUTffiNgv7HNkGU0lwFptk-JBg00tEs0I74G4sYXgG670cAdhSmXNcKlRU3-QaxW7Pflt-p8YAmyU5eC3EGJQfp7Mk7JpmP_P' \
  -H 'origin: https://live.douyin.com' \
  -H 'referer: https://live.douyin.com/444452144000' \
  -H 'sec-ch-ua: "Google Chrome";v="107", "Chromium";v="107", "Not=A?Brand";v="24"' \
  -H 'sec-ch-ua-mobile: ?0' \
  -H 'sec-ch-ua-platform: "macOS"' \
  -H 'sec-fetch-dest: empty' \
  -H 'sec-fetch-mode: cors' \
  -H 'sec-fetch-site: same-origin' \
  -H 'user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36' \
  -H 'x-secsdk-csrf-token: 000100000001c39118162052c6b50f6dadb067cc07c073c9b93f035ea22242c67c43f5951903172899f1bc355042' \
  --data-raw 'room_id=7167220081737468683&to_room_id=7167220081737468683&sec_to_user_id=MS4wLjABAAAAOgGR5D9qmmPglgaT08-30j8vnjeeAdmgXhJY_8Q7oLk&to_episode_id=0&send_type=4&send_scene=1&gift_source=0&buff_level=0&count=1&price=2&gift_id=2002&is_first_combo=true' \
  --compressed -iv
```

响应

 ```json
{
  "data": {
    "message": "Insufficient Fund",
    "prompts": "余额不足"
  },
  "extra": {
    "now": 1668755929438
  },
  "status_code": 40001
}
 ```



#### 快手接口

请求

```shell
curl 'https://live.kuaishou.com/live_graphql' \
  -H 'Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,zh-TW;q=0.7' \
  -H 'Connection: keep-alive' \
  -H 'Cookie: clientid=3; did=web_0d47b6546f1fd39fe4d1236703ae2b16; kuaishou.live.bfb1s=9b8f70844293bed778aade6e0a8f9942; client_key=65890b29; kpn=GAME_ZONE; userId=553458447; kuaishou.live.web_st=ChRrdWFpc2hvdS5saXZlLndlYi5zdBKgAV6EW8_dkcqveTyN6yy6uaMZd7O2c9rYOi19Fb3FhhOTPjtHtkb7lPQxQ4QaygTV0J-_Z0E7-4E5lFUZ2MRRzwjNAgbEEeSbf5duEVRtpGJnR_EEjJeZ3yyPMWPsJelIVcpSHGX02esKljXrWSXcbMWU709r4hxtaNHdMQtnvmLt1nijHxRE7lio0ZRYM5n-EK65VJq2EUpFqHFY_jFqxm0aEhrHsWfESUHgv806qk-5eqStgCIgOVwse70J74NPwA2TbGfOv-Ze0M-TVQJ0kHcHOHiTkKooBTAB; kuaishou.live.web_ph=75bafd83e69f9caf80b760c47f7b9c976d31; userId=553458447; ksliveShowClipTip=true' \
  -H 'Origin: https://live.kuaishou.com' \
  -H 'Referer: https://live.kuaishou.com/u/YiGe6666' \
  -H 'Sec-Fetch-Dest: empty' \
  -H 'Sec-Fetch-Mode: cors' \
  -H 'Sec-Fetch-Site: same-origin' \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36' \
  -H 'accept: */*' \
  -H 'content-type: application/json' \
  -H 'sec-ch-ua: "Google Chrome";v="107", "Chromium";v="107", "Not=A?Brand";v="24"' \
  -H 'sec-ch-ua-mobile: ?0' \
  -H 'sec-ch-ua-platform: "macOS"' \
  --data-raw '{"operationName":"SendTokenGift","variables":{"e":"UJfUx05OooLVLeaLzurAD7t0ycf7qHV57YPA64hWOh5Nslk4cNv5GCO5UIhUfygGimZNaBMDaQF7CWigSoxuOG4KmX6PBg7Nw/BxK3lCQ/MeVM8H3VRD7RIv7A9H4Zt+z/43c25jpTuaQrjLpxrWXzNYORIKJRjga9ZUGPCbNwatxYFMuVGEJcn8SZxFdd2rr1HMsQV2HXhl1PcILcXZ5fcnu7+VARIIj26snB4TOiQ=","iv":"yLelD2PBybOSK8LM","giftId":114,"liveStreamId":"sOuGkqrHrOs","count":1,"comboKey":"IZLFwC_9lDBi2YL6_1668754497884","giftToken":"CkMQj7b0hwIaNIECAoICgwLHAgmLAowCjQIOENsBnAGfAt8B4AEhoQLhAakCKXFysgLyAfYBtwL3AfgB+QEgNCi36OokEIzc0szIMBqQARbIi9FshY8J3yb72Pi3Kf3b4VlRZ8dHHr2d64OWA545YQ6SBsTaqA6ERQ9DQbGDCXK3L5MoVtVL/wy4cLlD+XTMYIYjEYUU/0IwTbvhrTXWdll64SIP1APvRQXCjDugMBShDAqlMCBPqREhchX0t8tXHhYO2h+h6k3+kwe3yAdSioapP7i5NxtuLxCFTYPiVA=="},"query":"mutation SendTokenGift($e: String, $iv: String, $giftId: Int, $liveStreamId: ID, $count: Int, $comboKey: String, $giftToken: String) {\n  sendTokenGift(e: $e, iv: $iv, giftId: $giftId, liveStreamId: $liveStreamId, count: $count, comboKey: $comboKey, giftToken: $giftToken) {\n    result\n    ksCoin\n    styleType\n    __typename\n  }\n}\n"}' \
  --compressed -iv
```

响应

```json
{
  "data": {
    "sendTokenGift": {
      "result": 1,//成功
      "ksCoin": 0,
      "styleType": null,
      "__typename": "SendGiftResult"
    }
  }
}
```

快手给端上的接口使用的[graphql](https://spec.graphql.org/draft/)统一中心化了入口，方便后续规范化管理接口；整体交互也有些差别，一个是后置判断，一个是前置判断礼物是否可以赠送，但是对于赠送礼物接口还是需要判断是否满足赠送的金额；前置对于用户体验会好些，少了些交互吧; 接口数据都有token加密验证，防止三方黑产中途拦截，后者对整体赠送数据也是压缩加密了(用户行为打点数据也是压缩上报的)；

Tips: 貌似礼物的价格在各个直播平台都一样的，不像商品价格有相对波动，只是会有些直播场景定制化的礼物，有种非理性情感冲动消费的感觉，搞直播类用户产品，社会心理学貌似挺重要的，老铁带一波节奏 666～



### 设计

#### 整体设计服务模块流程

![](https://raw.githubusercontent.com/weedge/mypic/master/%E7%9B%B4%E6%92%AD%E4%BA%92%E5%8A%A8-%E8%B5%A0%E9%80%81%E7%A4%BC%E7%89%A9%E8%AE%BE%E8%AE%A1.drawio.png)



#### DB

**gift 礼物表**：有限集，这个物料数目是固定的，没有SKU这一概念，可以直接定义好配置之后，直接存放在数据库中；便于后续缓存至远端或者服务本地；

| 字段         | 类型      | 描述                                          |
| ------------ | --------- | --------------------------------------------- |
| giftId       | int64     | UK  唯一键                                    |
| name         | string    | 礼物名称                                      |
| currencyCn   | int       | 价格：虚拟货币数目                            |
| unit         | int       | 虚拟货币度量单位，对应资产类型：金币/钻石/X币 |
| giftCategory | string    | 礼物类别                                      |
| iconUrl      | string    | 礼物icon地址                                  |
| sendRule     | string    | 赠送规则                                      |
| effectsUrl   | string    | 礼物特效地址                                  |
| createdAt    | timestamp | 创建时间                                      |
| updatedAt    | timestamp | 更新时间                                      |



**user_asset 用户拥有的虚拟资产表**：  用户当前拥有的虚拟币余额， 建库建表按照userId进行hash 分库分表分区/分片(Region) (写更新热点) 数据量按中国总人口计算，一张物理表存放1000w数据

| 字段      | 类型      | 描述                                          |
| --------- | --------- | --------------------------------------------- |
| userId    | int64     | 用户id                                        |
| assetCn   | int       | 资产数量                                      |
| assetType | int       | 资产类型 1. 金币 2.钻石 3. X币 ~~4. X优惠卷~~ |
| version   | int64     | 更新版本                                      |
| createdAt | timestamp | 创建时间                                      |
| updatedAt | timestamp | 更新时间                                      |

```sql
/* local mysql innodb(index B+TREE) */
create database `pay{$dbpartitions}`
CREATE TABLE `pay{$dbpartitions}`.`user_assert`
(
    `userId` bigint unsigned NOT NULL DEFAULT '0',
    `assetCn` bigint unsigned NOT NULL DEFAULT '0',
    `assetType` tinyint unsigned NOT NULL DEFAULT '0',
    `version` bigint unsigned NOT NULL DEFAULT '0',
    `createdAt` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updatedAt` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_user_assertType` (`userId`,`assetType`)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4 partition by hash ( `userId` ) partitions 16;
create table `pay{$dbpartitions}`.`user_assert${tbpartitions}` like  `user_assert`;


/* create table add this for tidb tikv(index LSM-TREE) */
/* eg: partitions*2^min(SHARD_ROW_ID_BITS,PRE_SPLIT_REGIONS) = 128 regions, echo regions 96MB(compressed) */
/* if don't use regions, those regions will be recycled */
/*T! SHARD_ROW_ID_BITS=4 PRE_SPLIT_REGIONS=3 */
CREATE TABLE `user_asset`
(
    `userId` bigint unsigned NOT NULL DEFAULT '0',
    `assetCn` bigint unsigned NOT NULL DEFAULT '0',
    `assetType` tinyint unsigned NOT NULL DEFAULT '0',
    `version` bigint unsigned NOT NULL DEFAULT '0',
    `createdAt` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updatedAt` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_user_assetType` (`userId`,`assetType`)
) DEFAULT CHARSET=utf8mb4 SHARD_ROW_ID_BITS=4 PRE_SPLIT_REGIONS=3 partition by hash ( `userId` ) partitions 16;
show table user_asset regions;


/* create db table for polardb-x innodb(index B+TREE) */
/* PARTITION_MODE drds/sharding (db,table), auto/partitioning use partitioning */
/* create database `pay` PARTITION_MODE=sharding; */
create database `pay` PARTITION_MODE=partitioning;
CREATE TABLE `pay`.`user_asset`
(
    `userId` bigint unsigned NOT NULL DEFAULT '0',
    `assetCn` bigint unsigned NOT NULL DEFAULT '0',
    `assetType` tinyint unsigned NOT NULL DEFAULT '0',
    `version` bigint unsigned NOT NULL DEFAULT '0',
    `createdAt` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updatedAt` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_user_assetType` (`userId`,`assetType`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 partition by hash ( `userId` ) partitions 128;
/*dbpartition by hash(`userId`) tbpartition by hash(`userId`) tbpartitions 16*/

show table info from `pay`.`user_asset`;
show topology from `pay`.`user_asset`;
show rule from `pay`.`user_asset`;
SHOW NODE;
show db status;
show table status;
```

对于**单个用户资产修改**；如果存在 **select 资产，然后根据不同产品策略进行业务逻辑计算出更新后资产，最后update 更新** 场景，在并发场景下，三种方式：

1. 服务直接通过分布式锁来处理，这种方式不能防住其他其他服务或者脚本直接操作数据库的情况，除非在操作之前也去获取一次锁，而且引入外部依赖；可以考虑自举方式，服务资源实例自己来上锁；

2. 悲观锁 select for update事务实现

 ```sql
SET AUTOCOMMIT=0; 
BEGIN；
	a = SELECT assetCn FROM user_asset WHERE userId={$userId} and assetType={$assetType} FOR UPDATE;  
	update user_asset set assetCn=a.assetCn+{$incrCn} where userId={$userId} and assetType={$assetType}; 
COMMIT;
 ```

3. 乐观锁 CAS的方式，没有更新继续循环，直到更新ok

```sql
a = SELECT assetCn FROM user_asset WHERE userId={$userId} and assetType={$assetType};
#$oldAssetCn = a.assetCn
#$newAssetCn = a.assetCn+$incrCn
#if $newAssetCn>=0 ->update
update user_asset set assetCn={$newAssetCn} where userId={$userId} and assetType={$assetType} and assetCn = {$oldAssetCn} 
```

存在ABA问题：

> 考虑如下操作：
>
> 并发1（上）：获取出数据的初始值是A，后续计划实施CAS乐观锁，期望数据仍是A的时候，修改才能成功
>
> 并发2：将数据修改成B
>
> 并发3：将数据修改回A
>
> 并发1（下）：CAS乐观锁，检测发现初始值还是A，进行数据修改
>
> 并发1在修改数据时，虽然还是A，但已经不是初始条件的A了，中间发生了A变B，B又变A的变化，此A已经非彼A，数据却成功修改，可能导致错误
>
> ABA问题导致的原因，是CAS过程中只简单进行了“值”的校验，再有些情况下，“值”相同不会引入错误的业务逻辑（例如库存），有些情况下，“值”虽然相同，却已经不是原来的数据了。

加上版本字段version, 对版本进行CAS更新

```sql
a = SELECT assetCn,version FROM user_asset WHERE userId={$userId} and assetType={$assetType};
#$oldAssetCn = a.assetCn
#$oldVersion = a.version
#$newAssetCn = a.assetCn+$incrCn
#if $newAssetCn>=0 ->update
#$newVersion = $oldVersion+1
update user_asset set assetCn={$newAssetCn} and version={$newVersion} where userId={$userId} and assetType={$assetType} and version = {$oldVersion} 
```

想一想：mysql为了高可用和读写分离，生产环境部署的实例集群是主从架构，存在主从延迟，其实这个是不影响的，最终都是CAS的update更新，更新成功会返回affect rows为1，没有更新则为0；

Notice: 

1. select for update 加排斥锁和所建的索引有关(间隔(gap)锁，临键(next-key)锁，锁行/表)

2. 如果使用mongodb来存放，直接使用findAndModify来操作即可，当然防止重复数据，需要加唯一索引(锁文档)

**user_asset_record 用户虚拟币交易流水表**：记录虚拟币增加和减少数据详情，这个提供后台查看，用于重复请求幂等处理， 建库建表按照userId进行hash 分库分表分区 / 分片(Region)； (写插入热点) , notice: 这里冗余设计了，可以按照第三范式，一个事件有多人参与，一个人可以参与多个事件，分出一个用户事件关联表(user_event)，一个事件表(event_record)；主要是方便按用户维度获取流水记录；这里如果标准字段换成存放操作用户opUser和接受用户toUser，如果按照opUser维度拆分，查询toUser的流水数据就不方便，虽然polardb-x可以使用[全局二级索引](https://help.aliyun.com/document_detail/311522.html)来解决这个问题，但是换个建表维度思路既可以满足当前这个业务场景，同时也通用统一使用userId进行拆分，如果是ToB场景，可以使用租户id(tenantId)分库(物理库)，用户userId分表，这里ToC场景用户维度，直接userId分区即可。

| 字段       | 类型      | 描述                                                         |
| ---------- | --------- | ------------------------------------------------------------ |
| userId     | int64     | 用户id                                                       |
| opUserType | int64     | 1.操作者，2.接收者                                           |
| bizId      | int64     | 直播场景：roomId，充值场景：订单Id                           |
| bizType    | int       | 业务类型：1.直播互动，2.充值                                 |
| eventId    | string    | 事件Id: 直播互动场景下，互动事件id 用于贯彻整个送礼物流水链路,进行幂等处理; 用户充值场景下，订单事件id,  UUID |
| eventType  | string    | 事件类型，interactGift,  orderApple, orderWX, orderAlipay, orderDouyin |
| objId      | string    | 操作对象id, giftId, transactionId/outOrderNo                 |
| record     | string    | 记录行为                                                     |
| recordOp   | string    | 操作记录 虚拟币增加和减少                                    |
| createdAt  | timestamp | 创建时间                                                     |
| updatedAt  | timestamp | 更新时间                                                     |

```sql
/* local mysql innodb(index B+TREE) */
create table `pay{$dbpartitions}`.`user_assert_record`
(
    `userId` bigint unsigned not null default '0',
    `opUserType` tinyint unsigned not null default '0',
    `bizId` bigint unsigned not null default '0',
    `bizType` tinyint unsigned not null default '0',
    `objId` varchar(128) not null default '',
    `eventId` varchar(128) not null default '',
    `eventType` varchar(128) not null default '',
    `record` varchar(256) not null default '',
    `recordOp` varchar(64) not null default '',
    `createdAt` timestamp not null  DEFAULT CURRENT_TIMESTAMP,
    `updatedAt` timestamp not null default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_user_opUserType_event` (`userId`,`opUserType`,`eventId`)
)engine=InnoDB default charset=utf8mb4 partition by hash(`userId`) PARTITIONS 16 ;
create table `pay{$dbpartitions}`.`user_assert_record${tbpartitions}` like  `user_assert_record`;


/* create table add this for tidb tikv(index LSM-TREE) */
/* eg: partitions*2^min(SHARD_ROW_ID_BITS,PRE_SPLIT_REGIONS) = 256 regions, echo regions 96MB(compressed) */
/* if don't use regions, those regions will be recycled */
/*T! SHARD_ROW_ID_BITS=4 PRE_SPLIT_REGIONS=5 */
create table `pay`.`user_asset_record`
(
    `userId` bigint unsigned not null default '0',
    `opUserType` tinyint unsigned not null default '0',
    `bizId` bigint unsigned not null default '0',
    `bizType` tinyint unsigned not null default '0',
    `objId` varchar(128) not null default '',
    `eventId` varchar(128) not null default '',
    `eventType` varchar(128) not null default '',
    `record` varchar(256) not null default '',
    `recordOp` varchar(64) not null default '',
    `createdAt` timestamp not null  DEFAULT CURRENT_TIMESTAMP,
    `updatedAt` timestamp not null default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_user_opUserType_event` (`userId`,`opUserType`,`eventId`)
)engine=InnoDB default charset=utf8mb4 SHARD_ROW_ID_BITS=4 PRE_SPLIT_REGIONS=5
    PARTITION BY HASH (`userId`) PARTITIONS 16;
show table user_asset_record regions;


/* create db table for polardb-x innodb(index B+TREE) or tokudb(index Fractal tree) or x-engine(index LSM-TREE) ,for this scene use innodb */
/* PARTITION_MODE drds/sharding (db,table), auto/partitioning use partitioning */
/* create database `pay` PARTITION_MODE=sharding; */
create database `pay` PARTITION_MODE=partitioning;
create table `pay`.`user_asset_record`
(
    `userId` bigint unsigned not null default '0',
    `opUserType` tinyint unsigned not null default '0',
    `bizId` bigint unsigned not null default '0',
    `bizType` tinyint unsigned not null default '0',
    `objId` varchar(128) not null default '',
    `eventId` varchar(128) not null default '',
    `eventType` varchar(128) not null default '',
    `record` varchar(256) not null default '',
    `recordOp` varchar(64) not null default '',
    `createdAt` timestamp not null  DEFAULT CURRENT_TIMESTAMP,
    `updatedAt` timestamp not null default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_user_opUserType_event` (`userId`,`opUserType`,`eventId`)
)engine=InnoDB /*tokudb/xengine*/ default PARTITION BY HASH (`userId`) PARTITIONS 256;
/*dbpartition by hash(`userId`) tbpartition by hash(`userId`) tbpartitions 16*/

show table info from `pay`.`user_asset_record`;
show topology from `pay`.`user_asset_record`;
show rule from `pay`.`user_asset_record`;
SHOW NODE;
show db status;
show table status;
```

资源评估：用抖音看下卡塔尔世界杯，支持下梅老板的阿根廷🇦🇷队直播，去估算下吧～～

Tips: 采用分布式数据库，对其资源评估，建表，索引，事务，查询优化等操作；请参考规范和最佳实践：[tibd 最佳实践](https://docs.pingcap.com/zh/tidb/stable/tidb-best-practices) [TiDB 高并发写入场景最佳实践](https://docs.pingcap.com/zh/tidb/stable/high-concurrency-best-practices) [tidb开发规范](https://cn.pingcap.com/best-practice-detail/best-practices-for-developing-applications-with-tidb) [palordb-x最佳实践](https://help.aliyun.com/document_detail/308293.html) ;

题外话：Tidb 底层存储tikv分片采用的范围分片，如果没有指定分片数，默认1个，当tikv上一个region写满了，会动态分裂出一个新的region, 可以提前分配好分片数，以防潮汐流量热点写入，比如这里的热门主播资产信息，以及资产流水记录，通过配置 SHARD_ROW_ID_BITS 参数通过tidb自身服务隐藏ROW_ID hash打散写入热点，类似mr任务数据倾斜时的再次hash打散，与redis/程序中的map rehash不同，redis rehash解决碰撞查找效率低和空间使用率低问题，通过loadFactor阈值，触发扩容和缩容，golang中map rehash仅触发扩容，这里所谈的是流量倾斜导致负载不均衡重新rehash； 而polardb-x的负载扩容 则类似于redis的rehash机制扩容，数据倾斜的问题是不能通过扩容来解决，可参考[polardb-x如何分析数据分布不均衡](https://help.aliyun.com/document_detail/309469.html)。

在支付中台，某个互动涉及到多个用户资产的变更，需要把多个写操作关联到一个本地事务进行处理，保证数据扣减和增减一致，需要开启本地事务来处理多表数据；以下gist为并发mysql本地事务处理测试demo

{{< gist weedge 1700dd7053a87a4ab35ba4fce0ebea6a >}}

{{< gist weedge fd731ce8549cd99ccfc9491f6025ae8e >}}



#### Cache

##### key设计：

| 名称                     | key                                                     | type        | value            | ex   | 是否频繁更新 | 说明                                                         |
| ------------------------ | ------------------------------------------------------- | ----------- | ---------------- | ---- | ------------ | ------------------------------------------------------------ |
| 用户虚拟资产信息         | I.asset.{userId}.{assetType.string()}                   | string/hash | user_asset(json) | 1 d  | Y            | type: String vs Hash tradeoff <br />1. String 对于redis读详情数据效率高，但是写操作资产数目需要json decode/encode get/set，如果操作是在业务应用逻辑服务器上操作可以使用(有对应优化json库)，结合watch+事务cas方式; <br />2. Hash 相对string读详情数据效率低些，但是写操作资产数目仅需hincrby(hget/hset)操作即可，如果操作是在数据中心redis上执行lua脚本原子操作可以使用，redis lua 使用cjson库相对解析效率低些，特别是大json，参考[sproto](https://blog.codingnow.com/2014/07/sproto.html)，并发量大容易增加redis数据中心负载，降低吞吐； 结合具体场景使用(尽量计算存储分离) |
| 礼物信息                 | I.gift.{giftId}                                         | string      | gift(json)       | 7 d  | N            |                                                              |
| 用户信息                 | I.user.{userId}                                         | string      | user(json)       | 1 d  | N            |                                                              |
| 直播间信息               | I.room.{roomId}                                         | string      | room(json)       | 1 d  | N            |                                                              |
| 获取用户虚拟资产分布式锁 | L.asset.{userId}.{tag}   <br />(tag:assetType.String()) | string      | token            | 60 s | N            | AP型锁，加锁单个线程从db中获取数据写入缓存，初始化送礼用户资产信息，锁的是热更新资源；存在数据复制的延迟可能带来的数据写后读（read-after-write）不一致问题，所以**从follower读取数据必须是强一致性读(tidb支持)/全局一致性读(polardb-x支持)，否则从leader/master上读取**；<br />这里使用redlock，如果运行锁失效多次读写入缓存幂等操作,竞争条件是可以接受，直接用单集群实例redlock就行；否则需要多集群redlock加锁，具体详情：[distributed-locks](https://redis.io/docs/manual/patterns/distributed-locks/) PS: 类似使用数据库锁也可以使用redlock算法来实现分布式可靠加锁;  锁的粒度，可以先分散到本地锁(需要加等待超时时间，防止夯主服务，超时上游可以重试)，然后分布式锁只有单个线程获取数据去设置缓存，降低分布式锁的竞争，最多部署进程实例数目竞争,高并发场景下很大幅度减少网络IO。 如下场景：**很多观众给热门女主播送礼物，这个女主播同时给热门男主播送礼物** |
| 资产变更消息             | M.asset.{userId}.{eventId}                              | string      | 1                | 1 d  | N            | 1. 保证幂等：类似Once done 原子操作(event事务维度)； <br />2.事务消息回调RC可见(原子操作) |
|                          |                                                         |             |                  |      |              |                                                              |

资源评估：用抖音看下卡塔尔世界杯，支持下梅老板的阿根廷🇦🇷队直播，去估算下吧～～

并发场景：

1. 缓存从db中获取

2. 频繁增/减用户虚拟资产缓存存量数据

热key:

1. 用户维度的用户虚拟资产信息，通过userId进行分散即可， 如果多用户/租户 对 共享缓存资源频繁操作(比如 优惠卷/商卷)，突破了单redis实例的读写瓶颈，需要对key进行再次切分；如果是读多场景，更新不频繁缓存可以缓存在本地服务进程中；

大key:

1. 存放的value值比较大，一般是list, set，zset, hash这些集合结构，存储的item/field数目一般在5000个左右，需要按比例切分，读放大的问题可以并发控制读取；

watch+事务，lua原子操作 demo代码如下：

{{< gist weedge 2b473c5baf0ac59d8d0c8b1ddc5692f5 >}}

{{< gist weedge c2d830dceef4163acc6dd749a05493db >}}

#### 消息队列

##### Topic

| 名称                     | 读/写队列数  | 消息类型 | 消息key | 消息tag                         | 说明             |
| ------------------------ | ------------ | -------- | ------- | ------------------------------- | ---------------- |
| TOPIC_ASSET_CHANGE_EVENT | default: 8/8 | 事务消息 | eventId | {eventType} ( \|\| 间隔多个tag) | 用户事件资产变更 |
|                          |              |          |         |                                 |                  |

Tips: 

1. topic tag: 长度不能超过 127 (Byte.MAX_VALUE )；
2. 读写队列数由消费集群和生产集群吞吐量决定，开发可以使用默认8/8；
3. 事务消息不支持批量生产和延时生产，即使在生产侧设置延时发送事务消息，rocketmq 4.7.0之后的不会生效，[commit](https://github.com/GenerousMan/rocketmq/commit/5a584d056a07350911954dd269bb1ee5c80dfd11) ;
4. [RocketMQ事务消息改进](https://github.com/apache/rocketmq/wiki/RIP-50-RocketMQ-Transaction-Message-Improvement) 这个改进提案暂时还未发布；

##### Message

| 字段        | 类型   | 说明                                                        |
| ----------- | ------ | ----------------------------------------------------------- |
| eventId     | string | 变更事件id                                                  |
| opUserId    | int64  | 操作者id                                                    |
| eventType   | string | 变更事件类型，interactGift                                  |
| messageBody | string | 消息数据，更具消息事件类型定义，格式可以为json 以便跟踪查看 |

producer GroupName/GroupID: P_GID_GIFT_ASSET_CHANGE

consumer GroupName/GroupID: C_GID_GIFT_ASSET_CHANGE   5.0版本尽量采用[POP consumer模式](https://mp.weixin.qq.com/s/UFWULymSblrs1hIGP5e7YQ) , 比如执行如下命令切换成`POP`模式：

```shell
./mqadmin setConsumeMode -c DefaultCluster -t test -g testGroup -m POP -n namesrv:9876
```

消息大小不能超过4M, 可参考 [5.0版限制](https://help.aliyun.com/document_detail/440347.html)

消息重试默认16次, 设置重试延迟级别(level)，设置的延迟级别下标(level-1)如下延迟数组`delayLevelArray`：

`[1s, 5s, 10s, 30s, 1m, 2m, 3m, 4m, 5m, 6m, 7m, 8m, 9m, 10m, 20m, 30m, 1h, 2h]`

如果不在指定延迟级别内，使用默认 `delayLevelArray[2:]`延迟数组依次重试16次；

##### 消费侧需要关注的参数

| 名称                       | 默认 | 说明                                                         |
| -------------------------- | ---- | ------------------------------------------------------------ |
| pullBatchSize              | 32   | 每次发起pull请求到broker，客户端需要指定一个最大batch size，表示这一次拉取消息最多批量拉取多少条；范围在 [1,1024]; 为了提高吞吐，一般都大于1 |
| consumeMessageBatchMaxSize | 1    | 批量消费的最大消息条数；范围在 [1,1024]；如果消费逻辑支持批量处理，可以设置值大于1； 处理业务逻辑的批量msgs的最大大小是consumeMessageBatchMaxSize和pullBatchSize的较小值。 |
| maxReconsumeTimes          | -1   | -1 默认16次重试；0或者小于-1 不重试；大于0，则为设置的重试次数 |
|                            |      |                                                              |

消费侧从broker pull message流程：

```
NewPushConsumer -> GetOrNewRocketMQClient -> GetOrNewRocketMQClient-> 注册RegisterRequestFunc  ReqResetConsumerOffset事件的回调func -> 触发 resetOffset -> ResetOffset-> resume -> doBalance -> updateProcessQueueTable写入拉去请求到prCH中

Start -> RegisterConsumer -> start
异步 select 轮训从prCh中获取拉去请求pr -> 异步 pullMessage(pr)->
	异步 select 轮训处理pr中的处理队列和消息队列 submitToConsume(pr.pq, pr.mq) (一个是顺序orderly处理，一个是普通currently处理)
	
	->pq.putMessage(msgs)从broker中请求的消息放入处理队列中的msgCh中
	
	->普通处理consumeMessageCurrently() -> pq.getMessages 从处理队列中的msgCh中获取消息-> 
		拉取到的一批消息会拆分成N（取决于consumeMessageBatchMaxSize）个小批消息subMsgs -> 
		异步 consumeInner(ctx,subMsgs) -> 调用业务定义的回调业务函数callback.f(ctx, subMsgs...)->
	处理完成返回响应
```

对于普通消息的处理，可以看出批量拉去拆分成N(PullBatchSize/min(ConsumeMessageBatchMaxSize,PullBatchSize))个批量msgs, 每次获取批量msgs处理都是并发的，不会相互等待；拉完一批触发偏移事件继续拉去下一批到本地pr队列中，直到broker队列中没有可消费的数据；重试有对应topic重试队列不会阻塞当前topic队列的正常消费；

其他参考：[客户端配置详解](https://zhuanlan.zhihu.com/p/27397055)

##### RocketMQ 重试, 死信, 系统 Topic

| 名称                           | 说明                                                         |      |
| ------------------------------ | ------------------------------------------------------------ | ---- |
| %RETRY%C_GID_GIFT_ASSET_CHANGE | 对应消费组重试topic，重试次数默认16次                        |      |
| %DLQ%C_GID_GIFT_ASSET_CHANGE   | 对应消费组死信队列topic, 超过重试次数放入该队列中            |      |
| SCHEDULE_TOPIC_XXXX            | 延迟消息队列topic                                            |      |
| *TBW102*                       | 默认用于创建不存在topic时使用这个默认topic来创建             |      |
| RMQ_SYS_TRACE_TOPIC            | 开启消息跟踪的topic 用于消息轨迹                             |      |
| RMQ_SYS_TRANS_HALF_TOPIC       | 记录所有的半事务消息，消费端不可见                           |      |
| RMQ_SYS_TRANS_OP_HALF_TOPIC    | 记录已经COMMIT或ROLLBACK的半事务消息，tags是"d" 逻辑删除     |      |
| TRANS_CHECK_MAX_TIME_TOPIC     | 未知状态的事务消息超过最大回查次数，默认15次，会存在这个队列 |      |

具体生产，消费用户资产事务消息demo如下: (demo中使用 [rocketmq-client-go](https://github.com/apache/rocketmq-client-go) **客户端生产消费配置了 trace，rocketmq broker也需要配置traceTopicEnable=true，用于查看消息轨迹**；除此之外，可以在消息属性中加上全链路追踪的traceId，用于整体系统服务进行串联，如果想把rocketmq客户端生产和消费加入OTEL进行全链路追踪, 可以参考 **[kafka-otel-sarama](https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/github.com/Shopify/sarama/otelsarama)** 实现不难)

{{< gist weedge fcbc8e4a2889c6230d62f6f35de6862d >}}

{{< gist weedge 37839a2fcb3c11f77228ce43a713dee7 >}}

#### API

前台统一入口接口参数安全验证

接口公共响应 BaseResp

| 字段    | 类型               | 说明                       |
| ------- | ------------------ | -------------------------- |
| errCode | int                | 错误码，0代表成功，非0错误 |
| errMsg  | string             | 错误信息                   |
| extra   | Map<string,string> | 额外信息                   |

errCode分配：

| 服务 | 分配范围      | 说明 |
| ---- | ------------- | ---- |
| 互动 | [10000,20000) |      |
| 支付 | [20000,30000) |      |
| 消息 | [30000,40000) |      |

互动中台：

1. 赠送礼物接口
2. 获取直播间礼物列表接口

支付中台：

1. **变更用户虚拟资产接口**

BizAssetChangesReq 请求：(批量并发控制统一收敛至支付中台，调用方无须并发获取)

| 字段            | 类型                        | 说明             |
| :-------------- | --------------------------- | ---------------- |
| bizAssetChanges | list<\BizEventAssetChange/> | 业务资产变更列表 |

BizEventAssetChange

| 字段              | 类型                | 说明                                                         |
| ----------------- | ------------------- | ------------------------------------------------------------ |
| eventId           | string              | 操作事件id                                                   |
| opUserId          | int64               | 操作者id                                                     |
| eventType         | string              | 事件类型: interactGift,  orderApple, orderWX, orderAlipay, orderDouyin |
| bizId             | int64               | 业务场景id: 直播(roomId), 充值(orderId)                      |
| bizType           | int                 | 业务类型 1.直播，2.充值                                      |
| objId             | string              | 操作对象id, giftId, transactionId/outOrderNo                 |
| opUserAssetChange | UserAssetChangeInfo | 操作用户资产变更                                             |
| toUserAssetChange | UserAssetChangeInfo | 对方用户资产变更                                             |

UserAssetChangeInfo

| 字段      | 类型  | 说明                |
| --------- | ----- | ------------------- |
| userId    | int64 | 用户id              |
| assetType | int   | 资产类型            |
| incr      | int   | +增加/-减少多少资产 |
|           |       |                     |

BizAssetChangesResp 响应

| 字段                  | 类型                            | 说明             |
| --------------------- | ------------------------------- | ---------------- |
| bizAssetChangeResList | list<\BizEventAssetChangerRes/> | 资产变更结果列表 |
| baseResp              | BaseResp                        | 公共响应信息     |

BizEventAssetChangerRes

| 字段        | 类型      | 说明             |
| ----------- | --------- | ---------------- |
| eventId     | string    | 操作事件id       |
| opUserAsset | UserAsset | 操作者的用户资产 |
| changeRes   | bool      | 1 成功， 0 失败  |
| failMsg     | string    | 失败信息         |

UserAsset

| 字段      | 类型  | 说明     |
| --------- | ----- | -------- |
| userId    | int64 | 用户id   |
| assetType | int   | 资产类型 |
| assetCn   | int   | 资产数   |

2. 获取用户虚拟资产接口

3. 获取用户资产变更流水接口
4. **更新数据库中的用户资产 消费逻辑**

消息服务： https://weedge.github.io/post/jxzbim/



### 开发

借助开源工具，从0到1开始搭建蓝图，>1由业务驱动宏图

1. 搭建本地基础环境，tidb/polardb-x, redis, rocketmq 1d

   ```shell
   #just for mac os local brew install
   brew install redis
   brew install mysql
   #list services to view is ok 
   brew services list
   ```

   ```yaml
   #redis-cluster-single-docker-compose.yaml
   #https://github.com/bitnami/bitnami-docker-redis-cluster/issues/3
   version: "3"
   name: "single-redis-cluster"
   services:
     redis-cluster:
       image: grokzen/redis-cluster:latest
       ports:
         - "26379-26384:26379-26384"
       environment:
         - "INITIAL_PORT=26379"
         - "MASTERS=3"
         - "SLAVES_PER_MASTER=1"
         - "SENTINEL=false"
         - "REDIS_CLUSTER_IP=0.0.0.0"
         - "IP=0.0.0.0"
         - "BIND_ADDRESS=0.0.0.0"
   ```

   ```shell
   # redis-cluster docker local deploy
   docker-compose -f redis-cluster-single-docker-compose.yaml up -d
   # check cluster state is ok 
   redis-cli -c -p 26379 cluster info
   
   # rocketmq docker local deploy    
   # https://github.com/apache/rocketmq-docker
   git clone https://github.com/apache/rocketmq-docker.git
   cd image-build
   sh build-image.sh 5.0.0 alpine
   cd ..
   sh stage.sh 5.0.0
   cd stages/5.0.0
   # change data/broker/conf/broker.conf add brokerIP1={localIp}, add traceTopicEnable=true
   # docker run -d -v `pwd`/data/broker/logs:/home/rocketmq/logs -v `pwd`/data/broker/store:/home/rocketmq/store -v `pwd`/data/broker/conf/broker.conf:/opt/rocketmq-5.0.0/conf/broker.conf --name rmqbroker --link rmqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" -p 10909:10909 -p 10911:10911 -p 10912:10912 apache/rocketmq:5.0.0${TAG_SUFFIX} sh mqbroker -c /opt/rocketmq-5.0.0/conf/broker.conf
   # run simple single node docker
   ./play-docker.sh alpine
   
   # rocketmq dashboard local deploy
   docker pull apacherocketmq/rocketmq-dashboard:latest
   docker run -d --name rocketmq-dashboard -e "JAVA_OPTS=-Drocketmq.namesrv.addr={localIp}:9876" -p 8181:8080 -t apacherocketmq/rocketmq-dashboard:latest
   # view http://127.0.0.1:8181/
   
   #tidb local deploy
   # notice:
   # docker is not support
   # u should use deploy local tidb by tiup tool
   # see this https://docs.pingcap.com/tidb/dev/quick-start-with-tidb
   tiup playground
   
   #polardb-x docker local deploy by pxd tool
   #see this 
   
   pxd tryout
   ```

2. 测试demo, redis cluster集群下的缓存事务和 tidb分布式缓存事务，rocketmq 分布式消息事务；

3. 核心服务模块开发支付中台接口 

4. 编写makefile, dockerfile, 打包成docker镜像， 整体解决方案依赖组件也可以通过docker-compose部署至docker容器中

5. 编写相关k8s资源 (configmap,secret,deployment+service,ingress)通过minikube/kind 部署至本地k8s集群 

6. 将服务网格化istio(xds流控, 可用于全链路监控, A/B测试，特别是产品新特性/模型策略场景)  (like this: https://mp.weixin.qq.com/s/SAn-H5p53IfvSy_Y3Mcz_Q)

7. k8s/istio operator

开源开发框架和组件选择(在标准化，规范模块化的前提下，尽量自动化，提高研发效能，focus on 核心业务逻辑)

1. 开发语言：golang + lua5.1([redis lua debugging](https://redis.io/docs/manual/programmability/lua-debugging/)) + shell 

2. 开发框架：[kitex](https://www.cloudwego.io/zh/docs/kitex/overview/)(统一规范化了RPC框架，支持gRPC和thrift的脚手架，常支持内部微服务，泛化调用+ proxyless xds流控, 参考[brpc](https://mp.weixin.qq.com/s/G8vmlJyaimux_K-548kFbA)) + [hertz](https://www.cloudwego.io/zh/docs/hertz/overview/)(http协议框架，常支持外部业务前/后台, 数据渲染,请求校验)

3. DB：mysql, 高可用集群方案：不推荐自建sharding+proxy的方式；推荐使用支持mysql协议的分布式数据库，且无需在业务代码中考虑分库分表操作，以及分库分表的分布式事务；比如：开源方案 [tidb](https://docs.pingcap.com/zh/)(shared nothing,scale out)，云厂商：[polardb-x](https://polardbx.com/document) (shared nothing, scale out)/[aurora mysql](https://docs.aws.amazon.com/zh_cn/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMySQL.html) (shared disk, scale up, aurora相关解读可以看下[mit 6.824 Cloud Replicated DB, Aurora](youtube.com/watch?v=jJSh54J1s5o)),   [gorm](https://gorm.io/)

   Tips: 对应分布式事务操作tidb有所不同，tidb和原生的mysql innodb引擎事务操作sql语句有些区别: [tidb transaction](https://docs.pingcap.com/zh/tidb/dev/transaction-overview), [tidb-gorm-sample](https://docs.pingcap.com/zh/tidb/dev/dev-guide-sample-application-golang)；[polardb-x 分布式事务](https://doc.polardbx.com/features/topics/distributed-transaction.html) 和 aurora 则原生支持事务操作sql语句;

4. Cache:  redis, 高可用集群方案 开源方案[redis-cluster](https://redis.io/docs/management/scaling/), [redis-cluster-proxy](https://github.com/RedisLabs/redis-cluster-proxy)；云厂商proxy代理方式：[阿里云redis](https://help.aliyun.com/document_detail/52228.html),  或者支持redis cluster协议 底层利用rocksdb/leveldb 作为kv存储引擎的开源方案，比如[kvrocks](https://github.com/apache/incubator-kvrocks)(watch命令暂未支持，可以使用lua脚本命令),  [go-redis](https://redis.uptrace.dev/guide/)

5. MQ: rocketMQ, [rocketmq-client-go](https://github.com/apache/rocketmq-client-go),  如果使用阿里云rocketMQ使用对应[client SDK](https://github.com/apache/rocketmq-clients/tree/master/golang) (producer已支持全部消息类型)

6. 服务编排流量治理：[k8s](https://kubernetes.io/zh-cn/docs/home/) + [istio](https://istio.io/latest/docs/) 

tips: 在本地开发，可以通过docker compose 本地部署； 还可以通过docker+minikube/kind+helm 来部署本地节点的多pod集群版本数据库(mysql/tidb/polardb-x)，redis-cluster，以及集群版rocketMQ；以及开发完业务服务应用后，也可以部署在本地pod的容器中, 以便后续CI/CD自动化集成部署至多云容器服务/自托管的容器服务中，具体可参考[k8s ci/cd](https://jimmysong.io/kubernetes-handbook/practice/ci-cd.html) , GitOps实践: [github-jenkins-argo](https://cloudyuga.guru/blog/jenkins-argo) , [使用gitlab,Jenkins和Argocd实现CI/CD](https://www.jokerbai.com/archives/ji-yu-jenkins-he-argocd-shi-xian-devops), [使用argo rollouts实现金丝雀发布](https://www.jokerbai.com/archives/shi-yong-argorollouts-shi-xian-jin-si-que-fa-bu)

无状态服务快速开发迭代大致自动化构建部署如下：

CI:  begin -> build(local makefile/test build, dockerfile build) -> push docker registry(自建[harbor](https://goharbor.io/docs/2.6.0/)或者云服务) -> end  (gitlab/jenkins CI)

CD: 分开发，测试，预发和生产环境，开发，测试直接单pod部署实例；预发双pod;  生产环境则分阶段部署和回滚：蓝绿部署/金丝雀(灰度)部署 ([argo CD](https://argo-cd.readthedocs.io/en/stable/), [argo Rollouts](https://argoproj.github.io/argo-rollouts/))

有状态数据存储服务通过operator部署：

polardb-x operator: https://doc.polardbx.com/operator/

Tidb-operator: https://github.com/pingcap/tidb-operator

Redis-cluster:   https://github.com/bitnami/charts/tree/main/bitnami/redis-cluster , https://kubedb.com/docs/v2022.10.18/guides/redis/clustering/redis-cluster/

RocketMQ-operator on k8s: https://github.com/apache/rocketmq-operator

题外话：或者进一步的直接使用无服务serverless 函数计算，开源方案[**Knative**](https://www.cncf.io/online-programs/event-driven-architecture-with-knative-events/) [openfaas on k8s](https://docs.openfaas.com/deployment/kubernetes/), 或者使用云服务比如[aws lambda](https://aws.amazon.com/cn/campaigns/lambda/)(transform, not transport data), 专注于业务逻辑，减少'胶水'代码，缺少了开发框架的依赖，常用于数据事件处理,比如直播音视频离线/实时处理（DevOps->AppOps)， **[AWS re:Invent 2022 - Best practices for advanced serverless developers](https://www.youtube.com/watch?v=PiQ_eZFO2GU&list=PL2yQDdvlhXf_lYR5Ntvr9V5iVYv5rcbNc&index=6)**, **[The Complete AWS SAM workshop](https://catalog.workshops.aws/complete-aws-sam/en-US)**。

**代码地址**： https://github.com/weedge/craftsman/tree/main/cloudwego

代码结构：(放在一个git仓库(svn)中主要是为了方便提交开发查看整体开发框架哈，如果想多人玩可以建个组织拆分哈,通过submodule组合在一起CP git flow，g*yhub)

```
├── aws	-------------- use aws cloud develop, focus on biz logic by use lambda / step functions
│   └── cdk ------ deploy infrastructure ECS, VPC, EKS, S3, cache,db/search,mq etc for severless 
├── cloudwego	----------------- use cloudwego framwork develop
│   ├── common	----------------- biz common for idl,dto,dict enum; rpc service/client, pkg
│   │   ├── idl
│   │   ├── kitex_gen
│   │   │   ├── base
│   │   │   ├── common
│   │   │   └── payment
│   │   │       ├── base
│   │   │       ├── da
│   │   │       │   └── paymentservice
│   │   │       └── station
│   │   │           └── paymentservice
│   │   └── pkg
│   │       └── constants
│   ├── kitex-contrib ------------------------ kitex rpc framwork contrib (add new)
│   │   ├── gorm
│   │   └── obs-opentelemetry
│   │       └── logging
│   │           └── zap
│   └── payment		---------------------------- app payment have http ui/gateway, rpc station,da service/server
│       ├── bin		---- go build cmd output to bin
│       ├── build	---- makefile, dockerfile,docker-compose.yml to build, run
│       ├── cmd		---- main.go to load da/station/gw cmd to run
│       │   ├── da
│       │   ├── gw
│       │   └── station
│       ├── conf	---------- env biz config for local/dev/test/pre/stress/gray/prod 
│       │   ├── dev
│       │   ├── gray
│       │   ├── local
│       │   ├── pre
│       │   ├── prod
│       │   ├── stress
│       │   └── test
│       ├── data	-------------- sql data, encode/decode meta data
│       ├── docs	-------------- app help doc
│       ├── internal ----------- internal don't be used by out package, code biz logic
│       │   ├── da	------------------------ dal 
│       │   │   ├── consumer ---- event drive consumer
│       │   │   ├── dao      ---- db table data persistence op
│       │   │   ├── domain   ---- domain entries, interface, errors
│       │   │   │   └── mocks ---- mock for test
│       │   │   ├── model    ---- table entry model 
│       │   │   ├── repository ---- impl domain db repository interface
│       │   │   │   └── mysql
│       │   │   └── usecase ----- biz logic use repositories 
│       │   ├── gw	------------------------ ui/gateway
│       │   │   └── middleware -------- middleware handle
│       │   └── station -------------------- biz logic station by using cache 
│       │       └── consumer ---- event drive consumer for cache
│       │       ├── domain   ---- domain entries, interface, errors
│       │       ├── repository ---- impl domain cache, mq produce repository interface
│       │       │   ├── redis
│       │       │   │   └── lua
│       │       │   └── rmq
│       │       └── usecase  ----- biz logic use repositories 
│       ├── manifests -------------- k8s/istio configmap,secret,deployment+service,ingress resource workload and traffic/flow control
│       │   ├── traffic
│       │   │   └── istio
│       │   └── workloads
│       └── pkg --------------------- app common pkg
│           ├── configparser
│           ├── constants
│           ├── injectors
│           ├── subscriber
│           ├── utils
│           │   ├── logutils
│           │   └── netutils
│           └── version 
├── kratos	----------------- use kratos framwork develop
└── opentelemetry-go-contrib ----- otel go contrib just for new one, if change, it's not origin, u can fork, go mod edit -replace
```

服务可以通过多个service组合成一个单体server，也可以从一个单体server拆分成多个微服务server，以便企业内部组织架构来调整服务(基于上下文软件架构原则 **[Conway's law](https://martinfowler.com/bliki/ConwaysLaw.html)**)

服务内部各个模块组件采用DDD [clean-architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) 架构，接口隔离实现，依赖反转(**[Dependency Inversion Principle](http://en.wikipedia.org/wiki/Dependency_inversion_principle)**)，木有循环依赖，既方便mock，也方便组装，golang都是以组合方式来构建，面向对象编程语言 java/c++ 通过接口/继承来实现多态，golang则没有继承通过接口的方式来实现多态，前者是运行时查找编译时生成的函数表(itable/vtable)进行动态绑定，后者则是编译时指针直接进行绑定(实现接口分指针类型和结构体类型，后者有参数拷贝操作，开发中使用前者指针类型；编译时非空interface iface中的itab.inter指向对应接口名, itab.\_type 指向对应实例对象以及函数数组指针fun指向对应结构实体中的实现方法；运行时将初始化实例对象分配在堆中，然后将实例化对象指针赋给itab._type, 调用对应实体的函数方法时在分配的栈空间中进行运行；通过 [go tool complie](https://golang.google.cn/src/cmd/compile/doc.go?h=go+tool+compile) 在对应硬件平台查看编译的Plan9汇编代码为准)，以组合的方式构建项目(原则: **任何构造函数都不应调用另一个构造函数**, 在程序初始化时进行构造注入实体)，以防类继承过度抽象设计；interface相关可参考：[golang-oop-tutorial](https://www.toptal.com/go/golang-oop-tutorial) [go-interface](https://halfrost.com/go_interface/) [golang-assembly](https://github.com/cch123/golang-notes/blob/master/assembly.md) **[go-internals-interfaces](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/README.md)** **[generics-can-make-your-go-code-slower](https://planetscale.com/blog/generics-can-make-your-go-code-slower)**

 尽量使用工具来生成规范化，结构化的代码; 常用工具如下：

1. [gorm-gen](https://gorm.io/gen/index.html) diy生成数据库持久访问层模型,可以用sql来推演业务逻辑，然后直接生成对应dao操作model/entry读写方法，提供给上层repository 进行接口实例化注入，如果业务逻辑非简单的CURD, 提供一层usecase来分离业务具体实现，以及组合repository来实现具体业务场景逻辑，实现 domain领域驱动 满足 业务对内对外的api接口 以及 事件驱动；
2.  [wire ](https://github.com/google/wire/blob/main/docs/guide.md)依赖注入，将db/cache/mq/rpc/http client -> dao, repository  -> usecase -> api/subscribe handler  -> service  => server 依次注入实体组装成server，提供服务；
3. [GoMock mockgen](https://github.com/golang/mock)(golang官方出品)，可以结合 [mockery](github.com/vektra/mockery) (相对开发友好)工具来mock 接口 ， 这样在前期不用实现具体场景实体类，直接可以测试驱动(BDD/TDD)进行业务逻辑推演；可以认为前期搭开发框架架子，不仅需要满足现在需求，而且需要对后续易变需求可进行扩展开发，无需改动以往逻辑(业务逻辑实体合理抽象以及对应方法接口)；也可以使用hack的方式 通过 [gomonkey](https://bou.ke/blog/monkey-patching-in-go/) 进行打桩注入进行mock，虽然简单直接，但是依赖底层编译硬件平台架构,有限制不推荐；
4. [ginkgo](https://onsi.github.io/ginkgo/) 编写公有函数测试用例，覆盖业务逻辑分支

总之保持 **KISS** 姿势，让工具来解放生产力（好的工具可以规范化写代码流程, 前提是工程最佳实践沉淀）。

> Keep it simple stupid “K.I.S.S”

附：[google golang style](https://google.github.io/styleguide/go/)  , [Deprecation notices in Go](https://rakyll.org/deprecated/) ,  [GOMM](https://research.swtch.com/gomm) , [Go Memory Allocator](https://medium.com/@ankur_anand/a-visual-guide-to-golang-memory-allocator-from-ground-up-e132258453ed) , [gc-guide](https://tip.golang.org/doc/gc-guide) , [garbage-collection-in-go](https://www.ardanlabs.com/blog/2018/12/garbage-collection-in-go-part1-semantics.html) , [scheduling-in-go](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html) , [channel-closing](https://go101.org/article/channel-closing.html) , [Understanding Channels](https://www.google.com.hk/search?q=kavya%20golang#fpstate=ive&vld=cid:089b5108,vid:KBZlN0izeiY)



### 测试

基本功能逻辑测试，并发场景下的扣减和新增数据一致；然后加上 业务/基础服务/中间件服务监控，日志，在开始压测，给出性能报告，调优；最终给出接口/整体服务吞吐和延时上限，采用服务流控对服务接口加上接口/服务限流，以及超时重试，服务相关降级策略；然后模拟线上场景继续压测，触发对应报警和限流，降级策略；

1. 模块测试, 主要是 pkg中的基础函数benchmark测试，以及internal中的业务逻辑测试

2. 加上服务监控，日志，报警，重点关注核心链路指标：资源U.S.E, 应用R.E.D; 参考: [**Monitoring SRE’s Golden Signals**](http://www.infoq.com/articles/monitoring-SRE-golden-signals), [monitoring_golden-signals L.E.T.S](https://sre.google/sre-book/monitoring-distributed-systems/#xref_monitoring_golden-signals)

3. 单个接口压测, 主要是接口中操作数据中心db的读写吞吐，以及引入缓存后，进行异步处理的读写吞吐

4. 核心前端接口整体压测, 需要流量和数据存储环境隔离，整体通过服务治理框架进行流控，打上stress标签染色，请求流量和数据流通过stress标签将数据存放于影子逻辑存储(cache KV, 表/集合/索引，队列)中; 参考：[Rhino](https://mp.weixin.qq.com/s/vofrpFGvnptj3MNAv1hQ-w) [Quake](https://tech.meituan.com/2018/09/27/quake-introduction.html)

5. 混沌测试，尽量覆盖触发边界场景，面向故障编程，测试; 参考：[混沌工程实践](https://mp.weixin.qq.com/s/kZ_sDdrbc-_trVLNCWXyYw)

6. 业务功能调优：优先业务功能大方向调优，分布式数据库下的查询语句，算子下推，索引效率 (空间换时间，k/v 操作 B+tree,LSM-tree)

7. Profiling 调优：通过相关trace/perf工具关注耗时，使用系统内核优化的调用函数，以及系统参数调整，内存(减少gc，频繁操作小对象池化复用，内存分配管理，局部性原则)，cpu(亲和性)，I/O(网络，磁盘I/O 尽量异步处理,) ，buff(缓冲，batch)等

   tips: 业务整体设计方案ok的情况下，同机房整体性能优化收益： 硬件(cpu/GPU浮点运算/TPU，内存/nvm，磁盘/ssd，网卡带宽拆解包校验) > 内核系统调用 > 业务代码(数据结构,语言层面编译优化opcode)

   

## 总结

面对大量并发请求的用户交互场景，涉及到用户金额等事务场景，整体思路是预热快速路径响应用户的交互请求行为，然后异步队列解耦，执行慢路径，慢路径上通过本地事务(同服务多表更新)，或者分布式事务(多服务对应表更新)，来保证数据整体一致，尽量通过批量处理(单分片事务组提交)提高吞吐；快路径尽量使用原子操作(服务主逻辑单线程执行)或者乐观锁用户程序自旋执行事务，按用户维度切分减少block, 以及通过提前规划分布式存储均匀打散读写热点(分库分表分区分片/槽Region/Slot)，规避节点负载倾斜不均衡问题(大流量,大数据),多维度查询的话可以采用数据异构方式(空间换时间)，如果涉及到跨数据中心则需要考虑数据同构迁移问题(比如国际化数据隐私*PIPL* *GDPR*)；对于数据并发同步获取至缓存，则需要加分布式互斥锁自旋block一段时间，防止并发操作加锁尽量前置处理(warm up to run fast~),减少block；本质上都是将并行变成串行，只不过内存操作比磁盘,网络IO操作要快很多(理论上，lua脚本在master主线程内存中原子执行相对watch+事务方式处理效率要高些)；还有服务异常时的降级措施，以及服务过载时的限流，以及消息数据流的反压措施。

题外话：没有银弹，充分利用开源(工程规范) ☁️红利(硬件计算存储资源)； *纸上得来终觉浅，绝知此事要躬行*， step by step, maybe day day up~  hope expand your horizons



## 参考

1. https://mp.weixin.qq.com/s/Cmw3QExqCfBAz9V0AlsS9A

2. https://aws.amazon.com/cn/builders-library/timeouts-retries-and-backoff-with-jitter/

3. https://mp.weixin.qq.com/s/jznfR9Jc-U-uCXioHXjeew , https://mp.weixin.qq.com/s/AV4E0Y9d4k5VYTL7n2TNug , https://mp.weixin.qq.com/s/cT9b2GDsUinVNoA6gyqs_g

4. http://www.52im.net/thread-3515-1-1.html#26 , http://www.52im.net/thread-3994-1-1.html , http://www.52im.net/thread-3376-1-1.html

5. https://coolshell.cn/articles/8239.html, https://time.geekbang.org/column/article/4050

6. **https://redis.io/docs/manual/transactions/ , https://redis.io/docs/manual/programmability/ , https://redis.io/docs/reference/cluster-spec/ , https://redis.com/blog/redis-clustering-best-practices-with-keys/**

7. https://martinfowler.com/articles/serverless.html  https://jimmysong.io/kubernetes-handbook/usecases/serverless.html , https://www.youtube.com/playlist?list=PL2yQDdvlhXf_lYR5Ntvr9V5iVYv5rcbNc

8. https://github.com/apache/incubator-kvrocks/wiki/Kvrocks-%E9%9B%86%E7%BE%A4%E6%96%B9%E6%A1%88%E7%AE%80%E4%BB%8B  https://www.qin.news/kvrocks-qian-xi/

9. **https://tidb.net/blog/09cc69f4 , https://cn.pingcap.com/best-practice-detail/best-practices-for-developing-applications-with-tidb** 

10. **https://mp.weixin.qq.com/s/UFWULymSblrs1hIGP5e7YQ , https://github.com/apache/rocketmq/wiki/RIP-50-RocketMQ-Transaction-Message-Improvement** , **https://xie.infoq.cn/article/ad16bac16b4c172e268225cfa**

11. https://www.youtube.com/playlist?list=PLy7NrYWoggjziYQIDorlXjTvvwweTYoNC , https://www.youtube.com/watch?v=vgPFzblBh7w&list=PL8t1FdN2Tj3ZVAzTY-FvsS0qy-mEfRdoj&index=5 , https://www.youtube.com/watch?v=-U9E1PhrM3o

12. **https://tonybai.com/2022/08/15/developing-kubernetes-operators-in-go-part1/**

13. **https://github.com/istio/istio/tree/master/samples/bookinfo , https://github.com/cloudwego/biz-demo**

14. https://mp.weixin.qq.com/s/n4eVf2KZRIp2yKACk88qJA
