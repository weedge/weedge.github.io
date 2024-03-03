---
title: "golang学习笔记"
date: 2017-11-01T01:16:30+08:00
tags: [
	"Golang",
]
categories: [
	"技术",
	"Golang",
]
lastmod: 2018-10-03T00:26:23+08:00
---

刚开始接触golang的时候是在大学时候，当时大概是2010年左右，对这门新语言比较好奇，但是没有深入去了解，只是道听途说这门语言在并发处理上很方便，对于协程这个东西也是第一次听说。自从工作之后，就一直没有接触过这门语言,最近公司想往golang上转，开发新的项目平台，毕竟golang运行效率和开发效率都要比其他语言要简单方便很多(听说c++都快20了)。

<!--more-->
吉祥物：（憨厚小地鼠 move~move~move~ Bui~~~~    go~go~go~ ）

![地鼠][1]

（萌萌の勤劳小地鼠~）并发协作：

![并发协作][2]

## 语言
看七周七语言中提到的，语言各有特色，用于不同的开发场景，对应团队的工具栈；经常用自己熟悉的编程语言，思维模式也会有所不同；总的来说，支持KISS原则，方便高效开发/测试/部署/运行，并发运行解决问题，流出更多时间关注业务或者`kick ass && Chew bubblegum (just a joke~)`

### 发展历史：
> 发明这个语言的初衷，「不忘初心，方得始终」

- 为毛需要这个语言：https://www.oschina.net/translate/go-at-google-language-design-in-the-service-of-software-engineering
- 初心：简单,开发简洁，方便快速开发编译发布(开发语言的作者来自c和java, 作者Rob Pike实现并发思想,布道者，他相关的视频和文章需要多看，精髓哦，This guy is a genius~ )
- 源码：https://github.com/golang/go
- 特殊节点：2008 6.6 ~1.5版本去除了c代码实现自举(1.5+直接用go来编译安装即可)， now 10年+ golang2.0 版  https://github.com/golang/go/wiki/Go2
- Hello,Gopher（golang作者之一Rob Pike 讲的golang发展史，及众家语言之所长，发明这个语言的初衷）:https://www.youtube.com/watch?v=VoS7DsT1rdM&list=PL3NQHgGj2vtsJkK6ZyTzogNUTqe4nFSWd [slide](https://talks.golang.org/2014/hellogophers.slide##12)
-  Go Concurrency Patterns(一些并发模式case, goroutine通过channel组合传递共享数据, no race like DAG): https://www.youtube.com/watch?v=f6kdp27TYZs&list=PL3NQHgGj2vtsJkK6ZyTzogNUTqe4nFSWd  [slide](https://talks.golang.org/2012/concurrency.slide)
- Concurrency Is Not Parallelism: https://blog.golang.org/concurrency-is-not-parallelism
- Simplicity is Complicated: https://www.youtube.com/watch?v=rFejpH_tAHM&list=PL3NQHgGj2vtsJkK6ZyTzogNUTqe4nFSWd [slide](https://talks.golang.org/2015/simplicity-is-complicated.slide#1)
- The Design of the Go Assembler：https://www.youtube.com/watch?v=KINIAgRpkDA&list=PL3NQHgGj2vtsJkK6ZyTzogNUTqe4nFSWd [slide](https://talks.golang.org/2016/asm.slide#1)



