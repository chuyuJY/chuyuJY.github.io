# Go-内存逃逸


[toc]

## 一、概述

Go 编译器会尽可能将变量分配到到栈上。但是，

- 当编译器无法证明函数返回后，**该变量没有被引用**，那么编译器就必须在堆上分配该变量，以此避免悬挂指针(dangling pointer)；
- 另外，如果**局部变量非常大**，也会将其分配在堆上。

有如下规则可以参考：

- 逃逸分析是在编译器完成的，这是不同于jvm的运行时逃逸分析;
- 如果变量在函数外部没有引用，则优先放到栈中；
- 如果变量在函数外部存在引用，则必定放在堆中。

## 二、案例

给出一些常见的内存逃逸情况。

可通过`go build -gcflags '-m -l'`命令来查看逃逸分析结果，其中，`-m` 打印逃逸分析信息，`-l` 禁止内联优化：

```go
go build -gcflags '-m -l' 01-type.go
go build -gcflags '-m -m -l' 01-type.go	// 更详细
```

### 变量类型不确定

```go
// 变量类型不确定
func main() {
	a := 123
	fmt.Println(a)
}
```

分析结果：

```go
❯ go build -gcflags '-m -l' 01-type.go
# command-line-arguments
.\01-type.go:11:13: ... argument does not escape
.\01-type.go:11:13: a escapes to heap

❯ go build -gcflags '-m -m -l' 01-type.go
# command-line-arguments
.\01-type.go:11:13: a escapes to heap:
.\01-type.go:11:13:   flow: {storage for ... argument} = &{storage for a}:
.\01-type.go:11:13:     from a (spill) at .\01-type.go:11:13
.\01-type.go:11:13:     from ... argument (slice-literal-element) at .\01-type.go:11:13
.\01-type.go:11:13:   flow: {heap} = {storage for ... argument}:
.\01-type.go:11:13:     from ... argument (spill) at .\01-type.go:11:13
.\01-type.go:11:13:     from fmt.Println(... argument...) (call parameter) at .\01-type.go:11:13
.\01-type.go:11:13: ... argument does not escape
.\01-type.go:11:13: a escapes to heap
```

分析结果告诉我们变量`a`逃逸到了堆上，`a`逃逸是因为它被传入了`fmt.Println`的参数中，这个方法参数自己发生了逃逸。

```go
func Println(a ...any) (n int, err error)
```

因为`fmt.Println`的函数参数为`interface`类型，**编译期不能确定其参数的具体类型**，也就**不确定开辟多大的空间**给它，所以将其分配于堆上。

### 暴露给外部指针

```go
func foo() *int {
	a := 123
	return &a
}

func main() {
	_ = foo()
}
```

分析结果：

```go
❯ go build -gcflags '-m -l' 02-expose.go
# command-line-arguments
.\02-expose.go:4:2: moved to heap: a

❯ go build -gcflags '-m -m -l' 02-expose.go
# command-line-arguments
.\02-expose.go:4:2: a escapes to heap:
.\02-expose.go:4:2:   flow: ~r0 = &a:
.\02-expose.go:4:2:     from &a (address-of) at .\02-expose.go:5:9
.\02-expose.go:4:2:     from return &a (return) at .\02-expose.go:5:2
.\02-expose.go:4:2: moved to heap: a
```

直接满足：**变量在函数外部存在引用**。这个很好理解，因为**当函数执行完毕，对应的栈帧就被销毁**，但是引用已经被返回到函数之外。如果这时外部从引用地址取值，虽然地址还在，但是这块内存已经被释放回收了，这就是非法内存，问题可就大了。所以，很明显，这种情况**必须分配到堆上**。

### 变量所占内存较大

```go
// []int 太大, 导致逃逸
func foo() {
	s := make([]int, 8193, 8193)	// > 8192 就是大对象了
	for i := 0; i < len(s); i++ {
		s[i] = i
	}
}

// 外部引用, 导致逃逸
//func foo() []int {
//	s := make([]int, 100, 100)
//	for i := 0; i < len(s); i++ {
//		s[i] = i
//	}
//	return s
//}

func main() {
	foo()
}
```

分析结果：

```go
❯ go build -gcflags '-m -l' 03-big.go
# command-line-arguments
.\03-big.go:5:11: make([]int, 10000, 10000) escapes to heap

❯ go build -gcflags '-m -m -l' 03-big.go
# command-line-arguments
.\03-big.go:5:11: make([]int, 10000, 10000) escapes to heap:
.\03-big.go:5:11:   flow: {heap} = &{storage for make([]int, 10000, 10000)}:
.\03-big.go:5:11:     from make([]int, 10000, 10000) (too large for stack) at .\03-big.go:5:11
.\03-big.go:5:11: make([]int, 10000, 10000) escapes to heap
```

