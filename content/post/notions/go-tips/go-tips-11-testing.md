---
author: "weedge"
title: "Go tips-笔记: 测试 82-90 mistakes"
date: 2023-02-20T14:26:23+08:00
tags: [
	"Golang"
]
categories: [
	"技术",
    "Golang"
]



---



## 引言

测试是项目生命周期的一个重要方面。它提供了无数的好处，例如建立对应用程序的信心、充当代码文档以及使重构更容易。与其他一些语言相比，Go 具有强大的编写测试原语。主要讨论测试过程变得脆弱、效率低下和准确性低的常见错误。这类问题属于工程规范实践，有些case同样适用于其他语言。

Go 中提供 go test 工具来执行测试，可以查看具体的开发文档： **https://pkg.go.dev/cmd/go#hdr-Testing_flags**  里面介绍了每个模式的具体使用方式，使用好这些测试模式flag，可以更快执行或更好地发现可能错误，进而保证代码质量，工程代码稳定性建设上的重要一环。Go中支持4种测试函数：单测函数，基准压测函数，模糊测试，以及打印输出样例测试。

```go
func TestXxx(t *testing.T) { ... }
func BenchmarkXxx(b *testing.B) { ... }
func FuzzXxx(f *testing.F) { ... }
func ExampleXxx() {
	Println("The output of\\nthis example.")
	// Output: The output of
	// this example.
}
```

<!--more-->

## 笔记

### 82.不对测试进行分类

功能测试大致分为单元测试，集成测试，以及端到端的测试，单元测试则是程序测试case覆盖率的保障，列举Go中3种常见的测试分类方法

#### Build tags

build tags 的一些使用场景：

- 测试环境使用 mock 服务；而正式环境使用真实数据
- 免费版、专业版和企业版提供不同的功能
- 不同操作系统的兼容性处理。通常用于跨平台，例如 windows，linux，mac 不同兼容处理逻辑。
- go 低版本的兼容处理
- 对测试用例进行分类测试

通过在测试文件中加上对应的测试分类标签，在测试的时候方便对一类tag进行测试，而不需要跑全部测试用例, 比如打上 mock 标签进行用于 mock 一类测试

```bash
//go:build !(mock1 && mock2) || mock3 || !(mock4 && mock5)
// +build !mock1 !mock2 mock3 !mock4 !mock5
```

tips: 这个是为了举个例子，具体测试tag需要因场景逻辑打上，确保测试文件tag无歧义。

从 Go 1.17 开始，语法`//+build foo`被替换为`//go:build foo`. 目前`gofmt`同步两种形式以帮助迁移。

```bash
# 默认仅运行包中tag为空的测试case
go test -v .
# 仅运行包中 mock3 的测试case
go test --tags=mock3 -v .
# 仅运行包中 mock1 && mock2 的测试case
go test --tags=mock1,mock2 -v .
```

#### 环境变量

使用build tag方式构建测试用，随着tag的增加，可能会隐藏掉其中的错误，而且需要去查看测试文件中的tag有哪些。环境变量这种方式是build tags 的补充吧，对于没有设置环境变量的情况下，明确显示哪些测试是跳过的测试，比如

```bash
if os.Getenv("INTEGRATION") != "true" {
		t.Skip("skipping integration test")
	}
```

tips: tag是测试文件粒度分类(Go 构建工具支持)，环境变量是测试代码粒度分类(手动代码逻辑)

#### Short 模式

另一种对测试进行分类的方法与它们的速度有关。可能不得不将短期运行的测试与长期运行的测试区分开来。

作为说明，假设有一组单元测试，其中一个是出了名的慢。想对慢速测试进行分类，这样就不必每次都运行它（尤其是在保存文件后触发），使用testing.Short区分如下：

```go
func TestLongRunning(t *testing.T) {
	if testing.Short() {
		t.Skip("skipping long-running test")
	}
	// ...
}
```

运行测试是，通过-short 参数来执行跳过

```bash
go test -v -short .
```

这三种方式可以组合使用，例如，项目包含长时间运行的单元测试，则使用构建标记或环境变量对测试进行分类（例如，作为单元测试，mock测试，或者集成测试）和使用短模式来跳过长时间运行的测试。

### 83.不启用 -race

对于并发程序代码的测试，需要检测是否存在data race， 需要使用 -race 模式来构建测试，比如

```go
go test -race ./...
```