#### 特征：
* KISS；简单朴实 （做工程的人最喜欢的就是简单实用的东西）；
* 运行占用资源低(可运行在嵌入式平台linux arm)，跨平台
* ((内存)数据结构+算法+抽象定义接口)；
* 语法简单，25个关键字；
* c语言结构体形式模拟继承方法重写机制；
* 基础数据结构类型：byte,string, array; slice, map,chanel 引用类型；(其他数据结构heap(优先队列),list,ring(circular list)这些在官方提供的公共库中)
* 强类型语言，interface{} 表示通用类型(相当于stl中的模板)；
* 异常处理: defer, panic, recover 抛出一个panic的异常（异常指的是意料之外的情况，比如引用了空指针，下标越界，除数为0，不应该出现的分支，比如default），然后在defer中通过recover捕获这个异常，最后正常处理；注意如果没有recover去捕获异常，程序就会退出；
* 错误机制: error，错误指的是意料之中的情况，比如文件可能会打开失败，网络连接可能断开导致连接失败等等，错误是业务过程的一部分，而异常不是 (业务需定义常见的错误码标识对应错误信息)
* 变量对象内存分配通过tcmalloc来申请一块span, 分成多个page提供使用，充分利用内存空间，通过gc回收runtime对象(随着版本迭代，对gc的效率逐渐提高，有专门的runtime团队支持)
* 并发程序通过channel共享内存(share memory by communicating)；
* 适合web后端开发(官方golang.org已提供的基础库 https://golang.org/pkg/ ，net/http(请求，响应，client，server), http/template, 数据库客户端驱动接口定义database/sql(三方根据接口标准可以自定义实现，也可以自己单独定义一套标准实现)，数据加密crypto）；
* 通过goroutine(协程，用户级别轻量级线程)支持并发， 协程通过channel通信( <-chan interface{} 接收chan；chan<-interface{} 发送chan；同步通过sync机制(lock(锁接口) mutex(某个goroutine加互斥锁) rwlock(对读写操作加锁) WaitGroup(pv操作,对goroutine组) ，Cond(条件变量，信号量))；
* Golang实现了 CSP(Communicating Sequential Process http://www.usingcsp.com/cspbook.pdf )并发模型做为并发基础，底层使用goroutine做为并发实体，goroutine非常轻量级可以创建几十万个实体；实体间通过 channel 继续匿名消息传递使之解耦，在语言层面实现了自动调度，这样屏蔽了很多内部细节，对外提供简单的语法关键字(go,select,channel)，大大简化了并发编程的思维转换和管理线程的复杂性；
* SO OPEN，import 三方库源(大部分git的方式来自github/gitlib，还支持hg)，源码直接可以修改编译(取之于源用之源)；
* Go 命令工具方便开发自测调试： https://golang.org/cmd/go/
    * 通过go test来单元测试，覆盖测试，性能测试(cpu,pprof)，直接可以测试用例先行，
    * go tool,  pprof (进行性能分析 http://cizixs.com/2017/09/11/profiling-golang-program ）objdump(dump出汇编代码）
    * go get -u ~=  download to go build and install , go run 单独运行
    * go fmt 代码格式化，统一编码风格，方便协同编程
    * go mod: https://github.com/golang/go/wiki/Modules 1.11新引入的功能，方便package依赖管理，类似nodejs中的npm
* Go 2.0正待发布，10年+发展，会有一些优化和新特性  https://www.youtube.com/watch?v=RIvL2ONhFBI
* More…..

#### 工作上使用语言进化

![coder][3]

> C/C++ -> \*.so -> php/python  |   java -> \*.jar  ==>>  go -> \*.a -import-> pkg -compile->  binary program  

编写运行：复杂  ->  简单

Go 相当于兼顾了c的执行效率，python的开发效率，择中方案，适合服务端开发(业务开发，基础组件开发(网关，db，AI)，自动化运维工具开发devops);
如果用go来调用.so动态链接库，需要用cgo机制

语法对比参考：http://hyperpolyglot.org/c

## 学习
0. 开始入门：https://golang.org/doc/code.html  
1. golang的基本语法知识，可以通过gotour来进行简单的学习入门，godoc查看具体基础文档和get到本地的三方库接口文档，
    Gotour简单入门 ——> [语法快速入门](https://github.com/qyuhen/book/blob/master/Go%20%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%20%E7%AC%AC%E5%9B%9B%E7%89%88.pdf)
2. 练习习题：
  - [Go语言经典笔试题](https://goquiz.github.io/)
  - [Go实例](https://gobyexample.com)
3. 最佳实践：(这个会根据实践场景逐渐完善，最终可能会形成一个标准规范)
  - [12条最佳实践](https://talks.golang.org/2013/bestpractices.slide) [中文解释](http://maiyang.me/2017/10/21/twelve-best-practices-in-golang/)
4. 面向对象设计SOLID：
  - [SOLID](https://blog.gokit.info/post/go-solid-design/)
5. 业务常用依赖阶级开发：(掌握一些三方库，方便快速集成开发)
  - Web开发框架：
     - beego https://github.com/astaxie/beego  支持mvc, orm; 适用于cms，mis管理后台开始开发 （重量点，开发框架已经搭好，基础web开发组件已有，RBAC规范后台权限）
       - gin https://github.com/gin-gonic/gin 利用httprouter实现高性能路由 (轻量点，适合diy)
  - 数据库 （client driver）
     - msyql: https://github.com/go-sql-driver/mysql 
       - Redis: https://github.com/garyburd/redigo
       - Mongo: https://github.com/go-mgo/mgo
  - ZK client: https://github.com/samuel/go-zookeeper/zk
  - Grpc: https://github.com/grpc/grpc-go
  - Log: https://github.com/sirupsen/logrus (以hook机制添加自定的日志，lograte:https://github.com/sirupsen/logrus##rotation，或者https://github.com/natefinch/lumberjack)
  - 队列: https://github.com/nsqio/nsq 
6. 订阅资源blog:
  - https://blog.golang.org  
  - https://www.goinggo.net  
  - https://gocn.io/  
7. 实践踩的坑：
  - https://zhuanlan.zhihu.com/p/29545675
  - https://zhuanlan.zhihu.com/p/31395716 (channel,引发的死锁）
8. 应用场景：
  - [新闻实时推送](https://zhuanlan.zhihu.com/p/26777189)
  - More:  DIY （u can do it）


9. 好的开源项目：
  - [golang-open-source-projects](https://github.com/hackstoic/golang-open-source-projects)
  - [awesome-go](https://github.com/avelino/awesome-go) (full-stack dev for venture company )
  - Web server服务：
     - Caddy：类似nginx,apache,middleware chain(责任链模式) 管理方式，独立的mw也可拆分单独运行，比如负载均衡模块，反向代理，fastcgi代理 https://github.com/mholt/caddy
  - 网关：
     - krakend: https://github.com/devopsfaith/krakend  （高性能，相比kong...)
  - 微服务：
     - go-micro (https://github.com/micro/go-micro  +  plugins: https://github.com/micro/go-plugins  )
       - Go-kit (https://github.com/go-kit/kit  very nice micro-service lib framework) diy 借鉴 相关介绍：https://www.youtube.com/watch?v=NX0sHF8ZZgw
  - 服务发现：
     - etcd: https://github.com/coreos/etcd (功能介绍: http://www.infoq.com/cn/articles/etcd-interpretation-application-scenario-implement-principle)
     - Consul: https://github.com/hashicorp/consul
  - Rpc：
     - grpc: https://github.com/grpc/grpc-go
     - thrift: https://github.com/apache/thrift/tree/master/lib/go
  - Web开发框架： 
     - beego(支持mvc模式开发,https://github.com/astaxie/beego)，
     - gin(高性能路由请求处理,https://github.com/gin-gonic/gin )
  - 数据库服务：
     - Tidb (NewSQL 数据库，水平扩展，强一致，分布式事务，多种数据引擎选择，mysql协议通信 https://github.com/pingcap/tidb 国内公司用的比较多）
     - Cockroachdb  (NewSQL 数据库，有百度的同学提交维护 https://github.com/cockroachdb/cockroach ）
     - Influxdb (时序数据库，应用于监控日志数据存储，提供实时数据展现，https://github.com/influxdata/influxdb）
     - Bboltdb (本地嵌入式数据库 , LMDB的go实现版本,https://github.com/coreos/bbolt)
  - 推送：
     - Gorush: https://github.com/appleboy/gorush
  - 爬虫：
     - Gocolly: https://github.com/gocolly/colly
     - Gocrawl: https://github.com/PuerkitoBio/gocrawl
  - 文件：
     - Upspin: https://github.com/upspin/upspin (Rob Pike 主导的一个实验项目，https://www.youtube.com/watch?v=ENLWEfi0Tkg)
  - 链路跟踪：
     - Opentracing: https://github.com/opentracing/opentracing-go  结合zipkin
  - 实时消息队列平台：
     - NSQ: https://github.com/nsqio/nsq 
  - 监控:
    - grafana：https://github.com/grafana/grafana
    - prometheus: https://github.com/prometheus/prometheus
  - 容器管理：
     - Kubernetes(K8S): https://github.com/kubernetes/kubernetes 
     - Kubernetes Handbook: https://jimmysong.io/kubernetes-handbook/
  - 代码仓库服务：
     - gogs (轻量级的代码工具，可以运行在pi上, 界面和github相似, https://github.com/gogits/gogs ）
  - CI/CD(Continuous Delivery):
     - Drone: https://github.com/drone/drone （可配套自研上线运维平台）
     - jenkins-x: https://github.com/jenkins-x/jx
  - 工具：
     - Gotty: https://github.com/yudai/gotty （webshell)
  - 静态网页生成器：
     - hugo: https://github.com/gohugoio/hugo (可以用来写个人技术博客)
  - 数据科学(cheat sheet)： https://www.cheatography.com/chewxy/cheat-sheets/data-science-in-go-a/pdf/

## 深入理解

![goroutine实现机制][4]

* goroutine实现机制：https://www.zhihu.com/question/20862617 （M:内核OS线程；G:goroutine用户态轻量级线程，自己的栈，正在等待的channel等；P: 调度的上下文，是调度协调的处理器，P的数量代表了真正的并发度)
        （tips：相对于java中并发库实现是不是要简洁高效些呢，https://golang.org/doc/faq##goroutines  这个解释了golang为什么用协成代替线程）

* 同步机制：sync lock(锁接口) mutex(某个goroutine加互斥锁) rwlock(对读写操作加锁) WaitGroup(pv操作,对goroutine组) ，Cond(条件变量，信号量) 用法参考：https://deepzz.com/post/golang-sync-package-usage.html  (其实就是OS中的同步机制实现)   锁的话是一个原子操作，要么锁住，要么不做，通过atomic操作底层的cpu指令(CHANGE\*, SWAP\* ....etc指令根据不同的cpu/gpu厂商定义，intel/amd)

* 内存管理： http://legendtkl.com/2017/04/02/golang-alloc/

* GC机制：http://legendtkl.com/2017/04/28/golang-gc/

* 深入了解golang的执行原理，https://github.com/qyuhen/book/blob/master/Go%201.5%20%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90.pdf (底层的运行时逻辑有点像php的执行过程, php7底层运行结构也借鉴了golang的设计思想，比如内存的申请)

## FAQ:

https://golang.org/doc/faq  相当有用值得一读

热更新方案： https://studygolang.com/topics/1194  (监控一个当前版本文件是否变动了，如果变动然后将老的执行文件软连指向新的执行文件(rm 软连,ln 创建新的软连，然后通过hup信号优雅的重启由endless启动的服务)


## 开发工具

- vim-go: https://github.com/fatih/vim-go
- vscode-go: https://github.com/Microsoft/vscode-go
- goland: https://www.jetbrains.com/go/

## DIY

(熟悉三方源码，基础库，自行diy服务)

- [ ] 网关服务
- [ ] 网关后台
- [ ] abtest
- [ ] abtest后台
- [ ] 微服务
  - [ ] passport
  - [ ] activity
  - More…
- [ ] 资源管理(K8S)
- [ ] 监控
- [ ] ci/cd构建发布上线
- [ ] 队列


[1]: https://raw.githubusercontent.com/weedge/weedge.github.io/3368e1e7132b25ac24459dd137ac57e7a3c0fe2e/image/go-mascot.png
[2]: https://raw.githubusercontent.com/weedge/weedge.github.io/3368e1e7132b25ac24459dd137ac57e7a3c0fe2e/image/go-concurrence.png
[3]: https://github.com/weedge/weedge.github.io/blob/3368e1e7132b25ac24459dd137ac57e7a3c0fe2e/image/coder.jpg?raw=true
[4]: https://raw.githubusercontent.com/weedge/weedge.github.io/3368e1e7132b25ac24459dd137ac57e7a3c0fe2e/image/goroutine.jpg

