# Go-基础操作


# **Go实践操作**

[toc]

## 二、基础

**记录遇到的问题：**

当使用`go get`安装包之后，在pkg中也可找到，但还是无法使用，显示unresolved。这其实是因为你的项目没有external libraries，所以`go get`或`go build`的包无法在当前目录找到，只需要：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220713214642714.png" alt="image-20220713214642714" style="zoom:50%;" />

### 1. 结构体

结构体变量在内存中的布局：

> 结构体的所有字段在内存中是连续的。

`2022/01/24`

> 看别的视频    **接口与多态**  

**多态：**

> 不同的类拥有同名的方法，方法有所不同。同一个语句 不同类型的对象可以调用各自类型的方法输出不同的结果

**接口：	**

> 1. 避免不必要的信息泄露 
>
> 2. 只能调用接口中有的方法 不能调用原对象拥有的方法
> 3. 给一个接口赋值的时候 需要采用指针   

``` go
	// 定义一个接口 注意类型是 interface
type IAttack interface {
	// 接口函数可以有多个 但是只能有函数原型 不可以有实现
	Attack()
}
	// 使用接口
	var player IAttack // 定义一个包含Attack的接口变量
	// 对player赋值为lowLevel  接口需要使用指针类型来赋值   接口只能访问接口中的方法
	player = &lowlevel
```

**常量**

**基础介绍：**

- 常量使用 `const` 修饰
- 常量在定义的时候必须初始化
- 常量一经定义就不能修改
- 常量只能修饰bool、数值类型(int, float等)、string类型
- 通过大小写控制常量的访问范围
- 语法：`const 常量名 常量type = value`
- 写法：

```go
1. 比较简洁的写法
const (
	a = 1
    b = 1
)

2. 更专业
const (
	a = iota  	// 表示给 a 赋值为 0， b 在 a 的基础上 +1，c 在 b 的基础上 +1 
    b
    c
)
```

```go
func main() {
	const (
		a = iota
		b
		c
	)
	fmt.Println(a, b, c) // 0, 1, 2
}
```

### 2. 接口

**核心中的核心。**

类似于**多态**。

1. **基本语法：**

```go
// 定义一个接口
type 接口名 interface {
    method1(参数列表) 返回值列表
    method2(参数列表) 返回值列表
}

// 实现接口
func (t 自定义类型) method1(参数列表) 返回值列表 {
    // 方法实现
} 

func (t 自定义类型) method2(参数列表) 返回值列表 {
    // 方法实现
} 
```

**注：interface不能包括任何变量**

```go
// 示例：
type Usb interface {
	Start()
	Stop()
}

// Phone 2. Phone实现Usb接口方法
type Phone struct {
}

func (p Phone) Start() {
	fmt.Println("Phone开始工作...")
}

func (p Phone) Stop() {
	fmt.Println("Phone停止工作...")
}
```

**注：**golang中的接口**不需要显式实现**。**只要一个变量（结构体），含有接口类型中的所有方法，那么这个变量就实现了这个接口。**。因此，golang中没有Java中的implement关键字。

**所有类型**都实现了**空接口**，我们**可以把任何一个变量赋给空接口**。

2. **接口vs继承**

接口可看作是继承的一种**补充**。**结构体是个体的抽象，接口是行为的抽象。**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220630095748505.png" alt="image-20220630095748505" style="zoom: 40%;" />

**接口和继承解决的问题不同：**

- **继承：**解决代码的复用性和可维护性
- **接口：**设计，设计好各种规范（方法），让其他自定义类型去实现这些方法。

接口比继承更加灵活，继承是满足 `is a` 的关系，而接口只需满足 `like a` 的关系。

接口在一定程度实现代码解耦。

### 3. 多态

Go中的多态通过接口实现。可以按照统一的接口来调用不同的实现，这时接口变量就呈现不同的形态。比如：不同结构体的排序。

#### 3.1 接口体现多态的两种形式

1. 多态参数

比如不同结构体的排序。

2. 多态数组

一个接口数组，可以存放不同的结构体对象。

#### 3.2 类型断言

> **类型断言：**由于一个接口是一般类型，不知道具体类型，如果要转成具体类型，就需要使用类型断言。

```go
type Point struct {
	x int
	y int
}

func main() {
	var a interface{}
	point := Point{1, 2}
	a = point
	var b Point
	// 这样是不可以的
	//b = a
    // 类型断言
	b = a.(Point)
	fmt.Println(b)
}
```

`b = a.(Point)` 就是类型断言，表示判断a是否指向Point类型的变量，如果是就转成Point类型并赋给b变量，否则报错。

**带检测机制的类型断言：**

```go
	b, ok := a.(Point)
	if ok {
		fmt.Println(b)
	} else {
		fmt.Println("类型断言出错！")
	}
```

**类型断言实践：**

```go
func typeJudge(items ...interface{}) {
	for index, x := range items {
		// 类型断言
		switch x.(type) {
		case bool:
			fmt.Printf("第%v个参数是bool类型，值是%v\n", index, x)
		case float64:
			fmt.Printf("第%v个参数是float64类型，值是%v\n", index, x)
		case float32:
			fmt.Printf("第%v个参数是float32类型，值是%v\n", index, x)
		case int, int32, int64:
			fmt.Printf("第%v个参数是int类型，值是%v\n", index, x)
		case string:
			fmt.Printf("第%v个参数是string类型，值是%v\n", index, x)
		default:
			fmt.Printf("第%v个参数是 不确定 类型，值是%v\n", index, x)
		}
	}
}

func main() {
	var n1 float32 = 1.1
	var n2 float64 = 2.2
	var n3 int32 = 3
	n4 := "北京"
	n5 := 300
	var n6 string = "nihao"
	typeJudge(n1, n2, n3, n4, n5, n6)
}
```

### 4. 文件操作

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220630110843132.png" alt="image-20220630110843132" style="zoom:50%;" />

#### 4.1 基本介绍

`os.File` 封装所有文件相关操作，`File` 是一个结构体。

1. **打开文件**

`os.Open()`

```go
	file, err := os.Open("./test.txt")
	if err != nil {
		fmt.Println(err)
	}
```

2. **关闭文件**

`file对象.Close()`

file 其实是个指针。

```go
	err = file.Close()
	if err != nil {
		fmt.Println(err)
	}
```

3. **读文件**

- 带缓冲
- 不带缓冲

**`ioutil.ReadFile`一次性读取：**

```go
	file := "./test.txt"
	content, err := ioutil.ReadFile(file)
	if err != nil {
		fmt.Println("read file err = ", err)
	}
	fmt.Println(string(content))	// 读取内容为[]byte
	// 没有显式open文件，此处也不必close，因为文件的open和close被封装到了ReadFile中
	// 这种方式很简洁，但文件很大的时候，将会效率很低，因为是一次读取所有内容
```

4. **写文件**

```
func OpenFile(name string, flag int, perm FileMode) (file *File, err error)
```

其中，`name`指定文件，`flag`代表打开模式（可以组合），`perm`权限控制（unix）

```go
const (
    O_RDONLY int = syscall.O_RDONLY // 只读模式打开文件
    O_WRONLY int = syscall.O_WRONLY // 只写模式打开文件
    O_RDWR   int = syscall.O_RDWR   // 读写模式打开文件
    O_APPEND int = syscall.O_APPEND // 写操作时将数据附加到文件尾部
    O_CREATE int = syscall.O_CREAT  // 如果不存在将创建一个新文件
    O_EXCL   int = syscall.O_EXCL   // 和O_CREATE配合使用，文件必须不存在
    O_SYNC   int = syscall.O_SYNC   // 打开文件用于同步I/O
    O_TRUNC  int = syscall.O_TRUNC  // 如果可能，打开时清空文件
)
```

实例：

- 创建一个新文件，写入 5 句 `hello, world!`.

```go
func writeNew() {
	// 1. 打开文件
	filePath := "testOpen.txt"
	file, _ := os.OpenFile(filePath, os.O_WRONLY|os.O_CREATE, 0666)
	// 2. 写入
	str := "hello, world!\n"
	// 带缓存的 *Writter
	writer := bufio.NewWriter(file)
	for i := 0; i < 5; i++ {
		writer.WriteString(str)
	}
	// 因为writter是带缓存的，因此在调用WritterString时，
	// 其实内容是先写到缓存的 ，所以需要调用Flush方法将缓存的数据，
	// 真正写到磁盘
	writer.Flush()
}
```

- 在已存在文件后，追加 `hello, JY!`.

```go
func writeAppend() {
	// 1. 打开文件
	filePath := "testOpen.txt"
	file, _ := os.OpenFile(filePath, os.O_WRONLY|os.O_APPEND, 0666)
	// 2. 写入
	str := "hello, JY!\r\n"
	// 带缓存的 *Writter
	writer := bufio.NewWriter(file)
	for i := 0; i < 5; i++ {
		writer.WriteString(str)
	}
	// 因为writter是带缓存的，因此在调用WritterString时，
	// 其实内容是先写到缓存的 ，所以需要调用Flush方法将缓存的数据，
	// 真正写到磁盘
	writer.Flush()
}
```

**注意，只是打开文件的模式不同。**

- 边读边追加

```go
func writeAndRead() {
	// 1. 打开文件
	filePath := "testOpen.txt"
	file, _ := os.OpenFile(filePath, os.O_RDWR|os.O_APPEND, 0666)
	defer file.Close()
	// 2. 先读取原来的内容并显示在终端
	reader := bufio.NewReader(file)
	for {
		str, err := reader.ReadString('\n')
		if err == io.EOF { // 若读取到文件末尾
			break
		}
		fmt.Print(str) // 打印
	}
	// 3. 追加
	str := "你好, JY!\r\n"
	// 带缓存的 *Writter
	writer := bufio.NewWriter(file)
	for i := 0; i < 5; i++ {
		writer.WriteString(str)
	}
	// 因为writter是带缓存的，因此在调用WritterString时，
	// 其实内容是先写到缓存的 ，所以需要调用Flush方法将缓存的数据，
	// 真正写到磁盘
	writer.Flush()
}
```

5. **判断文件是否存在**

自定义一个函数：

```go
func PathExists(path string) (bool, error) {
	_, err := os.Stat(path)
	if err != nil {	// 文件目录存在
		return true, nil
	}
	if os.IsNotExist(err) {
		return false, err
	}
	return false, err
}
```

6. **拷贝文件**

- 将一个已存在的文件内容写入到另一个文件中。

```go
// 将file1的内容复制到file2
func readAndWrite() {
	file1Path := "./test.txt"
	file2Path := "./testOpen.txt"

	data, err := ioutil.ReadFile(file1Path)
	if err != nil {
		fmt.Println("read file err = ", err)
		return
	}
	err = ioutil.WriteFile(file2Path, data, 0666)
	if err != nil {
		fmt.Println("write file err = ", err)
		return
	}
}
```

- 拷贝视频等

`func Copy(dst Writer, src Reader) (written int64, err error)`

```go
func copyFile(srcFileName, dstFileName string) (int64, error) {
	srcFile, err := os.Open(srcFileName)
	if err != nil {
		fmt.Println("open file err = ", err)
		return 0, err
	}
	defer srcFile.Close()
	dstFile, err := os.OpenFile(dstFileName, os.O_WRONLY|os.O_CREATE, 0666)
	if err != nil {
		fmt.Println("open file err = ", err)
		return 0, err
	}
	defer dstFile.Close()
	// 拿到reader和writer
	reader := bufio.NewReader(srcFile)
	writer := bufio.NewWriter(dstFile)
	// copy
	return io.Copy(writer, reader)
}
```

#### 4.2 实例

> 统计文件中有多少个英文、数字、空格和其他字符

```go
func fileCount(fileName string) (Record, error) {
	/*
		1. 打开一个文件，创建reader
		2. 逐行统计
		3. 记录在结构体
	*/
	file, err := os.Open(fileName)
	if err != nil {
		fmt.Println("open file err = ", err)
		return Record{}, err
	}
	defer file.Close()
	record := Record{}
	reader := bufio.NewReader(file)
	for {
		str, err := reader.ReadString('\n')
		// 为了兼容中文，可以将string转[]rune
		// 遍历str，进行统计
		for _, v := range str {
			switch {
			case 'a' <= v && v <= 'z':
				fallthrough // 穿透
			case 'A' <= v && v <= 'Z':
				record.CharCount++
			case v == ' ' || v == '\t':
				record.SpaceCount++
			case '0' <= v && v <= '9':
				record.NumCount++
			default:
				record.OtherCount++
			}
		}
		if err == io.EOF {
			break
		}
	}
	return record, nil
}
```

### 5. flag

> 解析命令行参数

```go
func parseCommand() {
	// 1. 定义几个变量用来存放命令行参数
	var (
		user string
		pwd  string
		host string
		port int
	)
	// 2. 转换
	/*
		&user: 接收用户命令行输入的 -u 后面的参数值
		"u": -u 指定参数
		"": 默认参数
		"用户名，默认为空": 说明
	*/
	flag.StringVar(&user, "u", "", "用户名，默认为空")
	flag.StringVar(&pwd, "pwd", "", "密码，默认为空")
	flag.StringVar(&host, "h", "localhost", "主机名，默认为localhost")
	flag.IntVar(&port, "port", 3306, "端口号，默认为3306")
	flag.Parse() // 一定要调用
	// 3. 输出结果
	fmt.Printf("user=%v pwd=%v host=%v port=%v", user, pwd, host, port)
}

$ go run 06-flag.go -u root -pwd 123456 -h 127.0.0.1 -port 3300
user=root pwd=123456 host=127.0.0.1 port=3300
$ go run 06-flag.go -u root -pwd 123456
user=root pwd=123456 host=localhost port=3306
```

### 6. json

#### 6.1 基础使用

> 轻量级的数据交换格式，易于机器解析，可提升网络传输效率。通常在网络传输时会先将数据（数据结构、map等）**序列化**成json字符串，到接收方得到json字符串时，再**反序列化**成原来的数据格式（数据结构、map等）。

**应用场景：**

1. 

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220707175741363.png" alt="image-20220707175741363" style="zoom: 50%;" />

2. 

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220707180257575.png" alt="image-20220707180257575" style="zoom:50%;" />

#### 6.2 序列化

> 将**数据结构**序列化为**json格式**。

```go
/*
	1. 将结构体序列化
	2. 将map序列化
	3. 对Slice序列化
	4. 对基本数据类型进行序列化（意义不大）
*/
```

```
data, err := json.Marshal()
```

#### 6.3 tag的使用

> **原理：**利用`reflect机制`。
>
> **目的：**如果我们希望序列化后`key`的名字是`指定的名字`，**同时不受go的命名限制，**可以指定`tag标签`。

```go
type Monster struct {
	Name     string  `json:"name" `
	Age      int     `json:"age" `
	Birthday string  `json:"birthday"`
	Salary   float64 `json:"salary"`
	Skill    string  `json:"skill"`
}

// 1. 将结构体序列化
type Monster struct {
	Name     string  `json:"name" `
	Age      int     `json:"age" `
	Birthday string  `json:"birthday"`
	Salary   float64 `json:"salary"`
	Skill    string  `json:"skill"`
}

func testStruct() {
	monster := Monster{
		Name:     "牛魔王",
		Age:      18,
		Birthday: "2011-11-11",
		Salary:   3000.0,
		Skill:    "牛魔拳",
	}

	// 将monster 序列化
	data, err := json.Marshal(&monster)
	if err != nil {
		fmt.Println("序列化错误 err = ", err)
	}
	fmt.Println("序列化的结果：", string(data))
}

// 2. 将map序列化
func testMap() {
	a := make(map[string]interface{})
	a["name"] = "红孩儿"
	a["age"] = 30
	a["address"] = "洪崖洞"

	data, err := json.Marshal(a)
	if err != nil {
		fmt.Println("序列化错误 err = ", err)
	}
	fmt.Println("a map 序列化后为：", string(data))
}

// 3. 对Slice序列化
func testSlice() {
	slice := []map[string]interface{}{}
	m1 := map[string]interface{}{}
	m1["name"] = "jack"
	m1["age"] = 10
	m1["address"] = []string{"墨西哥", "夏威夷"}
	m2 := map[string]interface{}{}
	m2["name"] = "chuyu"
	m2["age"] = 18
	m2["address"] = "南京"

	slice = append(slice, m1, m2)
	data, err := json.Marshal(slice)
	if err != nil {
		fmt.Println("序列化错误 err = ", err)
	}
	fmt.Println("slice 序列化后为：", string(data))
}

// 4. 对基本数据类型进行序列化(意义不大)
func testFloat64() {
	num1 := 1234.5678
	data, err := json.Marshal(num1)
	if err != nil {
		fmt.Println("序列化错误 err = ", err)
	}
	fmt.Println("num1 序列化结果为：", string(data))
}

序列化的结果： {"name":"牛魔王","age":18,"birthday":"2011-11-11","salary":3000,"skill":"牛魔拳"}
```

#### 6.4 反序列化

> 将**json格式**反序列化成对应的**数据结构**（结构体、map、Slice）。

```
err := json.Unmarshal([]byte(str), &monster)
```

**注意点：**

- 反序列化一个json字符串时，需要保证**反序列化后的数据类型**和**序列化前的数据类型**一致**（结构体须保证字段完全一致）**。
- json字符串如果是传输过来的（本来就是json字符串），无需进行转义处理。

