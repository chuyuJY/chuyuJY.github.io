# Go-GMP


[toc]

**参考文章**：

- [Go语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#65-%E8%B0%83%E5%BA%A6%E5%99%A8)
- [深入了解 Go 语言与并发编程](https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649770126&idx=1&sn=023a7c9af7828479b991168392cec15c&chksm=beccdbf589bb52e39586d2c8294b56ebfa36c73562d666421432974758d15c27d59d36fee76a&scene=126&sessionid=1681283099&subscene=227&key=83c848a1d31eeadc00ea32d2a9cb25e5ad9047c23185e70b8595cc4aabbb009a8a81a71499c769fe90b7b1f9305962aa18887026fc21b14d28c12c1d07cedd6ebdcdd8417b3c1612309593cfc769e5034a18ce3b062c1bf40779e78dbb60fa81b76a784010f80ea0992390542f215c27bc864980b80be9a3e64325e3b123f2a3&ascene=0&uin=MjY0NTQ5ODYwNg%253D%253D&devicetype=Windows+11+x64&version=63090214&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQw2EM0040jHSF5T68i9K3khLgAQIE97dBBAEAAAAAAOuOLjTIOloAAAAOpnltbLcz9gKNyK89dVj0batp40C3fMe2Y1xla%252FROFU4%252FE2kE77fx1Nm09aK%252F7ZGyaYEDVtl911%252Bj2AuSPxjLz9ZwTOzGjKCWI0V3j8AGaVSfZ%252FrxGFc4aSec96EkABq00oN0VXzpyZbO%252FIgKaQ%252B%252BhjliGY4JP6UJaoIlRg8rs6x0EVFWjvrFwqcueT2Jj%252B011iyBgCh26c02JEW4RhhFxpdBdxH759JCQtC5PW4G5GObnWPWACmdsmsJx2fKwff927daEtyRlbPH&acctmode=0&pass_ticket=7i96p0GGGPYUcWwVrJeEM%252FDiyTqOBKyr8%252ByDrCc1oxUlC3hgx%252F%252Bt2hb3g0O33WO59gJQbSn2GTfW1KKHZvtdbg%253D%253D&wx_header=1)
- [从 bug 中学习：六大开源项目告诉你 go 并发编程的那些坑](https://mp.weixin.qq.com/s/FDV77dO9nwtPltmx5cB7Lw)

## 一、Go 并发机制

Go 的调度器使用 G、M、P 三个结构体来实现 Goroutine 的调度，也称之为 **GMP 模型**。

### GMP 模型

**G**：表示 Goroutine。**每个 Goroutine 对应一个 G 结构体，G 存储 Goroutine 的运行堆栈、状态以及任务函数，可重用**。当 Goroutine 被调离 CPU 时，调度器代码负责把 CPU 寄存器的值保存在 G 对象的成员变量之中，当 Goroutine 被调度起来运行时，调度器代码又负责把 G 对象的成员变量所保存的寄存器的值恢复到 CPU 的寄存器；

**M**：OS 底层线程的抽象，它本身就与一个内核线程进行绑定，每个工作线程都有唯一的一个 M 结构体的实例对象与之对应，它代表着真正执行计算的资源，由操作系统的调度器调度和管理。**M 结构体对象除了记录着工作线程的诸如栈的起止位置、当前正在执行的 Goroutine 以及是否空闲等等状态信息之外，还通过指针维持着与 P 结构体的实例对象之间的绑定关系**；

**P**：表示逻辑处理器。对 G 来说，P 相当于 CPU 核，G 只有绑定到 P(在 P 的 local runq 中)才能被调度。对 M 来说，P 提供了相关的执行环境(Context)，如内存分配状态(mcache)，任务队列(G)等。它维护一个局部 Goroutine 可运行 G 队列，工作线程优先使用自己的局部运行队列，只有必要时才会去访问全局运行队列，这可以大大减少锁冲突，提高工作线程的并发性，并且可以良好的运用程序的局部性原理。

一个 G 的执行需要 P 和 M 的支持。一个 M 在与一个 P 关联之后，就形成了一个有效的 G 运行环境（内核线程+上下文）。每个 P 都包含一个可运行的 G 的队列（runq）。该队列中的 G 会被依次传递给与本地 P 关联的 M，并获得运行时机。

M 与 KSE 之间总是一一对应的关系，一个 M 仅能代表一个内核线程。M 与 KSE 之间的关联非常稳固，一个 M 在其生命周期内，会且仅会与一个 KSE 产生关联，而 M 与 P、P 与 G 之间的关联都是可变的，M 与 P 也是一对一的关系，P 与 G 则是一对多的关系。

#### G

运行时，G 在调度器中的地位与线程在操作系统中差不多，但是它占用了更小的内存空间，也降低了上下文切换的开销。它是 Go 语言**在用户态提供的线程**，作为一种粒度更细的资源调度单元。

g 结构体部分源码(src/runtime/runtime2.go)：

```go
type g struct {
    stack    stack  		// Goroutine的栈内存范围[stack.lo, stack.hi)
    stackguard0   uintptr 	// 用于调度器抢占式调度
    m     *m  				// Goroutine占用的线程
    sched    gobuf  		// Goroutine的调度相关数据
    atomicstatus  uint32 	// Goroutine的状态
    ...
}

type gobuf struct {
    sp  uintptr  			// 栈指针
    pc  uintptr  			// 程序计数器
    g  guintptr  			// gobuf对应的Goroutine
    ret  sys.Uintewg 		// 系统调用的返回值
    ...
}
```

- gobuf 中保存的内容会在调度器保存或恢复上下文时使用，其中栈指针和程序计数器会用来存储或恢复寄存器中的值，改变程序即将执行的代码。

- atomicstatus 字段存储了当前 Goroutine 的状态，Goroutine 主要可能处于以下几种状态：
  - 等待中：Goroutine 正在等待某些条件满足，例如：系统调用结束等，包括_Gwaiting、_Gsyscall 和_Gpreempted 几个状态
  - 可运行：Goroutine 已经准备就绪，可以在线程运行，如果当前程序中有非常多的 Goroutine，每个 Goroutine 就可能会等待更多的时间，即_Grunnable
  - 运行中：Goroutine 正在某个线程上运行，即_Grunning

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230429154517629.png" alt="image-20230429154517629" style="zoom:33%;" />

G 常见状态转换图：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230429154803958.png" alt="image-20230429154803958" style="zoom:40%;" />

#### M

Go 语言并发模型中的 M 是操作系统线程。调度器最多可以创建 10000 个线程，但是**最多只会有 GOMAXPROCS(P 的数量)个活跃线程能够正常运行**。在默认情况下，**运行时会将 GOMAXPROCS 设置成当前机器的核数**，我们也可以在程序中使用 runtime.GOMAXPROCS 来改变最大的活跃线程数。

m 结构体源码(部分)：

```go
type m struct {
    g0   *g   		// 一个特殊的goroutine，执行一些运行时任务
    gsignal  *g   	// 处理signal的G
    curg  *g   		// 当前M正在运行的G的指针
    p   puintptr 	// 正在与当前M关联的P
    nextp  puintptr // 与当前M潜在关联的P
    oldp  puintptr 	// 执行系统调用之前使用线程的P
    spinning bool  	// 当前M是否正在寻找可运行的G
    lockedg  *g   	// 与当前M锁定的G
}
```

#### P

调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率。

**P 的数量等于 GOMAXPROCS，设置 GOMAXPROCS 的值只能限制 P 的最大数量，对 M 和 G 的数量没有任何约束**。当 M 上运行的 G 进入系统调用导致 M 被阻塞时，运行时系统会把该 M 和与之关联的 P 分离开来，这时，如果该 P 的可运行 G 队列上还有未被运行的 G，那么运行时系统就会找一个空闲的 M，或者新建一个 M 与该 P 关联，满足这些 G 的运行需要。**因此，M 的数量很多时候都会比 P 多**。

p 结构体源码（部分）：

```go
type p struct {
    status   uint32			// p 的状态
    m        muintptr 		// 对应关联的 M
    runqhead uint32 		// 可运行的Goroutine队列，可无锁访问
    runqtail uint32
    runq     [256]guintptr
    runnext  guintptr 		// 缓存可立即执行的G
    gFree struct { 			// 可用的G列表，G状态等于Gdead
    	gList
    	n int32
	}
 ...
}
```

P 可能处于的状态如下：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230429155959520.png" alt="image-20230429155959520" style="zoom:40%;" />

#### 调度器

**调度的主要对象就是 G、M 和 P 的实例**。每个 M(即每个内核线程)在运行过程中都会执行一些调度任务，他们共同实现了 Go 调度器的调度功能。

##### g0 和 m0

运行时系统中的每个 M 都会拥有一个特殊的 G，一般**称为 M 的 g0**。M 的 g0 不是由 Go 程序中的代码间接生成的，而是**由 Go 运行时系统在初始化 M 时创建并分配给该 M 的。M 的 g0 一般用于执行调度、垃圾回收、栈管理等方面的任务**。M 还会拥有一个专用于处理信号的 G，称为 gsignal。

除了 g0 和 gsignal 之外，其他由 M 运行的 G 都可以视为用户级别的 G，简称用户 G，g0 和 gsignal 可称为系统 G。Go 运行时系统会进行切换，以使每个 M 都可以交替运行用户 G 和它的 g0。这就是前面所说的“每个 M 都会运行调度程序”的原因。

除了每个 M 都拥有属于它自己的 g0 外，**还存在一个 runtime.g0**。runtime.g0 用于执行引导程序，它运行在 Go 程序拥有的第一个内核线程之中，这个线程也称为 **runtime.m0**，**runtime.m0 的 g0 就是 runtime.g0**。

##### **核心元素的容器**

上面讲了 Go 的线程实现模型中的 3 个核心元素——G、M 和 P，下面看看承载这些元素实例的容器：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230429160858236.png" alt="image-20230429160858236" style="zoom:40%;" />

##### 调度循环

调用 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 进入调度循环：

1. 为了保证公平，当全局运行队列中有待执行的 Goroutine 时，通过 `schedtick` 保证有一定几率会从全局的运行队列中查找对应的 Goroutine；
2. 从处理器本地的运行队列中查找待执行的 Goroutine；
3. 如果前两种方法都没有找到 Goroutine，会通过 [`runtime.findrunnable`](https://draveness.me/golang/tree/runtime.findrunnable) 进行阻塞地查找 Goroutine；

[`runtime.findrunnable`](https://draveness.me/golang/tree/runtime.findrunnable) 的实现非常复杂，这个 300 多行的函数通过以下的过程获取可运行的 Goroutine：

1. 从本地运行队列、全局运行队列中查找；
2. 从网络轮询器中查找是否有 Goroutine 等待运行；
3. 通过 [`runtime.runqsteal`](https://draveness.me/golang/tree/runtime.runqsteal) 尝试从其他随机的处理器中窃取待运行的 Goroutine，该函数还可能窃取处理器的计时器；

因为函数的实现过于复杂，上述的执行过程是经过简化的，总而言之，当前函数一定会返回一个可执行的 Goroutine，如果当前不存在就会阻塞等待。

接下来由 [`runtime.execute`](https://draveness.me/golang/tree/runtime.execute) 执行获取到的 Goroutine，做好准备工作后，它会通过 [`runtime.gogo`](https://draveness.me/golang/tree/runtime.gogo) 将 Goroutine 调度到当前线程上；

最终在当前线程的 g0 的栈上调用 [`runtime.goexit0`](https://draveness.me/golang/tree/runtime.goexit0) 函数，该函数会将 Goroutine 转换会 `_Gdead` 状态；

在最后 [`runtime.goexit0`](https://draveness.me/golang/tree/runtime.goexit0) 会重新调用 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 触发新一轮的 Goroutine 调度，Go 语言中的运行时调度循环会从 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 开始，最终又回到 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule)，我们可以认为调度循环永远都不会返回。

## 二、从 Bug 中学习

> #### 无缓冲 channel，由于 receiver 退出导致 sender 侧阻塞

举例：

```go
func finishReq(timeout time.Duration) ob {
    ch := make(chan ob)
    go func() {
        result := fn()
        ch <- result // block
    }()
    select {
    case result = <-ch:
        return result
    case <-time.After(timeout):
        return nil
    }
}
```

本意是想在调用 fn() 时，加上超时的功能，如果 fn()在超时时间没有返回，则返回 nil。但是当超时发生的时候，针对无缓冲的 ch 来说，由于已经没有 receiver 了，第 5 行将会被 block 住，导致这个 goroutine 永远不会退出。

这个 bug 的修复方式也是非常的简单，把 unbuffered channel 修改成 buffered channel。

```go
func finishReq(timeout time.Duration) ob {
    ch := make(chan ob, 1)
    go func() {
        result := fn()
        ch <- result // block
    }()
    select {
    case result = <-ch:
        return result
    case <-time.After(timeout):
        return nil
    }
}
```

> **思考**：在上面的例子中，虽然这样不会 block 了，但是 channel 一直没有被关闭，channel 保持不关闭是否会导致资源的泄漏呢？

> #### 问：channel 被关闭多次引发的 bug

```go
select {
case <-c.closed:	
default:
    close(c.closed)
}
```

上面这块代码可能会被多个 goroutine 同时执行，这段代码的逻辑是，case 这个分支判断 closed 这个 channel 是否被关闭了，如果被关闭的话，就什么都不做；如果 closed 没有被关闭的话，就执行 default 分支关闭这个 channel，多个 goroutine 并发执行的时候，有可能会导致这个 channel 被关闭多次。

> For a channel c, the built-in function close(c) records that no more values will be sent on the channel. It is an error if c is a receive-only channel. Sending to or closing a closed channel causes a run-time panic.

这个 bug 的修复方式是：

```go
Once.Do(func() {
    close(c.closed)
})
```

把整个 select 语句块换成 Once.Do，保证 channel 只关闭一次。

> #### WaitGroup 误用导致阻塞

```go
var group sync.WaitGroup
group.Add(len(pm.plugins))
for _, p := range pm.plugins {
    go func(p *plugin) {
        defer group.Done()
    }(p)
    group.Wait()
}
```

当 `len(pm.plugins) >= 2` 时，第 7 行将会被卡住，因为这个时候只启动了一个异步的 goroutine，group.Done()只会被调用一次，group.Wait()将会永久阻塞。修复如下：

```go
var group sync.WaitGroup
group.Add(len(pm.plugins))
for _, p := range pm.plugins {
    go func(p *plugin) {
        defer group.Done()
    }(p)
}
group.Wait()
```

> #### context 误用导致资源泄漏

如下面的代码所示：

```go
hctx, hcancel := context.WithCancel(ctx)
if timeout > 0 {
    hctx, hcancel = context.WithTimeout(ctx, timeout)
}
```

第一行 context.WithCancel(ctx) 有可能会创建一个 goroutine，来等待 ctx 是否 Done，如果 parent 的 ctx.Done()的话，cancel 掉 child 的 context。也就是说 hcancel 绑定了一定的资源，不能直接覆盖。

> Canceling this context releases resources associated with it, so code should call cancel as soon as the operations running in this Context complete.

这个 bug 的修复方式是：

```go
var hctx context.Context
var hcancel context.CancelFunc
if timeout > 0 {
    hctx, hcancel = context.WithTimeout(ctx, timeout)
} else {
    hctx, hcancel = context.WithCancel(ctx)
}
```

或者

```go
hctx, hcancel := context.WithCancel(ctx)
if timeout > 0 {
    hcancel.Cancel()
    hctx, hcancel = context.WithTimeout(ctx, timeout)
}
```

> #### 问：多个 goroutine 同时读写共享变量导致的 bug

```go
for i := 17; i <= 21; i++ { // write
    go func() { /* Create a new goroutine */
        apiVersion := fmt.Sprintf("v1.%d", i) // read
    }()
}
```

第二行中的匿名函数形成了一个闭包(closures)，在闭包内部可以访问定义在外面的变量，如上面的例子中，第 1 行在写 i 这个变量，在第 3 行在读 i 这个变量。这里的关键的问题是对同一个变量的读写是在两个 goroutine 里面同时进行的，因此是不安全的。

其实这里会把 i 逃逸到堆上，然后都指向堆上同一个 i。因此，如下程序会输出很多个 6：

```go
func main() {
	for i := 0; i <= 5; i++ { // write
		go func() { 
			time.Sleep(100 * time.Millisecond)	// 等一下
			fmt.Printf("%d\n", i) // read
		}()
	}
	time.Sleep(3 * time.Second)
}
```

> Function literals are closures: they may refer to variables defined in a surrounding function. Those variables are then shared between the surrounding function and the function literal, and they survive as long as they are accessible.

可以修改成：

```go
for i := 17; i <= 21; i++ { // write
    go func(i int) { /* Create a new goroutine */
        apiVersion := fmt.Sprintf("v1.%d", i) // read
    }(i)
}
```

通过`passed by value`的方式规避了并发读写的问题。

> #### 问：timer 误用产生的 bug

如下面的例子：

```go
timer := time.NewTimer(0)
if dur > 0 {
    timer = time.NewTimer(dur)
}
select {
case <-timer.C:
case <-ctx.Done():
    return nil
}
```

原意是想 dur 大于 0 的时候，设置 timer 超时时间，但是 timer := time.NewTimer(0)导致 timer.C 立即触发。修复后：

```go
var timeout <-chan time.Time
if dur > 0 {
    timeout = time.NewTimer(dur).C
}
select {
case <-timeout:
case <-ctx.Done():
    return nil
}
```

> A nil channel is never ready for communication.

上面的代码中第一个 case 分支 timeout 有可能是个 nil 的 channel，select 在 nil 的 channel 上，这个分支不会被触发，因此不会有问题。
