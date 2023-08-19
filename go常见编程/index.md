# Go浅析-常见编程操作


## 使用方法名字符串，调用方法

思路：通过 `reflect`。

```go
type animal struct {
	name string
}

func (a *animal) Eat() {
	println("animal eat")
}

func main() {
	a := animal{"cat"}
	reflect.ValueOf(&a).MethodByName("Eat").Call([]reflect.Value{})
}
```

## 3个 goroutine 按照顺序分别打印10次"cat"，"fish"，"dog

思路：若限制只用 3 个 `goroutine`，如下

```go
var (
	catChan  = make(chan int)
	fishChan = make(chan int)
	dogChan  = make(chan int)
	allChan  = make(chan struct{})
)

func cat(ch chan int) {
	for v := range ch {
		fmt.Printf("%v cat\n", v)
		fishChan <- v
		if v == 10 {
			break
		}
	}
}

func fish(ch chan int) {
	for v := range ch {
		fmt.Printf("%v fish\n", v)
		dogChan <- v
		if v == 10 {
			break
		}
	}
}

func dog(ch chan int) {
	for v := range ch {
		fmt.Printf("%v dog\n", v)
		if v == 10 {
			allChan <- struct{}{}
			break
		}
		catChan <- v + 1
	}
}

func main() {
	go cat(catChan)
	go fish(fishChan)
	go dog(dogChan)
	catChan <- 1
	<-allChan
}
```

思路：若不限制 `goroutine` 数目

```go
var (
	catChan  = make(chan int)
	fishChan = make(chan int)
	dogChan  = make(chan int)
	allChan  = make(chan struct{})
)

func cat() {
	v := <-catChan
	fmt.Printf("%v cat\n", v)
	fishChan <- v
}

func fish() {
	v := <-fishChan
	fmt.Printf("%v fish\n", v)
	dogChan <- v
}

func dog() {
	v := <-dogChan
	fmt.Printf("%v dog\n", v)
	if v == 10 {
		allChan <- struct{}{}
		return
	}
	catChan <- v + 1	// 注意是无缓冲的 chan，若放到 if 前，就会死锁！
}

func main() {
	for i := 0; i < 10; i++ {
		go cat()
		go fish()
		go dog()
	}
	catChan <- 1
	<-allChan
}
```

## 启动 2 个 goroutine 2 秒后取消，第一个 1 秒执行完，第二个 3 秒执行完

```go
func f1(ch chan struct{}) {
	time.Sleep(time.Second * 1)
	ch <- struct{}{}
}

func f2(ch chan struct{}) {
	time.Sleep(time.Second * 3)
	ch <- struct{}{}
}

func main() {
	ctx, _ := context.WithTimeout(context.Background(), time.Second*2)
	ch := make(chan struct{}, 1)
	go func() {
		go f1(ch)
		select {
		case <-ctx.Done():
			println("f1 timeout")
		case <-ch:
			println("f1 done")
		}
	}()
	go func() {
		go f2(ch)
		select {
		case <-ctx.Done():
			println("f2 timeout")
		case <-ch:
			println("f2 done")
		}
	}()
	time.Sleep(time.Second * 5)
}
```

## 2 个 goroutine 交替打印 10 个字母和数字

```go
var (
	ch1 = make(chan byte)
	ch2 = make(chan int)
)

func letter() {
	for i := 0; i < 10; i++ {
		v := <-ch1
		fmt.Println(string(v))
		ch2 <- int(v - 'a' + 1)
	}

}

func number() {
	for i := 0; i < 10; i++ {
		v := <-ch2
		fmt.Println(v)
		if v == 10 {
			return
		}
		ch1 <- byte(v + 'a')
	}
}

func main() {
	go letter()
	go number()
	ch1 <- 'a'
	time.Sleep(time.Second * 2)
}
```

## 当 select 监控多个 chan 同时到达就绪态时，如何先执行某个任务？

可通过添加标签

```go
func prioritySelect(ch1, ch2 <-chan string) {
	for {
		select {
		case val := <-ch1:
			fmt.Println(val)
		case val2 := <-ch2:
		priority:	// 标签
			for {
				select {
				// 若 ch1 有值，先打印 ch1 的值，再打印 ch2 的值
				case val1 := <-ch1:
					fmt.Println(val1)
				default:
					break priority
				}
			}
			fmt.Println(val2)
		}
	}
}
```