```go
/*
	1. 反序列化为struct
	2. 反序列化为map
	3. 反序列化为slice
*/

// 1. 反序列化为 struct
type Monster struct {
	Name     string  `json:"name" `
	Age      int     `json:"age" `
	Birthday string  `json:"birthday"`
	Salary   float64 `json:"salary"`
	Skill    string  `json:"skill"`
}

func unMarshalStruct() {
	// 1. 得到序列化后的字符串，比如网络传输过来的
	str := "{\"name\":\"牛魔王\",\"age\":18," +
		"\"birthday\":\"2011-11-11\",\"salary\":3000,\"skill\":\"牛魔拳\"}"
	monster := Monster{}
	err := json.Unmarshal([]byte(str), &monster)
	if err != nil {
		fmt.Println("反序列化错误 err = ", err)
	}
	fmt.Printf("反序列化的结果：%v\n", monster)
}

// 2. 反序列化为Map
func unMarshalMap() {
	str := " {\"address\":\"洪崖洞\",\"age\":30,\"name\":\"红孩儿\"}"
	a := map[string]interface{}{}
	err := json.Unmarshal([]byte(str), &a)
	if err != nil {
		fmt.Println("反序列化失败 err = ", err)
	}
	fmt.Printf("序列化结果为：%v\n", a)
}

// 3. 反序列化为Slice
func unMarshalSlice() {
	str := "[{\"address\":[\"墨西哥\",\"夏威夷\"],\"age\":10," +
		"\"name\":\"jack\"},{\"address\":\"南京\",\"age\":18,\"name\":\"chuyu\"}]"
	slice := []map[string]interface{}{}
	err := json.Unmarshal([]byte(str), &slice)
	if err != nil {
		fmt.Println("反序列化失败 err = ", err)
	}
	fmt.Printf("反序列化结果为：%v\n", slice)
}
```

### 7. 单元测试

测试一个函数是否有效，传统做法就是调用一下他，观察结果是否符合预期，但这样具有缺点：

- 不方便，还需要自己编写实例去测试，需要在main函数中调用代码，也就是需要去改动代码，若项目正在运行，那么有可能还需要停下项目；
- 不利于管理，当测试多个函数或模块时，都需要写在main函数，不利于管理。

Go带有一个轻量级的测试框架`testing`和自带的`go test`来实现单元测试和性能测试，`testing`框架和其它语言中的测试框架相似。

- 单元测试主要找到逻辑漏洞；
- 性能测试主要测试程序在高并发高压力环境下的运行状况。

`testing` 提供对 Go 包的自动化测试的支持。通过 `go test` 命令，能够自动执行如下形式的任何函数：

```
func TestXxx(*testing.T)
```

其中 Xxx 可以是任何字母数字字符串（但第一个字母不能是 [a-z]），用于识别测试例程。

要编写一个新的测试套件，需要创建一个名称以 _test.go 结尾的文件，该文件包含 `TestXxx` 函数。

**示例：**

**文件名格式：`xxx_test.go`**

```go
package test

import "testing"
// 一个测试文件中可含有多个测试用例

// 1. 第一个测试函数
func addUpper(n int) int {
	res := 0
	for i := 1; i <= n; i++ {
		res += i
	}
	return res
}

// 第一个测试用例
func TestGetSub(t *testing.T) {
	res := getSub(10, 7)	// 写在别的文件中

	// 若错误
	if res != 5 {
		t.Fatalf("getSub(10, 7) 执行错误 期望值 = %v 实际值 = %v\n", 5, res)
	}
	// 若正确
	t.Logf("getSub(10, 7) 执行正确...")
}
// 第二个测试用例
func TestAddUpper(t *testing.T) {
	// 调用
	res := addUpper(10)
	// 若错误
	if res != 55 {
		t.Fatalf("addUpper(10) 执行错误 期望值 = %v 实际值 = %v\n", 55, res)
	}
	// 若正确
	t.Logf("addUpper(10) 执行正确...")
}
// 不需要main函数


$ go test -v
=== RUN   TestGetSub
    09-add_test.go:19: getSub(10, 7) 执行错误 期望值 = 5 实际值 = 3
--- FAIL: TestGetSub (0.00s)
=== RUN   TestAddUpper
    09-add_test.go:33: addUpper(10) 执行正确...
--- PASS: TestAddUpper (0.00s)
FAIL
exit status 1
FAIL    goLearn/test    0.157s

-v：无论正确与否，都输出日志
```

**注意点：**

- 测试文件名：`xxx_test.go`
- 测试用例名：`func TestXxx(*testing.T)`

- 若有多个测试文件，可以用`go test -v 指定测试文件名 原文件`测试指定的文件。

- 若一个测试文件中，有多个测试用例，可以用`go test -v -test.run 测试用例函数名`指定测试用例。

```go
$ go test -v -test.run TestGetSub
=== RUN   TestGetSub
    09-add_test.go:19: getSub(10, 7) 执行错误 期望值 = 5 实际值 = 3
--- FAIL: TestGetSub (0.00s)
FAIL
exit status 1
FAIL    goLearn/test    0.155s

```

### 8. 并发

#### 8.1 引出

**进程：**程序在操作系统中的一次执行过程，是系统进行资源分配和调度的基本单位。

**线程：**是进程的一个执行实例，是程序执行的最小单元，它是比进程更小的能独立运行的基本单位。

**关系：**

- 一个进程可以创建和销毁多个线程，同一个进程中的多个线程可以并发执行。

- 一个程序至少有一个进程，一个进程至少有一个线程。

----

> 假设，需要统计1-900000000000000000的数字中，哪些是素数？
>
> 1. 若按传统遍历，需要一个循环，从头到尾
> 2. 若按并发或并行，多个routine同时执行，将极大提升效率

#### 8.2 goroutine

##### 8.2.1 基础知识

**并发：多线程程序在单核上运行，就是并发。**一段时间内交替执行多个任务，但相同时间只能执行一个任务。（切换的很快，以至于让人感觉是"同时"进行）

**并行：多线程程序在多核上运行。**多个任务同时进行。（多个CPU，同时执行各自的任务）

----

**C语言**里面实现并发过程使用的是多线程（C++的最小资源单位），无法开启很多线程。

**Go主线程：**一个Go线程可以起多个goroutine，**goroutine是轻量级的线程。** ==> goroutine，每一个goroutine占用的系统资源远远小于线程，一个goroutine大约需要4k-5k的内存资源。一个程序可以启动大量的goroutine：

* 线程 ==> 几十个
* goroutine可以轻松启动成百上千上万个，==> 对于实现高并发，性能非常好
* 只需要在目标函数前加上go关键字即可

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220709155414503.png" alt="image-20220709155414503" style="zoom:50%;" />

**关系：**

- 主线程是一个物理线程，直接作用在CPU上，是重量级的，非常消耗CPU资源。
- goroutine是从主线程开启的，是轻量级的线程，是逻辑态的，对资源消耗较小。

**goroutine的特点（面试一定被问到）:star::star::star:：**

- 有独立的栈空间，数据是独立的，不会打架；
- 共享程序堆空间；
- 调度由用户控制；
- 协程是轻量级的线程。

**goroutined的调度模型MPG：**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220709155136557.png" alt="image-20220709155136557" style="zoom:50%;" />

1. M:操作系统的主线程（物理线程）
2. P:协程执行需要的上下文
3. G:协程

---

##### 8.2.2 demo

```go
/*
	1. 在主线程开启一个goroutine，每一秒输出一个"hello, world"
	2. 在主线程也每一秒输出一个"hello, world"
	3. 要求：主线程和goroutine同时执行
*/

func test() {
	for i := 0; i < 10; i++ {
		fmt.Println("test() hello, world" + strconv.Itoa(i))
		time.Sleep(time.Second)
	}
}

func main() {
	// 1. 在主线程开启一个goroutine
	go test()
	// 2. 主线程的输出
	for i := 0; i < 10; i++ {
		fmt.Println("main() hello, world" + strconv.Itoa(i))
		time.Sleep(time.Second)
	}

}

$ go run 01-goroutinedemo.go
main() hello, world0
test() hello, world0
test() hello, world1
main() hello, world1
main() hello, world2
test() hello, world2
.....
```

可以看出，主线程和goroutine同时在执行，由此引发问题：

- **谁执行完触发程序退出？**
  - 答：主线程执行完毕时，即使goroutine还未执行完毕，也依然会退出；goroutine执行完毕，主线程若还没执行完毕，依然会继续执行。**因此，按照主线程为准。**

#### 8.3 设置运行的CPU数

> 在golang里，设置运行的CPU数目。

```go
import "runtime"
```

1. `runtime.NumCPU()`

```go
// 1. 返回机器的cpu个数
cpuNum := runtime.NumCPU()
```

2. `runtime.GOMAXPROCS(cpuNum)`

```go
// 2. 设置CPU数目
runtime.GOMAXPROCS(cpuNum)
```

- go1.8后，默认程序运行于多核，不用手动设置
- go1.8前，需要手动设置

#### 8.4 channel

##### 8.4.1 先看一个需求

> **需求：**要计算1-200的各个数的阶乘，并且把各个数的阶乘放入到map中，最后显示出来。要求使用goroutine。

```go
/*
   思路：
   1. 编写一个函数，计算阶乘，并放入map
   2. 启动多个协程，统计结果放入map（全局共用）
   3. 所有的协程都对同一个map进行操作，会造成资源竞争，因此需要对全局变量上锁
*/
```

##### 8.4.2 上锁

```go
import "sync"
```

> sync包提供了基本的同步基元，如互斥锁。除了Once和WaitGroup类型，大部分都是适用于低水平程序线程，高水平的同步使用channel通信更好一些。

```go
var (
	myMap = map[int]int{}
	// 3. 全局互斥锁
	lock sync.Mutex
)

func calculate(n int) {
	res := 1
	for i := 1; i <= n; i++ {
		res *= i
	}

	// 加锁
	lock.Lock()
	myMap[n] = res
	// 解锁
	lock.Unlock()
}
func main() {
	// 起了20个协程
	for i := 1; i <= 20; i++ {
		go calculate(i)
	}
	// 此处不能马上遍历，因为不一定执行完毕
	time.Sleep(time.Second)
	for num, res := range myMap {
		fmt.Println(num, res)
	}
}
```

**低水平的，下面进一步**

- ==**使用无缓冲Channel**==

```go
package main

/*

        基于无缓冲的channel的main和 goroutine的同步

*/

import (
        "io"
        "log"
        "net"
        "os"
)

func main() {
        conn, err := net.Dial("tcp", "127.0.0.1:8001")
        if err != nil {
                log.Fatal(err)
        }

        done := make(chan string)
    // done := make(chan struct{})

        go func() {
                io.Copy(os.Stdout, conn)
                log.Println("groutine: done!")
                done <- "I am done"
            // close(done)
        }()

        //从客户端输入,将客户端标输入的数据发给客户端套接字
        io.Copy(conn, os.Stdin)

        conn.Close() //此时main要主动关闭conn, 否则goroutine里面的io.Copy()会一直阻塞等待conn

        log.Println("main wait goroutine...")
    	// 读阻塞在此处 
        <-done
        log.Println("main: done!")

        //这样我们就保证了 "main::done!"打印之前 一定先打印"groutine:done!"
}
```

基于channels发送消息有两个重要方面。首先每个消息都有一个值，但是有时候通讯的事实和发生的时刻也同样重要。当我们更希望强调通讯发生的时刻时，我们将它称为**消息事件**。有些消息事件并不携带额外的信息，它仅仅是用作两个goroutine之间的同步，这时候我们可以用`struct{}`空结构体作为channels元素的类型，虽然也可以使用bool或int类型实现同样的功能，`done <- 1`语句也比`done <- struct{}{}`更短。

##### 8.4.3 channel管道

1. **为什么需要channel？**

> - 主线程等待所有goroutine完成的时间很难确定，比如上例中设置了1s，仅仅是估算。
> - 若主线程等待时间长了，则会延长程序运行时间；若等待时间短了，则有可能还有goroutine未完成，而主线程的结束将导致未完成的goroutine销毁。
> - 通过加全局变量互斥锁来实现通讯，并不利于多个goroutine对全局变量的读写操作。

2. **channel的特点**

- channel本质就是一个数据结构-**队列**；
- 数据是**先进先出**的；
- 是**线程安全**的，多goroutine访问时，不需要加锁，也不会发生资源竞争问题；
- channel是有类型的，一个string的channel只能存放string类型的数据。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220709170356676.png" alt="image-20220709170356676" style="zoom:50%;" />

3. **基本语法**

> - 声明channel
> - channel遍历
> - channel关闭

---

- **声明channel**

```go
var chan名 chan 数据类型
chan名 := make(chan 数据类型, cap)
// example
var intChan chan int
var mapChan chan map[int]string
var perChan chan Person		
var perChan chan *Person
var allChan chan interface{}
```

**说明：**

- channel是引用类型的；
- channel必须初始化才能写入数据，即必须`make`；
- channel是有类型的，即intChan只能写入int型。**如果想啥类型都存，就声明成interface{}型，但是取的时候需要类型断言**；
- 向channel写入数据时，不能超过其cap（不会自动扩充），否则会deadlock；
- 从channel读取数据时，不能channel已经无数据了，还继续读，否则会deadlock；
- channel的写入和读出都符合**队列**。

**example1：**创建一个int型chan

```go
// 1. 创建一个int型chan
func example1() {
	/*
		演示channel的使用
	*/
	//	1. 创建一个int型chan
	//var intChan chan int
	//intChan = make(chan int, 3)
	intChan := make(chan int, 3)
	fmt.Println("看看intChan是什么？", intChan) // 输出intChan的地址 0xc000020100

	// 2. 向管道写入数据
	// 写数据时，不能超过cap；若超过则会deadlock
	intChan <- 10
	num := 211
	intChan <- num

	// 3. 看看管道的length和cap
	fmt.Println("channel len = ", len(intChan), ", cap = ", cap(intChan))

	// 4. 从管道读取数据
	// 读数据时，不能没数据了还继续读，否则deadlock
	num1 := <-intChan // 10， 先进先出
	fmt.Println("num1 = ", num1)
	fmt.Println("channel len = ", len(intChan), ", cap = ", cap(intChan))
}
```

**example2：**创建一个所有类型的channel

```go
// 2. 创建一个所有类型的chan
type Cat struct {
	Name string
	Age  int
}

func example2() {
	allChan := make(chan interface{}, 3)
	cat := Cat{
		Name: "小花猫",
		Age:  11,
	}
	allChan <- cat
	allChan <- 10
	allChan <- "chuyu"
	
	// 1. 类型断言 方式一
	getCat := (<-allChan).(Cat)
	fmt.Printf("getCat = %T, getCat = %v\n", getCat, getCat)
	// 不能直接取Name，因为interface{}
	fmt.Println("getCat.Name = ", getCat.Name)
	// 2. 类型断言 方式二
	//newCat := getCat.(Cat)
	//fmt.Println("getCat.Name = ", newCat.Name)
}
```

----

- **channel的遍历**

> channel支持`for-range`方式遍历，有两个细节：
>
>  	1. 在遍历时，若channel没有关闭，则会出现deadlock错误
>  	1. 在遍历时，若channel已经关闭，则会正常遍历数据，遍历完后就会退出遍历

- **channel的关闭**

> 内置函数`close`用来关闭channel，当关闭后，就不能再继续**写数据**，但依然可以继续**读数据**。

**close example：**

```go
func testClose() {
	intChan := make(chan int, 3)
	intChan <- 2
	// 1. 关闭一个channel
	close(intChan)
	// 2. 向关闭的channel中写入数据	// 会报错
	//intChan <- 3
	// 3. 从关闭的channel读取数据
	num := <-intChan
	fmt.Println(num)
}
```

**for-range example:**

```go
func testFR() {
	intChan := make(chan int, 100)
	for i := 1; i <= 100; i++ {
		intChan <- i
	}

	// 如果遍历管道，但不关闭管道，将会报错。因为还会傻乎乎的等着管道写数据，造成死锁
	close(intChan)
	// 1. 遍历管道
	for v := range intChan {
		fmt.Println("v = ", v)
	}
}
```

#### 8.5 goroutine和channel实例

1. **example1**

> 要求：
>
> 1. 开启一个writData协程，向管道intChan写入50个整数；
> 2. 开启一个readData协程，从管道intChan读取写入的数据；
> 3. 注意：
>    1. writData和readData操作的是同一个管道
>    2. 主线程需要等待writData和readData完成才能退出

```go
// 1. write data
func writData(intChan chan int) {
	for i := 1; i <= 50; i++ {
		intChan <- i
		fmt.Println("writData 写入数据：", i)
		time.Sleep(time.Second)
	}
	close(intChan)
}

// 2. read data
func readData(intChan chan int, exitChan chan bool) {
	for {
		if v, ok := <-intChan; ok {
			fmt.Println("readData 读到数据：", v)
            time.Sleep(time.Second)
		} else {
			break
		}
	}
	exitChan <- true
	close(exitChan)
}

func main() {
	// 3. 创建channel
	intChan := make(chan int, 50)
	exitChan := make(chan bool, 1)
	go writData(intChan)
	go readData(intChan, exitChan)

	for {
		if done := <-exitChan; done {
			break
		}
	}
}
```

**读和写的频率不一致是无所谓的**，当写满了，就会等着读，读一个就再写一个。

2. **example2**

> **需求：**统计1-20000的数字中，哪些是素数？
>
> **分析：**
>
> - 使用并发/并行的方式，将统计任务分配给多个（4个）goroutine完成；
> - 对比传统方式和goroutine方式的时间花费。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220709214727765.png" alt="image-20220709214727765" style="zoom:50%;" />

```go
func putNum(intChan chan int) {
	for i := 1; i <= 1000000; i++ {
		intChan <- i
	}
	close(intChan)
}

func primeNum(intChan chan int, primeChan chan int, exitChan chan bool, i int) {
	for num := range intChan {
		isPrime := true
		for i := 2; i < num; i++ {
			if num%i == 0 {
				isPrime = false
				break
			}
		}
		if isPrime {
			primeChan <- num
		}
	}
	// 所有的数字均读取完毕
	//time.Sleep(time.Millisecond * 10)
	fmt.Println("goruotine", i, "已完成...")
	exitChan <- true
}

func normal() {
	for num := 1; num <= 1000000; num++ {
		isPrime := true
		for i := 2; i < num; i++ {
			if num%i == 0 {
				isPrime = false
				break
			}
		}
		if isPrime {
			fmt.Println("prime:", num)
		}
	}
}

func main() {
	intChan := make(chan int, 20000)
	primeChan := make(chan int, 20000)
	exitChan := make(chan bool, 6)
	// 记录起始时间
	start := time.Now().Unix()
	end := time.Now().Unix()
	// 1. 开启写入数字的goroutine
	go putNum(intChan)
	// 2. 开启四个goroutine计算素数
	for i := 1; i <= 6; i++ {
		go primeNum(intChan, primeChan, exitChan, i)
	}
	// 3. 开启一个goroutine监测上述goroutine是否已全部完成
	// 当取出4个true，证明此时所有goroutine已完成
	go func() {
		for i := 0; i < 6; i++ {
			<-exitChan
		}
		end = time.Now().Unix()
		close(primeChan)
		close(exitChan)
	}()

	// 4. 主线程打印
	//for res := range primeChan {
	//	fmt.Println("prime:", res)
	//}
	// 测试时间专用，避免打印的时间
	for {
		if _, ok := <-primeChan; ok {
		} else {
			break
		}
	}
	fmt.Println("goroutine方式 花费时间：", end-start)

	// normal 方式花费时间
	//start = time.Now().Unix()
	//normal()
	//end = time.Now().Unix()
	//fmt.Println("normal方式 花费时间：", end-start)
}
```

