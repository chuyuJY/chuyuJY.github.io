# Go-Context


> **前言**：在 Go http包的Server中，每一个请求在都有一个对应的 goroutine 去处理。请求处理函数通常会启动额外的 goroutine 用来访问后端服务，比如数据库和RPC服务。用来处理一个请求的 goroutine 通常需要访问一些与请求特定的数据，比如终端用户的身份认证信息、验证相关的token、请求的截止时间。 当一个请求被取消或超时时，所有用来处理该请求的 goroutine 都应该迅速退出，然后系统才能释放这些 goroutine 占用的资源。

有空的时候，看看这篇讲Context实现的文章: https://wmf.im/p/%E5%88%86%E6%9E%90-go-%E6%A0%87%E5%87%86%E5%BA%93%E4%B8%AD%E7%9A%84-context-%E5%AE%9E%E7%8E%B0/

## 一、Context 概述

Go 1.7加入了一个新的标准库`context`，它定义了`Context`类型，用来简化 **处理单个用户请求**的**多个goroutine之间**与**元数据传递、截止时间、取消信号**等相关操作，这些操作可能涉及多个 API 调用。**当一个上下文被取消时，它派生的所有上下文也被取消**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230504145646646.png" alt="image-20230504145646646" style="zoom:50%;" />

主要内容概括为：

- 1 个接口：`Context`；
- 4 种类型实现：`emptyCtx`、`cancelCtx`、`timerCtx`、`valueCtx`；
- 6 个函数：`Background`、`TODO`、`WithCancel`、`WithDeadline`、`WithTimeout`、`WithValue`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230504151056768.png" alt="image-20230504151056768" style="zoom:50%;" />



首先要明确的是，**创建 Goroutine 和 Context 时**，都会按照树的结构，生成父节点到从节点的边，最终形成一种多叉树结构：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230430211219356.png" alt="image-20230430211219356" style="zoom:50%;" />

## 二、Context 接口

`context.Context`是一个接口，该接口定义了四个需要实现的方法。具体接口如下：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

- `Deadline`方法需要返回当前`Context`被取消的时间，也就是完成工作的截止时间（deadline）；
- `Done`方法需要返回一个`Channel`，这个Channel会在当前工作完成或者上下文被取消之后关闭;
- `Err`方法会返回当前`Context`结束的原因，它只会在`Done`返回的Channel被关闭时才会返回非空的值；

  - 如果当前`Context`被取消就会返回`Canceled`错误；

  - 如果当前`Context`超时就会返回`DeadlineExceeded`错误；
- `Value`方法会从`Context`中返回键对应的值，对于同一个上下文来说，多次调用`Value` 并传入相同的`Key`会返回相同的结果，该方法仅用于传递跨API和进程间跟请求域的数据；

## 三、类型实现

### emptyCtx

#### 数据结构

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}

func (*emptyCtx) Done() <-chan struct{} {
    return nil
}

func (*emptyCtx) Err() error {
    return nil
}

func (*emptyCtx) Value(key any) any {
    return 
}
```

- emptyCtx 是一个**空的 Context**，本质上类型为一个**整型**；
- Deadline 方法会返回一个公元元年时间以及 false 的 flag，标识当前 context 不存在过期时间；
- Done 方法返回一个 nil 值，用户无论往 nil 中写入或者读取数据，均会陷入阻塞；
- Err 方法返回的错误永远为 nil；
- Value 方法返回的 value 同样永远为 nil.

#### Background() 和 TODO()

Go内置两个函数：`Background()`和`TODO()`，这两个函数分别返回一个实现了`Context`接口的`background`和`todo`。我们代码中最开始都是以这两个内置的上下文对象作为最顶层的`partent context`，衍生出更多的子上下文对象。

```go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

func Background() Context {
    return background
}

func TODO() Context {
    return todo
}
```

- `Background()`主要用于初始化时创建一个`emptyCtx`，作为Context这个树结构的最顶层的Context，也就是**根Context**。

- `TODO()`：**官方文档**建议在本来应该使用外层传递的`ctx`，而外层却没有传递的地方使用。正如它的名字一样，留下一个 `TODO`。

`background`和`todo`本质上都是`emptyCtx`结构体类型，是一个不可取消，没有设置截止时间，没有携带任何值的Context。    

### cancelCtx

一种可取消的 Context。

#### 数据结构

```go
type cancelCtx struct {
    Context						   // embed 的父 Context，为空

    mu       sync.Mutex            	// 用于保护以下三个字段的锁，以保障cancelCtx是线程安全的
    done     atomic.Value			// 用于获取该Context的取消通知
    children map[canceler]struct{} 	// 用于存储以当前节点为根节点的所有可取消的Context
    err      error                 	// 用于存储取消时指定的错误信息
}