这种方法使启用竞争检测器，它可以检测代码以捕获潜在的数据竞争(没有打上!race标记的文件)。启用时，它对内存和性能有显着影响，因此必须在特定条件下使用，例如本地测试或 CI。在生产中，应该避免它（或者只在金丝雀发布的情况下使用)。

竞争检测器无法捕捉到误报（明显的数据竞争并不是真实的）。因此，如果收到警告，就知道代码包含数据竞争。相反，它有时会导致漏报（缺少实际的数据竞争）。对于漏报的情况，可以尽可能多的迭代来检测。

### 84.不使用测试执行模式

#### -parallel

默认情况下 go test 在不同的 package 之间是并行执行测试，在每个 package 内部是串行执行测试。如果想要在 package 内部开启并行测试，需要在测试函数中显式执行 t.Parallel() 告诉 go test 这个函数可以与其他测试并行执行，一旦开启并行测试，一定要确保测试函数之间的资源竞争的问题已经得到正确的解决。执行 go test -parallel n 来制定n个测试函数并行执行(需要再显示调用了t.Parrallel)。对于执行慢的测试函数，相互之间没有资源竞争，可以加入t.Parrallel 来同时执行，提高测试的执行速度。

#### -shuffle

从 Go 1.17 开始引入，可以随机化测试和基准测试的执行顺序， 设置为`on`或`off`启用或禁用随机测试；编写测试时的最佳做法是将它们隔离开来。例如，它们不应依赖于执行顺序或共享变量。这些隐藏的依赖关系可能意味着一个可能的测试错误，或者更糟的是，一个在测试期间不会被捕获的错误。

应该对现有的测试标志保持谨慎，并随时了解最新 Go 版本的新功能。运行parallel测试或者将测试分再不同的包中，可以减少运行所有测试的总体执行时间。shuffle测试可以帮助发现隐藏的依赖关系，这些依赖关系可能意味着在以相同顺序运行测试时出现测试错误，甚至不可见的错误。

### 85.不使用表驱动测试

这个在vscode, goland IDE中已经集成了，对应函数生成对应单测函数时，会自动给出表驱动测试模版，编写测试用例，用于覆盖函数分支场景；如果不是用表驱动测试的话， 会出现大量的冗余函数，而且表达含义也会相对模糊，直接放入一个测试函数中来编写用例测试即可，也便于对整个函数的测试覆盖。通过t.Run来执行这些测试用例，进行期望值比较，同时也可以使用t.Parallel() 通过parallel 模式来加速测试，以及通过shuffle来随机测试。

### 86.在单元测试中使用time.Sleep

在测试并发编程时，可能存在竞争条件race condition 的场景，导致程序的执行顺序不同，进入影响测试的准确性，如果使用time.Sleep 之后来断言值， 可能会有不同的结果，是不确定性的，所以，尽量使用管道同步的方式来进行断言测试；如果同步不可能做到的话， 可以重试进行断言，比如常用的testify 测试包，使用Eventually函数实现了最终应该成功的断言，这比使用被动睡眠更好的选择来消除测试中的非确定性。

### 87.没有有效地处理时间 API

对于函数中有time.Now()获取当前时间，而测试是也依赖当前时间的处理，导致那个时间点可能会存在差异，一种方式是提供全局共享变量，如果使用并行测试的，全部共享变量会引入数据竞争，导致无法并行测试，所以最好的方式，修改下所要测试的函数，去掉time.Now()的依赖，使用time.Time类型作为传入参数，有函数使用方一起来定义，这样可方便测试。

### 88.不使用测试实用程序包

httptest 和 iotest 是两个常用的包，应该利用起来，构造于http 和 io 相关函数的测试。

#### httptest

httptest 包不需要通过建立网络连接就可以进行测试，主要用来测试服务端的http api handler 函数 以及 客户端的http caller函数。具体查看开发文档：https://pkg.go.dev/net/http/httptest

对于测试服务中api Handler的场景，只需要通过httptest.NewRequest 来构建api的请求数据的Reader，以及使用httptest.NewRecorder 来创建一个往请求api中写入响应数据的Writer, 这样在写测试用例时候， 直接模拟接口请求数据，编写相关的测试case,  测试的api handler 返回的数据 可以从Writer中获取到，进而可以做接口响应数据的断言假设，比如 返回状态码，响应body数据， 响应头中的数据。