goroutine花费时间：18s

normal花费的时间：68s

**提高了约4倍。**增加到12个goroutine，时间均为11s，就算再增加goroutine，也不会提高，因为是6核12线程机器。另外，打印也会消耗时间。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220709224917287.png" alt="image-20220709224917287" style="zoom:33%;" />

#### 8.6 单向管道

> 管道可以是**单向**或**双向（默认）**的。

```go
intChan := make(chan int, 1)	// 可读可写
```

```go
intChan := make(chan<- int, 1) // 只写
intChan := make(<-chan int, 1) // 只读
```

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220710095958584.png" alt="image-20220710095958584" style="zoom: 46%;" />

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220710100206364.png" alt="image-20220710100206364" style="zoom:50%;" />

**应用：**可以将全局的双向管道当作参数赋给单向管道。

#### 8.7 select

> 使用select可以解决从管道取数据的阻塞问题，用来监听多管道。

```go
func main() {
	intChan := make(chan int, 10)
	for i := 0; i < 10; i++ {
		intChan <- i
	}
	stringChan := make(chan string, 5)
	for i := 0; i < 5; i++ {
		stringChan <- "hello" + fmt.Sprintf("%d", i)
	}
	// 1. 传统方法遍历管道时，如果不关闭管道将造成deadlock
	// 但实际中，很难确定何时关闭管道
	// 2. 使用select解决
	for {
		time.Sleep(time.Second)
		select {
		case v := <-intChan: // 注意，此处如果intChan一直没有关闭，也不会一直阻塞在这里，会继续往下走
			fmt.Println("从intChan中读取数据：", v)
		case v := <-stringChan:
			fmt.Println("从stringChan中读取数据：", v)
		default:
			fmt.Println("哪个管道都读不到数据了...")
			return
		}
	}
}
```

#### 8.8 recover

> **用法：**使用 `defer + recover()` 处理异常。
>
> **说明：**如果起了一个协程，但这个协程发生panic，如果没有捕获这个panic，将造成整个程序的崩溃。这时可以在协程中使用`recover()`来捕获panic，这样的话，即使该协程出现panic，也不会影响整个程序其它部分的运行。

```go
func sayHello() {
	for i := 0; i < 10; i++ {
		time.Sleep(time.Second)
		fmt.Println("hello, world", i)
	}
}

func testError() {
	// 使用defer + recover 捕获异常
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("testError 发生异常：", err)
		}
	}()

	// 写一个异常报错
	var myMap map[int]int
	myMap[0] = 0
}

func main() {
	go sayHello()
	go testError()

	// 主线程
	for i := 0; i < 10; i++ {
		fmt.Println("main() hello", i)
		time.Sleep(time.Second)
	}
}
```

### 9. reflect

#### 9.1 引出

**应用场景：**

- 结构体的tag标签
- 编写函数的适配器，桥连接
- ……

**基本介绍：**

1. 反射可以在运行时动态获取变量的各种信息，比如变量的类型(type)，类别(kind)；
2. 如果是结构体变量，还可以获取到结构体本身的信息（包括结构体的字段，方法）；
3. 通过反射，可以修改变量的值，可以调用关联的方法；
4. 使用反射，需要 `import "reflect"`
5. 示意图

> reflect包实现了运行时反射，允许程序操作任意类型的对象。
>
> **典型用法**是用静态类型interface{}保存一个值，
>
> - 通过调用TypeOf获取其动态类型信息，该函数返回一个**Type类型值**（和普通的Type不一样）；
> - 调用ValueOf函数返回一个**Value类型值**（和普通的值不一样），该值代表运行时的数据，且可以利用很多方法获取信息；
> - Zero接受一个Type类型参数并返回一个代表该类型零值的Value类型值。

#### 9.2 快速入门

1. **example1**

> 演示对（基本数据类型、interface {}、reflect.Value）进行反射的基本操作。

```go
// 1. example1
// 通过反射拿到data的type、kind、值
func reflectDemo1(data interface{}) {
	// 1. 获取reflect.Type
	refType := reflect.TypeOf(data)
	fmt.Println("refType:", refType, "refType's type:", reflect.TypeOf(refType))
	// 2. 获取reflect.Value
	refValue := reflect.ValueOf(data)
	fmt.Println("refValue:", refValue)
	// 操作refValue
	numInt64 := int64(1)
	// 不能直接操作rVal类型
	//num += refValue
	numInt64 += refValue.Int()
	fmt.Println("numInt64 =", numInt64)
	// 3. 将rVal转成interface{}
	iv := refValue.Interface()
	// 将 interface{} 通过类型断言转成需要的类型
	numInt := 1
	numInt += iv.(int)
	fmt.Println("numInt =", numInt)
}

func main() {
	// 1. example1
	reflectDemo1(12)
}
```

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220710111138923.png" alt="image-20220710111138923" style="zoom:33%;" />

2. **example2**

> 演示对（结构体、interface{}、reflect.Value）进行反射的基本操作。

```go
func reflectDemo2(data interface{}) {
	// 1. 获取reflect.Type
	refType := reflect.TypeOf(data)
	fmt.Println("refType:", refType, "  refType's type:", reflect.TypeOf(refType))
	// 2. 获取reflect.Value
	refValue := reflect.ValueOf(data)
	fmt.Println("refValue:", refValue)
	// 3. 获取变量对应的Kind
	fmt.Printf("refType kind = %v\n", refType.Kind())
	fmt.Printf("refValue kind = %v\n", refValue.Kind())
	// 4. 转成 interface{}
	iv := refValue.Interface()
	fmt.Printf("iv = %v, iv type = %T\n", iv, iv)
	// 即使当前输出是Student类型，也不能取出数据
	// 5. 通过类型断言转换成相应类型
	stu := iv.(Student)
	fmt.Println("stu.Name =", stu.Name, ", stu.Age =", stu.Age)
}

func main() {
	// 2. example2
	student := Student{
		Name: "Lily",
		Age:  12,
	}
	reflectDemo2(student)
}
```

#### 9.3 通过反射修改值

传入指针，利用 `rVal.Elem()`得到指针指向的值，然后利用 `Set` 系列函数更改值。

```go
// 通过反射，修改 num int 的值
func reflectDemo3(data interface{}) {
	rVal := reflect.ValueOf(data)
	fmt.Println("rVal kind =", rVal.Kind()) // ptr
	// 这样是不行的，因为rVal是指针，需要先取到指针的值
	//rVal.SetInt(20)
	rVal.Elem().SetInt(20) // .Elem() 可以获取ptr指向的值，然后再改
}

func main() {
	num := 10
	reflectDemo3(&num)
	fmt.Println("num =", num)
}
```

```go
func main() {
	str := "tom"
	rVal := reflect.ValueOf(&str)
	// 会报错
	//rVal.SetString("lily")
	rVal.Elem().SetString("lily")
	fmt.Println(str)
}
```

#### 9.4 实战

##### 9.4.1 example1

> 使用反射遍历结构体的字段，调用结构体的方法，并获取结构体标签的值。
>
> ```go
> rTyp := reflect.TypeOf(a)
> rVal := reflect.ValueOf(a)
> rVal.NumField() // 获取字段数
> rVal.Field(i)   // 获取第i个字段的值
> rTyp.Field(i).Tag.Get("json")	// 获取第i个字段的json tag
> rVal.NumMethod() // 获取方法数
> // Method调用方法时，按照函数字母(ASCII码)进行排序，和定义时候的顺序无关。
> rVal.Method(1).Call(nil)	// 调用第2个方法，并传入参数nil
> params := []reflect.Value{}
> rVal.Method(0).Call(params)	// 传入的参数要是[]reflect.Value{}格式
> ```

```go
// 1. 定义Monster结构体
type Monster struct {
	Name  string `json:"name,omitempty"`
	Age   int    `json:"age,omitempty"`
	Score float64
	Sex   string
}

func (s Monster) Print() {
	fmt.Println("---start---")
	fmt.Println(s)
	fmt.Println("---end---")
}

func (s Monster) GetSum(n1, n2 int) int {
	return n1 + n2
}

func (s *Monster) Set(name string, age int, score float64, sex string) {
	s.Name = name
	s.Age = age
	s.Score = score
	s.Sex = sex
}
```

```go
// 2. 反射
func testStruct(a interface{}) {
	rTyp := reflect.TypeOf(a)
	rVal := reflect.ValueOf(a)
	rKd := rVal.Kind()
	if rKd != reflect.Struct {
		fmt.Println("expect struct...")
		return
	}
	num := rVal.NumField()
	fmt.Printf("struct has %d fields.\n", num)
	for i := 0; i < num; i++ {
		fmt.Printf("Field %d: 字段值 = %v.\n", i, rVal.Field(i))
		tagVal := rTyp.Field(i).Tag.Get("json")
		if tagVal != "" {
			fmt.Printf("Field %d: 字段tag = %v.\n", i, tagVal)
		}
	}
	numOfMethod := rVal.NumMethod()
	fmt.Printf("struct has %d methods.\n", numOfMethod)
	// Method调用方法时，按照函数字母(ASCII码)进行排序，和定义时候的顺序无关。
	// 调用 Print 函数
	rVal.Method(1).Call(nil)
	// 赋值
	params := []reflect.Value{}
	params = append(params, reflect.ValueOf(10))
	params = append(params, reflect.ValueOf(40))
	// 调用 GetSum 函数
	res := rVal.Method(0).Call(params)
	fmt.Println("res =", res[0].Int()) // 返回结果为 []reflect.Value
}

func main() {
	a := Monster{
		Name:  "cat",
		Age:   12,
		Score: 20,
		Sex:   "unknown",
	}
	testStruct(a)
}
```

##### 9.4.2 example2

> 调用反射修改结构体的字段值。

```go
func changeStruct1(a interface{}) {
	rVal := reflect.ValueOf(a)
	ptr := rVal.Elem()
	// 通过字段名 修改 Age的值
	ptr.FieldByName("Age").SetInt(100)
}

func main() {
	a := Monster{
		Name:  "cat",
		Age:   12,
		Score: 20,
		Sex:   "unknown",
	}
	changeStruct1(&a)
	testStruct(a)
}
```

##### 9.4.3 example3

> 调用结构体方法修改字段值。

```go
func changeStruct2(a interface{}) {
	rVal := reflect.ValueOf(a)
	ptr := rVal.Interface().(*Monster)
	ptr.Set("mouse", 12, 30.0, "unknown")
}

func main() {
	a := Monster{
		Name:  "cat",
		Age:   12,
		Score: 20,
		Sex:   "unknown",
	}
	changeStruct2(&a)
	testStruct(a)
}
```

### 10.init函数

**init函数的主要特点：**

- init函数先于main函数自动执行，不能被其他函数调用；
- init函数没有输入参数、返回值；
- 每个包可以有多个init函数；
- **包的每个源文件也可以有多个init函数**，这点比较特殊；
- 同一个包的init执行顺序，golang没有明确定义，编程时要注意程序不要依赖这个执行顺序。
- 不同包的init函数按照包导入的依赖关系决定执行顺序。





## 三、并发

### 1. 基础

并发：一段时间内交替执行多个任务，但相同时间只能执行一个任务。（切换的很快，以至于让人感觉是"同时"进行）

并行：多个任务同时进行。（多个CPU，同时执行各自的任务）

C语言里面实现并发过程使用的是多线程（C++的最小资源单位），进程

Go语言里面不是线程，而是go程 == > goroutine，每一个go程占用的系统资源远远小于线程，一个go程大约需要4k-5k的内存资源。一个程序可以启动大量的go程：

* 线程 ==> 几十个
* go程可以启动成百上千个，==> 对于实现高并发，性能非常好
* 只需要在目标函数前加上go关键字即可

### 2. 提前退出

```go
// GOEXIT ===> 提前退出当前go程
// return ===> 返回当前函数
// exit   ===> 退出当前进程
func main() {
	// 三个匿名函数
	go func() {
		go func() { // 两个go程
			func() {
				fmt.Println("这是子go程内部的函数! ")
				//return // 这是返回当前函数 其他函数依然会执行 包括外层的函数也会继续  因此fmt.Println("子go程结束! ")也会执行
				//os.Exit(-1) // 这是退出进程 fmt.Println("子go程结束! ") 和 fmt.Println("Over! ")就不会执行了
				runtime.Goexit() // 只是退出当前子go程！ 注意是当前！！ 只退出一层！ 不会执行 fmt.Println("子go程结束! ")和 fmt.Println("go程22222222222") 但会继续执行 fmt.Println("go程11111111111")和fmt.Println("Over! ")
			}()
			fmt.Println("子go程结束! ")
			fmt.Println("go程22222222222")
		}()
		time.Sleep(2 * time.Second)
		fmt.Println("go程11111111111")
	}()
	fmt.Println("这是主go程! ")
	time.Sleep(5 * time.Second)
	fmt.Println("Over! ")
}
```

### 3. 缓冲通道

> 1. ```go 
>    sync.RWMutex{}
>    ```
>
> 2. 当C语言涉及多线程时，使用互斥量，通过上锁来保持资源同步，避免资源竞争问题
>
> 3. 而go语言支持这种方式，但有更好的方式 ===> 通道、管道（不需要自己进行加解锁）
>
> 4. A往通道里写数据，B从通道里读数据。go自动帮我们做好数据同步

```go 
	// 1. 无缓冲的通道
	numChan := make(chan int) // 装数字的管道，使用管道时一定要make，同map一样，否则是nil
	// 2. 有缓冲的通道
	numChan := make(chan int, 10)
```

__有缓冲通道：__

* 当缓冲写满后，写阻塞，当被读取后，再恢复写入
* 当缓冲读取完毕，读阻塞
* 如果管道没有使用make分配空间，那么管道默认是nil的，读取、写入都会阻塞
* 对于一个管道，读与写的次数要对等，否则会发生死锁。若不一致：
  * 阻塞发生在主go程，那么程序将锁死崩溃
  * 阻塞发生在子go程，那么会出现内存泄露，子go程会一直等待，当进程结束时，内存才能释放

``` go 
	numsChan1 := make(chan int, 10)
	// 写
	go func() {
		for i := 0; i < 50; i++ {
			numsChan1 <- i
			fmt.Println("写入数据: ", i)
		}
	}()
	// 读
	// 主程序的次数多  当主程序被通道阻塞时, 那么程序将锁死崩溃; 若是go程，将一直卡在go程，造成内存泄露
	// 一定要读写次数一致
	// 主go程
	func() {
		for i := 0; i < 60; i++ {
			fmt.Println("读数据: ", <-numsChan1)
		}
	}()
	// 子go程
	//go func() {
	//	for i := 0; i < 60; i++ {
	//		fmt.Println("读数据: ", <-numsChan1)
	//	}
	//}()
	for {
		fmt.Println("子go程正在阻塞")
		time.Sleep(3 * time.Second)
	}
```

* 为了避免不一致，在读取数据时可采用```for-range```读取管道。

* 	// 遍历管道时，只返回一个值
    	// 问题是: for-range不知道管道是否已经写完，所以会一直等待写入
    	// 解决: 在写入端将管道关闭，for range遍历关闭管道时，会退出

```go
// 遍历管道时，只返回一个值
// 问题是: for-range不知道管道是否已经写完，所以会一直等待写入
// 解决: 在写入端将管道关闭，for range遍历关闭管道时，会退出
func main() {
	numsChan2 := make(chan int, 10)
	// 写
	go func() {
		for i := 0; i < 50; i++ {
			numsChan2 <- i
			fmt.Println("写入数据: ", i)
		}
		fmt.Println("数据写入完毕, 即将关闭管道! ")
		close(numsChan2)
	}()
	// 遍历管道时，只返回一个值
	// 问题是: for-range不知道管道是否已经写完，所以会一直等待写入
	// 解决: 在写入端将管道关闭，for range遍历关闭管道时，会退出
	for v := range numsChan2 {
		fmt.Println("读取数据: ", v)
	}
    fmt.Println("Over!!!")
}
```

__判断管道是否已经关闭：__

需要知道一个管道的状态，如果已经关闭了，再进行写入或重复关闭将造成崩溃的风险。

```go 
  map:    ==>   v, ok := m1[0] 

channel:  ==>   v, ok := <- numChan
```

```go
ok-idom 模式判断
// 判断通道是否已经close
	//// 读
	//for v := range numChan{
	//	fmt.Println("v: ", v)
	//}

	for {
		v, ok := <-numChan // ok-idom 模式判断
		if !ok {
			fmt.Println("管道已经关闭了!!! 准备退出!!! ")
			break
		}
		fmt.Println("v: ", v)	// 加了读操作
	}

	fmt.Println("Over!!! ")
```

__单向通道：__

```go
numChan := make(chan int, 10)  ==>  双向通道，既可以读，也可以写
```

单向通道：为了明确语义，一般用于函数参数

* 单向读通道：```var numChanReadOnly <- chan int```
* 单向写通道：`var numChanWriteOnly <- chan int`