**逃逸信息**：`too large for stack`。当我们创建了一个容量为8193的`int`类型的底层数组对象时，由于对象过大，它也会被分配到堆上。

**那为啥大对象需要分配到堆上？**

`goroutine` 初始大小为`2KB`，其实说的是用户栈，它的最小和最大可以在`runtime/stack.go`中找到，分别是`2KB`和`1GB`，而**堆内存会大很多**。因此，为了**不造成栈溢出和频繁的扩缩容**，大对象分配在堆上更加合理(大对象的范围：`> 32KB`)，所以`s :=make([]int, n, n)`中，一旦`n > 8192`，就一定会逃逸。

### 变量大小不确定

```go
func foo() {
	n := 1
	s := make([]int, n)
	for i := 0; i < len(s); i++ {
		s[i] = i
	}
}

func main() {
	foo()
}
```

分析结果：

```go
❯ go build -gcflags '-m -l' 04-uncertain.go
# command-line-arguments
.\04-uncertain.go:5:11: make([]int, n) escapes to heap

❯ go build -gcflags '-m -m -l' 04-uncertain.go
# command-line-arguments
.\04-uncertain.go:5:11: make([]int, n) escapes to heap:
.\04-uncertain.go:5:11:   flow: {heap} = &{storage for make([]int, n)}:
.\04-uncertain.go:5:11:     from make([]int, n) (non-constant size) at .\04-uncertain.go:5:11
.\04-uncertain.go:5:11: make([]int, n) escapes to heap
```

**逃逸信息**：`non-constant size`。在`make`方法中，没有直接指定大小，而是填入了变量`n`，这时也会将其分配到堆区去。

可见，为了保证内存的绝对安全，Go的编译器可能会将一些变量不合时宜地分配到堆上，但是因为这些对象最终也会被垃圾收集器处理，所以也能接受。

### 闭包

```go
func foo() func() int {
	i := 0
	return func() int {
		i++
		return i
	}
}

func main() {
	foo()()
}
```

分析结果：

```go
❯ go build -gcflags '-m -l' 05-close.go
# command-line-arguments
.\05-close.go:4:2: moved to heap: i
.\05-close.go:5:9: func literal escapes to heap

❯ go build -gcflags '-m -m -l' 05-close.go
# command-line-arguments
.\05-close.go:4:2: foo capturing by ref: i (addr=false assign=true width=8)
.\05-close.go:5:9: func literal escapes to heap:
.\05-close.go:5:9:   flow: ~r0 = &{storage for func literal}:
.\05-close.go:5:9:     from func literal (spill) at .\05-close.go:5:9
.\05-close.go:5:9:     from return func literal (return) at .\05-close.go:5:2
.\05-close.go:4:2: i escapes to heap:
.\05-close.go:4:2:   flow: {storage for func literal} = &i:
.\05-close.go:4:2:     from i (captured by a closure) at .\05-close.go:6:3
.\05-close.go:4:2:     from i (reference) at .\05-close.go:6:3
.\05-close.go:4:2: moved to heap: i
.\05-close.go:5:9: func literal escapes to heap
```

原本在函数运行栈空间上分配的内存，由于闭包的关系，变量在函数的作用域之外使用。

## 三、总结

> #### 问：进行逃逸分析有啥用？

理解逃逸分析一定能帮助我们写出更好的程序。

知道变量分配在栈堆之上的差别，那么就要**尽量写出分配在栈上**的代码，堆上的变量变少了，可以减轻内存分配的开销，减小gc的压力，提高程序的运行速度。

所以，有些Go的项目，它们**在函数传参的时候**，**并没有传递结构体指针**，**而是直接传递的结构体**。这个做法，虽然它需要值拷贝，但这是**在栈上完成**的操作，**开销远比变量逃逸后动态地在堆上分配内存少的多**。当然该做法不是绝对的，**如果结构体较大，传递指针将更合适**。

因此，从GC的角度来看，指针传递是个双刃剑，需要谨慎使用，否则线上调优解决GC延时可能会让你崩溃。

> 参考文章：
>
> - [详解逃逸分析](https://mp.weixin.qq.com/s/58aqlX92ho0Z8KLrbURRfg)

