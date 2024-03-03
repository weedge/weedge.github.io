---
author: "weedge"
title: "Go tips-笔记: 并发实践 61-74 mistakes"
date: 2023-02-18T14:26:23+08:00
tags: [
	"Golang"
]
categories: [
	"技术",
    "Golang"
]


---



## 引言

这里主要介绍并发实践中遇到的问题，这些问题在golang开源项目中也经常会出现，如果编写并发也会一直伴随在开发当中出现，也有工程实践相关的论文进行统计归纳总结(PS: 用这种方式发个论文还是比较轻松的~)：

**[Understanding Real-World Concurrency Bugs in Go](https://songlh.github.io/paper/go-study.pdf)**

tips: 作者对golang和rust都有研究，结合相关的代码都可以一起学习下, 语言方面的小细节

[**A Study of Real-World Data Races in Golang**](https://arxiv.org/pdf/2204.00764.pdf)

Go 官方提供race工具来检查并发场景下的数据竞争问题： https://go.dev/doc/articles/race_detector

https://github.com/google/sanitizers/wiki/ThreadSanitizerGoManual

注：Go要使用-race，需启用CGO，依赖sanitizers；一般用于开发测试进行检测

如果想更加深入的了解并发并行，可以一起学习这本书： [**Is Parallel Programming Hard, And, If So, What Can You Do About It?**](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html)

<!--more-->

## 笔记

### 61.传播不适当的context

传播context应该谨慎进行。比如文中中通过一个基于与 http.Request关联的context处理异步操作的示例来说明这一点。因为一旦服务接口返回响应，这次请求会话的context就会被cancel，使用 http.Request关联的context的异步操作也可能会意外停止(请求已经结束，但是异步操作还没有执行完)。遇到这种情况，可以为特殊情况创建实现context.Context接口的自定义context结构，这个结构将原来的ctx context.Context作为成员wrap一层，实现主要的传递功能Value方法, 这样在服务的请求回话结束之后，异步操作可以继续执行完成。

tips：生产环境下，请求接口下的异步操作，必须避免因goroutine协程处理hang住，导致泄露，一般引入执行超时机制。

### 62.在不知道何时停止的情况下启动 goroutine (重要)

Goroutines 启动起来既简单又便宜——如此简单和便宜，以至于不考虑停止一个新的 goroutine，这可能会导致泄漏。不知道何时停止 goroutine 是一个设计问题，也是 Go 中常见的并发错误。了解原因以及如何预防它。

首先，量化一下 goroutine 泄漏的含义。在内存方面，一个 goroutine 以 2 KB 的最小堆栈大小开始，它可以根据需要增长和收缩（最大堆栈大小在 64 位上为 1 GB，在 32 位上为 250 MB）。在内存方面，goroutine 还可以保存分配给堆的变量引用。同时，goroutine 可以保存 HTTP 或数据库连接、打开的文件和网络套接字等资源，这些资源最终应该正常关闭。如果一个 goroutine 被泄露，那么这些资源也会被泄露。

goroutine 是一种资源，就像任何其他资源一样，最终必须关闭以释放内存或其他资源(通常通过cancel 信号量，ctx.Done方式让这些goroutine任务退出释放资源)。在不知道何时停止的情况下启动 goroutine 是一个设计问题。每当一个 goroutine 启动时，应该对它何时停止有一个明确的计划。如果一个 goroutine 创建资源并且它的生命周期与应用程序的生命周期绑定，那么在退出应用程序之前，等待这个 goroutine 完成可能更安全。这样可以确保资源可以被释放。

### 63.不注意 goroutines 和循环变量 (重要)

```go
for _, i := range s {      
    go func() {
        fmt.Print(i)
    }()
}
```

这个是Go新手经常会遇到问题，也是老生常谈的问题了，如果希望每个闭包都访问goroutine创建时的值，有什么解决方案？有两种方法: 每次迭代中，创建一个局部新变量i，并将i复制给新i；另外一种方法是不再依赖闭包，而是使用实际函数进行传参值拷贝，本质上一样。

### 64.使用 select 和 channels 期待确定性行为 （重要）

这里主要了解select语义的工作原理，可以从官方文档中进行了解：https://go.dev/ref/spec#Select_statements

如果一个或多个通信可以继续进行，则通过统一的伪随机选择一个可以继续进行的通信。否则，如果存在默认情况，则选择该情况。如果没有默认情况，则“select”语句将阻塞，直到至少有一个通信可以继续进行。

tips: 当使用无缓冲channel时，写入不想阻塞，使用select + case 写chan + default的方式来处理时非常好的办法，可以避免死锁的情况，比如 `fatal error: all goroutines are asleep - deadlock!`这个错误经常会遇到，这个是全部在执行的goroutine都进入了等待状态，Go 语言死锁检测会发现当前的 Goroutine 已经不可能被唤醒，就会直接报错退出；常见于 一组协程处理数据其中一个协程进入一直等待状态，调用sync.WaitGroup Wait方法(底层通过信号量值机制semacquire1)等待协程执行完成，这样出现相互等待，导致deadlock。

### 65.不使用通知channel

无数据channel应该用 `chan struct{}` 作为通知channel， struct{}{}不占内存空间。

### 66.不使用nil  channel

接受或发送到 nil 通道是一种阻塞行为，而且这种行为并非无用。正如文中合并两个通道的示例中看到的那样，即使close 通道， 接受方还是可以读取数据，通过返回的第二个参数判断是否关闭，关闭了将通道设置为nil，这样利用select不会选择阻塞的nil通道，可以使用 nil 通道来实现一个优雅的状态机，所以 nil 通道在某些情况下很有用，并且在处理并发代码时应该成为 Go 开发人员工具集的一部分。

### 67.对channel大小感到困惑

- 无缓冲通道支持同步。可以保证两个 goroutine 将处于已知状态：一个接收消息，另一个发送消息。
- 缓冲通道不提供任何强同步。实际上，生产者 goroutine 可以发送一条消息，然后在通道未满时继续执行。唯一的保证是 goroutine 在消息发送之前不会收到消息。

必须牢记这一基本区别。两种通道类型都支持通信，但只有一种提供同步。如果需要同步，必须使用无缓冲通道，无缓冲通道也可能更容易推理；缓冲通道可能导致模糊的死锁，这在无缓冲通道中会立即显现出来。在通知channel的情况下，通知是通过关闭channel ( `close(ch)`) 处理的，使用缓冲通道不会带来任何好处，close channel后，还可以继续从channel中读取数据。

使用缓冲通道的情况：

- 在使用类似工作池的模式下，创建的goroutine轮训从共享通道获取数据执行；可以将缓冲通道大小与创建的 goroutines 数量联系起来。
- 当使用通道来解决速率限制问题时。如果需要通过限制请求数量来强制资源利用，应该根据限制设置缓冲通道大小。例如，errorgroup 中的 sem chan struct{}(token) 就是用来设置最大执行的goroutine数目。

决定一个准确缓冲通道大小不是一个容易的问题。首先，它是 CPU 和内存之间的平衡。值越小，可以面对的 CPU 争用越多；但是这个值越大，需要分配的内存就越多；需要基于场景下，基准压测来衡量。

### 68.忘记字符串格式化可能产生的副作用

- 数据竞争(data race), 文中举了一个etcd 例子中 一个goroutine通过`fmt.Sprintf("%v", ctx)` 格式化成key, 对key进行watch操作， 通过ctx中的String方法读取ctx中的元数据进行格式化；另一个goroutine 通过context.WithValue 写入，这样产生了data race。修复 ( https://github.com/etcd-io/etcd/pull/7816 ) pr中, 直接实现wrap一层自定义ctx，不依赖通过context.WithValue写入改变值的ctx；
- 死锁(deadlock)，如果一个结构体的格式化String函数中使用了互斥锁，则对结构体对象格式化时，要考虑对应互斥锁的范围，如果上锁范围包括了格式化代码，则会重复上锁，导致相互等待，进而出现deadlock；

### 69.使用append操作产生数据竞争 (重要)

发生数据竞争(data race)的情况是多个并发goroutines至少有一个写操作发生在一个共享空间中；对于slice切片结构，append在扩容的时候是否重新分配了内存空间，如果发生扩容则在在切片副本上使用，而不是原始切片，这样就不会发生数据竞争；更合理情况是直接在goroutine中进行copy一份切片副本进行append操作。

多个goroutines 并发访问 slice和map时，发生数据竞争的情况：

- 使用至少一个更新值的 goroutine 访问同一个切片索引是一种数据竞争；goroutines 访问相同的内存位置。

- 无论操作如何访问不同的切片索引都不是数据竞争；不同的索引意味着不同的内存位置。

- 使用至少一个goroutine更新访问同一个map（不管它是相同的还是不同的key）是一种数据竞争。为什么这与切片数据结构不同？map底层结构是个桶数组，每个桶都是一个指向键值对数组的指针；哈希算法用于确定桶的数组索引。因为该算法在map初始化期间包含一些随机性，所以一次执行可能导致相同的数组索引(相同bucket)，而另一次执行可能不会。竞争检测器通过发出警告来处理这种情况，而不管实际的数据竞争是否发生。

  tips: 与slice不同，go在map实现中内置了对并发读写的检测，即便不加入-race，一旦发现存在数据竞争(至少有一个写操作)直接fatal error。

### 70.对slice和map使用mutex不准确 （重要）

在数据既可变又共享的并发上下文中工作时，通常使用mutex对操作数据的临界区域进行同步互斥访问；

具体map的内部结构在https://github.com/golang/go/blob/go1.20/src/runtime/map.go hmap查看源码(通过测试用例代码了解)；map是一个`runtime.hmap`主要包含元数据（counter,flags,B等）以及2个指向数据桶(bucket)的指针的结构。所以对与map变量之间赋值操作`mp:=m`不复制底层实际数据(buckets)。这个和slice切片的原理是一样，只不过需要注意append扩容情况，而map扩容的是底层buckets数据。

了解了slice和map的结构，对于mutex保护操作共享的slice或者map的临界区间很有帮助，对于map遍历操作进行互斥访问，如果遍历处理的时间长，考虑到性能问题，可以深拷贝一份出来进行耗时的计算操作；

在考虑使用mutex对slice或map进行互斥访问时，需要考虑好互斥的临界区域。

### 71.滥用 sync.WaitGroup  （重要）

sync.WaitGroup是Go并发程序常用的用于等待一组goroutine退出的机制。通过Add和Done方法实现内部计数的调整。而Wait方法用于等待，直到内部计数器为0才会返回。文中提到的例子是比较经典的坑，在论文[**A Study of Real-World Data Races in Golang**](https://arxiv.org/pdf/2204.00764.pdf)中也有提到

```go
wg := sync.WaitGroup{}
	var v uint64

	for i := 0; i < 3; i++ {
		go func() {
			wg.Add(1)
			atomic.AddUint64(&v, 1)
			wg.Done()
		}()
	}

	wg.Wait()
	fmt.Println(v)
```

在`sync.WaitGroup`结构中拥有一个默认初始化为 0的计数器。可以使用`Add(int)`方法递增此计数器，使用`Done()`或使用Add负值来递减此计数器。如果想要等到计数器为0，则使用`Wait()`阻塞等待并释放goroutine资源，这部分内容在 [#64](https://weedge.github.io/post/notions/go-tips/go-tips-09-concurrency-practice/#64%E4%BD%BF%E7%94%A8-select-%E5%92%8C-channels-%E6%9C%9F%E5%BE%85%E7%A1%AE%E5%AE%9A%E6%80%A7%E8%A1%8C%E4%B8%BA-%E9%87%8D%E8%A6%81) tips中也有提到，具体见源码客观分析：https://github.com/golang/go/blob/go1.20/src/sync/waitgroup.go (结合测试用例看疗效更好)

https://github.com/golang/go/blob/go1.20/src/runtime/sema.go (Semaphore实现，[类似Linux的futex机制](https://swtch.com/semaphore.pdf))

了解了WaitGroup,  不难理解例子中的代码问题，将wg.Add(1)放在了goroutine执行的函数中，而没有像正确方法那样，将Add(1)放在goroutine创建启动之前，这样会导致对WaitGroup内部计数器形成了数据竞争，很可能因goroutine调度问题，Add(1)还未来的及调用，从而导致Wait提前返回，这组goroutine中还有在执行中的。

在 论文[A Study of Real-World Data Races in Golang](https://arxiv.org/pdf/2204.00764.pdf) 中 还提到一个问题，就是goroutine中有多个defer 操作，defer Done 操作首先执行了，导致其他defer操作可能还未执行，Wait就已经返回了，导致后面依赖defer操作中的结果,进行判断处理的逻辑会出错。

tips: 上一节tips中有提到cpu有使用*内存屏障(memory barrier)*（也称为*内存栅栏(memory fence)*）来确保顺序。Go 为实现内存屏障定义了语言层面的内存模型规范，这里在使用的`sync.WaitGroup`，`wg.Add` 和 `wg.Wait`之间存在 happens-before 关系。

这个是Go开发人员常见错误。使用`sync.WaitGroup`，`Add`操作必须在父 goroutine 中启动 goroutine 之前完成，而`Done`操作必须在 goroutine 内完成。

### 72.忘记 sync.Cond

在同步原语`sync`包中，`sync.Cond`可能是最少使用和理解的。但是，它提供了无法通过channel实现的功能。实现类似pub/sub 的多通道广播机制，可以认为pub/sub机制是包括了单通道广播的，sync.Cond的内部实现，其结构中L Locker 用来互斥访问条件逻辑，如果条件不成立，则检测是否copy，copy则直接panic, 否则添加到通知列表中，进行等待；如果条件成立，则执行对应逻辑。唤醒方式分为两种：Signal() 唤醒等待队中的一个goroutine来执行判断； Broadcast 唤醒等待队列中的全部goroutine来执行对应判断逻辑；具体见源码客观分析：https://github.com/golang/go/blob/go1.20/src/sync/cond.go (结合测试用例看疗效更好)

https://github.com/golang/go/blob/go1.20/src/runtime/sema.go (Semaphore实现，[类似Linux的futex机制](https://swtch.com/semaphore.pdf))

对于Signal()方式，和使用 channel chan struct{} 非阻塞发送消息一样

```go
ch := make(chan struct{})
select {
case ch <- struct{}{}:
default:
}
```

### 73.不使用errorgroup

errorgroup这个包是google对go的一个扩展包：[golang.org/x/sync/errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup)

tips: `golang.org/x`是一个提供标准库扩展的库。为sync包扩展了一个包：errgroup

实现逻辑简单，主要是错误处理，如果其中有一个goroutine执行错误，则只记录第一次goroutine执行的错误，通过context告知了cancel状态，这个需要通过select+ctx.Done() 感知到；并且通知对应使用Wait()等待全部goroutine执行完成，并且返回记录的错误； 后面加入sem chan struct{}(token)，用来限制最大执行goroutine数，通过SetLimit来设置，并且提供了TryGo 非阻塞执行。

如果想加入goroutine的执行超时时间，也是可以做到，只需在使用errgroup前，使用cancelCtx就行，如下代码：

```go
func TestErrGroupWithTimeout(t *testing.T) {
	ctx, cancel := context.WithTimeout(context.TODO(), 5*time.Second)
	defer cancel()
	group, ctx := errgroup.WithContext(ctx)
	for i := 0; i < 10; i++ {
		index := i
		group.Go(func() error {
			select {
			case <-time.After(time.Duration(index) * time.Second):
				fmt.Printf("finished:%d\\n", index)
				return nil
			case <-ctx.Done():
				fmt.Printf("canceled:%d\\n", index)
				return ctx.Err()
			}
		})
	}
	if err := group.Wait(); err != nil {
		fmt.Println(err)
	}
}
```

当然如果想获取goroutine执行的全部错误则需要额外的错误数组来支持，Go中的函数返回错误必须为nil。

### 74.复制同步类型 （重要）

sync包提供基本同步原语，例如 mutex, rwmutex, condition variable，waitgroup，pool，map等。对于所有这些结构体，有一个硬性规则要遵循：它们永远不应该被复制。以下是一个常见的错误：

```go
type Counter struct {
	mu       sync.Mutex
	// mu      *sync.Mutex
	counters map[string]int
}

func NewCounter() Counter {
	return Counter{counters: map[string]int{}}
	// return Counter{counters: map[string]int{}, mu: &sync.Mutex{}}
}

func (c Counter) Increment1(name string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.counters[name]++
}
func (c *Counter) Increment2(name string) {
	// Same code
}
func main() {
	counter := NewCounter()

	go func() {
		counter.Increment1("foo")
	}()
	go func() {
		counter.Increment1("bar")
	}()

	time.Sleep(10 * time.Millisecond)
}
```

结构体接受者采用值传递，如果两个协程同时使用counter, 会复制一份结构，也会复制互斥锁，导致上锁失败，并发场景运行时出现data race,  data race可以 -race 进行检测；

通过linter类型工具检查，比如静态编译检查vet，可以直接检查出来进行提示，`passes lock by value` or `assignment copies lock value to` ；一般IDE开发工具安装了静态检查工具就可以检查出来提示(如果不扫描里面的noCopy成员，则扫不出来错误进行提示)，最好的办法直接使用 go vet 在CI阶段检查，进而保证代码质量；

这个noCopy的检测是怎么做到的呢？只要是实现了Locker 接口的Lock()和Unlock()方法的结构体，或者结构体成员实现了Locker接口，则可以通过go vet功能，来检查代码中该对象是否有被copy；比如自定义的结构体包涵值传递成员noCopy，noCopy结构体实现了Locker接口，则通过go vet检查是否copy

```go
// noCopy may be added to structs which must not be copied
// after the first use.
//
// See **<https://golang.org/issues/8005#issuecomment-190753527**>
// for details.
//
// Note that it must not be embedded, due to the Lock and Unlock methods.
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
type MyStruct struct {
   noValCopy noCopy
   // Copy *noCopy
}
```

每当多个 goroutine 访问一个同步共享元素，必须确保它们都依赖于同一个实例。此规则适用于定义的所有同步类型。可以使用指针解决这个问题，结构传递者对象是指针，或者结构成员中的同步共享元素是指针类型。本质上是值传递和指针传递的问题。

## 概括

- 了解context何时可以取消在传播它时需要注意，避免取消导致未执行完：例如，HTTP 处理程序在发送响应后取消context。
- 避免泄漏意味着要注意，无论何时启动 goroutine，都应该有一个最终停止它的计划。
- 为了避免 goroutines 和循环变量的错误，创建局部变量或调用函数而不是闭包。
- 如果多个选项是可能的，那么理解`select`多通道随机选择案例可以防止做出可能导致微妙的并发错误。
- 使用`chan struct{}`类型发送通知。
- 使用 nil channel应该成为并发工具集的一部分，从select语句中移除操作channel 的 case。
- 考虑到问题，仔细决定要使用的正确channel类型。只有无缓冲通道才能提供强大的同步保证。
- 应该有充分的理由为缓冲通道指定通道大小。
- 意识到字符串格式可能会导致调用现有函数意味着要注意可能的死锁和其他数据竞争。
- 并发 append并不总是没有数据竞争；因此，不应在共享切片上同时使用它。
- 了解slice和map结构体，具体底层数据结构；对防止常见的数据竞争处理有所帮助。
- 要准确使用`sync.WaitGroup`，`Add`在启动 goroutine 之前调用该方法。
- `sync.Cond`可以使用 广播方式向多个 goroutines 发送重复的通知(唤醒)，也可以单播方式想一个goroutine发送通知(唤醒)。
- 可以同步一组 goroutines 并使用`errgroup`包处理错误和context。
- 同步原语类型或者自定类型结构不应copy。