```go
func main() {
	// 生产者消费者模型
	// C: 数组+锁 thread1: 写， thread2: 读
	// GO: goroutine + channel

	// 1. 在主函数中创建一个双向通道numChan
	numChan := make(chan int, 5)
	// 2. 将numChan 传递给producer，负责生产
	go producer(numChan) // 双向通道可以赋值给同类型的单向通道，单向不能转双向
	// 3. 将numChan 传递给consumer，负责消费
	go consumer(numChan)

	time.Sleep(2 * time.Second)
}

// producer生产者  ==> 提供一个只写通道
func producer(out chan<- int) {
	for i := 0; i < 10; i++ {
		out <- i
		//data := <- out  // 会报错 写通道不允许有读取操作
		fmt.Println("======> 向管道中写入数据: ", i)
	}
}

// consunmer消费者 ==> 提供一个只读通道
func consumer(in <-chan int) {
	//in <- 10  // 读通道不允许写数据
	for v := range in {
		fmt.Println("从管道中读取数据: ", v)
	}
}
```

__管道总结：__

* 管道写满 ==> 写阻塞
* 管道读完 ==> 读阻塞
* 如果管道没有使用make分配空间，则默认是nil
* 从nil的管道中读、写数据，都会阻塞（__注：__不会崩溃）
* 从一个已经close的管道读取数据时，会返回零值（__注：__不会崩溃；此处的零值并非普通int 0，而是逻辑假，可以终止for循环）
* 对一个已经close的管道进行关闭或写数据操作时，程序将崩溃
* 关闭管道的动作，一定要放在写端，不能放在读端（读端怎么知道什么时候写完呢？）
* 读和写的次数一定要对等，否则：
  * 子go程中，资源泄露
  * 主go程中，程序崩溃 (deadlock)

### 4. Select

当程序中有多个channel协同工作：ch1、ch2，某时刻ch1或ch2被触发，程序要做相应的处理。

使用select来监听多通道，当管道被触发时（写入数据、读取数据、关闭通道）

select语法与switch case很像，但是所有的分支条件都必须是通道io

```go
func main() {
	// var chan1, chan2 chan int
	chan1 := make(chan int)
	chan2 := make(chan int)

	// 启动一个go程，负责监听两个channel
	go func() {
		for {
			fmt.Println("监听中.........")
			select {
			case data1 := <-chan1:
				fmt.Println(" 从chan1读取数据成功, data1: ", data1)
			case data2 := <-chan2:
				fmt.Println("------------> 从chan2读取数据成功, data2: ", data2)
			default:		// 没触发上边的就触发default
				fmt.Println("select default called")
				time.Sleep(time.Second)
			}
		}
	}()
	// 启动go1 写chan1
	go func() {
		for i := 0; i < 10; i++ {
			chan1 <- i
			time.Sleep(1 * time.Second / 2)
		}
	}()
	// 启动go2 写chan2
	go func() {
		for i := 0; i < 10; i++ {
			chan2 <- i
			time.Sleep(1 * time.Second)
		}
	}()

	for {
		fmt.Println("Over!!! ")
		time.Sleep(5 * time.Second)
	}
}
```

## 四、网络编程（韩）

### 4.1 基础知识

**端口：**

- 在计算机（尤其是做服务器），要尽可能少开端口
- 一个端口只能被一个程序监听
- 可以使用 netstat -an 查看本机有哪些端口在监听
- 可以使用 netstat -anb 查看监听端口的pid，再结合任务管理器关闭不安全的端口

服务端需要起多个goroutine处理多个客户端的请求。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220710215719207.png" alt="image-20220710215719207" style="zoom:50%;" />

### 4.2 实例

> **服务器：**
>
> 1. 可连接多个客户端；
> 2. 可多次接收客户端发送消息；
> 3. 接收 `exit` 关闭连接。
>
> **客户端：**
>
> 1. 可多次发送消息；
> 2. 发送 `exit` 关闭连接。

#### **server.go**

```go
func serverDemo() {
	fmt.Println("服务器开始监听......")
	// 1. 开启一个监听端口
	listener, err := net.Listen("tcp", "0.0.0.0:8888")
	if err != nil {
		fmt.Println("listen err =", err)
		return
	}
	defer listener.Close()
	fmt.Println("listener is:", listener)
	// 2. 等待连接
	for {
		fmt.Println("等待客户端连接......")
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("accept err =", err)
		}
		fmt.Println("conn is:", conn)
		fmt.Println("该连接的客户端ip =", conn.RemoteAddr()) // 获得客户端的ip地址
		// 接收数据
		go serverProcess(conn)
	}
}

// 接收客户端数据
func serverProcess(conn net.Conn) {
	// 循环接收
	defer conn.Close() // 一定要记得关闭
	fmt.Printf("server 等待 客户端:%v 发送信息......\n", conn.RemoteAddr())
	for {
		// 创建切片，用来接收
		buf := make([]byte, 1024)
		// 1. 等待客户端通过conn发送信息
		// 2. 如果客户端没有write[发送信息]，那么协程就阻塞在这里
		n, err := conn.Read(buf)
		if err != nil {
			fmt.Println("server conn.Read err =", err)
			return
		}
		//fmt.Println("读取message长度为:", n)
		// 显示message
		message := string(buf[:n]) // 注意要[:n]
		fmt.Printf("信息: %v 来自客户端:%v\n", message, conn.RemoteAddr())
		if message == "exit" {
			break
		}
	}
	fmt.Printf("客户端: %v 退出......\n", conn.RemoteAddr())
}

func main() {
	serverDemo()
}
```

#### **client.go**

```go
func clientDemo() {
	conn, err := net.Dial("tcp", "192.168.0.108:8888")
	if err != nil {
		fmt.Println("client dial err =", err)
		return
	}
	fmt.Println("client conn is:", conn)
	clientProcess(conn)
}

func clientProcess(conn net.Conn) {
	defer conn.Close()
	for {
		fmt.Print("请输入要发送的信息: ")
		// 功能1：客户端发送单行数据，然后退出
		reader := bufio.NewReader(os.Stdin) // os.Stdin 代表标准输入（终端）
		// 从终端读取一行用户输入，并准备发给服务器
		message, err := reader.ReadString('\n') // 会多读一个'\n'
		if err != nil {
			fmt.Println("ReadString err =", err)
		}
		message = strings.Trim(message, " \r\n") // 删掉 " \r\n"
		// 再将message发送给服务器
		n, err := conn.Write([]byte(message))
		if err != nil {
			fmt.Println("conn.Write err =", err)
		}
		fmt.Printf("message长度为%v 发送成功......\n", n)
		// 当输入 exit 退出
		if message == "exit" {
			break
		}
	}
	fmt.Println("客户端退出......")
}

func main() {
	clientDemo()
}
```

## 五、Redis

### 5.1 基本介绍

> Redis(REmote Dictionary Server(远程字典服务器)) 是NoSQL数据库，不是传统关系型数据库，也被称为数据结构数据库。
>
> 性能非常高，通常用作缓存、持久化。
>
> 开源，分布式，基于内存运行并支持持久化。

**Redis的五大数据类型：**

- String（字符串）
- Hash（哈希）
- List（列表）
- Set（集合）
- zset（sorted set: 有序集合）

### 5.2 安装配置

> 下载后直接解压就有Redis的服务端程序(redis-server.exe)和客户端程序(redis-cli.exe)，双击直接运行，无需安装。

**Redis操作原理图：**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220712231156519.png" alt="image-20220712231156519" style="zoom: 50%;" />

双击打开 `redis-server.exe`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220713171047377.png" alt="image-20220713171047377" style="zoom: 33%;" />

然后打开 `redis-cli.exe` 即可连接上。

### 5.3 基本使用

> **Redis使用手册：http://doc.redisfans.com/**
>
> **说明：**
>
> Redis安装好后，默认有16个数据库，**初始默认使用0号库**，编号是0…15.
>
> 1. 添加 key-value `[set]`
> 2. 查看当前redis的所有 key `[keys *]`
> 3. 获取 key 对应的值
> 4. 切换 redis 数据库 `[select index]`
> 5. 如何查看当前数据库的 key-val 数量 `[dbsize]`
> 6. 清空当前数据库的 key-val 和 清空所有数据库的 key-val `[flushdb]` `[flushall]`

```shell
127.0.0.1:6379> set key1 hello
OK
127.0.0.1:6379> keys *
1) "key1"
127.0.0.1:6379> get key1
"hello"
127.0.0.1:6379> select 1
OK
127.0.0.1:6379[1]> keys *
(empty list or set)

127.0.0.1:6379[1]> select 0
OK
127.0.0.1:6379> dbsize
(integer) 1

127.0.0.1:6379> flushdb
OK
127.0.0.1:6379> keys *
(empty list or set)
```

### 5.4 Redis的Crud操作

#### 5.4.1 基本数据类型

>**Redis的五大数据类型：**
>
>- String（字符串）
>- Hash（哈希）
>- List（列表）
>- Set（集合）
>- zset（sorted set: 有序集合）

---

#### 5.4.2 **String**

```shell
127.0.0.1:6379> set address beijing
OK
127.0.0.1:6379> get address
"beijing"
127.0.0.1:6379> del address
(integer) 1
127.0.0.1:6379> get address
(nil)
```

**注意：**此处的 `set`，如果存在就是修改，如果不存在就是添加。

**增改：set**

**查：get**

**删：del**

**限时的键值对：**`setex key seconds value`

```shell
127.0.0.1:6379> setex message 6 hello
OK
127.0.0.1:6379> get message
"hello"
# 6s 后
127.0.0.1:6379> get message
(nil)
```

**同时设置多个键值对：**`mset key1 value1 key2 value2`

**同时获取多个键值对：**`mget key1 key2`

```shell
127.0.0.1:6379> mset key1 hello key2 world
OK
127.0.0.1:6379> keys *
1) "key2"
2) "key1"
127.0.0.1:6379> get key1
"hello"
127.0.0.1:6379> get key2
"world"
```

```shell
127.0.0.1:6379> mget key1 key2
1) "hello"
2) "world"
```

---

#### 5.4.3 **Hash（类似map）**

> Redis hash 是一个**键值对(key-value)集合**(key 不能重复)。是一个string类型的field和value的映射表，hash特别适合用于存储对象。例如存放一个user信息：
>
> key: user1 
>
> value: name “smith” age 30 job “coder” 	# 这就是三对field-value

**增改：hset key field value**

**查：hget key field value**

**删：hdel key field..**

**同时设置多个field-value对：**`hmset key field value field value..`

**同时获取多个field-value对：**`hmget key field..`

**获取该key的所有信息：**`hgetall key`

**获取某key的field数：**`hlen key`

**判断某key的某field是否存在：**`hexists key field`

```shell
127.0.0.1:6379> hset user1 name "smith"
(integer) 1
127.0.0.1:6379> hset user1 age 30
(integer) 1
127.0.0.1:6379> hset user1 job coder
(integer) 1

127.0.0.1:6379> hget user1 name
"smith"
127.0.0.1:6379> hget user1 job
"coder"
127.0.0.1:6379> hget user1 age
"30"

127.0.0.1:6379> hgetall user1
1) "name"
2) "smith"
3) "age"			# 前边的是field，下边是value
4) "30"
5) "job"
6) "coder"

127.0.0.1:6379> hdel user1 name age job
(integer) 3

127.0.0.1:6379> hmset user1 name jerry age 24 job "java coder"
OK
127.0.0.1:6379> hmget user1 name age job
1) "jerry"
2) "24"
3) "java coder"

127.0.0.1:6379> hlen user1
(integer) 3

127.0.0.1:6379> hexists user1 name
(integer) 1
127.0.0.1:6379> hexists user1 work
(integer) 0
```

由上述输出也可看出，当redis存储的时候，类型都是string。

---

#### 5.4.4 List

> List是简单的字符串列表，按照插入顺序排序。**可以头插和尾插。**List本质是个链表，List的元素是有序的，且可以重复。

**增改：**`lpush key value..`/`rpush key value..`。

**查：**`lrange key start stop`，返回列表key中指定区间的元素，0表示第一个，-1表示最后一个，以此类推。

​	`lpop key`/`rpop key`，弹出左/右栈顶元素（查看并删除）。



<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220713205501333.png" alt="image-20220713205501333" style="zoom:50%;" />

**删：**`del key..`，删除多个List。

**查看List的长度：**`llen key`，若不存在，返回0。

```shell
127.0.0.1:6379> lpush city beijing shanghai tianjin
(integer) 3
127.0.0.1:6379> lrange city 0 -1
1) "tianjin"
2) "shanghai"
3) "beijing"
# 发现没？是倒着的，相当于栈(右边是底)。

127.0.0.1:6379> lpush heroList aaa bbb ccc
(integer) 3
127.0.0.1:6379> lrange heroList 0 -1
1) "ccc"
2) "bbb"
3) "aaa"
127.0.0.1:6379> rpush heroList ddd eee
(integer) 5
127.0.0.1:6379> lrange heroList 0 -1
1) "ccc"
2) "bbb"
3) "aaa"
4) "ddd"
5) "eee"

127.0.0.1:6379> lpop heroList
"ccc"
127.0.0.1:6379> rpop heroList
"eee"
127.0.0.1:6379> lrange heroList 0 -1
1) "bbb"
2) "aaa"
3) "ddd"

127.0.0.1:6379> del heroList
(integer) 1
127.0.0.1:6379> lpop heroList
(nil)

127.0.0.1:6379> llen heroList
(integer) 0
127.0.0.1:6379> lpush heroList aaa bbb ccc
(integer) 3
127.0.0.1:6379> llen heroList
(integer) 3
```

---

#### 5.4.5 Set（集合）

> Redis的Set是string类型的**无序集合**。底层是HashTable数据结构，Set也是存放很多字符串元素，但字符串元素是**无序**的，且**元素不可重复**。

**增改：**`sadd key member..`

**查：**`smembers key`

**删：**`del key`

**判断是否存在某个member：**`sismember key member`

**删除指定member：**`srem key member.. `

```shell
127.0.0.1:6379> sadd emails tom@sohu.com jack@qq.com
(integer) 2
127.0.0.1:6379> smembers emails
1) "jack@qq.com"
2) "tom@sohu.com"
127.0.0.1:6379> sadd emails aaa@bbb.com ccc@ddd.com
(integer) 2
127.0.0.1:6379> smembers emails
1) "ccc@ddd.com"
2) "jack@qq.com"
3) "aaa@bbb.com"
4) "tom@sohu.com"

127.0.0.1:6379> del emails
(integer) 1
127.0.0.1:6379> smembers emails
(empty list or set)

127.0.0.1:6379> sadd emails tom@sohu.com jack@qq.com
(integer) 2
127.0.0.1:6379> sismember emails tom@sohu.com
(integer) 1
127.0.0.1:6379> srem emails tom@sohu.com
(integer) 1
127.0.0.1:6379> sismember emails tom@sohu.com
(integer) 0
```

---

### 5.5 Golang操作Redis

#### 5.5.1 安装

安装第三方开源的Redis库：

```go
go get github.com/gomodule/redigo/redis
```

#### 5.5.2 基本操作

其实和直接用redis-cli一样，只不过是把命令写在 `conn.Do("命令", )`，然后各参数本来是按空格分开的，现在改成逗号隔开。

##### 5.5.2.1 添加和获取key-value

1. **建立连接：**

```go
conn, err := redis.Dial("tcp", "127.0.0.1:6379")
if err != nil {
    fmt.Println("redis.Dial err =", err)
    return
}
defer conn.Close()
```

2. **写数据：**

```go
// 2. 通过go 向redis写入数据 string [key-value]
_, err = conn.Do("set", "name", "tomjerry")
if err != nil {
    fmt.Println("set err =", err)
    return
}
```

3. **读数据：**

```go
// 3. 读取数据 string [key-value]
r, err := redis.String(conn.Do("get", "name")) // 转换成string
if err != nil {
    fmt.Println("get err =", err)
    return
}
fmt.Println("操作ok，返回数据:", r)
```

##### 5.5.2.2 操作hash

```go
func operateHash() {
	// 1. 连接到redis
	conn, err := redis.Dial("tcp", "127.0.0.1:6379")
	if err != nil {
		fmt.Println("redis.Dial err =", err)
		return
	}
	defer conn.Close()
	//fmt.Println("redis conn is", conn)
	// 2. 通过go 向redis写入数据 hset [key-field-value]
	/*	// 单个添加
		_, err = conn.Do("hset", "user1", "name", "tomjerry")
		if err != nil {
			fmt.Println("hset err =", err)
			return
		}
		_, err = conn.Do("hset", "user1", "age", "20")
		if err != nil {
			fmt.Println("hset err =", err)
			return
		}*/
	// 批量添加
	_, err = conn.Do("hmset", "user2", "name", "john", "age", "18")
	if err != nil {
		fmt.Println("hmset err =", err)
		return
	}
	// 3. 读取数据 string [key-value]
	/*	// 单个读取
		r1, err := redis.String(conn.Do("hget", "user1", "name")) // 转换成string
		if err != nil {
			fmt.Println("hget err =", err)
			return
		}
		r2, err := redis.Int(conn.Do("hget", "user1", "age")) // 转换成string
		if err != nil {
			fmt.Println("hget err =", err)
			return
		}
		fmt.Printf("操作ok，返回数据: %v: %v\n", r1, r2)*/

	// 批量读取
	r, err := redis.Strings(conn.Do("hmget", "user2", "name", "age")) // 转换成string
	if err != nil {
		fmt.Println("hmget err =", err)
		return
	}
	for i, v := range r {
		fmt.Printf("%v: %v\n", i, v)
	}
}
```

#### 5.5.3 连接池

> 上面的操作，需要每次都新开一个conn连接，然后用完就关闭。其实这样很浪费资源，因此采用Redis连接池。
>
> **说明：**
>
> 1. **事先初始化**一定数量的连接，放入连接池，即**初始化连接池**；
> 2. 当Go需要操作Redis时，就从连接池取出一个conn即可；
> 3. 等到完全不需要了，关闭连接池；
> 4. 这样可以节省临时建立conn的时间，提高效率。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220714102018226.png" alt="image-20220714102018226" style="zoom:50%;" />

**核心代码：**

```go
// 定义一个全局pool
var pool *redis.Pool
func init() {
	pool = &redis.Pool{
		Dial: func() (redis.Conn, error) { // 初始化连接的代码，连接哪个ip，端口
			return redis.Dial("tcp", "localhost:6379")
		},
		TestOnBorrow:    nil,
		MaxIdle:         8,   // 最大空闲连接数
		MaxActive:       0,   // 表示和数据库的最大连接数，0表示没有限制
		IdleTimeout:     100, // 最大空闲时间
		Wait:            false,
		MaxConnLifetime: 0,
	}
}
```

