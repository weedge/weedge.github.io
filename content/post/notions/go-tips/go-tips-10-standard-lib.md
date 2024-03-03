---
author: "weedge"
title: "Go tips-笔记: 标准库 75-81 mistakes"
date: 2023-02-19T14:26:23+08:00
tags: [
	"Golang"
]
categories: [
	"技术",
    "Golang"
]

---



## 引言

从程序中产生的错误中大多是使用姿势的不对，以及没有仔细阅读标准库相关包的开发文档，未查看源码导致，但是没有实践过这些问题，即使熟读文档和源码也可能避免不了。笔记中会以书中的mistake为切入点，结合源码升入分析其背后产生的原因，以及提出解决方案来避免。

<!--more-->

## 笔记

### 75.提供错误的持续时间

记住使用`time.Duration`API 和提供`int64`一个时间单位, 默认最小时间单位是微妙

### 76.time.After 和内存泄漏 （重要）

常见问题之一，将time.After函数进行循环调用，导致内存泄露。

```go
for {
		select {
		case event := <-ch:
			handle(event)
		case <-time.After(time.Hour):
			log.Println("warning: no messages received")
		}
	}
```

通过time.After源码可以看出，每次会通过time.NewTimer新建一个timer, 但是time.After返回的是一个C ← chan Time 只读channel，不能释放掉每次新建的timer, 可以使用Stop，如果一直循环使用，Go 1.15 中，每次调用使用大约 200 字节的内存，如果设置的时间间隔小，比如每小时500w条，则在一小时消耗1G左右的内存空间。

那如果直接使用time.NewTimer来处理，需要处理好Stop和Reset的情况：