对于测试客户端中相关的http client caller函数， 通过httptest.NewServer建立对应api handler服务, 客户端相关的http client [caller函数可以使用http.Client.Do](http://xn--callerhttp-uh4py1d60ohkhow7ewtwc.Client.Do) 对server.URL进行调用了，进而可以对返回的值进行断言测试。还可以使用httptest.NewTLSServer 建立一个TLS的测试服务。

grpc也有对应的测试库grpc/test 库，无需建立网络连接即可测试，具体查看开发文档：https://pkg.go.dev/google.golang.org/grpc/test

#### iotest

该iotest包 ( https://pkg.go.dev/testing/iotest ) 实现了io Reader ，Writer， Closer 等接口， 用于测试使用io 相关接口的方法测试；

### 89.编写不准确的基准

对于性能优化，不能盲猜，需要编写基准压测来 分析评估 具体性能，然而编写基准测试并不简单。编写不准确的基准并根据它们做出错误的假设可能非常简单。使用go test -bench `regexp` 匹配对应BenchmarkXxxx函数进行基准压测，默认运行1s, 运行时间通过-benchtime 来调整，其他性能分析的参数见开发文档。以下几种常见编写不准确基准压测的情况：

#### 不重置或暂停定时器

在基准压测时，可能会执行一个耗时长的初始准备逻辑，比如准备大量的数据用于基准压测，这个时候需要引入 testing.B.ResetTimer() 来重置时间，在开始基准压测，从测试结果中丢弃这部分耗时设置； 还有一种情况是在基准压测的循环里，这需要使用testing.B.StopTimer()停止时间，处理完准备逻辑，在使用testing.B.StartTimer()开始时间来处理基准测试函数。

#### 对微基准做出错误的假设

对于基准测试，不能只用一轮或几轮基准实验，就做出了假设；基准测试受当前机器运行时环境的影响，cpu，内存负载情况等；所以在进行基准压测试，如果测试的值有所偏失，应该按照概率论中的大数定律，将压测的时间放长(-benchtime 调整)，而且测试的次数可以增加些(-count 调整)， 将这些性能数据，通过benchstat https://pkg.go.dev/golang.org/x/perf/cmd/benchstat 工具来 对比分析 前后benchmark的统计数据, 如果对比结果误差很小，则说明性能无差异。基准测试必须基于在合理环境中通过多次样本取证进行A/B 比较之后才能给出比较正确的结果。

```go
go install golang.org/x/perf/cmd/benchstat@latest
go test -run=NONE -bench=BenchmarkAtomicStoreInt32 -count=10 -benchtime=3s -benchmem ./11-testing/89-benchmark/wrong-assumptions | tee -a smp1.txt
go test -run=NONE -bench=BenchmarkAtomicStoreInt32 -count=10 -benchtime=3s -benchmem ./11-testing/89-benchmark/wrong-assumptions | tee -a smp2.txt
benchstat smp1.txt smp2.txt

go test -run=NONE -bench=BenchmarkAtomicStore. -count=10 -benchtime=3s -benchmem ./11-testing/89-benchmark/wrong-assumptions | tee  smp.txt
benchstat smp.txt
```

#### 不注意编译器优化

这个case 在Go中有对应issue（https://github.com/golang/go/issues/14813）; benchmark的函数代码比较简单，被内联到基准压测文件中，导致测试为空的，这样导致空的基准压测在执行，每次op时间相当于一个时钟周期时间，如何避免编译器优化欺骗基准测试结果的模式：

将被测函数的结果分配给局部变量，然后将最新结果分配给全局变量，先分配局部变量在栈上，不影响测试，而将局部变量复制给全局变量，分配在堆上，以防编译器优化进行inline 处理。

还有一种方式是直接使用//go:noinline 标记函数，在编译阶段防止inline。

#### 被观察者效应愚弄

在基准压测一个CPU-Bound的函数时，需要注意cpu cache 局部性原理对基准压测的影响；为了防止cpu cache对基准压测的影响，可以在每次测试前，创建一个新的测试数据用于测试，比如基准压测矩阵运算函数，在每次测试前，新建一个测试矩阵数据。

### 90.没有探索所有的 Go 测试功能

工欲善其事必先利其器，充分掌握go test工具有助于写出高质量的代码。

#### 代码覆盖率

```bash
# 输出测试覆盖文件
go test -coverprofile=coverage.out ./...
# 分析测试覆盖文件
go tool cover -html=coverage.out
# 一个包在另外一个包中会测试到，需要表明测试覆盖到的包
go test -coverpkg=./... -coverprofile=coverage.out ./...
```

tips: 在追踪代码覆盖率时要保持谨慎。拥有 100% 的测试覆盖率并不意味着应用程序没有错误；需正确推理测试涵盖的内容。

#### 从不同的包进行测试

这种在业务功能函数测试，或者对外部进行测试， 关注包的公开api行为而不是内部的具体实现细节，专注于测试暴露的行为。 常用的测试如BDD 开发， 只关注包的公开行为， 比如[ginkgo](https://onsi.github.io/ginkgo/) ；编写的测试用例文件可以不用和测试函数在同一个包内，可以单独定义测试文件夹，对不同包来进行测试用例的开发。

#### helper功能函数

在进行测试是，需要测试前的准备，这些准备工作逻辑helper功能函数，用于初始化一些对象来测试，参数需要传入*testing.T,  在初始逻辑中判读初始的错误情况，如果错误直接调用t.Fatal退出即可，只返回对应测试对象，这样可以方便复用，无需在处理错误，方便其他测试场景使用。这个属于代码质量问题啦。

#### 安装(setup)和拆卸(teardown)

如果单个测试初始执行完测试后需要清理一些初始资源，可以使用 testing.T.Cleanup函数来做单测的收尾工作；多个调用入栈操作，单测完出栈执行。

如果测试文件都依赖于初始化之后才开始测试的话， 可以放在全部测试开始之前的位置进行测试的setup；在全部测试结束后对初始化的资源进行释放teardown； 可以通过在TestMain中定义整体逻辑如下：

```go
func TestMain(m *testing.M) {
	setupHelper()
	code := m.Run() // run all test func
	teardownHelper() 
	os.Exit(code)
}
func initHelper(t *testing.T) {
	t.Cleanup(func() { println(1) })
	t.Cleanup(func() { println(2) }) // first to cleanup
	// init
	return 
}
```

这种方式在测试开发框架包中经常使用到，比如ginkgo中的BeforeSuite和AfterSuite函数，在测试前后执行。

## 概括

- 使用构建标志、环境变量或 -short模式对测试进行分类可以使测试过程更加高效。可以使用构建标志或环境变量（例如，单元测试与集成测试）创建测试类别，并区分短期运行测试和长期运行测试以确定要执行的测试类型。
- `race`强烈建议在编写并发应用程序时启用该标志。这样做可以捕捉到可能导致软件错误的潜在数据竞争；开启race检测会消耗内存，一般在开发测试，CI, 金丝雀发布(预发环境)的时候使用。
- 使用`parallel`标志是加速测试的有效方法，尤其是长时间运行的测试。
- 使用`shuffle`标志帮助确保测试套件不依赖于可能隐藏错误的错误假设。
- 表驱动测试是将一组类似测试分组以防止代码重复并使未来更新更易于处理的有效方法。
- 避免time.Sleep使用同步来使测试更稳定、更健壮。使用channel来进行同步，如果无法同步，请考虑重试方法。
- 了解如何使用时间 API 处理函数是使测试不那么不稳定的另一种方法。可以使用标准技术，例如将时间作为隐藏依赖项的一部分处理或要求客户提供时间。
- `httptest`包有助于处理 HTTP 应用程序。它提供了一组实用程序来测试客户端和服务器。
- `iotest`包有助于编写`io.Reader`和测试应用程序是否容错。
- 关于基准：
  - 使用时间方法来保持基准的准确性。
  - 在处理微基准时，增加`benchtime`或使用诸如`benchstat`此类的工具会有所帮助。
  - 如果最终运行应用程序的系统与运行微基准测试的系统不同，请注意微基准测试的结果。
  - 确保被测函数会产生副作用，以防止编译器优化导致基准测试结果上有误差。
  - 为防止观察者效应，强制基准重新创建 CPU-Bound函数使用的数据。
- 使用带标志的代码覆盖率`coverprofile`可以快速查看代码的哪一部分需要更多关注。
- 将单元测试放在不同的包中，以强制编写专注于暴露行为而非内部的测试。
- 使用`testing.T`变量而不是经典变量来处理错误`if err != nil`使代码更短且更易于阅读。
- 可以使用设置安装和拆卸功能来配置复杂的环境，例如在集成测试的情况下。