```go
func operatePool() {
	// 先从一个pool取出一个连接
	conn := pool.Get()
	defer conn.Close()

	_, err := conn.Do("set", "name", "tom")
	if err != nil {
		fmt.Println("set err =", err)
		return
	}

	r, err := redis.String(conn.Do("get", "name"))
	if err != nil {
		fmt.Println("get err =", err)
		return
	}
	fmt.Println("get r =", r)

	// 关闭pool
	pool.Close()
	conn = pool.Get()
	// 此时取不出了，因为pool已关闭
}
```

## 六、MySQL

> Go语言中的`database/sql`包提供了保证SQL或类SQL数据库的泛用接口，并不提供具体的数据库驱动。使用`database/sql`包时必须注入（至少）一个数据库驱动。
>
> 我们常用的数据库基本上都有完整的第三方实现。例如：[MySQL驱动](https://github.com/go-sql-driver/mysql)

### 6.1 安装配置

1. **下载MySQL**

https://dev.mysql.com/downloads/windows/installer/8.0.html

选择8.0版本下载。

2. **下载驱动**

```go
go get -u github.com/go-sql-driver/mysql
```

3. **导入驱动**

```go
import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)
```



### 6.2 基础操作

#### 6.2.1 准备database

```mysql
create database go_db;

use go_db;

create table user_tb1(id integer primary key auto_increment, username varchar(20), password varchar(20));
insert into user_tb1(username, password) values("tom", "123");
insert into user_tb1(username, password) values("kite", "456");
select * from user_tb1;
```

#### 6.2.2 初始化连接

> Open函数可能只是验证其参数格式是否正确，实际上并不创建与数据库的连接。如果要检查数据源的名称是否真实有效，应该调用Ping方法。
>
> 返回的DB对象可以安全地被多个goroutine并发使用，并且维护其自己的空闲连接池。因此，Open函数应该仅被调用一次，很少需要关闭这个DB对象。

```go
dsn := "user:password@tcp(127.0.0.1:3306)/dbname"
db, err := sql.Open("mysql", dsn)
if err != nil {
    panic(err)
}
defer db.Close()
```

>接下来**定义一个全局变量`db`，用来保存数据库连接对象。**

```go
var db *sql.DB
func initDB() (err error) {
	dsn := "root:123456@tcp(127.0.0.1:3306)/go_db?charset=utf8mb4&parseTime=True"
	// 不会校验账号密码是否正确
	// 注意此处不用：:=
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err
	}
	// 尝试与数据库建立连接
	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}
func main() {
	err := initDB()
	if err != nil {
		fmt.Println("init db failed, err:", err)
		return
	}
}
```

其中，`sql.DB`是表示**连接的数据库对象（结构体实例）**，它保存了连接数据库相关的所有信息。它内部维护着一个具有零到多个底层连接的连接池，**它可以安全地被多个goroutine同时使用**。

> **其他初始化函数**

 **SetMaxOpenConns**

```go
func (db *DB) SetMaxOpenConns(n int)
```

`SetMaxOpenConns`设置与数据库建立连接的最大数目。 如果n大于0且小于最大闲置连接数，会将最大闲置连接数减小到匹配最大开启连接数的限制。 如果n<=0，不会限制最大开启连接数，默认为0（无限制）。

**SetMaxIdleConns**

```go
func (db *DB) SetMaxIdleConns(n int)
```

SetMaxIdleConns设置连接池中的最大闲置连接数。 如果n大于最大开启连接数，则新的最大闲置连接数会减小到匹配最大开启连接数的限制。 如果n<=0，不会保留闲置连接。

#### 6.2.3 Crud

事先我们已经准备好了一个`go_db`用于测试。

##### 6.2.3.1 查询数据

> 分为两种：
>
> - **单行查询**
> - **多行查询**

通常查询数据前，要先定义结构体来接收数据。

```go
// 先定义一个结构体，用于接收数据
type User struct {
	id       int
	username string
	password string
}
```

1. **单行查询**

单行查询`db.QueryRow()`执行一次查询，并期望返回最多一行结果（即Row）。QueryRow总是返回非nil的值，直到返回值的Scan方法被调用时，才会返回被延迟的错误。（如：未找到结果）

```go
func (db *DB) QueryRow(query string, args ...interface{}) *Row
```

```go
// 1.1 查询单行数据
func queryOneRow() {
	s := "select * from user_tb1 where id = ?"
	user := User{}
	// 非常重要：确保QueryRow之后调用Scan方法，否则持有的数据库链接不会被释放
	err := db.QueryRow(s, 1).Scan(&user.id, &user.username, &user.password)
	if err != nil {
		fmt.Println("queryOneRow failed, err:", err)
		return
	}
	fmt.Println("query result:", user)
}
```

2. **多行查询**

多行查询`db.Query()`执行一次查询，返回多行结果（即Rows），一般用于执行select命令。参数args表示query中的占位参数。

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
```

```go
// 1.2 多行查询
func queryRows() {
	s := "select * from user_tb1 where id > ?"
	rows, err := db.Query(s, 0) // 取出 id > 0 的 row
	if err != nil {
		fmt.Println("queryRows failed, err:", err)
		return
	}
	defer rows.Close()
	// 循环读取
	for rows.Next() {
		user := User{}
		err = rows.Scan(&user.id, &user.username, &user.password)
		if err != nil {
			fmt.Println("scan failed, err:", err)
			return
		}
		fmt.Println("query result:", user)
	}
}
```

##### 6.2.3.2 插入数据

**插入、更新和删除**操作都使用`Exec`方法。

```go
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
```

```go
// 2. 插入数据
func insert() {
	s := "insert into user_tb1 (username, password) values(?, ?)"
	r, err := db.Exec(s, "jerry", "789")
	if err != nil {
		fmt.Println("insert failed, err:", err)
		return
	}
	theId, _ := r.LastInsertId() // 新插入数据的id
	fmt.Printf("insert %v success...", theId)
}
```

##### 6.2.3.3 更新数据

```go
// 3. 更新数据
func updateRow() {
	s := "update user_tb1 set username=?, password=? where id=?"
	ret, err := db.Exec(s, "chuu", "0404", 1)
	if err != nil {
		fmt.Println("update failed, err:", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Println("get rowAffected failed, err:", err)
		return
	}
	fmt.Println("update success, affected rows:", n)
}
```

##### 6.2.3.4 删除数据

```go
// 4. 删除数据
func deleteRow() {
	s := "delete from user_tb1 where id=?"
	ret, err := db.Exec(s, 2)
	if err != nil {
		fmt.Println("delete failed, err:", err)
		return
	}
	n, err := ret.RowsAffected()
	if err != nil {
		fmt.Println("get rowAffected failed, err:", err)
		return
	}
	fmt.Println("delete success, affected rows:", n)
}
```

## 七、MongoDB

> [mongoDB](https://www.mongodb.com/)是目前比较流行的一个基于分布式文件存储的数据库。mongoDB中将一条数据存储为一个文档（document），数据结构由键值（key-value）对组成。 其中文档类似于我们平常编程中用到的JSON对象。 文档中的字段值可以包含其他文档，数组及文档数组。

### 7.1 相关概念

| MongoDB术语/概念 |                说明                 | 对比SQL术语/概念 |
| :--------------: | :---------------------------------: | :--------------: |
|     database     |               数据库                |     database     |
|    collection    |                集合                 |      table       |
|     document     |                文档                 |       row        |
|      field       |                字段                 |      column      |
|      index       |                index                |       索引       |
|   primary key    | 主键 MongoDB自动将_id字段设置为主键 |   primary key    |

### 7.2 常用命令

#### 7.1 collection

```shell
# 1. 创建新collection
> db.createCollection("student");
{ "ok" : 1 }
# 2. 查看所有collection
> show collections;
student
# 3. 删除某条collection
> db.student.drop();
true
> show collections;
>

```

#### 7.2 document

```shell
# 1. 插入一条document
> db.student.insertOne({name:"小王子", age:18});
{
        "acknowledged" : true,
        "insertedId" : ObjectId("62e251bd229c90eb53214c33")
}
# 2. 插入多条document 
> db.student.insertMany([
... {name:"张三", age:20},
... {name:"李四", age:25},
... ])
{
        "acknowledged" : true,
        "insertedIds" : [
                ObjectId("62e25228229c90eb53214c34"),
                ObjectId("62e25228229c90eb53214c35")
        ]
}
# 3. 查找student的所有document
> db.student.find()
{ "_id" : ObjectId("62e251bd229c90eb53214c33"), "name" : "小王子", "age" : 18 }
{ "_id" : ObjectId("62e25228229c90eb53214c34"), "name" : "张三", "age" : 20 }
{ "_id" : ObjectId("62e25228229c90eb53214c35"), "name" : "李四", "age" : 25 }
# 4. 查找age>20的所有document
> db.student.find( 
... {age:{$gt:20}}
... )
{ "_id" : ObjectId("62e25228229c90eb53214c35"), "name" : "李四", "age" : 25 }
# 5. 更新document
> db.student.update( 
... {name:"小王子"},
... {name:"老王子",age:98}
... );
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
> db.student.find();
{ "_id" : ObjectId("62e251bd229c90eb53214c33"), "name" : "老王子", "age" : 98 }
{ "_id" : ObjectId("62e25228229c90eb53214c34"), "name" : "张三", "age" : 20 }
{ "_id" : ObjectId("62e25228229c90eb53214c35"), "name" : "李四", "age" : 25 }
# 6. 删除document
> db.student.deleteOne({name:"李四"});
{ "acknowledged" : true, "deletedCount" : 1 }
> db.student.find();
{ "_id" : ObjectId("62e251bd229c90eb53214c33"), "name" : "老王子", "age" : 98 }
{ "_id" : ObjectId("62e25228229c90eb53214c34"), "name" : "张三", "age" : 20 }
```

**更多命令见：**[官方文档：shell命令](https://docs.mongodb.com/manual/mongo/)和[官方文档：CRUD操作](https://docs.mongodb.com/manual/crud/)。

### 7.3 Go操作MongoDB

**安装mongoDB Go驱动包**

```bash
go get github.com/mongodb/mongo-go-driver
```

#### 7.3.1 初始化

**启动连接：**

```go
var client *mongo.Client

func initDB() error {
	// 设置客户端连接
	clientOptions := options.Client().ApplyURI("mongodb://root:123456@127.0.0.1:27017")
	// 连接到db
	c, err := mongo.Connect(context.TODO(), clientOptions)
	if err != nil {
		log.Fatalln(err)
		return err
	} else {
		fmt.Printf("client: %v\n", c)
	}
	err = c.Ping(context.TODO(), nil)
	if err != nil {
		log.Fatalln(err)
		return err
	} else {
		fmt.Println("连接成功!")
	}
	client = c
	return nil
}
```

**关闭连接：**

```go
func mongoClose() {
	err := client.Disconnect(context.TODO())
	if err != nil {
		log.Fatalln(err)
	}
	fmt.Println("Connection to MongoDB closed.")
}
```

**连接池模式：**

```go
func connectToDB(uri, name string, timeout time.Duration, num uint64) (*mongo.Database, error) {
	ctx, cancel := context.WithTimeout(context.Background(), timeout)
	defer cancel()
	o := options.Client().ApplyURI(uri)
	o.SetMaxPoolSize(num)
	client, err := mongo.Connect(ctx, o)
	if err != nil {
		return nil, err
	}
	return client.Database(name), nil
}
```

#### 7.3.2 BSON

> MongoDB中的JSON文档存储在名为BSON(二进制编码的JSON)的二进制表示中。与其他将JSON数据存储为简单字符串和数字的数据库不同，BSON编码扩展了JSON表示，使其包含额外的类型，如int、long、date、浮点数和decimal128。这使得应用程序更容易可靠地处理、排序和比较数据。

连接MongoDB的Go驱动程序中有两大类型表示BSON数据：`D`和`Raw`。

类型`D`家族被用来简洁地构建使用本地Go类型的BSON对象。这对于构造传递给MongoDB的命令特别有用。`D`家族包括四类:

- D：一个BSON文档。这种类型应该在顺序重要的情况下使用，比如MongoDB命令。
- M：一张无序的map。它和D是一样的，只是它不保持顺序。
- A：一个BSON数组。
- E：D里面的一个元素。

要使用BSON，需要先导入下面的包：

```go
import "go.mongodb.org/mongo-driver/bson"
```

下面是一个使用D类型构建的过滤器文档的例子，它可以用来查找name字段与’张三’或’李四’匹配的文档:

```go
bson.D{{
	"name",
	bson.D{{
		"$in",
		bson.A{"张三", "李四"},
	}},
}}
```

`Raw`类型家族用于验证字节切片。你还可以使用`Lookup()`从原始类型检索单个元素。如果你不想要将BSON反序列化成另一种类型的开销，那么这是非常有用的。这个教程我们将只使用D类型。

#### 7.3.3 Crud

首先，定义一个Student类型：





## 八、Template

`html/template`包实现了数据驱动的模板，用于生成可防止代码注入的安全的HTML内容。它提供了和`text/template`包相同的接口，Go语言中输出HTML的场景都应使用`html/template`这个包。

### 8.1 模板与渲染

在一些前后端不分离的Web架构中，我们通常需要在后端将一些数据渲染到HTML文档中，从而实现动态的网页（网页的布局和样式大致一样，但展示的内容并不一样）效果。

我们这里说的模板可以理解为事先定义好的HTML文档文件，模板渲染的作用机制可以简单理解为文本替换操作–使用相应的数据去替换HTML文档中事先准备好的标记。

很多编程语言的Web框架中都使用各种模板引擎，比如Python语言中Flask框架中使用的jinja2模板引擎。

### 8.2 Go语言的模板引擎

Go语言内置了文本模板引擎`text/template`和用于HTML文档的`html/template`。它们的作用机制可以简单归纳如下：

1. 模板文件通常定义为`.tmpl`和`.tpl`为后缀（也可以使用其他的后缀），必须使用`UTF8`编码。
2. 模板文件中使用`{{`和`}}`包裹和标识需要传入的数据。
3. 传给模板这样的数据就可以通过点号（`.`）来访问，如果数据是复杂类型的数据，可以通过{ { .FieldName }}来访问它的字段。
4. 除`{{`和`}}`包裹的内容外，其他内容均不做修改原样输出。

### 8.3 模板引擎的使用

Go语言模板引擎的使用可以分为三部分：定义模板文件、解析模板文件和模板渲染.

#### 8.3.1 定义模板文件

其中，定义模板文件时需要我们按照相关语法规则去编写，后文会详细介绍。

#### 8.3.2 解析模板文件

上面定义好了模板文件之后，可以使用下面的常用方法去解析模板文件，得到模板对象：

```go
func (t *Template) Parse(src string) (*Template, error)
func ParseFiles(filenames ...string) (*Template, error)
func ParseGlob(pattern string) (*Template, error)
```

当然，你也可以使用`func New(name string) *Template`函数创建一个名为`name`的模板，然后对其调用上面的方法去解析模板字符串或模板文件。

#### 8.3.3 模板渲染

渲染模板简单来说就是使用数据去填充模板，当然实际上可能会复杂很多。

```go
func (t *Template) Execute(wr io.Writer, data interface{}) error
func (t *Template) ExecuteTemplate(wr io.Writer, name string, data interface{}) error
```

#### 8.3.4 基本示例

##### 8.3.4.1 定义模板文件

我们按照Go模板语法定义一个`hello.tmpl`的模板文件，内容如下：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Hello</title>
</head>
<body>
    <p>Hello {{.}}</p>
</body>
</html>
```

##### 8.3.4.2 解析和渲染模板文件

然后我们创建一个`main.go`文件，在其中写下HTTP server端代码如下：

```go
// main.go

func sayHello(w http.ResponseWriter, r *http.Request) {
	// 解析指定文件生成模板对象
	tmpl, err := template.ParseFiles("./hello.tmpl")
	if err != nil {
		fmt.Println("create template failed, err:", err)
		return
	}
	// 利用给定数据渲染模板，并将结果写入w
	tmpl.Execute(w, "沙河小王子")
}
func main() {
	http.HandleFunc("/", sayHello)
	err := http.ListenAndServe(":9090", nil)
	if err != nil {
		fmt.Println("HTTP server failed,err:", err)
		return
	}
}
```

将上面的`main.go`文件编译执行，然后使用浏览器访问`http://127.0.0.1:9090`就能看到页面上显示了“Hello 沙河小王子”。 这就是一个最简单的模板渲染的示例，Go语言模板引擎详细用法请往下阅读。

### 8.4 模板语法

#### 8.4.1 {{.}}

模板语法都包含在`{{`和`}}`中间，其中`{{.}}`中的点表示当前对象。

当我们传入一个结构体对象时，我们可以根据`.`来访问结构体的对应字段。例如：

```go
// main.go

type UserInfo struct {
	Name   string
	Gender string
	Age    int
}

func sayHello(w http.ResponseWriter, r *http.Request) {
	// 解析指定文件生成模板对象
	tmpl, err := template.ParseFiles("./hello.tmpl")
	if err != nil {
		fmt.Println("create template failed, err:", err)
		return
	}
	// 利用给定数据渲染模板，并将结果写入w
	user := UserInfo{
		Name:   "小王子",
		Gender: "男",
		Age:    18,
	}
	tmpl.Execute(w, user)
}
```

模板文件`hello.tmpl`内容如下：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Hello</title>
</head>
<body>
    <p>Hello {{.Name}}</p>
    <p>性别：{{.Gender}}</p>
    <p>年龄：{{.Age}}</p>
</body>
</html>
```

同理，当我们传入的变量是map时，也可以在模板文件中通过`.`根据key来取值。

#### 8.4.2 注释

```template
{{/* a comment */}}
注释，执行时会忽略。可以多行。注释不能嵌套，并且必须紧贴分界符始止。
```

#### 8.4.3 pipeline

`pipeline`是指产生数据的操作。比如`{{.}}`、`{{.Name}}`等。Go的模板语法中支持使用管道符号`|`链接多个命令，用法和unix下的管道类似：`|`前面的命令会将运算结果(或返回值)传递给后一个命令的最后一个位置。

**注意：**并不是只有使用了`|`才是pipeline。Go的模板语法中，`pipeline的`概念是传递数据，只要能产生数据的，都是`pipeline`。

#### 8.4.4 变量

我们还可以在模板中声明变量，用来保存传入模板的数据或其他语句生成的结果。具体语法如下：

```template
$obj := {{.}}
```

其中`$obj`是变量的名字，在后续的代码中就可以使用该变量了。

#### 8.4.5 移除空格

有时候我们在使用模板语法的时候会不可避免的引入一下空格或者换行符，这样模板最终渲染出来的内容可能就和我们想的不一样，这个时候可以使用`{{-`语法去除模板内容左侧的所有空白符号， 使用`-}}`去除模板内容右侧的所有空白符号。

例如：

```template
{{- .Name -}}
```

**注意：**`-`要紧挨`{{`和`}}`，同时与模板值之间需要使用空格分隔。

#### 8.4.6 条件判断

Go模板语法中的条件判断有以下几种:

```template
{{if pipeline}} T1 {{end}}

{{if pipeline}} T1 {{else}} T0 {{end}}

{{if pipeline}} T1 {{else if pipeline}} T0 {{end}}
```

#### 8.4.7 range

Go的模板语法中使用`range`关键字进行遍历，有以下两种写法，其中`pipeline`的值必须是数组、切片、字典或者通道。

```template
{{range pipeline}} T1 {{end}}
如果pipeline的值其长度为0，不会有任何输出

{{range pipeline}} T1 {{else}} T0 {{end}}
如果pipeline的值其长度为0，则会执行T0。
```

#### 8.4.8 with

```template
{{with pipeline}} T1 {{end}}
如果pipeline为empty不产生输出，否则将dot设为pipeline的值并执行T1。不修改外面的dot。

{{with pipeline}} T1 {{else}} T0 {{end}}
如果pipeline为empty，不改变dot并执行T0，否则dot设为pipeline的值并执行T1。
```

#### 8.4.9 预定义函数

执行模板时，函数从两个函数字典中查找：首先是模板函数字典，然后是全局函数字典。一般不在模板内定义函数，而是使用Funcs方法添加函数到模板里。

预定义的全局函数如下：

```template
and
    函数返回它的第一个empty参数或者最后一个参数；
    就是说"and x y"等价于"if x then y else x"；所有参数都会执行；
or
    返回第一个非empty参数或者最后一个参数；
    亦即"or x y"等价于"if x then x else y"；所有参数都会执行；
not
    返回它的单个参数的布尔值的否定
len
    返回它的参数的整数类型长度
index
    执行结果为第一个参数以剩下的参数为索引/键指向的值；
    如"index x 1 2 3"返回x[1][2][3]的值；每个被索引的主体必须是数组、切片或者字典。
print
    即fmt.Sprint
printf
    即fmt.Sprintf
println
    即fmt.Sprintln
html
    返回与其参数的文本表示形式等效的转义HTML。
    这个函数在html/template中不可用。
urlquery
    以适合嵌入到网址查询中的形式返回其参数的文本表示的转义值。
    这个函数在html/template中不可用。
js
    返回与其参数的文本表示形式等效的转义JavaScript。
call
    执行结果是调用第一个参数的返回值，该参数必须是函数类型，其余参数作为调用该函数的参数；
    如"call .X.Y 1 2"等价于go语言里的dot.X.Y(1, 2)；
    其中Y是函数类型的字段或者字典的值，或者其他类似情况；
    call的第一个参数的执行结果必须是函数类型的值（和预定义函数如print明显不同）；
    该函数类型值必须有1到2个返回值，如果有2个则后一个必须是error接口类型；
    如果有2个返回值的方法返回的error非nil，模板执行会中断并返回给调用模板执行者该错误；
```

#### 8.4.10 比较函数

布尔函数会将任何类型的零值视为假，其余视为真。

下面是定义为函数的二元比较运算的集合：

```template
eq      如果arg1 == arg2则返回真
ne      如果arg1 != arg2则返回真
lt      如果arg1 < arg2则返回真
le      如果arg1 <= arg2则返回真
gt      如果arg1 > arg2则返回真
ge      如果arg1 >= arg2则返回真
```

为了简化多参数相等检测，eq（只有eq）可以接受2个或更多个参数，它会将第一个参数和其余参数依次比较，返回下式的结果：

```template
{{eq arg1 arg2 arg3}}
```

比较函数只适用于基本类型（或重定义的基本类型，如”type Celsius float32”）。但是，整数和浮点数不能互相比较。

#### 8.4.11 自定义函数

Go的模板支持自定义函数。

```go
func sayHello(w http.ResponseWriter, r *http.Request) {
	htmlByte, err := ioutil.ReadFile("./hello.tmpl")
	if err != nil {
		fmt.Println("read html failed, err:", err)
		return	
	}
	// 自定义一个夸人的模板函数
	kua := func(arg string) (string, error) {
		return arg + "真帅", nil
	}
	// 采用链式操作在Parse之前调用Funcs添加自定义的kua函数
	tmpl, err := template.New("hello").Funcs(template.FuncMap{"kua": kua}).Parse(string(htmlByte))
	if err != nil {
		fmt.Println("create template failed, err:", err)
		return
	}

	user := UserInfo{
		Name:   "小王子",
		Gender: "男",
		Age:    18,
	}
	// 使用user渲染模板，并将结果写入w
	tmpl.Execute(w, user)
}
```

我们可以在模板文件`hello.tmpl`中按照如下方式使用我们自定义的`kua`函数了。

```template
{{kua .Name}}
```

#### 8.4.12 嵌套template

我们可以在template中嵌套其他的template。这个template可以是单独的文件，也可以是通过`define`定义的template。

举个例子： `test.tmpl`文件内容如下：

```template
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>tmpl test</title>
</head>
<body>
    
    <h1>测试嵌套template语法</h1>
    <hr>
    {{template "ul.tmpl" .}}
    <hr>
    {{template "ol.tmpl"}}
</body>
</html>

{{ define "ol.tmpl"}}
<ol>
    <li>吃饭</li>
    <li>睡觉</li>
    <li>打豆豆</li>
</ol>
{{end}}
```

`ul.tmpl`文件内容如下：

```template
<ul>
    <li>注释</li>
    <li>日志</li>
    <li>测试</li>
    <li>{{ .Name}}
</ul>
```

我们注册一个`templDemo`路由处理函数.

```go
http.HandleFunc("/tmpl", tmplDemo)
```

`tmplDemo`函数的具体内容如下：

```go
func tmplDemo(w http.ResponseWriter, r *http.Request) {
	tmpl, err := template.ParseFiles("./test.tmpl", "./ul.tmpl")
	if err != nil {
		fmt.Println("create template failed, err:", err)
		return
	}
	user := UserInfo{
		Name:   "小王子",
		Gender: "男",
		Age:    18,
	}
	tmpl.Execute(w, user)
}
```

**注意**：在解析模板时，被嵌套的模板一定要在后面解析，例如上面的示例中`test.tmpl`模板中嵌套了`ul.tmpl`，所以`ul.tmpl`要在`t.tmpl`后进行解析。

**若要给模板传值，**正确写法是 {{ template "ul.tmpl" . }} ，需要在后边加个 `.`。

#### 8.4.13 block

```template
{{block "name" pipeline}} T1 {{end}}
```

`block`是定义模板`{{define "name"}} T1 {{end}}`和执行`{{template "name" pipeline}}`缩写，典型的用法是定义一组根模板，然后通过在其中重新定义块模板进行自定义。

定义一个根模板`templates/base.tmpl`，内容如下：

```template
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <title>Go Templates</title>
</head>
<body>
<div class="container-fluid">
    {{block "content" . }}{{end}}
</div>
</body>
</html>
```

然后定义一个`templates/index.tmpl`，”继承”`base.tmpl`：

```tempalte
{{template "base.tmpl"}}

{{define "content"}}
    <div>Hello world!</div>
{{end}}
```

然后使用`template.ParseGlob`按照正则匹配规则解析模板文件，然后通过`ExecuteTemplate`渲染指定的模板：

```go
func index(w http.ResponseWriter, r *http.Request){
	tmpl, err := template.ParseGlob("templates/*.tmpl")
	if err != nil {
		fmt.Println("create template failed, err:", err)
		return
	}
	err = tmpl.ExecuteTemplate(w, "index.tmpl", nil)
	if err != nil {
		fmt.Println("render template failed, err:", err)
		return
	}
}
```

如果我们的模板名称冲突了，例如不同业务线下都定义了一个`index.tmpl`模板，我们可以通过下面两种方法来解决。

1. 在模板文件开头使用`{{define 模板名}}`语句显式的为模板命名。
2. 可以把模板文件存放在`templates`文件夹下面的不同目录中，然后使用`template.ParseGlob("templates/**/*.tmpl")`解析模板。

#### 8.4.14 修改默认的标识符

Go标准库的模板引擎使用的花括号`{{`和`}}`作为标识，而许多前端框架（如`Vue`和 `AngularJS`）也使用`{{`和`}}`作为标识符，所以当我们同时使用Go语言模板引擎和以上前端框架时就会出现冲突，这个时候我们需要修改标识符，修改前端的或者修改Go语言的。这里演示如何修改Go语言模板引擎默认的标识符：

```go
template.New("test").Delims("{[", "]}").ParseFiles("./t.tmpl")
```

### 8.5 text/template与html/tempalte的区别

`html/template`针对的是需要返回HTML内容的场景，在模板渲染过程中会对一些有风险的内容进行转义，以此来防范跨站脚本攻击。

例如，我定义下面的模板文件：

```template
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Hello</title>
</head>
<body>
    {{.}}
</body>
</html>
```

这个时候传入一段JS代码并使用`html/template`去渲染该文件，会在页面上显示出转义后的JS内容。 `<script>alert('嘿嘿嘿')</script>` 这就是`html/template`为我们做的事。

但是在某些场景下，我们如果相信用户输入的内容，不想转义的话，可以自行编写一个safe函数，手动返回一个`template.HTML`类型的内容。示例如下：

```go
func xss(w http.ResponseWriter, r *http.Request){
	tmpl, err := template.New("xss.tmpl").Funcs(template.FuncMap{
		"safe": func(s string)template.HTML {
			return template.HTML(s)
		},
	}).ParseFiles("./xss.tmpl")
	if err != nil {
		fmt.Println("create template failed, err:", err)
		return
	}
	jsStr := `<script>alert('嘿嘿嘿')</script>`
	err = tmpl.Execute(w, jsStr)
	if err != nil {
		fmt.Println(err)
	}
}
```

这样我们只需要在模板文件不需要转义的内容后面使用我们定义好的safe函数就可以了。

```template
{{ . | safe }}
```

即，信任的用safe，不信任的用.

## 九、Gin

`Gin`是一个用Go语言编写的web框架。它是一个类似于`martini`但拥有更好性能的API框架, 由于使用了`httprouter`，速度提高了近40倍。 如果你是性能和高效的追求者, 你会爱上`Gin`。

### 9.1 Gin框架介绍

Go世界里最流行的Web框架，[Github](https://github.com/gin-gonic/gin)上有`32K+`star。 基于[httprouter](https://github.com/julienschmidt/httprouter)开发的Web框架。 [中文文档](https://gin-gonic.com/zh-cn/docs/)齐全，简单易用的轻量级框架。

### 9.2 Gin框架安装与使用

#### 9.2.1 安装

下载并安装`Gin`:

```bash
go get -u github.com/gin-gonic/gin
```

#### 9.2.2 第一个Gin示例：

```golang
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	// 创建一个默认的路由引擎
	r := gin.Default()
	// GET：请求方式；/hello：请求的路径
	// 当客户端以GET方法请求/hello路径时，会执行后面的匿名函数
	r.GET("/hello", func(c *gin.Context) {
		// c.JSON：返回JSON格式的数据
		c.JSON(200, gin.H{
			"message": "Hello world!",
		})
	})
	// 启动HTTP服务，默认在0.0.0.0:8080启动服务
	r.Run()
}
```

将上面的代码保存并编译执行，然后使用浏览器打开`127.0.0.1:8080/hello`就能看到一串JSON字符串。

### 9.3 RESTful API

REST与技术无关，代表的是一种软件架构风格，REST是Representational State Transfer的简称，中文翻译为“表征状态转移”或“表现层状态转化”。

推荐阅读[阮一峰 理解RESTful架构](http://www.ruanyifeng.com/blog/2011/09/restful.html)

简单来说，REST的含义就是客户端与Web服务器之间进行交互的时候，使用HTTP协议中的4个请求方法代表不同的动作。

- `GET`用来获取资源
- `POST`用来新建资源
- `PUT`用来更新资源
- `DELETE`用来删除资源。

只要API程序遵循了REST风格，那就可以称其为RESTful API。目前在前后端分离的架构中，前后端基本都是通过RESTful API来进行交互。

例如，我们现在要编写一个管理书籍的系统，我们可以查询对一本书进行查询、创建、更新和删除等操作，我们在编写程序的时候就要设计客户端浏览器与我们Web服务端交互的方式和路径。按照经验我们通常会设计成如下模式：

| 请求方法 |     URL      |     含义     |
| :------: | :----------: | :----------: |
|   GET    |    /book     | 查询书籍信息 |
|   POST   | /create_book | 创建书籍记录 |
|   POST   | /update_book | 更新书籍信息 |
|   POST   | /delete_book | 删除书籍信息 |

同样的需求我们按照RESTful API设计如下：

| 请求方法 |  URL  |     含义     |
| :------: | :---: | :----------: |
|   GET    | /book | 查询书籍信息 |
|   POST   | /book | 创建书籍记录 |
|   PUT    | /book | 更新书籍信息 |
|  DELETE  | /book | 删除书籍信息 |

Gin框架支持开发RESTful API的开发。

```go
func main() {
	r := gin.Default()
	r.GET("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "GET",
		})
	})

	r.POST("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "POST",
		})
	})

	r.PUT("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "PUT",
		})
	})

	r.DELETE("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "DELETE",
		})
	})
}
```

开发RESTful API的时候我们通常使用[Postman](https://www.getpostman.com/)来作为客户端的测试工具。

### 9.4 Gin渲染

#### 9.4.1 HTML渲染

我们首先定义一个存放模板文件的`templates`文件夹，然后在其内部按照业务分别定义一个`posts`文件夹和一个`users`文件夹。 `posts/index.html`文件的内容如下：

```template
{{define "posts/index.html"}}
<!DOCTYPE html>
<html lang="en">	

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>posts/index</title>
</head>
<body>
    {{.title}}