一种方式是直接每次循环中NewTimer, 然后使用Stop方法从最小堆timer数组中删除底层的运行时timer(如果timer 没有expire 到期 以及有复用timer reuse active timer)，这样可以防止内存泄露，但是这些timer结构对象需要GC来标记扫描释放，带来了额外的GC压力以及最小堆timer管理压力；这里需要注意Stop方法的使用，按照 [Timer.Stop 文档](https://golang.org/pkg/time/#Timer.Stop) 的使用说明，每次调用 Stop 后需要判断返回值，如果返回 false（表示 Stop 失败，Timer 已经在 Stop 前到期）则需要排掉（drain）一次 channel 中的Time数据(C 是长度为1的缓冲channel)：

```go
if !t.Stop() {
	<-t.C
}
```

但是如果之前程序已经从 channel 中接收过事件，那么上述 `<-t.C` 就会发生阻塞。可能的解决办法是借助 select 进行 **非阻塞** 排放（draining）：

```go
if !t.Stop() {
	select {
	case <-t.C: // try to drain the channel
	default:
	}
}
```

但是因为 channel 的发送和接收发生在不同的 goroutine，所以 [存在竞争条件](https://github.com/golang/go/issues/14383)（race condition），最终可能导致 channel 中的事件未被排掉。因为sendTimer 和 操作Stop函数是在两个goroutine中执行，当timer刚好到期，已从最小堆中删除，操作Stop函数返回false,  在执行 ←t.C 接受操作和 sendTime 发送操作 分别在两个goroutine中执行相互之间执行是无序的，可能会发生先从t.C接受数据，没有，由于是非阻塞继续执行，这个时候sendTime发送一条Time数据到C中，后面执行Reset虽然重置一个Timer, 但是在select + case ←timer.C时，C中有数据选中直接执行了，和通过Reset重置的一个Timer间隔时间执行的预期期望不同，这样存在race condition，但是这种情况出现机率比较低，可参考 [Russ Cox 的回复](https://github.com/golang/go/issues/11513#issuecomment-157062583) ，目前 Timer 可能合理的使用方式是：程序需要维护一个状态变量(在同一个goroutine中)，用于记录它是否已经从 channel 中接收过事件，进而作为 Stop 中 draining 操作的判断依据。可以订阅[golang-dev](https://groups.google.com/g/golang-dev/c/c9UUfASVPoU)组查看相关进展。

另外一种方式是把 NewTimer 放在循环外，在for循环中通过Reset函数来复用原有Timer结构，按照 [Timer.Reset 文档](https://golang.org/pkg/time/#Timer.Reset) 的使用说明，要正确地 Reset Timer，首先需要正确地 Stop Timer；因此 Reset 的问题跟 Stop 基本相同。

tips： 具体详情见源码客观分析：

[go1.20/src/time/sleep.go](https://github.com/golang/go/blob/go1.20/src/time/sleep.go) (time标准中提供使用的Timer)

[go1.20/src/runtime/time.go](https://github.com/golang/go/blob/go1.20/src/runtime/time.go) (运行时的timer)

[go1.20/src/runtime/runtime2.go](https://github.com/golang/go/blob/go1.20/src/runtime/runtime2.go) , [go1.20/src/runtime/proc.go](https://github.com/golang/go/blob/go1.20/src/runtime/proc.go)(p结构上最小四叉堆 timer数组, 以及运行时相关timer的调度；调用流程：findRunnable/stealWork → checkTimers → runtimer → runOneTimer → f (sendTime or goFunc) , lock free的方式调用f, CAS原子操作timer的状态)

每次新生成Timer的时候，会往p上的最小堆上添加timer(O(logN))，将等待可读事件放入netpoll异步事件中监听，netpoll是在程序启动时初始化绑定一个单独的M进行事件轮训；Go1.14之前使用timerproc函数会调用一些系统调用来来让 goroutine 进入睡眠状态并唤醒 goroutine，系统调用意味着它为此生成OS线程，如果创建timer比较多，那就会发生比较多的系统调用，大大降低性能；之后改成异步事件轮训机制netpoll的方式多路复用，只需要一个OS线程来监听事件即可；系统调用因系统平台而异，通过runtime.nanotime1函数进行了封装；

如果时间到了，将最小堆顶timer删除(O(logN))，通过netpoll 异步事件机制 将 可执行的G调度到runnext中，然后绑定M运行f；

### 77.常见的 JSON 处理错误 (重要)

#### case1 类型嵌入导致的意外行为

需要了解json.Marshal 方法，在对结构类型对象进行Marshal操作时，如果实现了json.Marshaler接口方法MarshalJSON， 则会调用对应MarshalJSON方法进行encode操作，可以看具体源码： [go1.20/src/encoding/json/encode.go](https://github.com/golang/go/blob/go1.20/src/runtime/proc.go) （调用流程：Marshal→ marshal → reflectValue → valueEncoder → typeEncoder → newTypeEncoder → marshalerEncoder → MarshalJSON)  ; 所以在json.Marshal操作的时候需要注意结构体的嵌入成员是否实现了json.Marshaler接口方法MarshalJSON， 比如： time.Time 实现了MarshalJSON这个方法； 如果不想直接使用组合嵌入成员的方法，则将其定义为对应类型成员，或者实现MarshalJSON方法覆盖嵌入成员的实现；

tips: 对结构类型对象进行UnMarshal操作也是同样情况。

#### case2 JSON 和单调时钟

对包含一个time.Time类型的结构encode或decode，有时会遇到意想不到的比较错误。

首先需要弄清楚操作系统处理两种不同的时钟类型：wall clock(挂钟)和 monotonic clock(单调时钟)。挂钟用于确定一天中的当前时间。此时钟可能会有所变化。例如，如果时钟使用同步网络时间协议 (NTP)，它可以及时向后或向前跳转。不应该使用挂钟测量持续时间，因为可能会遇到奇怪的行为，例如负持续时间(润秒重置的情况)。这就是操作系统提供第二种时钟类型的原因：单调时钟。单调时钟保证时间总是向前移动并且不受时间跳跃的影响。它可能会受到频率调整的影响（例如，如果服务器检测到本地石英钟的移动速度与 NTP 服务器不同），但不会受到时间跳跃的影响。

以前Go Time包的相关时间读取函数实现仅读取系统挂钟，从不读取单调时钟，从而在时钟重置时导致测量不正确。比如 一个 Go 程序在闰秒期间测量负的经过时间导致[CloudFlare 最近的 DNS 中断](https://blog.cloudflare.com/how-and-why-the-leap-second-affected-cloudflare-dns/). 维基百科[与闰秒相关的问题示例列表](https://en.wikipedia.org/wiki/Leap_second#Examples_of_problems_associated_with_the_leap_second)现在包括 CloudFlare 的中断，并将 Go 的时间 API 列为根本原因。除了闰秒问题之外，Go 还扩展到非生产环境中的系统，这些环境中的时钟可能不太好调节，因此时钟重置更频繁。Go 必须优雅地处理时钟重置。Go语言作者Russ Cox提出了提案设计**[Proposal: Monotonic Elapsed Time Measurements in Go](https://github.com/golang/proposal/blob/master/design/12914-monotonic.md)**  (golang的开发规范和提案设计文档值得借鉴学习的，背景原因，验证评估影响面，尽量向前兼容，提案通过，再安排开发计划)； 将monotonic clock 单调时钟引入time.Time结构体中，具体CR: https://go-review.googlesource.com/c/go/+/36255 ， HN也有对应讨论： https://news.ycombinator.com/item?id=13566110 ;

tips： 测量持续时间，使用单调时钟；仅对**本地持续时间测量**有效；两个不同服务器的单调时钟根据定义是不同步的。因此，基于这些时钟测量分布式执行将不准确；这就涉及到分布式时钟同步的问题了。

ok了解了背景，回归正题，比如对一个结构体有time.Time类型成员，time.Time可能同时包含一个挂钟和一个单调时间，使用time.Now方法返回的时间就包括挂钟读数和单调时钟读数，具体见time包开发文档：https://pkg.go.dev/time#section-documentation; 以及查看源码客观分析：[go1.20/src/time/time.go](https://github.com/golang/go/blob/go1.20/src/time/time.go) （Now→time.now→runtime.now in assembly → 如果可以使用vdso 调用 runtime·vdsoClockgettimeSym 减少系统调用提升性能，否则执行系统调用SYSCALL SYS_clock_gettime(228) 指令，见：**[linux系统调用指令集](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl)**）。time.Now返回的Duration值打印如下：

```go
2023-02-19 15:37:08.218505 +0800 CST m=+0.000118444
------------------------------------ --------------
             Wall time               Monotonic time
```

在执行json.UnMarshal 解码包涵time.Time类型公开成员结构体进行格式化解析时， 也会调用time.Time的UnMarshalJSON函数，最终会调用Time.stripMono, 去掉Monotonic time；导致前后结构体对象不一致，一个从time.Now中返回有Monotonic time，解析后的没有；通过Time.Truncate方法，去掉Monotonic time，可以解决；需要注意带有time.Time的结构体在encode/decode时，前后对象会不一致的情况；

#### case3 any map

any是空接口interface{}的别名，在对map[string]any类型对象进行 json.UnMarshal时，json字符串中的整数类型会解析成默认的float64类型，这样可能会导致数据判断时出现问题，对类型转换做出错误的假设可能会导致 goroutine panic。

### 78.常见的 SQL 错误

该`database/sql`包提供SQL（或类似 SQL）数据库的标准通用接口；依赖具体数据操作，由三方来实现；接口与实现分离的很好例子；

tips：在设计通用中台和平台项目中的模块时，经常需要将抽象与实现分离，驱动化设计，方便具体领域场景的定制化开发。

具体查看开发文档：https://pkg.go.dev/database/sql；在使用这个包时看到一些模式或错误也很常见；深入研究五个常见错误case。

#### case1 忘记 sql.Open 不一定建立到数据库的连接 (工程规范)

Open 可能只是验证其参数而不创建与数据库的连接。要验证数据源名称是否有效，请调用 Ping。在使用的时候，和数据库进行交互的时候才建立连接。比如go-redis [issues-2085](https://github.com/redis/go-redis/issues/2085) , 这个issue是因为使用go-redis v8 版本 通过ping请求访问 7.0 redis redis-cluster， v8版本还不支持新的协议返回的数据导致，需要升级使用go-redis v9版本来支持，所以使用ping功能即可以测试生成有效连接，而且可以验证客户端和服务端协议的一致性。

#### case2 忘记使用连接池 (工程规范)

应为数据库是底层存储数据资源，如果不限制使用有限的底层数据库连接资源，会增加底层数据库服务的负载；需要设置连接池，进行连接复用，以及结合数据库服务能力限制设置最大连接数，具体参数：

- `SetMaxOpenConns`最大限度打开的数据库连接数（默认值`unlimited`）；设置`SetMaxOpenConns`对于生产级应用程序很重要。因为默认值是无限的，应该设置它以确保它适合底层数据库可以处理的内容。
- `SetMaxIdleConns`最大限度空闲连接数（默认值`2`）；如果应用程序生成大量并发请求，则应增加`SetMaxIdleConns`(default: )的值。`2`否则，应用程序可能会经历频繁的重新连接。
- `SetConnMaxIdleTime`最大限度连接关闭前可以空闲的时间量（默认值`unlimited`）；如果应用程序可能面临大量请求，那么设置就很重要。当应用程序返回到更和平的状态时，希望确保创建的连接最终被释放。
- `SetConnMaxLifetime`最大限度连接在关闭之前可以保持打开状态的时间（默认值`unlimited`）；如果连接到负载平衡的数据库服务器，设置会很有帮助。在这种情况下，要确保应用程序永远不会使用连接太久。

如果应用程序面临不同的用例，可以使用多个连接池。这些值需要根据不同环境进行配置，对这些值进行可配置化管理，或者放在配置中心。

#### case3 不使用Prepare语句 (工程规范)

生产环境中，应该使用Prepare对sql 进行预处理，以防sql 注入，并且重复的sql语句不需要重新解析处理。

#### case4 错误处理空值   (工程规范)

在设计数据库表时，如果允许字段为NULL的话；查询这个字段scan row时，需要考虑NULL的情况，如果直接使用类型，则会报错； 解决方法，使用指针类型，以及sql包中封装的类型sql.NullXXX

比指针类型更清楚地表达了意图。

#### case5 不处理行迭代错误  (工程规范)

这是要牢记的最佳实践：因为`rows.Next`可以在遍历所有行或准备下一行时发生错误时停止，所以应该在迭代后使用`rows.Err`进行检查。

### 79.不关闭临时资源

开发者经常在代码中的某个点关闭申请的临时资源，以避免磁盘或内存，连接等资源泄漏。结构通常实现`io.Closer`接口表示必须关闭临时资源。列举3个不关闭临时资源的case:

#### case1 HTTP Response body （重要）

如果使用Go语言编写HTTP协议相关的代码，经常会遇到的问题，忘记关闭返回的http.Response.Body,  导致资源泄露，其实开发文档中已经给出了说明 https://pkg.go.dev/net/http#Response.Body

```go
The http Client and Transport guarantee that Body is always non-nil, even on responses without a body or responses with a zero-length body.  It is the caller's responsibility to close Body.  The default HTTP client's Transport may not reuse HTTP/1. x "keep-alive" TCP connections if the Body is not read to completion and closed.
```

http客户端和传输保证Body总是非空的，即使响应没有Body或者响应的Body长度为零。关闭Body是调用者的责任。如果**Body没有读到完成并且关闭，缺省HTTP客户端的传输(DefaultTransport 默认打开了Keep-Alive)不能复用HTTP/1.x "keep-alive"tcp 连接**。并且查看源码分析：

[go1.20/src/net/http/client.go](https://github.com/golang/go/blob/go1.20/src/net/http/client.go)

[go1.20/src/net/http/transport.go](https://github.com/golang/go/blob/go1.20/src/net/http/transport.go)

[go1.20/src/net/http/transfer.go](https://github.com/golang/go/blob/go1.20/src/net/http/transfer.go) (body Read from bufio Read)

[go1.20/src/bufio/bufio.go](https://github.com/golang/go/blob/go1.20/src/bufio/bufio.go)

调用流程：初始化Client, 调用 Client.Do/do (Get/Post/Head方法NewRequest之后都会调用Do方法）→ Client.send → send →  Transport.RoundTrip 接口方法 →  Transport.roundTrip

→ Transport.getConn → Transport.queueForDial -》 go Transport.dialConnFor → go persistConn.readLoop （将连接响应数据写入transferReader Body中, 发送responseAndError给roundTrip） 和  go persistConn.writeLoop  (往连接中写请求数据,将writeErr结果分别发送一份到writeErrCh中，由readLoop接收处理，发送一份给roundTrip处理)

→ persistConn.roundTrip (发送persistConn.requestAndChan 到 reqch中,用于readLoop接收；发送writeRequest到writech中，由writeLoop 接收；从writeErrCh 处理write错误；从responseAndError chan中处理read错误)

整体过程是一个建立长连接(KeepAlive开启), 并在长连接中通过读写管道和错误结果管道来协同处理，管道是可缓冲的，长度是1个buffer，刚好用于存放一个数据，发送和接收等待管道中的数据进行处理。

在KeepAlive开启的情况下，长连接如果不关闭Response.Body，并且不读取Body中的数据，不会复用原有长连接，通过上面分析，会导致协程泄露；如下代码：

```go
for i := 0; i < 10; i++ {
		fmt.Println("go nums", runtime.NumGoroutine())
		resp, _ := http.Get("<http://www.baidu.com>")
		if resp != nil && resp.Body != nil {
			//_, _ = ioutil.ReadAll(resp.Body)
			//_ = resp.Body.Close()
		}
	}
	fmt.Println("go nums", runtime.NumGoroutine())
```

如果复用的话，这里请求是串行处理，会复用同一个连接，所以只会有3个协程在工作；如果不能复用连接的话,每处理一个请求会新开连接，导致协程泄露。Client不初始化，Transport默认是开启keep-alive；

```go
// DefaultTransport is the default implementation of Transport and is
// used by DefaultClient. It establishes network connections as needed
// and caches them for reuse by subsequent calls. It uses HTTP proxies
// as directed by the $HTTP_PROXY and $NO_PROXY (or $http_proxy and
// $no_proxy) environment variables.
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: defaultTransportDialContext(&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
	}),
	ForceAttemptHTTP2:     true,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}
```

生产环境中，使用tcp连接资源都是需要根据调用 资源服务放的系统负载吞吐能力来配置的。也是需要配置化。

- 如果在没有读取的情况下也没有关闭Body，会发生协程泄露，同时tcp连接也不会复用,本质上是连接资源未释放至连接池中，存在连接泄露。

还需要记住的重要事情是，如开发文档net/http中提到的，当关闭 Response Body时，是否复用连接，这取决于是否从中读取完body中的值：

- 如果在没有读取的情况下关闭Body，虽然不会发生协程泄露，但是默认的 HTTP 传输可能会关闭连接。
- 如果在读取后关闭Body，默认的 HTTP 传输不会关闭连接；因此，它可以重复使用。

所以不管如何，最好的方式是都应该关闭Response Body,  尽管Body没有数据，或者已经读取完了，都应该关闭。

tips: 是否连接复用的判定，可以通过tcpdump 或者 wireshark 来抓包，通过是否使用同一个连接四元组来确定是否复用了同一连接。可以使用类似如下命令：

```bash
tcpdump -i utun2 -tnn dst host www.baidu.com //per host pool
```

#### case2 sql.Rows

`sql.Rows`是用作 SQL 查询结果的结构。因为这个结构实现了`io.Closer`，所以它必须被关闭。忘记关闭行意味着连接泄漏，这会阻止数据库连接被放回连接池。

#### case3 os.File

如果最终没有关闭一个`os.File`，它本身不会导致泄漏：文件将在`os.File`垃圾收集时自动关闭。但是，最好`Close`显式调用，因为不知道下一次 GC 何时会被触发（除非手动运行它）。

总结本节，已经看到关闭临时资源从而避免泄漏的重要性。临时资源必须在正确的时间和特定情况下关闭。事先并不总是清楚什么必须关闭。只能通过仔细阅读 API 文档和/或通过经验来获取这些信息。但是应该记住，如果一个结构实现了`io.Closer`接口，最终必须调用`Close`方法。最后但并非最不重要的一点是，了解如果闭包失败该怎么办非常重要：是否足以记录一条消息，或者是否也应该传播它？适当的操作取决于具体错误err是否需要处理。

### 80.在回复 HTTP 请求后忘记返回语句 （凑数）

如果有适当的覆盖率，这样的问题可以而且应该在测试期间被发现。这个属于err≠nil, 需要check遇到错误不为nil，是否直接return返回。这总低级错误，可以交给测试用例来覆盖到。

### 81.使用默认的 HTTP 客户端和服务器 (工程规范)

在讨论http包的时候提到, 如果不初始化http.Client，Client结构中的RoundTripper会默认使用DefaultTransport, 而DefaultTransport 只能用于开发测试时使用；对于生产环境， 需要更具依赖的资源服务进行配置，保证其配置过大的连接数而超出资源服务的负载能力，以及在网络不稳定情况下，连接超时，读写超时的设定，以便是否重试，这样不会一直hang住连接不释放，并发场景下，会导致服务负载增加， 连接过多导致服务拒绝。所以对于网络tcp请求，都需要根据具体的生产情况进行合理配置，而且是可配置化， 或者引入配置中心动态下发配置。对于服务端的tcp连接配置也是如此，也需要配置读写超时时间，进行可配置化管理。

## 概括

- 对接受`time.Duration`. 即使允许传递整数，也要尽量使用时间 API 来防止任何可能的混淆。
- 避免调用`time.After`重复函数（例如循环或 HTTP 处理程序）可以避免峰值内存消耗。由创建的资源`time.After`只有在定时器到期时才会被释放。
- 在 Go 结构中使用嵌入式字段时要小心。这样做可能会导致偷偷摸摸的错误，例如`time.Time`实现`json .Marshaler`接口的嵌入式字段，从而覆盖默认的封送处理行为。
- 比较两个`time.Time`结构时，回想一下它`time.Time`同时包含一个挂钟和一个单调时钟，并且使用运算符的比较`==`是在两个时钟上完成的。
- 为避免在解组 JSON 数据时提供地图时出现错误假设，请记住`float64`默认情况下会将数字转换为。
- 如果需要测试配置并确保数据库可访问，请调用`Ping`or方法。`PingContext`
- 配置生产级应用程序的数据库连接参数。
- 使用 SQL 预处理语句可以使查询更高效、更安全。
- 使用指针或类型处理表中可为空的列`sql.NullXXX`。
- 调用行后迭代`Err`的方法`sql.Rows`以确保您在准备下一行时没有遗漏任何错误。
- 最终关闭所有实现的结构`io.Closer`以避免可能的泄漏。
- `return`为避免 HTTP 处理程序实现中的意外行为，如果您希望处理程序在 之后停止，请确保您没有错过该语句`http.Error`。
- 对于生产级应用程序，不要使用默认的 HTTP 客户端和服务器实现。这些实现缺少在生产中应该强制执行的超时和行为。