type canceler interface {
    cancel(removeFromParent bool, err error)
    Done() <-chan struct{}
}
```

- embed 了一个 context 作为其父 context. 可见，cancelCtx 必然为某个 context 的子 context；
- mu：用于保护以下三个字段的锁，以保障 `cancelCtx` 是线程安全的；
- done：用于获取该 Context 的取消通知；
- children：用于存储以当前节点为根节点的所有可取消的 Context，以便在根 Context 取消时，把它的子节点一并取消；
- err：用于存储取消时指定的错误信息。

#### WithCancel

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

`WithCancel`函数可以**将一个 Context 包装为`cancelCtx`**，并**提供一个取消函数**，调用该取消函数可以 Cancel 对应的Context。

`WithCancel`返回带有新Done通道的父节点的副本。当调用返回的cancel函数或当关闭父上下文的Done通道时，将关闭返回上下文的Done通道。

#### Done

```go
func (c *cancelCtx) Done() <-chan struct{} {
    d := c.done.Load()
    if d != nil {
        return d.(chan struct{})
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    d = c.done.Load()
    if d == nil {
        d = make(chan struct{})
        c.done.Store(d)
    }
    return d.(chan struct{})
}
```

- 基于 atomic 包，读取 cancelCtx 中的 chan；倘若已存在，则直接返回；
- 加锁后，再次检查 chan 是否存在，若存在则返回；（double check）
- 初始化 chan 存储到 aotmic.Value 当中，并返回。（懒加载机制）

### timerCtx

#### 数据结构

```go
type timerCtx struct {
    cancelCtx
    
    timer 		*time.Timer 	// 由 cancelCtx.mu 来保护，确保取消操作时线程安全的
    deadline 	time.Time
}
```

在`cancelCtx`基础上，又封装了一个**定时器`timer`**和一个**截止时间`deadline`**，这样及可以根据需要主动取消，也可以在到达 deadline 时通过`timer`触发取消动作。

注意，`timer` 由 `cancelCtx.mu` 来保护，确保取消操作时线程安全的。

#### WithDeadline 和 WithTimeout

`WithDeadline` 和 `WithTimeout` 的函数签名如下：

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

这两个函数都可以创建 `timerCtx`，区别是：

- 前者传入一个**时间点**；
- 后者传入一个**时间段**，函数内再调用`WithDeadline(parent, time.Now().Add(timeout))`。

通常用于**数据库或者网络连接的超时控制**。

### valueCtx

#### 数据结构

```go
type valueCtx struct {
    Context
    
    key, val any
}
```

一个 valueCtx 中仅有一组 kv 对。

#### WithValue

`WithValue`函数签名如下：

```go
func WithValue(parent Context, key, val interface{}) Context
```

`WithValue`创建一个`valueCtx`，其中与key关联的值为val。

#### valueCtx.Value()

```go
func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    return value(c.Context, key)
}
```

- 假如当前 valueCtx 的 key 等于用户传入的 key，则直接返回其 value；
- 假如不等，则**从 parent context 中向上遍历寻找**.

例如，会出现子Context中的key-value覆盖父节点的：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230504162223634.png" alt="image-20230504162223634" style="zoom:50%;" />

- 首先会比较当前Context中的key是否等于要查找的key；
- 此处`keyA==keyC`，所以会直接返回ctxC中的val，因此出现了子节点"覆盖"父节点数据的情况。

为了规避子节点"覆盖"父节点数据的情况，最好不要直接使用string、int等基础类型作为key，而是**用自定义类型包装一下**：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230504162807950.png" alt="image-20230504162807950" style="zoom:50%;" />

- 首先比较当前Context中的key是否等于要查找的key；
- 此处由于类型不同，因此检查不通过，于是向父节点继续查找，进而找到正确的val。

#### 使用规范

- 为了规避子节点"覆盖"父节点数据的情况，最好不要直接使用string、int等基础类型作为key，而是**用自定义类型包装一下**；

- 可以看出，**valueCtx 不适合视为存储介质，存放大量的 kv 数据**，原因有三：

  - 一个 valueCtx 实例只能存一个 kv 对，因此 n 个 kv 对会嵌套 n 个 valueCtx，造成空间浪费；

  - 基于 k 寻找 v 的过程是线性的，时间复杂度 O(N)；

  - 不支持基于 k 的去重，相同 k 可能重复存在，并基于起点的不同，返回不同的 v. 由此得知，valueContext 的定位类似于请求头，只适合存放少量作用域较大的全局 meta 数据.

- Context本身是本着不可改变(immutable)的模式设计的，所以不要视图修改ctx中保存的值。

## 四、注意事项

- 推荐以参数的方式显示传递Context
- 以Context作为参数的函数方法，应该把Context作为第一个参数。
- 给一个函数方法传递Context的时候，不要传递nil，如果不知道传递什么，就使用context.TODO()
- Context的Value相关方法应该传递请求域的必要数据，不应该用于传递可选参数
- Context是线程安全的，可以放心的在多个goroutine中传递

## 五、使用示例

`context`包中定义了四个With系列函数。

### WithCancel

`WithCancel`返回带有新Done通道的父节点的副本。当调用返回的cancel函数或当关闭父上下文的Done通道时，将关闭返回上下文的Done通道。

取消此上下文将释放与其关联的资源，通常使用`defer`来立即调用`cancel()`：

```go
func gen(ctx context.Context) <-chan int {
	dst := make(chan int)
	cnt := 0
	go func() {
		for {
			select {
			case <-ctx.Done():
				log.Println("gen over!")
				return // return结束该goroutine，防止泄露
			case dst <- cnt:
				cnt++
			}
		}
	}()
	return dst
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel() // 当我们取完需要的整数后调用cancel
	for cnt := range gen(ctx) {	
		fmt.Println(cnt)
		if cnt == 5 {
			break
		}
	}
}
```

上面的示例代码中，`gen`函数在单独的goroutine中生成整数并将它们发送到返回的通道。gen的调用者在使用生成的整数之后需要取消上下文，以免`gen`启动的内部goroutine发生泄漏。

下面演示，如何通过 `context` 关闭多个 `goroutine`：

```go
func watchDog(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(name, "已收到停止指令, 马上停止")
			return
		default:
			fmt.Println(name, "正在监控...")
		}
		time.Sleep(1 * time.Second)
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	var wg sync.WaitGroup
	wg.Add(2)
	go func() {
		defer wg.Done()
		watchDog(ctx, "dahuang")
	}()
	go func() {
		defer wg.Done()
		watchDog(ctx, "dabai")
	}()
	time.Sleep(5 * time.Second) // 先让监控狗监控5秒
	cancel()                    // 通知多个 goroutine 退出
	wg.Wait()
}
```

### WithDeadline

取消此上下文将释放与其关联的资源，因此代码应该在此上下文中运行的操作完成后立即调用cancel。

```go
func main() {
	n, d := time.Now(), time.Now().Add(50*time.Millisecond)
	fmt.Printf("当前时间为：%v\n截止时间为：%v\n", n, d)
	ctx, cancel := context.WithDeadline(context.Background(), d)
	// 尽管ctx会过期，但在任何情况下调用它的cancel函数都是很好的习惯。
	// 如果不这样做，可能会使上下文及其父类存活的时间超过必要的时间。
	defer cancel()
	select {
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```

输出：

```bash
$ go run test.go 
当前时间为：2022-09-28 16:58:04.381338 +0800 CST m=+0.001487201
截止时间为：2022-09-28 16:58:04.431338 +0800 CST m=+0.051487201
context deadline exceeded
```

上面的代码中，定义了一个50毫秒之后过期的deadline，然后我们调用`context.WithDeadline(context.Background(), d)`得到一个上下文（ctx）和一个取消函数（cancel），然后使用一个select让主程序陷入等待：等待1秒后打印`overslept`退出或者等待ctx过期后退出。

在上面的示例代码中，因为ctx 50毫秒后就会过期，所以`ctx.Done()`会先接收到context到期通知，并且会打印ctx.Err()的内容。若d在`time.After`后，那么将输出`overslept`。

### WithTimeout

```go
func dbConnect(ctx context.Context) {
	fmt.Printf("db connecting...\n")
	time.Sleep(100 * time.Millisecond)	// 1. 超时
    //time.Sleep(10 * time.Millisecond)	// 2. 不超时
	select {	// 若在ctx规定时间内未连接上，则超时；否则连接成功。
	case <-ctx.Done():
		fmt.Println("db connect failed, err:", ctx.Err())
	default:
		fmt.Println("db connect success!")
	}
	wg.Done()
}

func main() {
	wg.Add(1)
	// 设置数据库超时期限为50ms
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel()
	go dbConnect(ctx)
	wg.Wait() // 等待dbConnect
	fmt.Println("main is over...")
}
```

输出：

```bash
# 超时情况
$ go run test.go 
db connecting...
db connect failed, err: context deadline exceeded
main is over...

# 不超时情况
$ go run test.go 
db connecting...
db connect success!
main is over...
```

### WithValue

```go
func worker(ctx context.Context) {
	key := TraceCode("TRACE_CODE")
	if traceCode, ok := ctx.Value(key).(string); ok {
		log.Println("trade code:", traceCode)
	} else {
		log.Println("invalid key")
	}
	wg.Done()
}

func main() {
	wg.Add(1)
	// 设置数据库超时期限为50ms
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	// 在系统的入口中设置trace code传递给后续启动的goroutine实现日志数据聚合
	ctx = context.WithValue(ctx, TraceCode("TRACE_CODE"), "12512312234")
	defer cancel()
	go worker(ctx)
	wg.Wait() // 等待worker
	fmt.Println("main is over...")
}
```

输出：

```bash
$ go run test.go 
2022/09/28 21:06:02 trade code: 12512312234
main is over...
```

## 六、客户端超时取消实例

客户端调用服务端API时，如何实现超时控制？

**server端：**

```go
func indexHandler(w http.ResponseWriter, r *http.Request) {
	number := rand.Intn(2)
	if number == 0 {
		fmt.Fprintf(w, "slow response")
		time.Sleep(time.Second * 10) // 耗时10s的慢响应
		return
	}
	fmt.Fprintf(w, "quick response")
}

func main() {
	http.HandleFunc("/", indexHandler)
	err := http.ListenAndServe(":8000", nil)
	if err != nil {
		panic(err)
	}
}
```

**client端：**

```go
var wg sync.WaitGroup

type respData struct {
	resp *http.Response
	err  error
}

func doCall(ctx context.Context) {
	transport := http.Transport{
		DisableKeepAlives: true,
	}
	client := http.Client{Transport: &transport}
	req, err := http.NewRequest("GET", "http://127.0.0.1:8000/", nil)
	if err != nil {
		fmt.Println("client create new request failed, err:", err)
		return
	}
	// 使用带超时的ctx创建新的request
	req = req.WithContext(ctx)
	wg.Add(1)
	defer wg.Wait()
	// 调用server端api
	respChan := make(chan *respData, 1)
	go func() {
		resp, err := client.Do(req) // 可能发生超时错误
		if err != nil {
			fmt.Println("client request api failed, err:", err)
			wg.Done()
			return
		}
		respChan <- &respData{
			resp: resp,
			err:  err,
		}
		wg.Done()
	}()
    
	select {
	case <-ctx.Done():	// ctx超时时
		fmt.Println("client call server api timeout...")
	case result := <-respChan:	// 正常调用时
		fmt.Println("client call server api success!")
		if result.err != nil {
			fmt.Println("client call server api failed, err:", result.err)
			return
		}
		data, _ := ioutil.ReadAll(result.resp.Body)
		defer result.resp.Body.Close()
		fmt.Println("resp:", string(data))
	}
}

func main() {
	// 定义一个1s的超时ctx, 超过1s则返回ctx deadline exceed错误
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel() // 调用cancel释放子goroutine资源
	doCall(ctx)
}
```

输出：

```bash
# 1. 正常调用
$ go run client.go 
client call server api success!
resp: quick response

# 2. 超时
$ go run client.go 
client call server api timeout...
client request api failed, err: Get "http://127.0.0.1:8000/": context deadline exceeded
```

## 七、底层原理(有空看吧, 没空算了)

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230430211219356.png" alt="image-20230430211219356" style="zoom:50%;" />

首先要明确的是，**创建 Goroutine 和 Context 时**，都会按照如上的结构，生成父节点到从节点的边，最终形成一种多叉树结构。

### 6.1 核心数据结构

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)	
    Done() <-chan struct{}		// 只读的，用作结束信号
    Err() error					// 只在context结束时，才会产生
    Value(key any) any			// 数据存储，类似 map
}
```

Context 为 interface，定义了四个核心 api：

- Deadline：返回 context 的过期时间；
- Done：返回 context 中的 channel；
- Err：返回错误；
- Value：返回 context 中的对应 key 的值.

#### 6.1.1 error

```go
var Canceled = errors.New("context canceled")

var DeadlineExceeded error = deadlineExceededError{}

type deadlineExceededError struct{}

func (deadlineExceededError) Error() string   { return "context deadline exceeded" }
func (deadlineExceededError) Timeout() bool   { return true }
func (deadlineExceededError) Temporary() bool { return true
```

- Canceled：context 被 cancel 时会报此错误；
- DeadlineExceeded：context 超时时会报此错误.

### 6.2 empty context

#### 6.2.1 类的实现

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}

func (*emptyCtx) Done() <-chan struct{} {
    return nil
}

func (*emptyCtx) Err() error {
    return nil
}

func (*emptyCtx) Value(key any) any {
    return 
}
```

- emptyCtx 是一个空的 context，本质上类型为一个整型；
- Deadline 方法会返回一个公元元年时间以及 false 的 flag，标识当前 context 不存在过期时间；
- Done 方法返回一个 nil 值，用户无论往 nil 中写入或者读取数据，均会陷入阻塞；
- Err 方法返回的错误永远为 nil；
- Value 方法返回的 value 同样永远为 nil.

#### 6.2.2 context.Background() & context.TODO()

```go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

func Background() Context {
    return background
}

func TODO() Context {
    return todo
}
```

我们所常用的 context.Background() 和 context.TODO() 方法返回的**均是 emptyCtx 类型的一个实例**.

### 6.3 cancelCtx

#### 6.3.1 数据结构

```go
type cancelCtx struct {
    Context				// embed 的父 Context，为空

    mu       sync.Mutex            // protects following fields
    done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}

type canceler interface {
    cancel(removeFromParent bool, err error)
    Done() <-chan struct{}
}
```

- embed 了一个 context 作为其父 context. 可见，cancelCtx 必然为某个 context 的子 context；
- 内置了一把锁，用以协调并发场景下的资源获取；
- done：实际类型为 chan struct{}，即用以反映 cancelCtx 生命周期的通道；
- children：一个 set，指向 cancelCtx 的所有子 context；
- err：记录了当前 cancelCtx 的错误. 必然为某个 context 的子 context；

#### 6.3.2 Deadline 方法

cancelCtx 未实现该方法，仅是 embed 了一个带有 Deadline 方法的 Context interface，因此倘若直接调用会报错.

#### 6.3.3 Done 方法

```go
func (c *cancelCtx) Done() <-chan struct{} {
    d := c.done.Load()
    if d != nil {
        return d.(chan struct{})
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    d = c.done.Load()
    if d == nil {
        d = make(chan struct{})
        c.done.Store(d)
    }
    return d.(chan struct{})
}
```

- 基于 atomic 包，读取 cancelCtx 中的 chan；倘若已存在，则直接返回；
- 加锁后，再次检查 chan 是否存在，若存在则返回；（double check）
- 初始化 chan 存储到 aotmic.Value 当中，并返回.（懒加载机制）

#### 6.3.4 Error 方法

```go
func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}
```

- 加锁；
- 读取 cancelCtx.err；
- 解锁；
- 返回结果.

#### 6.3.5 Value 方法

```go
func (c *cancelCtx) Value(key any) any {
    if key == &cancelCtxKey {	 // 系统内部调用的
        return c
    }
    return value(c.Context, key) // 普通用户取数据，就像是 key-value
}
```

- 倘若 key 特定值 &cancelCtxKey，则返回 cancelCtx 自身的指针；
- 否则遵循 valueCtx 的思路取值返回，具体见 2.1.6 小节.

#### 6.3.6 context.WithCancel

##### 6.3.6.1 context.WithCancel()

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}
```

- 校验父 context 非空；
- 注入父 context 构造好一个新的 cancelCtx；
- 在 propagateCancel 方法内启动一个**守护协程**，以保证父 context 终止时，该 cancelCtx 也会被终止；
- 将 cancelCtx 返回，连带返回一个用以终止该 cancelCtx 的闭包函数.

##### 6.3.6.2 newCancelCtx

```go
func newCancelCtx(parent Context) cancelCtx {
    return cancelCtx{Context: parent}
}
```

- 注入父 context 后，返回一个新的 cancelCtx.

##### 6.3.6.3 propagateCancel

```go
func propagateCancel(parent Context, child canceler) {
    done := parent.Done()
    if done == nil {
        return // parent is never canceled
    }

    select {
    case <-done:
        // parent is already canceled
        child.cancel(false, parent.Err())
        return
    default:
    }

    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil {
            // parent has already been canceled
            child.cancel(false, p.err)
        } else {
            if p.children == nil {
                p.children = make(map[canceler]struct{})	// 添加到 children map 即可
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
    } else {
        atomic.AddInt32(&goroutines, +1)
        go func() {
            select {
            case <-parent.Done():	// parent is already canceled
                child.cancel(false, parent.Err())
            case <-child.Done():
            }
        }()
    }
}
```

propagateCancel 方法顾名思义，用以传递父子 context 之间的 cancel 事件：

- 倘若 parent 是不会被 cancel 的类型（如 emptyCtx），则直接返回；
- 倘若 parent 已经被 cancel，则直接终止子 context，并以 parent 的 err 作为子 context 的 err；
- 假如 parent 是 cancelCtx 的类型，则加锁，并将子 context 添加到 parent 的 children map 当中；
- 假如 parent 不是 cancelCtx 类型，但又存在 cancel 的能力（比如用户自定义实现的 context），则启动一个协程，通过多路复用的方式监控 parent 状态，倘若其终止，则同时终止子 context，并透传 parent 的 err.

进一步观察 parentCancelCtx 是如何校验 parent 是否为 cancelCtx 的类型：

```go
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    done := parent.Done()
    if done == closedchan || done == nil {
        return nil, false
    }
    p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
    if !ok {
        return nil, false
    }
    pdone, _ := p.done.Load().(chan struct{})
    if pdone != done {
        return nil, false
    }
    return p, true
}
```

- 倘若 parent 的 channel 已关闭或者是不会被 cancel 的类型，则返回 false；
- 倘若以特定的 cancelCtxKey 从 parent 中取值，取得的 value 是 parent 本身，则返回 true. （基于 cancelCtxKey 为 key 取值时返回 cancelCtx 自身，是 cancelCtx 特有的协议）.

##### 6.3.6.4 cancelCtx.cancel

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // already canceled
    }
    c.err = err
    d, _ := c.done.Load().(chan struct{})
    if d == nil {
        c.done.Store(closedchan)
    } else {
        close(d)
    }
    for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