</body>
</html>
{{end}}
```

`users/index.html`文件的内容如下：

```template
{{define "users/index.html"}}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>users/index</title>
</head>
<body>
    {{.title}}
</body>
</html>
{{end}}
```

Gin框架中使用`LoadHTMLGlob()`或者`LoadHTMLFiles()`方法进行HTML模板渲染。

```go
func main() {
	r := gin.Default()
	r.LoadHTMLGlob("templates/**/*")
	//r.LoadHTMLFiles("templates/posts/index.html", "templates/users/index.html")
	r.GET("/posts/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "posts/index.html", gin.H{
			"title": "posts/index",
		})
	})

	r.GET("users/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "users/index.html", gin.H{
			"title": "users/index",
		})
	})

	r.Run(":8080")
}
```

#### 9.4.2 自定义模板函数

定义一个不转义相应内容的`safe`模板函数如下：

```go
func main() {
	router := gin.Default()
	router.SetFuncMap(template.FuncMap{
		"safe": func(str string) template.HTML{
			return template.HTML(str)
		},
	})
	router.LoadHTMLFiles("./index.tmpl")

	router.GET("/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.tmpl", "<a href='https://liwenzhou.com'>李文周的博客</a>")
	})

	router.Run(":8080")
}
```

在`index.tmpl`中使用定义好的`safe`模板函数：

```template
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <title>修改模板引擎的标识符</title>
</head>
<body>
<div>{{ . | safe }}</div>
</body>
</html>
```

#### 9.4.3 静态文件处理

当我们渲染的HTML文件中引用了静态文件时，我们只需要按照以下方式在渲染页面前调用`gin.Static`方法即可。

```go
func main() {
	r := gin.Default()
	r.Static("/static", "./static")
	r.LoadHTMLGlob("templates/**/*")
   // ...
	r.Run(":8080")
}
```

#### 9.4.4 使用模板继承

Gin框架默认都是使用单模板，如果需要使用`block template`功能，可以通过`"github.com/gin-contrib/multitemplate"`库实现，具体示例如下：

首先，假设我们项目目录下的templates文件夹下有以下模板文件，其中`home.tmpl`和`index.tmpl`继承了`base.tmpl`：

```bash
templates
├── includes
│   ├── home.tmpl
│   └── index.tmpl
├── layouts
│   └── base.tmpl
└── scripts.tmpl
```

然后我们定义一个`loadTemplates`函数如下：

```go
func loadTemplates(templatesDir string) multitemplate.Renderer {
	r := multitemplate.NewRenderer()
	layouts, err := filepath.Glob(templatesDir + "/layouts/*.tmpl")
	if err != nil {
		panic(err.Error())
	}
	includes, err := filepath.Glob(templatesDir + "/includes/*.tmpl")
	if err != nil {
		panic(err.Error())
	}
	// 为layouts/和includes/目录生成 templates map
	for _, include := range includes {
		layoutCopy := make([]string, len(layouts))
		copy(layoutCopy, layouts)
		files := append(layoutCopy, include)
		r.AddFromFiles(filepath.Base(include), files...)
	}
	return r
}
```

我们在`main`函数中

```go
func indexFunc(c *gin.Context){
	c.HTML(http.StatusOK, "index.tmpl", nil)
}

func homeFunc(c *gin.Context){
	c.HTML(http.StatusOK, "home.tmpl", nil)
}