- cancelCtx.cancel 方法有两个入参，第一个 removeFromParent 是一个 bool 值，表示当前 context 是否需要从父 context 的 children set 中删除；第二个 err 则是 cancel 后需要展示的错误；
- 进入方法主体，首先校验传入的 err 是否为空，若为空则 panic；
- 加锁；
- 校验 cancelCtx 自带的 err 是否已经非空，若非空说明已被 cancel，则解锁返回；
- 将传入的 err 赋给 cancelCtx.err；
- 处理 cancelCtx 的 channel，若 channel 此前未初始化，则直接注入一个 closedChan，否则关闭该 channel；
- 遍历当前 cancelCtx 的 children set，依次将 children context 都进行 cancel；
- 解锁.
- 根据传入的 removeFromParent flag 判断是否需要手动把 cancelCtx 从 parent 的 children set 中移除.

走进 removeChild 方法中，观察如何将 cancelCtx 从 parent 的 children set 中移除：

```go
func removeChild(parent Context, child canceler) {
    p, ok := parentCancelCtx(parent)
    if !ok {
        return
    }
    p.mu.Lock()
    if p.children != nil {
        delete(p.children, child)
    }
    p.mu.Unlock()
}
```

- 如果 parent 不是 cancelCtx，直接返回（因为只有 cancelCtx 才有 children set） 
- 加锁；
- 从 parent 的 children set 中删除对应 child
- 解锁返回.

### 6.4 timerCtx

#### 6.4.1 数据结构

```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.
    deadline time.Time
}
```

timerCtx 在 cancelCtx 基础上又做了一层封装，除了继承 cancelCtx 的能力之外，新增了一个 time.Timer 用于定时终止 context；另外新增了一个 deadline 字段用于字段 timerCtx 的过期时间.

#### 6.4.2 timerCtx.Deadline()

```go
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
    return c.deadline, true
}
```

context.Context interface 下的 Deadline api 仅在 timerCtx 中有效，展示其过期时间.

#### 6.4.3 timerCtx.cancel

```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

- 复用继承的 cancelCtx 的 cancel 能力，进行 cancel 处理；
- 判断是否需要手动从 parent 的 children set 中移除，若是则进行处理
- 加锁；
- 停止 time.Timer
- 解锁返回.

#### 6.4.4 context.WithTimeout & context.WithDeadline

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```

context.WithTimeout 方法用于构造一个 timerCtx，本质上会调用 context.WithDeadline 方法：

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // The current deadline is already sooner than the new one.
        return WithCancel(parent)
    }
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    propagateCancel(parent, c)
    dur := time.Until(d)
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded) // deadline has already passed
        return c, func() { c.cancel(false, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
```

- 校验 parent context 非空；
- 校验 parent 的过期时间是否早于自己，若是，则构造一个 cancelCtx 返回即可；
- 构造出一个新的 timerCtx；
- 启动守护方法，同步 parent 的 cancel 事件到子 context；
- 判断过期时间是否已到，若是，直接 cancel timerCtx，并返回 DeadlineExceeded 的错误；
- 加锁；
- 启动 time.Timer，设定一个延时时间，即达到过期时间后会终止该 timerCtx，并返回 DeadlineExceeded 的错误；
- 解锁；
- 返回 timerCtx，已经一个封装了 cancel 逻辑的闭包 cancel 函数.

### 6.5 valueCtx

#### 6.5.1 数据结构

```go
type valueCtx struct {
    Context
    key, val any
}
```

- valueCtx 同样继承了一个 parent context；
- 一个 valueCtx 中仅有一组 kv 对.

#### 6.5.2 valueCtx.Value()

```go
func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    return value(c.Context, key)
}
```

- 假如当前 valueCtx 的 key 等于用户传入的 key，则直接返回其 value；
- 假如不等，则**从 parent context 中向上遍历寻找**.

```go
func value(c Context, key any) any {
    for {
        switch ctx := c.(type) {
        case *valueCtx:
            if key == ctx.key {
                return ctx.val
            }
            c = ctx.Context
        case *cancelCtx:
            if key == &cancelCtxKey {
                return c
            }
            c = ctx.Context
        case *timerCtx:
            if key == &cancelCtxKey {
                return &ctx.cancelCtx
            }
            c = ctx.Context
        case *emptyCtx:
            return nil
        default:
            return c.Value(key)
        }
    }
}
```

- 启动一个 for 循环，**由下而上，由子及父，依次对 key 进行匹配**；
- 其中 cancelCtx、timerCtx、emptyCtx 类型会有特殊的处理方式；
- 找到匹配的 key，则将该组 value 进行返回.

#### 6.5.3 valueCtx 用法小结

可以看出，**valueCtx 不适合视为存储介质，存放大量的 kv 数据**，原因有三：

- 一个 valueCtx 实例只能存一个 kv 对，因此 n 个 kv 对会嵌套 n 个 valueCtx，造成空间浪费；
- 基于 k 寻找 v 的过程是线性的，时间复杂度 O(N)；
- 不支持基于 k 的去重，相同 k 可能重复存在，并基于起点的不同，返回不同的 v. 由此得知，valueContext 的定位类似于请求头，只适合存放少量作用域较大的全局 meta 数据.

#### 6.5.4 context.WithValue()

```go
func WithValue(parent Context, key, val any) Context {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    if key == nil {
        panic("nil key")
    }
    if !reflectlite.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}
}
```

- 倘若 parent context 为空，panic；
- 倘若 key 为空 panic；
- 倘若 key 的类型不可比较，panic；
- 包括 parent context 以及 kv对，返回一个新的 valueCtx.

参考文章：

- https://mp.weixin.qq.com/s?__biz=MzkxMjQzMjA0OQ==&mid=2247483677&idx=1&sn=d1c0e52b1fd31932867ec9b1d00f4ec2&chksm=c10c4fc3f67bc6d590141040342153004ea4e420d83f4e2b2afc068e8bb904778b02b5be3f97&scene=126&sessionid=1682837915#rd