func main(){
	r := gin.Default()
	r.HTMLRender = loadTemplates("./templates")
	r.GET("/index", indexFunc)
	r.GET("/home", homeFunc)
	r.Run()
}
```

#### 9.4.5 补充文件路径处理

关于模板文件和静态文件的路径，我们需要根据公司/项目的要求进行设置。可以使用下面的函数获取当前执行程序的路径。

```go
func getCurrentPath() string {
	if ex, err := os.Executable(); err == nil {
		return filepath.Dir(ex)
	}
	return "./"
}
```

#### 9.4.6 JSON渲染

```go
func main() {
	r := gin.Default()

	// gin.H 是map[string]interface{}的缩写
	r.GET("/someJSON", func(c *gin.Context) {
		// 方式一：自己拼接JSON
		c.JSON(http.StatusOK, gin.H{"message": "Hello world!"})
	})
	r.GET("/moreJSON", func(c *gin.Context) {
		// 方法二：使用结构体
		var msg struct {
			Name    string `json:"user"`
			Message string
			Age     int
		}
		msg.Name = "小王子"
		msg.Message = "Hello world!"
		msg.Age = 18
		c.JSON(http.StatusOK, msg)
	})
	r.Run(":8080")
}
```

#### 9.4.7 XML渲染

注意需要使用具名的结构体类型。

```go
func main() {
	r := gin.Default()
	// gin.H 是map[string]interface{}的缩写
	r.GET("/someXML", func(c *gin.Context) {
		// 方式一：自己拼接JSON
		c.XML(http.StatusOK, gin.H{"message": "Hello world!"})
	})
	r.GET("/moreXML", func(c *gin.Context) {
		// 方法二：使用结构体
		type MessageRecord struct {
			Name    string
			Message string
			Age     int
		}
		var msg MessageRecord
		msg.Name = "小王子"
		msg.Message = "Hello world!"
		msg.Age = 18
		c.XML(http.StatusOK, msg)
	})
	r.Run(":8080")
}
```

#### 9.4.8 YAML渲染

```go
r.GET("/someYAML", func(c *gin.Context) {
	c.YAML(http.StatusOK, gin.H{"message": "ok", "status": http.StatusOK})
})
```

#### 9.4.9 protobuf渲染

```go
r.GET("/someProtoBuf", func(c *gin.Context) {
	reps := []int64{int64(1), int64(2)}
	label := "test"
	// protobuf 的具体定义写在 testdata/protoexample 文件中。
	data := &protoexample.Test{
		Label: &label,
		Reps:  reps,
	}
	// 请注意，数据在响应中变为二进制数据
	// 将输出被 protoexample.Test protobuf 序列化了的数据
	c.ProtoBuf(http.StatusOK, data)
})
```

### 9.5 获取参数

#### 9.5.1 获取querystring参数

`querystring`指的是URL中`?`后面携带的参数，例如：`/user/search?username=小王子&address=沙河`。 获取请求的querystring参数的方法如下：

```go
func main() {
	//Default返回一个默认的路由引擎
	r := gin.Default()
	r.GET("/user/search", func(c *gin.Context) {
		username := c.DefaultQuery("username", "小王子")
		//username := c.Query("username")
		address := c.Query("address")
		//输出json结果给调用方
		c.JSON(http.StatusOK, gin.H{
			"message":  "ok",
			"username": username,
			"address":  address,
		})
	})
	r.Run()
}
```

#### 9.5.2 获取form参数

当前端请求的数据通过form表单提交时，例如向`/user/search`发送一个POST请求，获取请求数据的方式如下：

```go
func main() {
	//Default返回一个默认的路由引擎
	r := gin.Default()
	r.POST("/user/search", func(c *gin.Context) {
		// DefaultPostForm取不到值时会返回指定的默认值
		//username := c.DefaultPostForm("username", "小王子")
		username := c.PostForm("username")
		address := c.PostForm("address")
		//输出json结果给调用方
		c.JSON(http.StatusOK, gin.H{
			"message":  "ok",
			"username": username,
			"address":  address,
		})
	})
	r.Run(":8080")
}
```

#### 9.5.3 获取json参数

当前端请求的数据通过JSON提交时，例如向`/json`发送一个POST请求，则获取请求参数的方式如下：

```go
r.POST("/json", func(c *gin.Context) {
	// 注意：下面为了举例子方便，暂时忽略了错误处理
	b, _ := c.GetRawData()  // 从c.Request.Body读取请求数据
	// 定义map或结构体
	var m map[string]interface{}
	// 反序列化
	_ = json.Unmarshal(b, &m)

	c.JSON(http.StatusOK, m)
})
```

更便利的获取请求参数的方式，参见下面的 **参数绑定** 小节。

#### 9.5.4 获取path参数

请求的参数通过URL路径传递，例如：`/user/search/小王子/沙河`。 获取请求URL路径中的参数的方式如下。

```go
func main() {
	//Default返回一个默认的路由引擎
	r := gin.Default()
	r.GET("/user/search/:username/:address", func(c *gin.Context) {
		username := c.Param("username")
		address := c.Param("address")
		//输出json结果给调用方
		c.JSON(http.StatusOK, gin.H{
			"message":  "ok",
			"username": username,
			"address":  address,
		})
	})

	r.Run(":8080")
}
```

#### 9.5.5 参数绑定

为了能够更方便的获取请求相关参数，提高开发效率，我们可以基于请求的`Content-Type`识别请求数据类型并利用反射机制自动提取请求中`QueryString`、`form表单`、`JSON`、`XML`等参数到结构体中。 下面的示例代码演示了`.ShouldBind()`强大的功能，它能够基于请求自动提取`JSON`、`form表单`和`QueryString`类型的数据，并把值绑定到指定的结构体对象。

```go
// Binding from JSON
type Login struct {
	User     string `form:"user" json:"user" binding:"required"`
	Password string `form:"password" json:"password" binding:"required"`
}

func main() {
	router := gin.Default()

	// 绑定JSON的示例 ({"user": "q1mi", "password": "123456"})
	router.POST("/loginJSON", func(c *gin.Context) {
		var login Login

		if err := c.ShouldBind(&login); err == nil {
			fmt.Printf("login info:%#v\n", login)
			c.JSON(http.StatusOK, gin.H{
				"user":     login.User,
				"password": login.Password,
			})
		} else {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		}
	})

	// 绑定form表单示例 (user=q1mi&password=123456)
	router.POST("/loginForm", func(c *gin.Context) {
		var login Login
		// ShouldBind()会根据请求的Content-Type自行选择绑定器
		if err := c.ShouldBind(&login); err == nil {
			c.JSON(http.StatusOK, gin.H{
				"user":     login.User,
				"password": login.Password,
			})
		} else {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		}
	})

	// 绑定QueryString示例 (/loginQuery?user=q1mi&password=123456)
	router.GET("/loginForm", func(c *gin.Context) {
		var login Login
		// ShouldBind()会根据请求的Content-Type自行选择绑定器
		if err := c.ShouldBind(&login); err == nil {
			c.JSON(http.StatusOK, gin.H{
				"user":     login.User,
				"password": login.Password,
			})
		} else {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		}
	})

	// Listen and serve on 0.0.0.0:8080
	router.Run(":8080")
}
```

`ShouldBind`会按照下面的顺序解析请求中的数据完成绑定：

1. 如果是 `GET` 请求，只使用 `Form` 绑定引擎（`query`）。
2. 如果是 `POST` 请求，首先检查 `content-type` 是否为 `JSON` 或 `XML`，然后再使用 `Form`（`form-data`）。

### 9.6 文件上传

#### 9.6.1 单个文件上传

文件上传前端页面代码：

```html
{{define "upload_files.html"}}
<!DOCTYPE html>
<html lang="zn-CN">
<head>
  <meta charset="UTF-8">
  <title>上传文件</title>
</head>
<body>
<form action="/uploadSingle" method="post" enctype="multipart/form-data">
  上传单个文件：
  <input type="file" name="file">
  <input type="submit" value="上传">
</form>

<form action="/uploadMulti" method="post" enctype="multipart/form-data">
  上传多个文件：
  <input type="file" name="files[]" multiple>
  <input type="submit" value="上传">
</form>
</body>
</html>
{{end}}
```

后端gin框架部分代码：

```go
func uploadSingleFile(r *gin.Engine) {
	r.POST("/uploadSingle", func(context *gin.Context) {
		// 单个文件
		file, err := context.FormFile("file")
		if err != nil {
			context.JSON(http.StatusInternalServerError, gin.H{"message": err.Error()})
			return
		}
		log.Println(file.Filename)
		dst := fmt.Sprintf("./upload_files/%s", file.Filename)
		// 上传文件到指定文件夹
		err = context.SaveUploadedFile(file, dst)
		if err != nil {
			context.JSON(http.StatusInternalServerError, gin.H{"message": "upload failed."})
		}
		context.JSON(http.StatusOK, gin.H{"message": fmt.Sprintf("%s upload success.", file.Filename)})
	})
	//r.Run()
}
```

#### 9.6.2 多个文件上传

```go
func uploadMultiFiles(r *gin.Engine) {
	r.POST("/uploadMulti", func(context *gin.Context) {
		form, _ := context.MultipartForm()
		files := form.File["files[]"]
		for index, file := range files {
			log.Println(file.Filename)
			dst := fmt.Sprintf("./upload_files/%d_%s", index, file.Filename)
			// 上传文件到指定目录
			context.SaveUploadedFile(file, dst)
		}
		context.JSON(http.StatusOK, gin.H{"message": fmt.Sprintf("%d files upload success.", len(files))})
	})
	//r.Run()
}

func main() {
	r := gin.Default()
	// 1. 静态页面	默认显示./upload下的index.html页面。注意，必须是index命名
	//r.Static("/", "./upload_files")
	// 2. 动态显示
	r.LoadHTMLFiles("./upload_files/upload_files.html")
	r.GET("/", func(context *gin.Context) {
		context.HTML(http.StatusOK, "upload_files.html", gin.H{
			"title": "upload_files",
		})
	})

	// 处理multipart forms提交文件时默认的内存限制是32 MiB
	// 可以通过下面的方式修改
	r.MaxMultipartMemory = 8 << 20 // 8 MiB
	uploadSingleFile(r)
	uploadMultiFiles(r)
	r.Run()
}
```

### 9.7 重定向

#### 9.7.1 HTTP重定向

HTTP 重定向很容易。 内部、外部重定向均支持。

```go
r.GET("/test", func(c *gin.Context) {
	c.Redirect(http.StatusMovedPermanently, "http://www.sogo.com/")
})
```

#### 9.7.2 路由重定向

路由重定向，使用`HandleContext`：

```go
r.GET("/test", func(c *gin.Context) {
    // 指定重定向的URL
    c.Request.URL.Path = "/test2"
    r.HandleContext(c)
})
r.GET("/test2", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"hello": "world"})
})
```

### 9.8 Gin路由

#### 9.8.1 普通路由

```go
r.GET("/index", func(c *gin.Context) {...})
r.GET("/login", func(c *gin.Context) {...})
r.POST("/login", func(c *gin.Context) {...})
```

此外，还有一个可以匹配所有请求方法的`Any`方法如下：

```go
r.Any("/test", func(c *gin.Context) {...})
```

为没有配置处理函数的路由添加处理程序，默认情况下它返回404代码，下面的代码为没有匹配到路由的请求都返回`views/404.html`页面。

```go
r.NoRoute(func(c *gin.Context) {
		c.HTML(http.StatusNotFound, "views/404.html", nil)
	})
```

#### 9.8.2 路由组

我们可以将拥有共同URL前缀的路由划分为一个路由组。习惯性一对`{}`包裹同组的路由，这只是为了看着清晰，你用不用`{}`包裹功能上没什么区别。

```go
func main() {
	r := gin.Default()
	userGroup := r.Group("/user")
	{
		userGroup.GET("/index", func(c *gin.Context) {...})
		userGroup.GET("/login", func(c *gin.Context) {...})
		userGroup.POST("/login", func(c *gin.Context) {...})

	}
	shopGroup := r.Group("/shop")
	{
		shopGroup.GET("/index", func(c *gin.Context) {...})
		shopGroup.GET("/cart", func(c *gin.Context) {...})
		shopGroup.POST("/checkout", func(c *gin.Context) {...})
	}
	r.Run()
}
```

路由组也是支持嵌套的，例如：

```go
shopGroup := r.Group("/shop")
	{
		shopGroup.GET("/index", func(c *gin.Context) {...})
		shopGroup.GET("/cart", func(c *gin.Context) {...})
		shopGroup.POST("/checkout", func(c *gin.Context) {...})
		// 嵌套路由组
		xx := shopGroup.Group("xx")
		xx.GET("/oo", func(c *gin.Context) {...})
	}
```

通常我们将路由分组用在划分业务逻辑或划分API版本时。

#### 9.8.3 路由原理

Gin框架中的路由使用的是[httprouter](https://github.com/julienschmidt/httprouter)这个库。

其基本原理就是构造一个路由地址的前缀树。

### 9.9 Gin中间件

Gin框架允许开发者在处理请求的过程中，加入用户自己的钩子（Hook）函数。这个钩子函数就叫中间件，中间件适合处理一些公共的业务逻辑，比如登录认证、权限校验、数据分页、记录日志、耗时统计等。

#### 9.9.1 定义中间件

Gin中的中间件必须是一个`gin.HandlerFunc`类型。

##### 9.9.1.1 记录接口耗时的中间件

例如我们像下面的代码一样定义一个统计请求耗时的中间件。

```go
// StatCost 是一个统计耗时请求耗时的中间件
func StatCost() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Set("name", "小王子") // 可以通过c.Set在请求上下文中设置值，后续的处理函数能够取到该值
		// 调用该请求的剩余处理程序
		c.Next()
		// 不调用该请求的剩余处理程序
		// c.Abort()
		// 计算耗时
		cost := time.Since(start)
		log.Println(cost)
	}
}
```

##### 9.9.1.2 记录响应体的中间件

我们有时候可能会想要记录下某些情况下返回给客户端的响应数据，这个时候就可以编写一个中间件来搞定。

```go
type bodyLogWriter struct {
	gin.ResponseWriter               // 嵌入gin框架ResponseWriter
	body               *bytes.Buffer // 我们记录用的response
}

// Write 写入响应体数据
func (w bodyLogWriter) Write(b []byte) (int, error) {
	w.body.Write(b)                  // 我们记录一份
	return w.ResponseWriter.Write(b) // 真正写入响应
}

// ginBodyLogMiddleware 一个记录返回给客户端响应体的中间件
// https://stackoverflow.com/questions/38501325/how-to-log-response-body-in-gin
func ginBodyLogMiddleware(c *gin.Context) {
	blw := &bodyLogWriter{body: bytes.NewBuffer([]byte{}), ResponseWriter: c.Writer}
	c.Writer = blw // 使用我们自定义的类型替换默认的

	c.Next() // 执行业务逻辑

	fmt.Println("Response body: " + blw.body.String()) // 事后按需记录返回的响应
}
```

##### 9.9.1.3 跨域中间件cors

推荐使用社区的https://github.com/gin-contrib/cors 库，一行代码解决前后端分离架构下的跨域问题。

**注意：** 该中间件需要注册在业务处理函数前面。

这个库支持各种常用的配置项，具体使用方法如下。

```go
package main

import (
  "time"

  "github.com/gin-contrib/cors"
  "github.com/gin-gonic/gin"
)

func main() {
  router := gin.Default()
  // CORS for https://foo.com and https://github.com origins, allowing:
  // - PUT and PATCH methods
  // - Origin header
  // - Credentials share
  // - Preflight requests cached for 12 hours
  router.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"https://foo.com"},  // 允许跨域发来请求的网站
    AllowMethods:     []string{"GET", "POST", "PUT", "DELETE",  "OPTIONS"},  // 允许的请求方法
    AllowHeaders:     []string{"Origin", "Authorization", "Content-Type"},
    ExposeHeaders:    []string{"Content-Length"},
    AllowCredentials: true,
    AllowOriginFunc: func(origin string) bool {  // 自定义过滤源站的方法
      return origin == "https://github.com"
    },
    MaxAge: 12 * time.Hour,
  }))
  router.Run()
}
```

当然你可以简单的像下面的示例代码那样使用默认配置，允许所有的跨域请求。

```go
func main() {
  router := gin.Default()
  // same as
  // config := cors.DefaultConfig()
  // config.AllowAllOrigins = true
  // router.Use(cors.New(config))
  router.Use(cors.Default())
  router.Run()
}
```

#### 9.9.2 注册中间件

在gin框架中，我们可以为每个路由添加任意数量的中间件。

##### 9.9.2.1 为全局路由注册

```go
func main() {
	// 新建一个没有任何默认中间件的路由
	r := gin.New()
	// 注册一个全局中间件
	r.Use(StatCost())
	
	r.GET("/test", func(c *gin.Context) {
		name := c.MustGet("name").(string) // 从上下文取值
		log.Println(name)
		c.JSON(http.StatusOK, gin.H{
			"message": "Hello world!",
		})
	})
	r.Run()
}
```

##### 9.9.2.2 为某个路由单独注册

```go
// 给/test2路由单独注册中间件（可注册多个）
	r.GET("/test2", StatCost(), func(c *gin.Context) {
		name := c.MustGet("name").(string) // 从上下文取值
		log.Println(name)
		c.JSON(http.StatusOK, gin.H{
			"message": "Hello world!",
		})
	})
```

##### 9.9.2.3 为路由组注册中间件

为路由组注册中间件有以下两种写法。

写法1：

```go
shopGroup := r.Group("/shop", StatCost())
{
    shopGroup.GET("/index", func(c *gin.Context) {...})
    ...
}
```

写法2：

```go
shopGroup := r.Group("/shop")
shopGroup.Use(StatCost())
{
    shopGroup.GET("/index", func(c *gin.Context) {...})
    ...
}
```

#### 9.9.3 中间件注意事项

##### 9.9.3.1 gin默认中间件

`gin.Default()`默认使用了`Logger`和`Recovery`中间件，其中：

- `Logger`中间件将日志写入`gin.DefaultWriter`，即使配置了`GIN_MODE=release`。
- `Recovery`中间件会recover任何`panic`。如果有panic的话，会写入500响应码。

如果不想使用上面两个默认的中间件，可以使用`gin.New()`新建一个没有任何默认中间件的路由。

##### 9.9.3.2 gin中间件中使用goroutine

当在中间件或`handler`中启动新的`goroutine`时，**不能使用**原始的上下文（c *gin.Context），必须使用其只读副本（`c.Copy()`）。

### 9.10 运行多个服务

我们可以在多个端口启动服务，例如：

```go
package main

import (
	"log"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"golang.org/x/sync/errgroup"
)

var (
	g errgroup.Group
)

func router01() http.Handler {
	e := gin.New()
	e.Use(gin.Recovery())
	e.GET("/", func(c *gin.Context) {
		c.JSON(
			http.StatusOK,
			gin.H{
				"code":  http.StatusOK,
				"error": "Welcome server 01",
			},
		)
	})

	return e
}

func router02() http.Handler {
	e := gin.New()
	e.Use(gin.Recovery())
	e.GET("/", func(c *gin.Context) {
		c.JSON(
			http.StatusOK,
			gin.H{
				"code":  http.StatusOK,
				"error": "Welcome server 02",
			},
		)
	})

	return e
}

func main() {
	server01 := &http.Server{
		Addr:         ":8080",
		Handler:      router01(),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	server02 := &http.Server{
		Addr:         ":8081",
		Handler:      router02(),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}
   // 借助errgroup.Group或者自行开启两个goroutine分别启动两个服务
	g.Go(func() error {
		return server01.ListenAndServe()
	})

	g.Go(func() error {
		return server02.ListenAndServe()
	})

	if err := g.Wait(); err != nil {
		log.Fatal(err)
	}
}
```



## 五、HTTP

### 1. 概述

编写web的语言：

1. Java
2. php，现在都在尝试用go重写
3. python，豆瓣
4. go，beego，gin两个主流的web框架

https协议：我们使用浏览器访问的时候发送的就是http请求

1. http是应用层协议，底层还是依赖传输层；tcp（短连接），网络层（ip）
2. 无状态的，每一次请求都是独立的，下次请求需要重新建立连接
3. https：
   1. http是标准协议
   2. https = http + ssl（非对称加密，数字证书）
   3. 现在所有网站都尽量要求使用https开发，为了安全

### 2. http请求报文格式

![QQ图片20220125200745](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/202201252008476.png)

一个http请求可以分为4部分：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/202201291100055.png" alt="HTTP消息结构 HTTP请求报文和响应报文的格式" style="zoom:80%;" />

1. 请求行
   1. 格式：方法 + URL + 协议版本号
   2. 实例：POST + /chapter17/user.html + HTTP/1.1
   3. 请求方法：
      1. GET：获取数据
      2. POST：上传数据（表单格式，json格式）
      3. PUT：修改数据
      4. DELETE：用于删除数据
2. 请求头
   1. 格式：key：value
   2. 可以有很多个键值对（包含协议自带，也包含用户自定义的）
   3. 重要的头：
      1. Accept：接收数据的格式
      2. User-Agent：描述用户浏览器的信息
      3. Connection：Keep-Alive（长连接），Close（短连接）
      4. Accept-Encoding：gzip，xxx，描述可接受的编码
      5. Cookie：由服务器设置的key = value数据，客户端下次请求可以携带以验证
      6. Content-Type：
         1. application/-form（表示上传的数据是表单格式）
         2. application/json（表示body的数据是json格式）
      7. 用户可以自定义的：
         1. name：DUKE
         2. age：18
3. 空行
   1. 告诉服务器，请求头结束了，用于分割
4. 请求包体（可选的）
   1. 一般在POST方法时，会配套提供BODY
   2. 在GET的时候也可以提供BODY，但这样容易让人混淆，建议不要这样使用
   3. 上传两种数据格式：
      1. 表单：姓名，性别，年龄
      2. json数据格式

### 3. http响应消息格式

![image-20220129171217268](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/202201291712539.png)

http响应消息格式分为四个部分：

1. 状态行
   1. 协议格式：协议版本号 + 状态码 + 状态描述
   2. 实例：HTTP/1.1 + 200 + OK
   3. 实例：HTTP/1.1 + 404 + Page not found 
   4. 常用状态码
      1. 1xx：客户端可以即时发送请求（一般感知不到）
      2. 2xx：正常访问，200
      3. 3xx：重定向
      4. 4xx：
         1. 401：未授权 not authorized
         2. 404：Not found
      5. 5xx：
         1. 501：Internal Error（服务器内部错误）
2. 响应头
   1. Content-Type：application/json
   2. Server：Apache
   3. Data：Mon，12 Sep
   4. ...
3. 空行
   1. 用于分割，表示下面没有响应头了
4. 响应体
   1. 通常是返回json格式数据

```go
func main() {
	// http包
	client := http.Client{}

	// func (c *Client) Get(url string) (resp *Response, err error) {
	resp, err := client.Get("https://www.baidu.com")
	if err != nil {
		fmt.Println("client.Get err: ", err)
		return
	}
	// 读取Header
	ct := resp.Header.Get("Content-Type")
	data := resp.Header.Get("Date")
	ser := resp.Header.Get("Server")
	fmt.Println("content-type: ", ct)
	fmt.Println("date: ", data)
	fmt.Println("server: ", ser)
	fmt.Println("----------------------")

	// 读取状态行
	url := resp.Request.URL
	code := resp.StatusCode
	status := resp.Status
	fmt.Println("url: ", url)
	fmt.Println("code: ", code)
	fmt.Println("status: ", status)
	fmt.Println("----------------------")

	// 读取响应体
	body := resp.Body
	// func ReadAll(r io.Reader) ([]byte, error)
	readBodyStr, err := ioutil.ReadAll(body)
	if err != nil {
		fmt.Println("read body err: ", err)
		return
	}
	fmt.Println("body string: ", string(readBodyStr))
}
```

### 4. http-server

```go
func main() {
	// 注册路由  router
	// xxx/user  ==>  func1
	// xxx/name  ==>  func2
	// xxx/id  ==>  func3

	// https://127.0.0.1:8080/user	func是回调函数，用于路由的响应，这个回调函数原型是固定的
	http.HandleFunc("/user", func(writer http.ResponseWriter, request *http.Request) {
		// request:   ===> 包含客户端发来的数据
		fmt.Println("用户请求详情: ")
		fmt.Println("request: ", request)
		
        // 这里是具体处理业务逻辑
		// writer:    ===> 通过writer将数据返回给客户端
		_, _ = io.WriteString(writer, "这是/user请求返回的数据!")
	})
	// https://127.0.0.1:8080/name
	http.HandleFunc("/name", func(writer http.ResponseWriter, request *http.Request) {
		_, _ = io.WriteString(writer, "这是/name请求返回的数据!")
	})
	// https://127.0.0.1:8080/id
	http.HandleFunc("/id", func(writer http.ResponseWriter, request *http.Request) {
		_, _ = io.WriteString(writer, "这是/id请求返回的数据!")
	})
	// func ListenAndServe(addr string, handler Handler) error
	// 常规写法
	fmt.Println("Http Server Start ...")
	err := http.ListenAndServe("127.0.0.1:8080", nil)
	if err != nil {
		fmt.Println("http start failed, err: ", err)
		return
	}

	/*// 简便写法
	if err := http.ListenAndServe("127.0.0.1:8080", nil); err != nil {
		fmt.Println("http start failed, err: ", err)
		return
	}*/

}
```

### 5. json

```json
{
    "name": "矮大紧",
    "sex": "man",
    "age": 131,
    "girls": ["金莲", "凤姐", "马融", "春哥"],
    "成绩": [2, 14, 9, 78, 96],
    "家电": {"彩电": "海尔", "洗衣机": "三星"},
    "stars": [
        {"name": "Faye", "address": "北京"},
        {"name": "Andy", "address": "香港"},
        {"name": "Eddie", "address": "台湾"}
    ]
}
```

记住：json最后一个元素后面不能加逗号

#### json编解码

在网络传输的时候，把结构体，编码成json字符串，再进行传输

接收字符串，需要把字符串转换成结构体，然后操作  ==> 字符串 ==> 结构体 ==> 解密

```go
type Student struct {
	Id     int
	Name   string
	Age    int
	gender string // 注意此处是小写  看区别   小写开头的，json编码时会忽略掉
}

func main() {
	lily := Student{
		Id:     1,
		Name:   "lily",
		Age:    18,
		gender: "女",
	}
	// 编码（序列化），结构 => 字符串
	// func Marshal(v interface{}) ([]byte, error)
	encodeInfo, err := json.Marshal(&lily)
	if err != nil {
		fmt.Println("json.Marshal err: ", err)
		return
	}
	fmt.Println("encodeInfo: ", string(encodeInfo)) // encodeInfo:  {"Id":1,"Name":"lily","Age":18}

	// 对端接收到数据
	// 反序列化（解码）：字符串 ==> 结构体

	var lily2 Student
	// func Unmarshal(data []byte, v interface{}) error
	err = json.Unmarshal([]byte(encodeInfo), &lily2)
	if err != nil {
		fmt.Println("json.Unmarshal err: ", err)
		return
	}
	fmt.Println("name: ", lily2.Name)
	fmt.Println("gender: ", lily2.gender)	// 打印空白
	fmt.Println("id: ", lily2.Id)
	fmt.Println("age: ", lily2.Age)
}

```

**注：**struct中小写字母开头的，json编码时会自动忽略掉！！！

![image-20220129211736684](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/202201292117746.png)

#### 结构体标签

```go
type Teacher struct {
	Name    string `json:"-"`                 // 在使用json编码时，这个不参与编码  注意此处是反引号``
	Subject string `json:"Subject_name"`      // 在json编码时，这个字段会编码成Subject_name
	Age     int    `json:"age,string"`        // 在json编码中，将名字转换成age并转换成string类型
	Address string `json:"address,omitempty"` // 在json编码中，如果这个字段为空，那么忽略掉，不参与编码
	gender  string		// 在json编码中，小写字母的不参与编码
}

func main() {
	t1 := Teacher{
		Name:    "Duke",
		Subject: "Golang",
		Age:     18,
		Address: "北京",
		gender:  "男",
	}

	fmt.Println("t1: ", t1)
	encodeInfo, _ := json.Marshal(&t1)
	fmt.Println("encodeInfo: ", string(encodeInfo))
}
```

![image-20220129214534570](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/202201292145927.png)

**总结：**

1. 对于结构体进行编码时，字段首字母必须是大写，否则不参与编码
2. 如果json格式要求key小写，那么可以通过标签（tag）来解决
3. tag细节：

```go
	Name    string `json:"-"`                 // 在json编码时，这个不参与编码  注意此处是反引号``
	Subject string `json:"Subject_name"`      // 在json编码时，这个字段会编码成Subject_name
	Age     int    `json:"age,string"`        // 在json编码中，将名字转换成age并转换成string类型
	Address string `json:"address,omitempty"` // 在json编码中，如果这个字段为空，那么忽略掉，不参与编码
```

## 六、项目Demo

### 6.1 聊天室

#### 1. 功能分析

实现一个网络聊天室（群）：

* 上线下线
* 聊天，其他人、自己都可以看到聊天信息
* 查询当前聊天室用户名字
* 可以修改自己的名字
* 超时踢出

#### 2. 技术分析

技术点分析：

1. socket tcp编程
2. map结构：
   1. 存储所有用户
   2. map遍历
   3. map删除

3. go程、channel
4. select（超时退出，主动退出）
5. timer定时器

#### 3. 实现基础

1. 思路分析

   * tcp socket，建立多连接

   ```go
   for {
   		fmt.Println("主go程监听中...")
   		// 监听连接
   		conn, err := listener.Accept()
   		if err != nil {
   			fmt.Println("listener.Accept err: ", err)
   			return
   		}
   		fmt.Println("建立连接成功!")
   
   		// 业务处理
   		go handle(conn)
   	}
   ```

   * 定义User/map结构

   ```go
   		newuser := User{
   			name: clientAddr,
   			id:   clientAddr,
   			msg:  make(chan string), // 需要make空间 否则无法写入数据
   		}
   		// 将user添加到map
   		allUsers[newuser.id] = newuser
   ```

   * 定义message通道

   ```go
   	// 定义一个message全局通道，用于接收任何人发送来的消息
   	var message = make(chan string, 10)
   ```

   * 创建监听广播过程函数

   ```go
   // 向所有的用户广播消息，启动一个全局唯一的go程
   func broadcast() {
   	fmt.Println("广播go程启动成功! ")
   	// 1. 从message中读取数据
   	info := <-message
   	for _, user := range allUsers {
   		user.msg <- info
   	}
   }
   ```

   * 启动，全局唯一

   ```go
   	// 启动全局唯一的go程，负责监听message通道，写给所有的用户
   	go broadcast()
   	fmt.Println("服务器启动成功! ")
   ```

   * 写入上线信息

   ```go
   		// 将用户上线的消息写入message，用于广播给所有用户
   		loginInfo := fmt.Sprintf("[%s]:[%s] ===> 上线了login! ", newuser.id, 		newuser.name) // 拼接字符串时，最好都用这种 少用 +
   		message <- loginInfo
   ```

   * user监听通道

   每个用户还需要一个用来监听各自msg通道的go程，负责将数据返回给客户端
   
   ```go
   func writeBackToClient(user *User, conn net.Conn) {
   	//TODO
   	fmt.Printf("user: %s 的go程正在监听自己的msg管道: \n", user.name)
   	for data := range user.msg {
   		fmt.Printf("user: %s 写回给客户端的数据为: %s\n", user.name, data)
   		// Write(b []byte) (n int, err error)
   		_, _ = conn.Write([]byte(data))
   	}
   }
   ```
   
   ![image-20220216102836156](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/202202161028309.png)

#### 4. 增加功能

##### 1. 查询用户

查询命令：who  ==> 将当前所有登录的用户，展示出来，id，name

##### 2. 重命名

规则：rename|Duke

1. 读取数据判断长度7，判断字符是rename
2. 使用 | 进行分割，获取 | 后面的部分作为名字
3. 更新用户名字newUser.name = Duke
4. 通知客户端，更新成功

```go
else if len(userInput) > 8 && userInput[:7] == "\\rename" {
			/*
				1. 读取数据判断长度7，判断字符是rename
				2. 使用 | 进行分割，获取 | 后面的部分作为名字
			*/
			//arry := strings.Split(userInput, "|")
			//name := arry[1]
			/*
				3. 更新用户名字newUser.name = Duke
				4. 通知客户端，更新成功
			*/
			newUser.name = strings.Split(userInput, "|")[1]
			allUsers[newUser.id] = newUser // 更新map中的user
			newUser.msg <- "rename success! "
```

##### 3. 主动退出

两种形式：==> quit，ctrl+c

用户退出：清理工作

1. 从map中删除
2. 对应的conn要close

每个用户都有自己的watch go程，负责监听退出信号

```go
// 启动一个go程负责监听退出信号，触发后，进行清零工作: delete map, close conn都在这里进行处理
func watch(user *User, conn net.Conn, isQuit <-chan bool) {
	fmt.Println("-2- 启动监听退出信号的go程...")
	defer fmt.Println("watch go程退出! ")
	for {
		select {
		case <-isQuit:
			logoutInfo := fmt.Sprintf("%s exit already! ", user.name)
			fmt.Println("删除当前用户: ", user.name)
			delete(allUsers, user.id)
			message <- logoutInfo
			conn.Close()
			return // 防止内存泄露
		}
	}
}
```

在handle中启动watch go程

```go
	// 定义一个退出信号，用于监听client的状态
	var isQuit = make(chan bool)
	// 启动go程，负责监听退出信号
	go watch(&newUser, conn, isQuit)
```

在read之后，通过读取的cnt判断用户退出，向isQuit中写入信号：

```go
	// 读取客户端发来的数据
		cnt, err := conn.Read(buf)
		if cnt == 0 {
			fmt.Println("客户端主动关闭ctrl+c, 准备退出! ")
			// map删除，conn close掉
			// 服务器还可以主动退出
			// 在这里不进行真正的退出动作，而是发出一个退出信号，统一做退出处理，可以使用新的管道来做信号传递
			isQuit <- true
		}
```

##### 4. 超时退出

使用定时器来进行超时管理

如果60s内没有发送任何数据，那么直接将这个连接关闭掉：

```go
<- time.After(60 * time.second)  // chan time
```

创建restTimer管道用来当作重置器：

```go
	//创建一个用于重置计数器的管道，用于告知watch函数，当前用户正在输入
	var restTimer = make(chan bool)
	
	// 启动go程，负责监听退出信号
	go watch(&newUser, conn, isQuit, restTimer)
```

每次写完数据后进行重置：

```go
		restTimer <- true // 重置
```

```
		case <-time.After(5 * time.Second):
			logoutInfo := fmt.Sprintf("%s exit timeout!\n", user.name)
			fmt.Println("删除当前用户: ", user.name)
			delete(allUsers, user.id)
			message <- logoutInfo
			conn.Close()
			return // 防止内存泄露
		case <-restTimer:
			fmt.Printf("连接%s 重置计数器!\n", user.name)

```

##### 5.上锁

```go
```

### 6.2 用户管理系统

> **需求：**实现用户的插入、修改、删除，并能够打印用户明细表。

#### 6.2.1 程序框架图

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20220715232200624.png" alt="image-20220715232200624" style="zoom:50%;" />

#### 6.2.2 显示主菜单、退出软件

> **功能说明：**
>
> - 用户打开可以看到页面
>

#### 6.2.3 显示客户列表



---

## 八、高并发与微服务

**==《Go语言高并发与微服务实战》==**

### 1. 云原生

#### 1.1 什么是云原生？

```mermaid
graph
云原生 --> DevOps
云原生 --> 持续集成
云原生 --> 微服务架构
云原生 --> 容器化
```

### 1.2 微服务

##### 1.2.1 系统架构的演进

```mermaid
graph LR
单体架构 --> 垂直分层架构 --> SOA面向服务架构 --> 微服务架构 --> 云原生架构
```

1. 单体架构

缺点：局部改动就需要重新部署，不易扩展，不支持多语言技术栈。

2. 垂直分层架构

##### 1.2.2 常见微服务框架

- Java

  - Spring Cloud
  - Dubbo

- Go

  - Go Kit
  - Go Micro

- NodeJS

  - Seneca

  

