# 

前言：**用来记录Go的语言特性，主要参考为B站《幼麟实验室》及《深度探索Go语言》。

# 一、基础知识

## 1.1 数据结构

### 1.1.1 String

**编码**：**定长编码非常浪费内存**，所以采用**变长编码**。**那么怎么划分边界呢？**最高几位空出来作为标识位，标记该字符占用几个字节。

这是 Go 默认的编码方式：UTF-8 编码：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230405223347820.png" alt="image-20230405223347820" style="zoom: 33%;" />

---

==**string的结构：**==

- **一个起始地址**：用来标记字符串起始位置；但是怎么找到结尾呢？C 是在字符串结尾处放个`\0`，但这限制了内容不能出现这个字符，所以 Go 并不这样做；
- **一个字符串长度：**用来标记该字符有多长(**字节个数**，而**不是字符个数**！)，这样就能够找到字符串在哪结束了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230405223805558.png" alt="image-20230405223805558" style="zoom:33%;" />

**Go 的字符串内容不能修改**，所以 Go 的编译器会把定义好的字符串内容分配到只读内存段。可以通过`[]byte()`将字符串转换为字节slice，这样会为slice变量重新分配一段内存，并拷贝之前字符串的内容，可脱离只读内存的限制。

> #### 问：rune 和 byte 的区别？

**本质区别就是：**

```go
type byte = uint8
type rune = int32
```

- rune 等同于 `int32`，即4个字节长度，常用来处理 unicode 或 utf-8 字符。比如用来处理**中文字符**。
- byte 等同于 `uint8`，即一个字节长度，常用来处理 ascii 字符(共128个)。

在go中修改字符串，需要先将字符串转化成数组，[]byte 或 []rune，然后再转换成 string 型。

```go
str := "你好 world"
// str[i] 其实是 byte
for i := 0; i < len(str); i++ {
    fmt.Printf("%c", str[i]) // ä½&nbsp;å¥½ world
}
// 使用range，其实是使用rune类型来编码的，rune类型用来表示utf8字符，一个rune字符由一个或多个byte组成。
for _, value := range str {
    fmt.Printf("%c", value) // 你好 world
}
```

### 1.1.2 Slice

#### 1.1.2.1 数据结构

**Slice 有三个部分：**

- 元素存哪里(data)
- 存了多少个元素(len)
- 可以存多少个元素(cap)

例如：

- **若通过`var ints []int`或`new`创建数组**，data是该数组的起始地址，但初始化为nil，len=0，cap=0，因为不会开辟底层内存给它；

- **若通过`make([]int, 2, 5)`创建数组**，不仅会分配上述三部分，还会**开辟一段内存**作为它的底层数组；

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222212427138.png" alt="image-20230222212427138" style="zoom:33%;" />

#### 1.1.2.2 `append`

`append`可以为没有底层内存空间的Slice开辟一段内存，并赋值。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222212806168.png" alt="image-20230222212806168" style="zoom: 33%;" />

可以把不同的Slice关联到同一个数组，它们会**共用底层数组**，例如：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222213129238.png" alt="image-20230222213129238" style="zoom: 33%;" />

这三个Slice访问和修改的都是**同一个底层数组**，`s1`和`s2`若访问超过其len的元素，会产生访问越界。

**上图中，若再给`s2`添加元素会怎样？**这个底层数组是不能用了，得开辟新数组，原来的元素要拷贝过去，还要添加新元素，此时`s2`就不指向**原底层数组**了：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222213434400.png" alt="image-20230222213434400" style="zoom:33%;" />

#### 1.1.2.3 ==扩容规则==

当使用`append`添加元素，容量不够时，该怎么重新**分配容量**呢？

1. **预估扩容后的容量：**

   **预估规则：**

   - `oldLen*2 < cap`：`旧容量*2` 还是小于最少要分配的容量，那么就分配最少要分配的容量，此处为5；
   - 否则：`oldLen < 1024`时，直接 *2，`oldLen >= 1024`时，就先扩 1/4。

   <img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222213854160.png" alt="image-20230222213854160" style="zoom:33%;" />

2. **预估元素需要占用多大内存：**直接分配 `预估容量 * 元素类型大小` 内存可以嘛？**不可以的！**因为不一定有刚好一样大的**预置规格**的内存块。这就是第三步要做的事。

3. **匹配到合适的内存规格：**之前例子中，预估容量为 5，64位下就需要申请 40 字节内存，而**最接近的内存规格**为 48 字节。也就是能装 6 个该元素，所以 **扩容后容量为 6**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222214929976.png" alt="image-20230222214929976" style="zoom:33%;" />

4. 分配完新的底层 Array 空间后，就把原 Array 中的数据拷贝到新的上面。

> #### 问：Slice 和 Array 有啥区别？

- ```go
  arr := [2]int{1, 2} 	// 声明了一个数组
  sli := []int{1, 2}		// 声明了一个Slice
  ```

- Array 长度是固定的；Slice 是动态数组，长度可变；

- Slice 是在 Array 基础上实现的，Slice有仨字段，首个就是对底层 Array 的**引用**，所以 Slice 是引用型。

在 C 语言中，数组变量是指向第一个元素的指针，但是 Go 语言中并不是。

由于值传递，数组进行赋值、传递时，实际上会复制整个数组：

```go
a := [...]int{1, 2, 3} 	// ... 会自动计算数组长度
b := a					// 值拷贝，复制整个数组
a[0] = 100
fmt.Println(a, b) 		// [100 2 3] [1 2 3]

// 为了避免复制数组，一般会传递指向数组的指针。
a := [...]int{1, 2, 3}
b := &a
(*b)[0] = 100
fmt.Println(a, *b)		// [100 2 3] [100 2 3]
```

> #### 问：Slice 的性能陷阱？

1. **大量内存得不到释放**

**在已有切片的基础上进行切片，不会创建新的底层数组。因为原来的底层数组没有发生变化，内存会一直占用，直到没有变量引用该数组**。因此很可能出现这么一种情况，原切片由大量的元素构成，但是我们在原切片的基础上切片，虽然只使用了很小一段，但底层数组在内存中仍然占据了大量空间，得不到释放。比较推荐的做法，使用 `copy` 替代 `re-slice`。

```go
// 1. 无法释放 origin，造成内存浪费
func lastNumsBySlice(origin []int) []int {
	return origin[len(origin)-2:]
}

// 2. 自动回收 origin，节省内存
func lastNumsByCopy(origin []int) []int {
	result := make([]int, 2)
	copy(result, origin[len(origin)-2:])
	return result
}
```



### 1.1.3 结构体与内存对齐

太强了，看视频吧去。拨云见日，茅塞顿开！

---

#### 1.1.3.1 访问内存

CPU是将**内存地址**通过**地址总线**传输给**内存**，**内存**准备好数据后通过**数据总线**传给CPU：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222220413977.png" alt="image-20230222220413977" style="zoom:25%;" />

若想一次读取 8字节 的数据，就需要64位数据总线，这里的数据总线位数就是**机器字长**。**如何能只传输一个地址，读取 8字节 数据呢？**

为了更高的访问效率，内存布局如下：就是8个`chip`排列在一起(每个`chip`由8个`bank`组成，即 1字节 内存)，共用同一个内存地址，各自寻找 1字节，然后组合起来成为 8字节：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222221021697.png" alt="image-20230222221021697" style="zoom:50%;" />

所以每次访问内存都只能从`起始地址%8==0`处开始访问。

#### 1.1.3.2 内存对齐

为了提高访问效率(**保证一次读取**)，编译器会把各种类型的数据**安排到合适的地址**，并**占用合适的长度**。

内存对齐要求**数据存储起始地址**，及**占用的字节数**都要是它**对齐边界**的**倍数**：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222222250526.png" alt="image-20230222222250526" style="zoom:33%;" />

#### 1.1.3.3 结构体对齐

**共两个条件：**

- 各成员要对齐边界；
- 结构体总内存大小 % 对齐边界 == 0。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222223638378.png" alt="image-20230222223638378" style="zoom: 50%;" />

可以看出，**结构体各字段的顺序**会影响**结构体占用内存的大小**。

==**为什么要有`结构体总内存大小 % 对齐边界 == 0`？**==

如下情况，若不是整数倍的话，只有第一个T的内存是对齐的，第2个T就没有对齐。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222223858695.png" alt="image-20230222223858695" style="zoom:33%;" />

> #### 问：什么是内存对齐？

**答：**为了提高访问效率，编译器会把各种类型的数据**安排到合适的地址**，并**占用合适的长度**。内存对齐要求**数据存储起始地址**，及**占用的字节数**都要是它**==对其边界==**的**倍数**。

> #### 问：如何确定数据类型的对齐边界？

**机器字长**是**最大对齐边界**，而**数据类型的对齐边界**是：`min(类型大小, 最大对齐边界)`。

==为什么不**统一按照最大对齐边界**或**类型大小**分配呢？==

**为了减少浪费，提高性能。**

如果是`int8`类型，只占 1字节，若对齐到最大边界，就会浪费 7字节，所以对齐到 1字节 最合适；

如果是`int64`类型，占用 8字节，若对齐到 8字节，如下情况中就会浪费前面的 6字节，所以对齐到最大对齐边界最合适。

如果是结构体类型，对齐边界就是包含类型中最大的对齐边界。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230222223422115.png" alt="image-20230222223422115" style="zoom: 33%;" />

> #### 问：为什么要内存对齐？

**有些CPU能访问任意地址**，是因为做了处理：比如读`1-8`的内存：会先读`0-7`，只取`1-7`，再读`8-15`，只取`8`，组合起来就是`1-8`，但是这样会降低访问效率。**为了避免这样读取，因此要内存对齐**。

### 1.1.4 Map

#### 1.1.4.1 概述

Map主要由**哈希函数**和**桶**组成。

- **哈希函数**用来实现`key-value`的映射，是决定哈希表的读写性能的关键，**哈希函数映射的结果一定要尽可能均匀**。如果使用结果分布较为均匀的哈希函数，那么哈希的增删改查的时间复杂度为 `O(1)`；但是如果哈希函数的结果分布不均匀，那么所有操作的时间复杂度可能会达到 `O(n)`。

- **桶**用来解决哈希映射的冲突问题，常见方法有：**开放寻址法**和**拉链法**。

==**Point 1：**==

因为哈希之后的地址空间通常远大于实际地址空间，因此需要对哈希值进行处理，常用有两种方法：

- 取模法：`hash % m`；
- **与运算(Go采用)：**`hash & (m-1)`。这里`m`必须是`2`的整数次幂，这样可以确保不会出现`空桶`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221203151836038.png" alt="image-20221203151836038" style="zoom: 25%;" />

**==Point 2：==**

上述对哈希值的操作会造成哈希冲突，因此需要对哈希冲突进行处理，常用有两种方法，如图所示：

- **开放寻址法：**
  - **写入：**对当前元素进行哈希映射得到地址，若该地址空闲，可以直接填入；若已被占用，则需要将当前元素分配在该地址**后**的空闲地址；
  - **读取：**若映射地址中存储元素的`Key`与当前元素的`Key`不同，则需要遍历该地址之后的地址空间，直到地址为空或找到目标`Key`。
- **拉链法（最常用）：**
  - **写入：**对当前元素进行哈希映射得到地址，若该地址空闲，可以直接填入；若已被占用，则需要在链表的**末尾追加**新的键值对；
  - **读取：**遍历映射地址的键值对链表，直到搜索到相同的`Key`(存在)或链表末尾(不存在)。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221203152250505.png" alt="image-20221203152250505" style="zoom: 25%;" />

#### 1.1.4.2 数据结构

[`runtime.hmap`](https://draveness.me/golang/tree/runtime.hmap) 是最核心的结构体：

```go
type hmap struct {
	count     int		// 记录 已存储键值对的数目
	flags     uint8		
    B         uint8		// 记录 桶(buckets)的数目是2的多少次幂
	noverflow uint16	// 记录 使用的溢出桶的数目
	hash0     uint32	// 哈希的种子，为哈希函数的结果引入随机性

	buckets    unsafe.Pointer	// 记录 桶的位置
    oldbuckets unsafe.Pointer	// 记录 扩容阶段旧桶的位置，大小是当前 buckets 的一半
	nevacuate  uintptr			// 记录 扩容阶段旧桶迁移进度：下一个要进行迁移的旧桶编号

	extra *mapextra				// 记录 溢出桶相关信息
}

// 记录溢出桶相关信息
type mapextra struct {
	overflow    *[]*bmap		// 记录 目前已被使用的溢出桶的地址
	oldoverflow *[]*bmap		// 记录 扩容阶段旧桶使用到的溢出桶的地址
	nextOverflow *bmap			// 下一个空闲溢出桶
}

type bmap struct {
    topbits  [8]uint8		// 每个topbit都是对应hash值的高8位，索引
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr		// 溢出桶，布局与常规bmap相同，是为了减少扩容次数
}
```

Go语言中，**Map类型的变量**本质上是一个指向`hmap`结构体的**指针**。

使用的**桶的数据结构**为`bmap`结构，每一个`bmap`都能存储` 8 `个键值对，当哈希表中存储的数据过多，单个桶已经装满时就会使用 `extra.nextOverflow` 中的桶存储溢出的数据。随着哈希表存储的数据逐渐增多，我们会**扩容哈希表**或者**使用更多溢出桶**存储溢出的数据。

#### 1.1.4.3 初始化

##### ==字面量：==

Go语言中通过`key: value`的方法表示键值对，可以通过如下方式初始化：

```go
hash := map[string]int{
	"1": 2,
	"3": 4,
	"5": 6,
}
```

- 当**`哈希表中的元素数量 <= 25 `**时，编译器会将字面量初始化的结构体转换成以下的代码，将所有的键值对一次加入到哈希表中：

```go
hash := make(map[string]int, 3)
hash["1"] = 2
hash["3"] = 4
hash["5"] = 6
```

- 当**`哈希表中的元素数量 > 25 `**时，编译器会将字面量初始化的结构体转换成以下的代码，将所有的键值对一次加入到哈希表中：

```go
hash := make(map[string]int, 26)
vstatk := []string{"1", "2", "3", ... ， "26"}
vstatv := []int{1, 2, 3, ... , 26}
for i := 0; i < len(vstak); i++ {
    hash[vstatk[i]] = vstatv[i]
}
```

##### ==运行时：==

根据传入的`B`来确定需要创建的桶的数量：

- 若`桶的数量 < 2^4`，此时认为数据量较小，使用溢出桶的概率较低，因此不创建溢出桶；
- 若`桶的数量 > 2^4`，会额外创建`2^(B-4)`个溢出桶。

**注意：**正常桶和溢出桶在内存中的存储空间是连续的。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204155115482.png" alt="image-20221204155115482" style="zoom: 50%;" />

#### 1.1.4.4 读写操作

##### ==访问：==

共有两种访问方式：

```go
v     := hash[key] // => v     := *mapaccess1(maptype, hash, &key)
v, ok := hash[key] // => v, ok := mapaccess2(maptype, hash, &key)
```

赋值语句左侧接受参数的个数会决定使用的运行时方法：

- 当接受一个参数时，会使用 [`runtime.mapaccess1`](https://draveness.me/golang/tree/runtime.mapaccess1)，该函数仅会返回一个指向目标值的**指针**（感觉这里的指针并不是一个地址，感觉还是目标值）；
- 当接受两个参数时，会使用 [`runtime.mapaccess2`](https://draveness.me/golang/tree/runtime.mapaccess2)，除了返回目标值之外，它还会返回一个用于表示当前键对应的值是否存在的 `bool` 值。

**查找过程:star:：**哈希会依次遍历正常桶和溢出桶中的数据，它会先比较哈希的高 8 位和桶中存储的 `tophash`(**==减少key的对比次数，缩小查找成本==**)，后**比较传入的和桶中的`key`**(**==因此key需要是可比较类型==**)以加速数据的读写。

**每一个桶都是一整片的内存空间**，当发现桶中的 `tophash` 与传入键的 `tophash` 匹配之后，我们会通过指针和偏移量获取哈希中存储的键 `keys[x]` 并与 `key` 比较，如果两者相同就会获取目标值的指针 `values[x]` 并返回。

##### ==写入：==

首先会根据传入的`key`拿到对应的哈希和桶，然后通过遍历比较桶中存储的 `tophash` 和`key`的哈希，

- 如果当前键值对在哈希中**存在**，那么就会直接返回目标区域的内存地址；
- 如果当前键值对在哈希中**不存在**，哈希会为新键值对规划存储的内存地址并存入。

#### 1.1.4.5 :star:扩容

除了用**散列均匀的哈希函数**来提高读写性能，还可以通过**对地址空间适时扩容**减少哈希冲突以提高性能。

**负载因子：**`count/(2^B)`，即`存储键值对的数目/桶的数目`，**用来判断是否需要进行扩容**。**不难理解**，装载因子越大，哈希的读写性能就越差。

**扩容规则：**

- `count/(2^B) > 6.5` –> **翻倍扩容**(`hamp.B++`)；
- 使用了"过多"的溢出桶 -> **等量扩容**。

**注：**"过多" 指：

	1. `B <= 15`时，`noverflow >= 2^B`；
	1. `B > 15`时，`noverflow >= 2^15`。

**翻倍扩容：**会创建旧桶数目2倍的新桶，然后将旧桶中的键值对分流到对应的两个新桶中，b = hash(key) & (2^B-1)，(这个地方很有意思)。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204163810403.png" alt="image-20221204163810403" style="zoom: 33%;" />

**等量扩容：**所谓**等量扩容**就是创建和旧桶数目一样多的新桶，然后把原来的键值对迁移到新桶中。**这里有一个问题：**既然是等量的，那何必扩容呢？

**答：**当发生大规模删除操作时，旧桶中存放的键值对可能非常稀疏，因此为了紧凑内存，需要将这些键值对重新排列到新桶中。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204163312783.png" alt="image-20221204163312783" style="zoom: 33%;" />

**==Point 3==**

扩容时，**需要把旧桶中的数据迁移到新桶**，但并不需要一次性迁移完。

**渐进式扩容：**把**键值对迁移的时间**分摊到**多次哈希表操作**中，可以**避免瞬时的性能抖动**。在哈希表每次进行读写操作时，如果检测到当前处于**扩容阶段**，就完成一部分**键值对迁移任务**，直到所有的旧桶迁移完毕。

**具体过程：**扩容时，字段`oldbuckets`指向旧桶，`nevacuate`记录下一个待迁移的旧桶。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221203160841963.png" alt="image-20221203160841963" style="zoom:25%;" />

个人觉得，Go本质上还是拉链法，但是它做了一些内存上的优化，以空间换时间，给每个桶预先分配可以放8个数据的空间，和传统的拉链法相比，**好处就是不需要频繁的分配内存**，同时在某些极限情况下，也可以节省一些空间，传统的拉链法还需要存放前后数据的指针，在64位机器上又是16个字节的开销，但是用数组的方式组织的话，就不需要存储前后的指针。如果一个桶里面存放了超过8个数据，还是需要另一个bmap来放多余的数，然后把两个bmap连接起来。我觉得对比一下，传统的拉链法就是一个节点只能放一个数据，而go是一个节点可以放8个数据，这8个数据是按照数组来组织的。

#### 1.1.4.5 小结

Go 语言使用**拉链法**来解决哈希碰撞的问题实现了哈希表，它的访问、写入和删除等操作都在编译期间转换成了运行时的函数或者方法。哈希在每一个桶中存储键对应哈希的前 8 位，当对哈希进行操作时，这些 `tophash` 就成为可以帮助哈希快速遍历桶中元素的缓存。

**哈希表的每个桶都只能存储 8 个键值对**，一旦当前哈希的某个桶超出 8 个，新的键值对就会存储到哈希的**溢出桶**中。随着键值对数量的增加，溢出桶的数量和哈希的装载因子也会逐渐升高，超过一定范围就会触发扩容，扩容会将桶的数量翻倍，元素再分配的过程也是在调用**写操作**时增量进行的，不会造成性能的瞬时巨大抖动。

### 1.1.5 sync.map

> #### 问：sync.map 和 mutex+map 有啥区别？sync.map 为啥要有俩 map(read+dirty)？一个不行吗？

一个是不行的，你总不能读不加锁，写加锁吧？这样照样会有并发访问的问题(是会报错的)。但是如果是 read+dirty，就可以对 read 的所有操作都不加锁，对 dirty 的所有操作都要加锁，就可以避免并发问题。我觉得最关键的差别就是这个地方！也是很精妙的地方！

---

参考文章：

- https://eddycjy.com/posts/go/sync-map/

- https://mp.weixin.qq.com/s/vtvlye801ePWRY7RvtkDmA :star:

先来看看 `sync.map` 的底层数据结构：

```go
type Map struct {
    mu Mutex						// 保护 dirty 和 read
    read atomic.Value 				// readOnly，只读类型，所以是并发安全的
    dirty map[interface{}]*entry	// 一个非线程安全的原始 map
    misses int						// 计数作用。当读数据时，该字段不在 read 中，尝试从 dirty 中读取，不管是否在 dirty 中读取到数据，misses+1。当 misses 累计到 len(dirty) 时，会将 dirty 拷贝到 read 中，并将 dirty 清空，以此提升读性能。
}

// read 的类型为 atomic.Value，它会通过 atomic.Value 的 Load 方法将其断言为 readOnly 对象
read, _ := m.read.Load().(readOnly) // m 为 sync.Map
// read 的真实类型即是 readOnly
type readOnly struct {
    m       map[interface{}]*entry	// read 中的 go 内置 map 类型，但是它不需要锁。
    amended bool	// 当 sync.Map.diry 中的包含了某些不在 m 中的 key 时，amended 的值为 true.
}

// map 存储的值类型是 *entry，它包含一个指针 p，指向用户存储的 value 值。
type entry struct {
    p unsafe.Pointer // *interface{}
}
```

- **`mu`**：**保护 dirty**；
- **`read`**：**存只读数据**。读是并发安全的，但如果要更新 `read`，则需要加锁保护；
- **`dirty`**：**包含最新写入的数据**。当 misses 计数达到一定值，将其赋值给 read；
- **`misses`**：**计数作用**。当读数据时，该字段不在 read 中，尝试从 dirty 中读取，不管是否在 dirty 中读取到数据，misses+1。当 misses 累计到 len(dirty) 时，会将 dirty 拷贝到 read 中，并将 dirty 清空，以此提升读性能。

#### 1.1.5.1 查询数据

`Load()`:

- 首先，**从只读数据`read`中读取**(因为不用加锁)，若有就直接返回，若无再继续往下；
- 若`read`没有，且`dirty`中有新数据，就会**去`dirty`查找**；
- **先加锁**，然后再次检查`read`，若还没有，才**真正去`dirty`查找**，并且对`miss`计数器+1(无论是否找到)；
- 若`miss`的值 >= `dirty`中的元素数量，就把`dirty`赋给`read`，因为穿透次数太多了，然后就可以把`dirty`置为空了。

```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    // 首先从 m.read 中通过 Load 方法得到 readOnly
    read, _ := m.read.Load().(readOnly)
    // 从 read 中的 map 中查找 key
    e, ok := read.m[key]
    
    // 如果 read 没有，并且 dirty 有新数据，那么去 dirty 中查找
    if !ok && read.amended {
        m.mu.Lock()
        // 双重检查：避免在本次加锁的时候，有其他 goroutine 正好将 Map 中的 dirty 数据复制到了 read 中。
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        
        // 如果 read 中还是不存在，并且 dirty 中有新数据
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // m 计数 +1
            m.missLocked()
        }
        
        m.mu.Unlock()
    }
    
    if !ok {
        return nil, false
    }
    return e.load()	// 返回指针指向的值
}

func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    
    // 将dirty置给read，因为穿透概率太大了(原子操作，耗时很小)
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil	// 清空 dirty
    m.misses = 0	// 重置 misses
}
```

#### 1.1.5.2 删除数据

`Delete()`：

- 首先，从只读数据`read`中读取，若有的话，就直接从`read`中"删除"(并非真的删除，只是标记一下)；
- 若`read`中没有，就获得锁，然后再检查一遍`read`，然后就从`dirty`中删除；

```go
func (m *Map) Delete(key interface{}) {
    // 读出 read，断言为readOnly 类型
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    // 如果 read 中没有，并且 dirty 中有新元素，那么就去 dirty 中去找。这里用到了 amended，当 read 与 dirty 不同时为 true，说明 dirty 中有 read 没有的数据。
    if !ok && read.amended {
        m.mu.Lock()
        // 再检查一次，因为前文的判断和锁不是原子操作，防止期间发生了变化。
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        
        if !ok && read.amended {
            // 直接删除
            delete(m.dirty, key)
        }
        m.mu.Unlock()
    }
    
    if ok {
        // 如果 read 中存在该 key，则将该 value 赋值 nil(采用标记的方式删除！)
        e.delete()
    }
}

// 如果 read 中有该键，则从 read 中删除，其删除方式是通过原子操作
func (e *entry) delete() (hadValue bool) {
    for {
        p := atomic.LoadPointer(&e.p)
        // 如果 p 指针为空，或者被标记清除
        if p == nil || p == expunged {
            return false
        }
        
        // 通过原子操作，将 e.p 标记为 nil
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return true
        }
    }
}
```

#### 1.1.5.3 增改数据

`Store()`：

- 首先，从只读数据`read`中获取该`key`，若存在，并且没有被标记删除，就尝试更新；
- 如果在`read`中不存在或已被标记删除，就在`dirty`中判断是否存在，若已存在，就尝试更新；
- 若在`dirty`中不存在，就加入`dirty`；

```go
func (m *Map) Store(key, value interface{}) {
    // 如果 m.read 中存在该键，且该键没有被标记删除(expunged)
    // 则尝试直接存储(见 entry 的 tryStore 方法)
    // 注意：如果 m.dirty 中也有该键(key 对应的 entry)，由于都是通过指针指向，所有 m.dirty 中也会保持最新 entry 值。
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }
    
    // 如果不满足上述条件，即 read 不存在或者已经被标记删除
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
   
    if e, ok := read.m[key]; ok { // read 存在该 key
    // 如果 entry 被标记 expunge，则表明 dirty 没有 key，可添加入 dirty，并更新 entry。
        if e.unexpungeLocked() { 
            // 加入 dirty 中，这儿是指针
            m.dirty[key] = e
        }
        // 更新 entry 指向新的 value 地址
        e.storeLocked(&value) 
        
    } else if e, ok := m.dirty[key]; ok { // read 不存在该 key，但 dirty 存在该 key，更新
        e.storeLocked(&value)
        
    } else { // read 和 dirty 都没有
        // 如果 read 与 dirty 相同，则触发一次 dirty 刷新(因为当 read 重置的时候，dirty 已置为 nil 了)
        if !read.amended { 
            // 将 read 中未删除的数据加入到 dirty 中
            m.dirtyLocked() 
            // amended 标记为 read 与 dirty 不相同，因为后面即将加入新数据。
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value) 
    }
    m.mu.Unlock()
}

// 将 read 中未删除的数据加入到 dirty 中
func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }
    
    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    
    // 遍历 read
    for k, e := range read.m {
        // 通过此次操作，dirty 中的元素都是未被删除的，可见标记为 expunged 的元素不在 dirty 中！！！
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

// 判断 entry 是否被标记删除，并且将标记为 nil 的 entry 更新标记为 expunge
func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    
    for p == nil {
        // 将已经删除标记为 nil 的数据标记为 expunged
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}

// 对 entry 尝试更新(原子 cas 操作)
func (e *entry) tryStore(i *interface{}) bool {
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
        return false
    }
    for {
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
    }
}

// read 里将标记为 expunge 的更新为 nil
func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

// 更新 entry
func (e *entry) storeLocked(i *interface{}) {
    atomic.StorePointer(&e.p, unsafe.Pointer(i))
}
```

---

在此，给出 map+锁 和 sync.map 的性能测试代码：

`benchmark-for-map.go`：

```go
package mapbench

import "sync"

type MyMap struct {
	sync.Mutex
	m map[int]int
}

type MyRwMap struct {
	sync.RWMutex
	m map[int]int
}

var myMap *MyMap
var myRwMap *MyRwMap
var syncMap *sync.Map

func init() {
	myMap = &MyMap{
		m: make(map[int]int, 100),
	}
	myRwMap = &MyRwMap{
		m: make(map[int]int, 100),
	}

	syncMap = &sync.Map{}
}

func builtinMapStore(k, v int) {
	myMap.Lock()
	defer myMap.Unlock()
	myMap.m[k] = v
}

func builtinMapLookup(k int) int {
	myMap.Lock()
	defer myMap.Unlock()
	if v, ok := myMap.m[k]; !ok {
		return -1
	} else {
		return v
	}
}

func builtinMapDelete(k int) {
	myMap.Lock()
	defer myMap.Unlock()
	if _, ok := myMap.m[k]; !ok {
		return
	} else {
		delete(myMap.m, k)
	}
}

func builtinRwMapStore(k, v int) {
	myRwMap.Lock()
	defer myRwMap.Unlock()
	myRwMap.m[k] = v
}

func builtinRwMapLookup(k int) int {
	myRwMap.RLock()
	defer myRwMap.RUnlock()
	if v, ok := myRwMap.m[k]; !ok {
		return -1
	} else {
		return v
	}
}

func builtinRwMapDelete(k int) {
	myRwMap.Lock()
	defer myRwMap.Unlock()
	if _, ok := myRwMap.m[k]; !ok {
		return
	} else {
		delete(myRwMap.m, k)
	}
}

func syncMapStore(k, v int) {
	syncMap.Store(k, v)
}

func syncMapLookup(k int) int {
	v, ok := syncMap.Load(k)
	if !ok {
		return -1
	}

	return v.(int)
}

func syncMapDelete(k int) {
	syncMap.Delete(k)
}
```

`benchmark-for-map_test.go`：

```go
package mapbench

import (
	"math/rand"
	"testing"
	"time"
)

func BenchmarkBuiltinMapStoreParalell(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		r := rand.New(rand.NewSource(time.Now().Unix()))
		for pb.Next() {
			// The loop body is executed b.N times total across all goroutines.
			k := r.Intn(100000000)
			builtinMapStore(k, k)
		}
	})
}

func BenchmarkBuiltinRwMapStoreParalell(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		r := rand.New(rand.NewSource(time.Now().Unix()))
		for pb.Next() {
			// The loop body is executed b.N times total across all goroutines.
			k := r.Intn(100000000)
			builtinRwMapStore(k, k)
		}
	})
}

func BenchmarkSyncMapStoreParalell(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		r := rand.New(rand.NewSource(time.Now().Unix()))
		for pb.Next() {
			// The loop body is executed b.N times total across all goroutines.
			k := r.Intn(100000000)
			syncMapStore(k, k)
		}
	})
}

func BenchmarkBuiltinMapLookupParalell(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		r := rand.New(rand.NewSource(time.Now().Unix()))
		for pb.Next() {
			// The loop body is executed b.N times total across all goroutines.
			//k := r.Int()
			k := r.Intn(100000000)
			builtinMapLookup(k)
		}
	})
}

func BenchmarkBuiltinRwMapLookupParalell(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		r := rand.New(rand.NewSource(time.Now().Unix()))
		for pb.Next() {
			// The loop body is executed b.N times total across all goroutines.
			//k := r.Int()
			k := r.Intn(100000000)
			builtinRwMapLookup(k)
		}
	})
}

func BenchmarkSyncMapLookupParalell(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		r := rand.New(rand.NewSource(time.Now().Unix()))
		for pb.Next() {
			// The loop body is executed b.N times total across all goroutines.
			k := r.Intn(100000000)
			syncMapLookup(k)
		}
	})
}

func BenchmarkBuiltinMapDeleteParalell(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		r := rand.New(rand.NewSource(time.Now().Unix()))
		for pb.Next() {
			// The loop body is executed b.N times total across all goroutines.
			k := r.Intn(100000000)
			builtinMapDelete(k)
		}
	})
}

func BenchmarkBuiltinRwMapDeleteParalell(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		r := rand.New(rand.NewSource(time.Now().Unix()))
		for pb.Next() {
			// The loop body is executed b.N times total across all goroutines.
			k := r.Intn(100000000)
			builtinRwMapDelete(k)
		}
	})
}

func BenchmarkSyncMapDeleteParalell(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		r := rand.New(rand.NewSource(time.Now().Unix()))
		for pb.Next() {
			// The loop body is executed b.N times total across all goroutines.
			k := r.Intn(100000000)
			syncMapDelete(k)
		}
	})
}
```

## 1.2 语言基础

### 1.2.1 函数调用

#### 1.2.1.1 栈帧布局

按照编程语言定义的**函数**，会被编译器编译为一堆**机器指令**，写入**可执行文件**，在**运行时**被加载到内存，位于虚拟地址空间的**代码段**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204170330812.png" alt="image-20221204170330812" style="zoom:33%;" />

当出现**函数调用**时，编译器就会对应生成一条`call`指令，程序执行到这条指令时，就会跳转到对应**函数入口处**执行，而每个函数最后都有一个`ret`指令，负责在函数调用结束后，**跳转回调用处**继续执行。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204170633999.png" alt="image-20221204170633999" style="zoom:33%;" />

函数执行时需要足够的内存空间来存放局部变量、参数、返回值等数据，这些数据对应到虚拟地址空间的**栈**。

栈的执行顺序是从**低地址到高地址**，所有函数的**栈帧**格式都是统一的。执行到`call`指令会做两件事情：

- 首先是将下一条指令入栈，也即**返回地址**，当执行完调用函数就会跳转回这里；
- 然后会跳转到被调用函数入口处开始执行，这个过程是通过**偏移量+栈指针sp**完成的。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204171042711.png" alt="image-20221204171042711" style="zoom:33%;" />

指令运行时，CPU用特定寄存器来存储运行时的**栈基和栈指针**，同时也有指令指针寄存器用来存放**下一条要运行的指令**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204171752850.png" alt="image-20221204171752850" style="zoom:33%;" />

Go语言中**不是逐步扩张栈帧**的，而是**一次性分配**。然后通过**栈指针+偏移值**使用栈帧。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204172009476.png" alt="image-20221204172009476" style="zoom:33%;" />

**==Point：==**

**一次性分配栈帧**主要是为了**避免栈访问越界**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204172323096.png" alt="image-20221204172323096" style="zoom: 33%;" />

而Go语言编译器会在**函数头部插入检测代码**，当发现需要进行**"栈增长"**，就会另外分配一段足够大的栈空间，并把原来栈上的数据拷贝过来，同时释放原来那段栈空间。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204172553847.png" alt="image-20221204172553847" style="zoom:33%;" />

**==call和ret指令==：**此处最好是看视频理解。

需要注意的是，可执行文件存放在**代码段**，所用的参数、变量等存放在**栈空间**，**寄存器**是指向栈空间的！

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221204173040262.png" alt="image-20221204173040262" style="zoom:33%;" />

#### 1.2.1.2 传参与返回值

##### ==传值与传引用：==

- **传值：**函数调用时会对参数进行拷贝，被调用方和调用方两者持有不相关的两份数据；
- **传引用：**函数调用时会传递参数的指针，被调用方和调用方两者持有相同的数据，任意一方做出的修改都会影响另一方。

不同语言会选择不同的方式传递参数，Go 语言选择了**传值**的方式，**无论是传递基本类型、结构体还是指针，都会对传递的参数进行拷贝**。

##### ==传参：==

**过程：**下图`swap`函数并不能实现交换a, b的作用。

1. 首先需要分配`main`的栈帧空间，即`BP of main` 和 `SP of main`中间的空间；
2. 将`main`中的局部变量入栈；
3. **从右至左**依次将参数入栈(**值拷贝**)；
4. `call`会将返回地址入栈，即`return addr`；
5. 然后就是`swap`的栈帧了；
6. 可以发现，执行交换操作只是对参数(同时也是`swap`的内部变量)进行操作，对`main`中原本的数据并不能造成影响，因此交换失败。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205104655719.png" alt="image-20221205104655719" style="zoom: 33%;" />

**下图`swap`函数能实现交换a, b的作用。**

1. 前两步同上；
2. **从右至左**依次将参数入栈，这里同样是**值拷贝**，只不过值为a、b的地址；
3. 交换时，是直接将`addrA`和`addrB`指向的数据进行交换，因此可以交换成功。



<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205105352049.png" alt="image-20221205105352049" style="zoom:33%;" />

**==Point：==**

**Go 语言中传指针也是传值。**将指针作为参数传入某个函数时，**函数内部会复制指针**，也就是会同时出现两个指针指向原有的内存空间。因此，在传递数组或者内存占用非常大的结构体时，我们应该尽量使用**指针**作为参数类型来避免发生数据拷贝进而影响性能。

##### ==返回值：==

通常返回值是通过寄存器返回，但是Go支持返回**多个返回值**，也就是有可能**返回值的数目大于寄存器个数**，因此Go选择在**栈上存储返回值**。

**匿名返回值：**下图返回结果为1。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205110334567.png" alt="image-20221205110334567" style="zoom:33%;" />

**被调用函数**执行完毕，`ret`会给**返回值赋值**，并**执行`defer`函数**，这里有一个问题：是先给**返回值赋值**还是先**执行`defer`函数**呢？

**答：**先给返回值赋值，再执行`defer`函数。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205110746761.png" alt="image-20221205110746761" style="zoom:33%;" />

**命名返回值：**

和上边那个完全相同，**只改动一个地方**，返回结果为2。**==这是因为被调用函数的返回值b一直在调用者栈帧中。==**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205111156927.png" alt="image-20221205111156927" style="zoom:33%;" />

**参数空间分配：**

若调用多个函数，将以**最大的参数+返回值空间**为标准来分配。如下图，将会按照`B`的参数+返回值空间进行分配，`B`执行完后，C的参数+返回值会**从低地址到高地址**进行分配，**不满的空间就空着**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205112136866.png" alt="image-20221205112136866" style="zoom: 33%;" />

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205112622817.png" alt="image-20221205112622817" style="zoom: 50%;" />

#### 1.2.1.3 小结

Go 通过**栈**传递函数的**参数和返回值**，==**参数**和**返回值**居然是在**调用者的栈帧**中！！！==在**调用函数之前**会在栈上为返回值分配合适的内存空间，随后将入参**从右到左**按顺序压栈并拷贝参数，返回值会被存储到调用方预留好的栈空间上，我们可以简单总结出以下几条规则：

1. 通过堆栈传递参数，入栈的顺序是**从右到左**，而参数的计算是**从左到右**；
2. 函数返回值通过堆栈传递并由调用者**预先**分配内存空间；
3. 调用函数时都是**传值**，接收方会对入参进行复制再计算；

### 1.2.2 闭包

函数，可以作为**参数传递**，可以做**函数返回值**，也可以绑定到**变量**。称这样的参数、返回值或变量为**`function value`**。

`function value`本质上是一个指针，但是并不直接指向函数指令入口，而是指向一个`runtime.funcval`的结构体，这个结构体里只有一个地址，就是这个二函数指令的入口地址。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230223210607640.png" alt="image-20230223210607640" style="zoom:33%;" />

将一个函数赋值给多个变量，这些变量会共用同一个`funcval`。

那么既然`funcval`中只有一个地址，为啥不直接使用这个地址呢？

**这是为了处理闭包。**

闭包的一个示例：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230223211130381.png" alt="image-20230223211130381" style="zoom:33%;" />

Go 语言中，闭包就是有捕获列表的`Function Value`。捕获列表中存储的值，通过`funcval`的地址+偏移量来获取。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230223212134241.png" alt="image-20230223212134241" style="zoom:25%;" />

**==闭包需要维持捕获变量在外层函数和内层函数中的一致性。==**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230223212638218.png" alt="image-20230223212638218" style="zoom: 25%;" />

### 1.2.3 方法

#### 1.2.3.1 方法的本质

如果定义一个类型A，并给他关联一个方法，然后就可以通过类型A的**变量**来调用这个方法，这种调用方式其实是"语法糖"，实际上和下边那个调用方式一样。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205145309906.png" alt="image-20221205145309906" style="zoom:33%;" />

Go语言中，**函数类型**只和**参数与返回值**相关，所以下边输出为`True`，可以说明==**方法**本质上就是**普通函数**，**接收者就是隐含的第一个参数**==。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205150521000.png" alt="image-20221205150521000" style="zoom:33%;" />

#### 1.2.3.2 方法调用

其实和函数调用一样。

**举例说明过程：**

1. Go语言中传参值拷贝，此处为**值接收者**，所以参数首先为：`data=addr1 4`；
2. 执行`Name()`中第一行时，`data`指向新的`string`，更新为：`data=addr2 8`；
3. 返回值就是值拷贝的参数，因此`main`中的局部变量并未受到影响。

![image-20221205151005651](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205151005651.png)

**再举个例子对比：**

1. Go语言中传参值拷贝，此处为**指针接收者**，所以参数首先为：`pa=&a`；
2. 执行`Name()`中第一行时，修改`pa`地址处的变量，也就是`a`，`a`指向新的`string`，更新为：`data=addr2 8`；
3. 返回值就是值拷贝的参数，即`a`指向的变量。此处`main`中的局部变量`a`也会被修改。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205152132567.png" alt="image-20221205152132567" style="zoom:50%;" />

上述例子中，通过**值**调用**值接收者的方法**，通过**指针**调用**指针接收者的方法**，那么如果用**值**调用**指针接收者的方法**或用**指针**调用**值接收者的方法**，是否可行呢？

**答：**可行，这些也是"语法糖"，在**编译阶段**，编译器会进行转换。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205152718495.png" alt="image-20221205152718495" style="zoom:50%;" />

#### 1.2.3.3 方法表达式和方法变量

来看看将一个**方法**赋给一个**变量**是怎么一回事？

Go语言中，**函数**作为**变量、参数和返回值**时都是以`Function Value`形式存在的，**闭包**也只是有**捕获列表**的`Function Value`而已。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205154108887.png" alt="image-20221205154108887" style="zoom: 50%;" />

先来看看什么是**方法表达式**和**方法变量**：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205154459886.png" alt="image-20221205154459886" style="zoom:50%;" />

**从本质上讲，方法表达式和方法变量都是`Function Value`。**

看这段代码：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221205155047495.png" alt="image-20221205155047495" style="zoom:50%;" />

### 1.2.4 接口

#### 1.2.4.1 概述

**计算机科学**中，接口是计算机系统中多个组件共享的边界，不同的组件能够在边界上交换信息。如下图所示，接口的本质是引入一个新的中间层，调用方可以通过接口与具体实现分离，解除上下游的耦合，上层的模块不再需要依赖下层的具体模块，只需要依赖一个约定好的接口。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221206204054161.png" alt="image-20221206204054161" style="zoom:50%;" />

**Go 语言**中的接口是一种内置的类型，它定义了一组方法的签名。Go 语言中**接口的实现都是隐式的**，类型实现接口时只需要实现接口中的全部方法。

- 在 Java 中：实现接口需要显式地声明接口并实现所有方法；
- 在 Go 中：实现接口的所有方法就隐式地实现了接口；

#### 1.2.4.2 数据结构

Go 语言根据接口类型**是否**包含**一组方法**将接口类型分成了两类：

- 使用 [`runtime.iface`](https://draveness.me/golang/tree/runtime.iface) 结构体表示**包含方法**的接口，又称**非空接口类型**；
- 使用 [`runtime.eface`](https://draveness.me/golang/tree/runtime.eface) 结构体表示**不包含任何方法**的 `interface{}` 类型，又称**空接口类型**，可以接收任意类型的数据；

**[`runtime.eface`](https://draveness.me/golang/tree/runtime.eface) 结构体：**

```go
type eface struct { // 16 字节
	_type *_type			// 动态类型：表示类型元数据
	data  unsafe.Pointer	// 动态值：记录在哪
}
```

举例说明：赋值前`_type`和`data`字段都为`nil`，赋值后：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221206210352897.png" alt="image-20221206210352897" style="zoom:50%;" />

**[`runtime.iface`](https://draveness.me/golang/tree/runtime.iface) 结构体：**

```go
type iface struct { // 16 字节
	tab  *itab				// 记录动态类型和方法列表
	data unsafe.Pointer		// 动态值
}

type itab struct { // 32 字节
	inter *interfacetype	// 接口的类型元数据，包含接口的方法列表：需要实现的方法
	_type *_type			// 接口的动态类型元数据
	hash  uint32			// 类型哈希值：用于快速判断类型是否相等，类型断言时会用到
	_     [4]byte
	fun   [1]uintptr		// 方法地址数组：用于快速调用方法，无需再去接口的类型元数据寻找方法地址
}
```

- `hash` 是对 `_type.hash` 的拷贝，当我们想将 `interface` 类型转换成具体类型时，可以使用该字段快速判断目标类型和具体类型 [`runtime._type`](https://draveness.me/golang/tree/runtime._type) 是否一致；
- `fun` 是一个动态大小的数组，它是一个用于动态派发的虚函数表，存储了一组函数指针。虽然该变量被声明成大小固定的数组，但是在使用时会通过原始指针获取其中的数据，所以 `fun` 数组中保存的元素数量是不确定的。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221206212446139.png" alt="image-20221206212446139" style="zoom:50%;" />

举例说明：赋值前`tab`和`data`字段都为`nil`，赋值后：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221206211141071.png" alt="image-20221206211141071" style="zoom:50%;" />

**==Point：==**

`itab`结构体内容一旦确定(接口类型和动态类型)，实际上是不会改变的，因此是可复用的。Go语言会将`itab`缓存起来，并且以接口类型和动态类型的组合为`key`(`接口类型的hash` ^ `动态类型的hash`)，以`&itab`为`value`构造一个哈希表。需要一个`itab`时，会先在这里边寻找，如果已经有对应的`itab`，就直接拿来用，如果没有，就新创建并添加。

### 1.2.5 类型断言

类型断言作用在**抽象类型**上，包括：**空接口和非空接口**。而**断言的目标类型**可以是**具体类型**或**非空接口类型**。这样就组合出来了四种类型断言：

- 空接口.(具体类型)
- 非空接口.(具体类型)
- 空接口.(非空接口)
- 非空接口.(非空接口)

#### 1.2.5.1 空接口.(具体类型)

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221209152915781.png" alt="image-20221209152915781" style="zoom:50%;" />

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221209153010631.png" alt="image-20221209153010631" style="zoom:50%;" />

#### 1.2.5.2 非空接口.(具体类型)

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221209153420393.png" alt="image-20221209153420393" style="zoom:50%;" />

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221209153459034.png" alt="image-20221209153459034" style="zoom:50%;" />

#### 1.2.5.3 空接口.(非空接口)

![image-20221209154140906](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221209154140906.png)

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221209154454602.png" alt="image-20221209154454602" style="zoom:50%;" />

#### 1.2.5.4 非空接口.(非空接口)

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221209154710761.png" alt="image-20221209154710761" style="zoom:50%;" />

####  1.2.5.5 小结

类型断言的关键是：明确接口的**==动态类型==**及对应的**动态类型实现了哪些方法**，而明确这些的关键又在于动态类型的**类型元数据**，以及**空接口**与**非空接口**的**"数据结构"**。

### 1.2.6 反射

**反射的作用**就是**把类型元数据暴露给用户使用**。

反射包中有两对非常重要的函数和类型，两个函数分别是：

- [`reflect.TypeOf`](https://draveness.me/golang/tree/reflect.TypeOf) 能获取类型信息；
- [`reflect.ValueOf`](https://draveness.me/golang/tree/reflect.ValueOf) 能获取数据的运行时表示。

两个类型是 [`reflect.Type`](https://draveness.me/golang/tree/reflect.Type) 和 [`reflect.Value`](https://draveness.me/golang/tree/reflect.Value)，它们与函数是一一对应的关系：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221209173456424.png" alt="image-20221209173456424" style="zoom:50%;" />

#### 1.2.6.1 三大法则

 **Go 语言反射的三大法则：**

1. 从 `interface{}` 变量可以反射出反射对象；
2. 从反射对象可以获取 `interface{}` 变量；
3. 要修改反射对象，其值必须可设置；

##### 法则1

为什么是从 `interface{}` 变量到反射对象？我们也可以执行`reflect.ValueOf(1)`呀，1是`int`类型呀，并不是`interface{}`？

由于 [`reflect.TypeOf`](https://draveness.me/golang/tree/reflect.TypeOf)、[`reflect.ValueOf`](https://draveness.me/golang/tree/reflect.ValueOf) 两个方法的入参都是 `interface{}` 类型，所以在方法执行的过程中发生了类型转换。因为 Go 语言的函数调用都是值传递的，所以变量会在**函数调用时**进行类型转换。基本类型 `int` 会转换成 `interface{}` 类型，这也就是为什么第一条法则是从接口到反射对象。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221210154544265.png" alt="image-20221210154544265" style="zoom:50%;" />

- 通过`TypeOf`获得变量类型；
- 通过`ValueOf`获得变量值；
- 然后就可以通过`Method`获得类型实现的方法；
- 通过`Field`获得类型包含的全部字段。

##### 法则2

从反射对象可以获取 `interface{}` 变量。[`reflect`](https://golang.org/pkg/reflect/) 中的 [`reflect.Value.Interface`](https://draveness.me/golang/tree/reflect.Value.Interface) 就能完成这项工作。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221210155135702.png" alt="image-20221210155135702" style="zoom:50%;" />

不过调用 [`reflect.Value.Interface`](https://draveness.me/golang/tree/reflect.Value.Interface) 方法只能获得 `interface{}` 类型的变量，还需要进行类型断言才能变成原类型。

**从反射对象到接口值的过程是从接口值到反射对象的镜面过程，两个过程都需要经历两次转换：**

- 从接口值到反射对象：
  - 从基本类型到接口类型的类型转换；
  - 从接口类型到反射对象的转换；
- 从反射对象到接口值：
  - 反射对象转换成接口类型；
  - 通过显式类型转换变成原始类型(如果原来就是接口类型，那么不必这一步)。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221210155748212.png" alt="image-20221210155748212" style="zoom:50%;" />

##### 法则3

假如想要更新原变量的值，如果这样写，是不行的：

```go
func main() {
	i := 1
	v := reflect.ValueOf(i)
	v.SetInt(10)
	fmt.Println(i)
}
```

需要这样写：

```go
func main() {
	i := 1
	v := reflect.ValueOf(&i)
	v.Elem().SetInt(10)
	fmt.Println(i)
}
```

1. 调用 [`reflect.ValueOf`](https://draveness.me/golang/tree/reflect.ValueOf) 获取变量指针；
2. 调用 [`reflect.Value.Elem`](https://draveness.me/golang/tree/reflect.Value.Elem) 获取指针指向的变量；
3. 调用 [`reflect.Value.SetInt`](https://draveness.me/golang/tree/reflect.Value.SetInt) 更新变量的值：

#### 1.2.6.2 类型和值

##### TypeOf

###### 如何传递参数

可以思考一下此处栈上参数列表应该是怎样的？

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221210162119892.png" alt="image-20221210162119892" style="zoom:50%;" />

由于`TypeOf`的参数是空接口类型，**空接口是动态类型**，实际大小不知道，因此需要传递地址，但是如果传递`a`的地址进去，就会违背Go语言值拷贝的特性，可能会修改原`a`的值，那么应该怎么做呢？

实际上在编译阶段，会对`a`进行copy，实际传的是`copy of a`的地址，**所有参数为空接口类型的情况，都是这样**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221210162601767.png" alt="image-20221210162601767" style="zoom:50%;" />

###### 返回值是什么？

通过`TypeOf`返回的，其实是一个非空接口变量：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221210163130131.png" alt="image-20221210163130131" style="zoom:50%;" />

##### ValueOf

###### 参数传递

同`TypeOf`，同时，`ValueOf`会将这个临时变量地址显式的**逃逸到堆上**。

例：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221210163726208.png" alt="image-20221210163726208" style="zoom:50%;" />

`a`的地址被显式的逃逸到堆，注意此处的返回值`ptr=&a`：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221210163931017.png" alt="image-20221210163931017" style="zoom:50%;" />

接下来调用`v.Elem()`，会拿到`v.ptr`指向的变量`a`，并把它**包装成`reflect.Value`类型的返回值**(ptr又变为&a)，赋值给`v`：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221210164411855.png" alt="image-20221210164411855" style="zoom:50%;" />

然后在调用`v.SetString()`，此时通过参数`v`拿到`a`的地址，修改的就是原来的`a`：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221210164522130.png" alt="image-20221210164522130" style="zoom:50%;" />

### 1.2.7 方法集

> #### **问题：**`T`和`*T`的方法集是啥关系？

`T`和`*T`是两种类型，分别有着自己的类型元数据，而根据自定义类型的类型元数据，可以找到该类型关联的方法列表，既然`T`和`*T`各有各的方法集，**那为什么还要限制`T`和`*T`不能定义同名方法？**又怎么会有**"`*T`的方法集包含`T`的方法集"这种说法？**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230104202426028.png" alt="image-20230104202426028" style="zoom:33%;" />

**答：**首先，可以确定的是，`T`的方法集里，全部都是有明确定义的接收者为`T`类型的方法；而`*T`的方法集里，除了有明确定义的接收者为`*T`的方法以外，**还会有编译器生成的一些"包装方法"：**这些包装方法是对接收者为`T`类型的同名方法的"包装"，**为什么编译器要为接收者为`T`的方法包装一个接收者为`*T`的同名方法呢？**

你可能会想到 Go 支持通过指针变量访问方法。**但这里首先要明确一点**：通过`*T`类型的变量直接调用`T`类型接收者的方法只是一种**语法糖**，经验证，这种调用方式，编译器会再调用端进行指针解引用，**并不会用到这里的包装方法**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230104203305015.png" alt="image-20230104203305015" style="zoom:33%;" />

==**实际上，**编译器生成包装方法主要是**为了支持接口**。==

非空接口的数据结构只包含两个指针，一个和类型元数据相关，一个和接口装载的数据相关，虽然有数据指针，但却不能像上述语法糖那样，通过指针解引用来调用值接收者的方法。**原因：**方法的接收者是方法调用时隐含的第一个参数，Go 中的函数参数是通过栈来传递的，如果参数指针类型，那就很好实现：平台确定了，指针大小就确定了。但如果要解引用为值类型，就要有明确的类型信息，编译器才能确定这个参数要在栈上占用多大的内存空间。**而对于接口**，编译阶段并不能确定它会装载哪一类的数据，所以编译器并不能生成对应的指令来解引用。

**总而言之，==接口不能直接使用接收者为值类型的方法==。**

针对这个问题，编译器选择为值类型接收者的方法，**生成指针接收者"同名"包装方法**这一解决方案。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230104204206737.png" alt="image-20230104204206737" style="zoom: 33%;" />

**因此，**回到最初的问题：**"为什么还要限制`T`和`*T`不能定义同名方法？"**，如果给`T`和`*T`定义了同名方法，就有可能和编译器生成的包装方法发生冲突，所以 Go 干脆不允许为`T`和`*T`定义同名方法。

至于，**"`*T`的方法集包含`T`的方法集的所有方法"**，这种说法可以这样理解。虽然**编译器会为所有接收者为`T`的方法生成接收者为`*T`的包装方法**，但是**链接器会把程序中确定用不到的方法都裁剪掉**，所以如果去分析可执行文件的话，就会发现不止是这些包装方法，就连我们明确定义的方法，也不一定会存在于可执行文件中。不过，一定要从可执行文件中去分析，**不能通过反射去验证**，因为反射的实现也是基于接口。若通过反射来验证，会被链接器认为用到了这个方法，从而把它保留下来，这就"测不准"了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230104205023358.png" alt="image-20230104205023358" style="zoom:33%;" />

### 1.2.8 泛型

#### go 1.17

在代码中经常会用到一些本地缓存组件，它们是复用性极高的基础组件，在使用体验上和`map`差不多，都提供了`Set`和`Get`方法。**为了支持任意类型**，这些方法都使用了空接口类型的参数，内部实际存储数据的是个值类型为空接口的`map`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230110162400295.png" alt="image-20230110162400295" style="zoom:33%;" />

使用`Set`方法时，一般不会觉得有什么不方便，因为从具体类型或接口类型，到空接口的赋值不需要额外处理。但是`Get`方法使用时，需要通过类型断言，把取出来的数据转换成预期的类型。比如想从本地缓存`c`里面取出来一个`string`，就需要这样写：

```go
if v, ok := c.Get("key"); ok {
    if s, ok := v.(string); ok {
        // use s
    }
}
```

如果是仅仅多这一步，那也无可厚非，可实际上并不这么简单。

**空接口本质上是一对指针，用来装载值类型时会发生装箱，造成变量逃逸。**例如用`Cache`来缓存`int64`类型，缓存对象`c`底层是个`map`，在`map`的`buckets`中存储着元素的哈希值、`key`和`value`，对于`c`而言，`bucket`这里存储的`value`是一个一个的空接口，而实际上的`int64`会在堆上单独分配，空接口的`data`指针指向堆上的`int64`，相较于直接把`int64`存储在`map`的`bucket`里，堆分配方式凭空多出来一次堆分配，而且还多占用了两倍的内存空间。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230111162036321.png" alt="image-20230111162036321" style="zoom:33%;" />

针对这个问题，改造上述缓存为泛型缓存。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230111162647627.png" alt="image-20230111162647627" style="zoom:33%;" />

- 将`Cache`改为`cache`，因为 Go 1.17 的泛型实现还不支持导出，泛型相关的类型和函数只能在当前包中使用；
- Go 1.17 的泛型支持默认是关闭的，构建可执行文件时应指定参数来显式的开启，而且，据观察`build`命令只有在编译`main`包时，才会透传这个参数，这就限制了只能在`main`包中使用泛型。

改造好后，同样用来存储`int64`类型的数值，然后，通过反射观察底层`map`存储的是什么样类型的元素，下边的代码会打印`int64`，也就是说，泛型缓存`cache`的底层`map`会直接在`bucket`中存储`int64`类型的数值，没有额外的堆分配。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230111162701301.png" alt="image-20230111162701301" style="zoom:33%;" />

可以看出，泛型能够解决这个问题。**但是，这不是没有代价的。**

**泛型的实现：**

使用泛型最直接的代价就是：**编译器会为同一套模板的每个类型，都生成一套代码**，可能会导致可执行文件大小有所增加；而且，即便使用泛型，要想在一个缓存对象里面存储多种不同类型的值，依然要使用空接口，否则，一个缓存对象就只能存储一种类型的值。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230111163205226.png" alt="image-20230111163205226" style="zoom:33%;" />

所以说，**泛型本质上是**编译阶段的代码生成，它并不会替代空接口，**空接口主要用来实现语言的动态特性**，它们的适用场景根本不同~

#### go 1.18 

Go 从 1.18 正式开始支持泛型，这里的`T`就是类型参数，与`java`、`c++`对比，没有使用`<>`，而是使用`[]`：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230117214908572.png" alt="image-20230117214908572" style="zoom: 33%;" />

同时，为了让编译器能够更高效的检查类型参数的合法性，还引入了**类型参数的约束条件**。这个约束条件在语法层面是通过**接口**来实现的，它明确描述了调用方传入的类型参数，需要符合什么样的要求。

这里的`fmt.Stringer`接口限制了调用`ToString`方法时，传入的参数必须实现了`String()`方法，如果传入的参数不符合要求，就无法通过编译。当然，此处的接口也并不是原来的接口。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230117215636985.png" alt="image-20230117215636985" style="zoom: 33%;" />

若使用**原来的接口**，就只能通过**接口的方法集**来约束类型参数，但这有时是行不通的。因为，这些内置类型是没有实现任何方法的，比如`int32`、`int64`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230117215956554.png" alt="image-20230117215956554" style="zoom: 33%;" />

例如，写了一个`Sum()`函数，想让它支持所有整型类型，该如何实现呢？

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230117220236547.png" alt="image-20230117220236547" style="zoom:33%;" />

Go 1.18 将接口扩展了一下。原来的接口只能定义`方法集`，**扩展后的接口增加了`类型集`，通过`类型集`指明哪些类型是被支持的**。

例如，`Integer`接口用作约束条件时，就可以支持所有的有符号整型。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230117220749258.png" alt="image-20230117220749258" style="zoom:25%;" />

**但这依然不够**，如果我们自定义了`type MyInt int`类型，同样也希望被支持，这个约束条件又该怎么写呢？总不能每次新增加自定义类型都去改一下接口的类型集吧？

为了解决这个问题，Go 新增加了符号`~`，只要写`~int`，就可以支持**`int`类型**以及**基于`int`创建的所有自定义类型**，

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230117220812122.png" alt="image-20230117220812122" style="zoom:33%;" />

**注意：**这种扩展后的接口，目前只用于泛型的约束条件。

现在有了泛型，只需要实现一遍功能函数，就可以通过多种类型参数来调用。

**问题：**编译器会不会给每种支持的类型都生成一套代码呢？// ==TODO==

貌似不用。通过字典和gcshape实现。没听懂。先这样吧。。。

## 1.3 常用关键字

### 1.3.1 defer

使用`defer`的最常见场景是在函数调用结束后完成一些收尾工作，通常用来：

- 关闭文件描述符；
- 关闭数据库连接；
- 解锁资源

通常使用`defer`会遇到两个常见的问题：

- `defer` 关键字的**调用时机**以及多次调用 `defer` 时**执行顺序**是如何确定的；
- `defer` 关键字使用**传值**的方式传递参数时会进行预计算，导致不符合预期的结果。

##### ==执行顺序==

`deferProc()`负责把要执行的函数信息保存起来，称之为**`defer`注册**；`defer`注册完成后，会继续执行后面的逻辑，直到返回之前通过`deferreturn`执行注册的`defer`函数。**即：先注册，再延迟调用。**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221211161841348.png" alt="image-20221211161841348" style="zoom:50%;" />

`defer`会在函数返回之前，按照**倒序**执行。这是因为`goroutine`运行时会有一个对应的`结构体g`，其中有一个字段指向`defer`链表头，`defer`链表链起来的是一个个`_defer`结构体，**新注册的`defer`会添加到链表头，执行时从头开始**，因此执行顺序是**注册顺序的倒序**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221206203611957.png" alt="image-20221206203611957" style="zoom:50%;" />

##### ==预计算参数==

Go 语言中所有的函数调用都是传值的，虽然 `defer` 是关键字，但是也继承了这个特性。如下代码：

```go
func main() {
	startedAt := time.Now()
	defer fmt.Println(time.Since(startedAt))
	
	time.Sleep(time.Second)
}

$ go run main.go
0s
```

为什么会输出0呢？**这是因为：**调用 `defer` 关键字会**立刻拷贝函数中引用的外部参数**，所以 `time.Since(startedAt)` 的结果不是在 `main` 函数退出之前计算的，而是在 `defer` 关键字调用时计算的，最终导致上述代码输出 0s。

想要解决这个问题的方法非常简单，我们只需要向 `defer` 关键字传入匿名函数：

```go
func main() {
	startedAt := time.Now()
	defer func() { fmt.Println(time.Since(startedAt)) }()
	
	time.Sleep(time.Second)
}

$ go run main.go
1s
```

#### 1.3.1.1 数据结构

`defer`在Go语言中的结构：

```go
type _defer struct {
	siz       int32		// 参数和返回值共占多少字节
	started   bool		// 标记defer是否已经执行
	openDefer bool		// 
	sp        uintptr	// 调用者栈指针
	pc        uintptr	// 返回地址
	fn        *funcval	// defer 关键字中传入的函数，即注册函数
	_panic    *_panic	// 触发延迟调用的结构体，可能为空
	link      *_defer	// 下一个_defer指针
}
```

可以看出[`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体其实是`_defer`调用链表上的一个元素，所有的结构体都会通过 `link` 字段串联成链表。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221211155916102.png" alt="image-20221211155916102" style="zoom:50%;" />

#### 1.3.1.2 执行机制

**堆分配**、**栈分配**和**开放编码**是处理 `defer` 关键字的三种方法。

- Go 1.12 引入：堆分配[`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体；

- Go 1.13 引入：栈分配的结构体；
- Go 1.14- 引入：基于开放编码的 `defer`。

#### 1.3.1.3 堆分配

`deferproc`函数执行时，需要堆分配一段空间，存放`_defer`结构体及参数与返回值。实际上Go语言也会预分配不同规格的`_defer`池，执行时从空闲`_defer`池中取出一个，没有合适的或空闲的就会进行堆分配，用完之后再放入`_defer`池，以避免频繁的堆分配与回收。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221211163551700.png" alt="image-20221211163551700" style="zoom:50%;" />

Go 1.12 `defer`很慢！原因：

- `_defer`结构体堆分配：即使有预分配的`deferpool`，也需要去堆上获取和释放，而且，参数还要在堆栈间来回拷贝；
- `_defer`注册通过链表：链表本身的操作就慢！

#### 1.3.1.4 栈上分配

先来对比一下1.12版本和1.13版本中，`defer`指令编译后有什么不同？

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221212160435327.png" alt="image-20221212160435327" style="zoom:50%;" />

1.12 将`_defer`结构体分配在堆上。在1.13中，通过在编译阶段，增加局部变量，把`defer`信息保存到**当前函数栈**的局部变量区域，再通过`deferprocStack`把栈上这个`_defer`结构体注册到`_defer`链表中，执行依然是通过`deferreturn`实现。**优化点：减少`defer`信息的堆分配。**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221212161026409.png" alt="image-20221212161026409" style="zoom:50%;" />



1.13中的`defer`，官方提供的性能提升是30%。

#### 1.3.1.5 开放编码

Go 1.14是通过在编译阶段插入代码，把`defer`函数的执行逻辑展开在所属函数内，从而免于创建`_defer`结构体，而且不需要注册到`_defer`链表。这种方式省去了构造`_defer`链表项，并注册到链表的过程。举例说明：

`A1`可以简单的通过：定义局部变量，并在`return`前插入调用`A1`的指令 来实现延迟调用效果。但是对于`A2`呢？他在编译阶段是无法确定是否被调用的，因此需要一些处理。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221212161710592.png" alt="image-20221212161710592" style="zoom:50%;" />

Go语言通过一个**标识变量`df`**来解决这个问题。

**`df`变量中的每一位对应一个`defer`函数是否要被执行。**

**这里先以`A1`举例：**

- `df`变量首先需要通过异或运算`|=`将第1位置为1；
- `return`前插入的调用指令也应该进行修改，需要判断`df`变量第1位是否>0，若>0，在调用`A1`前，需要将`df`变量的第1位置为0，这是为了避免重复调用。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221212162439970.png" alt="image-20221212162439970" style="zoom:50%;" />

**再以`A2`举例：**

- 首先插入 **判断是否需要将`A2`的标志位置为1** 的代码；
- 然后在`return`前插入 **检查`df`标志位** 的代码。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221212162804767.png" alt="image-20221212162804767" style="zoom:50%;" />

**这种方法将性能提高了一个数量级，但不是没有代价。**

如果在`code to do something`，也就是还未执行到`defer`函数调用处，就发生了`panic`或`runtime.Goexit`函数，后面这些代码根本无法执行到，因此需要通过**栈扫描**的方式来发现。**所以，`defer`变快了，但`panic`变得更慢了。**

#### 1.3.1.6 小结

**需要注意的是，栈上分配和开放编码的方法均不适用于循环中的`defer`，因此也保留1.12中的堆分配方法。**

### 1.3.2 panic和recover

- **`panic`** 能够改变程序的控制流，调用 `panic` 后会**立刻停止执行当前函数**的剩余代码，并在当前 Goroutine 中递归执行调用方的 `defer`；
- **`recover`** 可以中止 `panic` 造成的程序崩溃。它是一个**只能在 `defer` 中发挥作用**的函数，在其他作用域中调用不会发挥作用。

#### 1.3.2.1 现象

- **跨协程失效：**`panic` 只会触发当前 Goroutine 的 `defer`；
- **失效的崩溃恢复：**`recover` 只有在 `defer` 中调用才会生效；
- **嵌套崩溃：**`panic` 允许在 `defer` 中嵌套多次调用。

##### 跨协程失效

```go
func main() {
	defer println("in main")
	go func() {
		defer println("in goroutine")
		panic("")
	}()

	time.Sleep(1 * time.Second)
}

❯ go run test_go.go
in goroutine
panic:
...
```

上述代码没有执行`main`中的`defer`，只执行了 goroutine 中的`defer`。

**总结：**当程序发生崩溃时只会调用当前 goroutine 的`defer`。

##### 失效的崩溃恢复

```go
func main() {
	defer println("in main")
	if err := recover(); err != nil {
		fmt.Println(err)
	}
	panic("unknown err")
}

❯ go run test_go.go
in main
panic: unknown err
...
```

在主程序中调用 `recover` 试图中止程序的崩溃，但是从运行的结果中我们也能看出，程序没有正常退出。应该如下面这样写：

```go
func main() {
	defer println("in main")
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("错误发现:", err)
		}
	}()
	panic("unknown err")
}

❯ go run test_go.go
错误发现: unknown err
in main
```

可以发现程序正常执行。

**总结：`recover` 只有在发生 `panic` 之后调用才会生效。**然而在上面的控制流中，`recover` 是在 `panic` 之前调用的，并不满足生效的条件，所以**需要在 `defer` 中使用 `recover` 关键字。**

##### 嵌套崩溃

```go
func main() {
	defer println("in main")
	defer func() {
		defer func() {
			panic("panic again and again")
		}()
		panic("panic again")
	}()

	panic("panic once")
}

❯ go run test_go.go
in main
panic: panic once
        panic: panic again
        panic: panic again and again
...
```

可以确定程序多次调用 `panic` 也不会影响 `defer` 函数的正常执行，所以**使用 `defer` 进行收尾工作**一般来说都是安全的。

#### 1.3.2.2 数据结构

`panic` 关键字在 Go 语言的源代码是由数据结构 [`runtime._panic`](https://draveness.me/golang/tree/runtime._panic) 表示的：

```go
type _panic struct {
	argp      unsafe.Pointer	// defer的参数空间
	arg       interface{}		// panic的参数
	link      *_panic			// link to earlier panic
	recovered bool				// 是否被 recover 恢复
	aborted   bool				// 是否被终止
	pc        uintptr
	sp        unsafe.Pointer
	goexit    bool
}
```

和`defer`一样，`panic`也是一个链表，当有新的`panic`出现，也是在链表头插入新的`_panic`结构体。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221213213343330.png" alt="image-20221213213343330" style="zoom:33%;" />

`panic`执行`defer`时，是从头开始执行的。首先把`_defer`的`started`字段置为`true`，标记该`defer`已经开始执行，并且会把`panic`字段指向当前执行的`panic`，表示这个`defer`是由这个`panic`触发的。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221214152803635.png" alt="image-20221214152803635" style="zoom:50%;" />

如果函数`A2`能够正常执行，那么就会移除`A2`。

之所以这样设计，是**为了应对`defer`函数没有正常结束的情况**。假如此时`A2`顺利执行，那么执行`A1`：

因为`A1`也会触发`panic`，那么需要将`panicA1`链在`panic`头部，此时正在执行的`panic`就变成了`panicA1`，然后也会去执行`defer`链表，从标记字段会发现`A1`已经在执行了，且触发他的`panic`是`panicA`，所以会根据该指针找到`panicA`，把他**`aborted`标记为已终止**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221214153050041.png" alt="image-20221214153050041" style="zoom:50%;" />

所以现在`panicA`已终止，`A1`也执行完毕，`A1`会被移除，当前`defer`链表为空。

接下来就该打印`panic`信息了，注意：**`panic`打印异常信息时，会从链表尾开始，即`panic`的发生顺序逐个输出。**所以此处会先`panicA`，再`panic A1`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221214153611418.png" alt="image-20221214153611418" style="zoom:50%;" />

#### 1.3.2.3 recover

上述过程没有添加`recover`，接下来看看添加`recover`之后的情况。

`recover`只做一件事，就是**把当前`panic`的`recovered`字段变为`true`。**

---

**以下为例。**当执行完`A2`时，会将当前`panicA`的`recovered=true`。**每个`defer`执行完毕，`panic`都会检查当前`recovered`是否为`true`**。此时会发现`panicA`已经被恢复了，就会把他从`panic`链表中移除，并且移除`A2`。**不过`A2`被移除之前，要保存`_defer.sp`和`_defer.pc`**，接下来就要利用这两个值跳出`panicA`的处理流程。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221214154559842.png" alt="image-20221214154559842" style="zoom:50%;" />

**`sp`和`pc`是注册`defer`函数时保存的**。这里`sp`就是**函数`A`的栈指针**，而`pc`就是**调用`deferproc`函数的返回地址**。

通过`sp`可以返回到函数`A`的栈帧，通过`pc`可以返回到调用`deferproc`处(即判定r>0)处。若此时`r=0`，那么`code to do something`就会被重复执行，所以会将寄存器中的`r=1`，这样就可以跳转到`deferreturn`处，继续执行`defer`链表。**注意：`deferreturn`只负责执行当前函数`A`注册的`defer`函数，**他是通过栈指针来判断的。

![image-20221214155631217](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221214155631217.png)

所以，`A2`执行完毕，还会继续执行`A1`，执行完毕就结束了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221214160115738.png" alt="image-20221214160115738" style="zoom:50%;" />

---

**这里要注意：**只有当`defer`函数执行完毕，`panic`才会去检查是否被恢复。但是如果`defer`先`recover`，然后又`panic`了呢？

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221214160550712.png" alt="image-20221214160550712" style="zoom:50%;" />

此时，由于发生了`panic`，所以会将`panicA2`注册到`panic`链表头，并成为当前的`panic`，他会去执行`defer`链表，当执行`A2`时，从标记字段发现`A2`已经开始执行，并且触发者是`panicA`，那么会将`panicA`终止(`aborted=true`)，并把`A2`从`defer`链表移除。继续执行下一个`defer`，`A1`就是由`panicA2`触发的了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221214160906485.png" alt="image-20221214160906485" style="zoom:50%;" />

`A1`执行完毕后，会被移除，此时`defer`链表为空。

接下来就要输出`panic`信息了。

**注意：在输出已经被`recover`的`panic`时，打印时会带上`recovered`标记。**`panic`每一项被输出后，程序退出。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221214161119978.png" alt="image-20221214161119978" style="zoom:50%;" />

#### 小结

- 用`defer`进行收尾工作通常是安全的，这是因为`panic`可以终止其他代码的执行，但不会影响到`defer`的执行。

- 在判断`panic`的执行过程时，只需要把握住两个链表`defer`和`panic`的执行顺序即可。

# 二、运行时

## 2.1 并发编程

### 2.1.1 GMP

#### 2.1.1.1 概述

Go语言在并发编程方面有强大的能力，这离不开语言层面对并发编程的支持。谈到Go语言调度器，绕不开的是操作系统、进程与线程这些概念。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221217160458921.png" alt="image-20221217160458921" style="zoom: 50%;" />

**多个线程可以属于同一个进程并共享内存空间。**因为多线程不需要创建新的虚拟内存空间，所以它们也不需要内存管理单元处理上下文的切换，线程之间的通信也正是基于共享的内存进行的，与重量级的进程相比，线程显得比较轻量。这不是挺好的嘛，为啥还要引入`goroutine`？

**引入`goroutine`的原因：**虽然线程比较轻量，但是在调度时也有比较大的额外开销。每个线程都会占用 1M 以上的内存空间，在切换线程时不止会消耗较多的内存，恢复寄存器中的内容还需要向操作系统申请或者销毁资源，每一次线程上下文的切换都需要消耗 ~1us 左右的时间，但是 Go 调度器对 Goroutine 的上下文切换约为 ~0.2us，减少了 80% 的额外开销。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221217161508540.png" alt="image-20221217161508540" style="zoom:50%;" />



**==Point：==**

Go语言中：

- **协程**对应的数据结构是`runtime.g`；
- **工作线程**对应的数据结构是`runtime.m`；
- 后来引入了`runtine.p`，**引入原因：**一开始所有的`g`都在一个全局队列中，多个`m`从全局队列中获取`g`时需要频繁的加解锁及等待；引入`p`后，`m`就可以直接从关联的`p`处获取待执行的`g`，不用每次都和众多`m`从一个全局队列中争抢任务，提高了并发性能。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220154053343.png" alt="image-20221220154053343" style="zoom:50%;" />

其中，`allgs`、`allm`、`allp`分别记录所有的`g`、`m`和`p`。

##### 首先看一个简单的场景

只有一个`mian.main`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220154815913.png" alt="image-20221220154815913" style="zoom:50%;" />

在`main goroutine`创建前，`G`、`P`、`M`的情况如上图。

- `main goroutine`创建后，被加入当前`P`的本地队列中；
- 然后通过`mstart`开启调度循环。这个`mstart`是所有工作线程的入口，主要就是调用`schedule`函数，也就是执行调度循环。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220155225867.png" alt="image-20221220155225867" style="zoom:50%;" />

- 当前队列中只有`main goroutine`等待执行，所以`m0`切换到`main goroutine`；

- 执行入口自然是`runtime.main`，它会做很多事情：创建监控线程、进行包初始化等，包括调用`main.main`，然后就可以执行`main.main`了！

  <img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220155629894.png" alt="image-20221220155629894" style="zoom:50%;" />

##### 一个改进的场景

在`main.main`中又创建新的`goroutine`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220155829901.png" alt="image-20221220155829901" style="zoom:33%;" />

- 我们**通过`go`关键字创建`goroutine`，会被编译器转换为`newproc`函数调用。(`main goroutine`也是由`newproc`创建的)**
- 创建`goroutine`时，我们只负责指定入口、参数，而`newproc`会给`goroutine`构造一个栈帧，目的是让协程任务结束后，返回到`go exit`函数中，进行协程资源回收处理等工作。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220160301974.png" alt="image-20221220160301974" style="zoom:50%;" />

- 新创建的`goroutine`，此处叫做`hello goroutine`，会被添加到当前`P`的本地`runq`中；
- 然后`main.main`就会返回了，然后`exit()`函数被调用，进程就结束了。所以此处`hello goroutine`并未能执行。

**问题：**问题在于，当`main.main`返回后，直接会调用`exit()`函数，会把进程都结束掉，没给`hello goroutine`调度执行的时间。所以应该在`main.main`返回前，拖延点时间给`hello goroutine`执行。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220160532266.png" alt="image-20221220160532266" style="zoom:50%;" />

#### 2.1.1.2 goroutine创建、让出与恢复

还通过一个例子来看一下。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220171138313.png" alt="image-20221220171138313" style="zoom:33%;" />

通过函数栈帧看一下`newproc`的调用过程：

- `main`函数栈帧自然分配在`main goroutine`的协程栈中；

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220171515758.png" alt="image-20221220171515758" style="zoom:50%;" />

- `newproc`主要做的就是**切换到`g0`栈去调用`newproc1`函数**；**为什么要切换到`g0`栈呢？**简单来说，`g0`栈空间大。因为`runtime`中很多函数都有`no-split`标记，意味着这个函数不支持栈增长，而**协程栈**空间本来就小，容易栈溢出，而`g0`的栈直接分配在**线程栈**上，栈空间足够大。

  <img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220172217709.png" alt="image-20221220172217709" style="zoom:50%;" />

- `newproc1(协程入口, 参数地址, 参数大小, 父协程, 返回地址)`；
- `newproc1`首先通过`acquirem()`禁止当前`m`被抢占。**为什么不能被抢占？**因为接下来要执行的程序中，可能会把当前`p`保存到局部变量中，若此时`m`被抢占，`p`关联到别的`m`，等再次恢复时，继续使用这个局部变量里保存的`p`，就会造成问题。所以为了保持数据的一致性，会暂时禁止`m`被抢占。
- 接下来，会尝试获取一个空闲的`g`，如果当前`p`和调度器中都没有空闲的`g`，就创建一个并添加到全局变量`allgs`中。此处我们依然将此添加的`g`记为`hello goroutine`，此时它的状态是`_Gdead`，而且已然拥有自己的协程栈。
- 接下来，如果协程入口函数有参数，就把参数移动(拷贝)到协程栈上。
- 接下来，会把`goexit()`的地址+1，压入协程栈，即返回地址；
- 再把协程(`hello goroutine`)对应的`g`的`startpc`置为协程入口函数的起始地址，`gopc`置为父协程调用`newproc`后的返回地址，`g.sched`结构体用于保存现场：`g.sched.sp`置为协程栈指针，`g.sched.pc`置为协程入口函数的起始地址。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220173903125.png" alt="image-20221220173903125" style="zoom:50%;" />

- 等到这个协程得到调度执行的时候，通过`g.sched`恢复现场，就会从协程入口函数处开始执行，而函数结束后便会返回到`goexit()`中，执行协程资源回收等收尾工作。**到此，协程如何出场与收场就都有了着落。**
- 接下来，`newproc1`还会给新建的`goroutine`赋予一个唯一`id`。给`g.goid`赋值前，会把协程的状态置为`_Grunnable`，这个状态意味着这个`g`可以进到`run queue`中了。
- 所以接下来会调用`runqput`把这个`g`放到当前`p`的本地队列中。
- 接下来判断，如果当前有空闲`p`，而且没有处于`spinning`状态的`m`，即所有`m`都忙，而且主协程已经开始执行了，那么就调用`wakep()`：启动一个`m`并把它置为`spinning`状态。
- 最后与一开始的`acquirem()`呼应，会调用`releasem()`允许当前`m`被抢占；而`spinning`状态的`m`启动后，会一直执行调度循环寻找任务，从本地`runq`到全局`runq`再到其他`p`的`runq`，只为找到个待执行的`g`。但此时，若`main.main`已经返回，那也不会执行剩余的`g`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220175604483.png" alt="image-20221220175604483" style="zoom:50%;" />

**此处我们通过等待一个`channel`来实现调度执行`g`。**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220175922653.png" alt="image-20221220175922653" style="zoom:33%;" />

- 当前`main goroutine`会阻塞在`<-ch`这里等待数据；
- 然后`chanrecv()`通过`gopark()`函数挂起当前`goroutine`，让出`cpu`；
- `gopark()`首先会调用`acquirem()`禁止当前`m`被抢占，然后把`main goroutine`的状态从`_Grunning`修改为`_Gwaiting`，`main goroutine`就不再是执行中状态了；
- 接下来调用`releasem()`解除`m`的抢占禁令；
- 最后调用`mcall(park_m)`：负责保存当前协程的执行现场；
- 然后切换到`g0`栈，调用由`mcall`的参数传入的这个函数，对应到这里就是`park_m()`函数；
- `park_m()`函数会根据`g0`找到当前`m`，把`m.curg`置为`nil`。此时当前`m`正在执行的`g`便不再是`main goroutine`了；
- 最后会调用`schedule()`寻找下一个待执行的`g`。
- 然后，`hello goroutine`要么被当前`m0`调度执行，要么被其他`m`调度执行，总归是能执行了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220181026564.png" alt="image-20221220181026564" style="zoom:50%;" />

- 等到`hello goroutine`执行完毕，关闭`main gorutine`等待的`channel`时，不止会修改`channel`的`closed`状态，还会处理等待队列中的`g`，最终调用`goready()`函数来结束这个`g`的等待状态；
- 而`goready()`函数会切换到`g0`栈，并执行`runtime.ready()`函数，目前待`ready`的协程自然是`main goroutine`，此时它的状态是`_Gwaiting`，接下来会被修改为`_Grunnable`，表示它又可以被调度执行了。
- 然后，它会被放入当前`p`的本地`runq`中，同协程创建时一样，接下来也会检查是否有空闲的`p`，并且没有`spinning`状态的`m`，是的话，也会调用`weakp()`函数启动新的`m`；
- 接下来`hello goroutine`结束，`main goroutine`得到调度执行，最终结束进程。

**总结：底层通过`newproc`创建`goroutine`，通过`gopark`实现协程让出，使用`goready`把协程恢复到`runnable`状态放回到`runq`中**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221220203946776.png" alt="image-20221220203946776" style="zoom:50%;" />

接下来，就需要了解调度循环`schedule()`是做啥的了，以及如何把一个可运行的协程真正的运行起来？

#### 2.1.1.3 goroutine让出、抢占、监控和调度

协程让出CPU分为两种：**主动让出**和**被动让出(抢占)**。其中，主动让出指协程自身要等待某种条件而主动让出，被动让出指调度器通过某种规则强制使协程让出。

##### 主动让出

**主动让出**主要包括三种形式：`time.Sleep()`，`<-chan`和`I/O操作`。

1. `time.Sleep()`

**整体过程：**协程执行`time.Sleep()`时，状态会从`_Grunning`变为`_Gwaiting`，并进入到对应的`timer`中等待，而`timer`中持有一个回调函数，在指定时间到达后**调用**这个回调函数，把等在这里的协程恢复到`_Grunnable`状态并放回到`runq`中。

**这里有一个问题？**谁负责触发`timer`注册的回调函数呢？

**答：**其实每个`p`都有一个**最小堆**，存储在`p.timers`中，用于管理自己的`timer`，堆顶的`timer`就是接下来要触发的那一个。而每次调度时，都会调用`checkTimers()`函数，检查并执行那些已经到时间的`timer`。**不过这不够稳妥**，万一所有的`m`都在忙，那么就不能及时触发调度了，可能会导致`timer`执行时间发生较大偏差。所以还会通过**监控线程**来增加一层保障。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221221161626609.png" alt="image-20221221161626609" style="zoom:50%;" />

(接上)。**监控线程**是由`main goroutine`创建的，与`GPM`中的工作线程不同，并**不需要依赖`p`，也不由`GPM`调度**。

监控线程有多个任务，其中一项便是**保障`timer`的正常执行**。监控线程检测到接下来有`timer`要执行时，若此时无空闲`m`，便会创建新的工作线程`m`以保障`timer`可以顺利执行。

2. `<-chan`

**整体过程：**协程等待一个`channel`时，其状态也会从`_Grunning`变为`_Gwaiting`，并进入到对应`channel`的读队列或写队列中等待，等待的`channel`可读、可写了会通知到相关协程。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221221164504027.png" alt="image-20221221164504027" style="zoom:33%;" />

3. `I/O`

**整体过程：**协程等待一个`I/O事件`，也需要进行让出。以`epoll`为例，若`I/O事件`尚未就绪，需要注册要等待的`I/O事件`到监听队列中，而每个监听对象都可以关联一个`event data`，所以就在这里记录是哪个协程在等待，等到事件就绪时再把它恢复到`runq`中即可。与`timer`和`channel`不同的是，没有谁能通知协程是否就绪，因此想要获取`I/O事件`就绪情况需要**主动轮询(netpoll)**，这里的**主动轮询**是指非阻塞的查询是否有已就绪的IO事件。全局变量`sched`会记录上次`netpoll`执行的时间`lastpoll`，**监控线程**检测到距离上次轮询已经超过`10ms`便会再次执行`netpoll`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221221165530425.png" alt="image-20221221165530425" style="zoom:33%;" />

##### 抢占

**监控线程**要本着**公平调度**的原则，对运行时间过长的`p`实行**"抢占"**操作。就是告诉那些运行时间超过特定阈值(10ms)的`g`，该让出了！

**问题：那么，怎么知道运行时间过长了呢？**

`p`里面有一个`schedtick`字段，每当调度执行一个新的`g`，并且不继承上个`g`的时间片时，就会把`p.schedtick++`，而`sysmontick.schedwhen`记录上一次调度的时间。监控线程如果监测到`sysmontick.schedtick`与`p.schedtick`不相等，说明这个`p`发生了新的调度，就会同步`sysmontick.schedtick`的值，并更新调度时间`sysmontick.schedwhen`；但若二者相等，说明没发生新的调度，或者即使发生了新的调度，也沿用了之前的时间片，所以可以通过当前时间与`sysmontick.schedwhen`的差值来判断当前`p`上的`g`是否运行时间过长。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221221172659457.png" alt="image-20221221172659457" style="zoom: 33%;" />

**问题：那如果真的运行时间过长了，怎么通知它让出呢？**

**答：**`Go 1.14`之前使用的是**栈增长**。栈增长共三种情况：

其中，和**协程调度**相关的是第三种。当`runtime`希望某个协程让出CPU时，就会把他的`stackguard0`(栈下界)赋值为`stackPreempt`，这是一个非常大的值，真正的栈指针不会指向这个位置，因此用作**特殊标识**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221221171258248.png" alt="image-20221221171258248" style="zoom: 50%;" />

进而会跳转到`morestack`处，而`morestack`会调用`runtime.newstack()`函数，负责栈增长工作。不过他在进行栈增长工作前会先**判断`stackguard0`是否等于`stackPreempt`**，等于的话就不进行栈增长了，**而是执行一次协程调度**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221221171537047.png" alt="image-20221221171537047" style="zoom:33%;" />

所以，`Go 1.14`之前实际上是通过设置`stackguard`为`stackPreempt`来完成协程让出的。

这种方式的缺点是过度依赖栈增长代码，如果来个空的`for{}`循环，因为与栈增长无关，程序就会卡死在这个地方。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221221171902137.png" alt="image-20221221171902137" style="zoom:50%;" />

这一问题在`Go 1.14`版本得到解决。因为实现了**异步抢占**。

不同系统中的实现不尽相同，如`Unix`中。会向协程关联的`m`发送信号(sigPreempt)，接下来目标线程会被信号中断，转去执行`runtime.sighandler()`，在`sighandler()`函数中检测到信号为`sigPreempt`后，就会调用`runtime.doSigPreempt()`，它会向当前被打断的协程上下文中注入一个异步抢占函数调用，处理完信号后`sighandler()`返回，**被中断的协程得到恢复**，**立刻执行被注入的异步抢占函数**，该函数最终会调用`runtime`中的调度逻辑，这就让出了！

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221221172540730.png" alt="image-20221221172540730" style="zoom:33%;" />

上述已经回答了问题。

其实，为了充分利用CPU，监控线程还会抢占处在系统调用中的`p`，因为一个协程要执行系统调用，就要切换到`g0`栈，在系统调用没执行完之前，这个**`m`和`g`其实绑定了**，不能被分开，也就**用不到`p`**，所以在陷入系统调用前，当前`m`会让出`p`，解除`m.p`与当前`p`的强关联，只在`m.oldp`中记录这个`p`。`p`的数目毕竟有限，如果有其他协程正在等待执行，那就会把他关联到其他`m`。不过如果当前`m`从系统调用中恢复，会先检测之前的`p`是否被占用，没有的话就继续使用，否则再去申请一个，没申请到的话，就把当前`g`放入全局`runq`中，然后当前线程就睡眠了。

##### 调度

协程让出了、抢占了之后，总需要**给这个`m`再找个待执行的`g`来执行**吧？这就用到了**`schedule()`**。

- 首先，要确定当前`m`是否和当前`g`绑定了。如果绑定了，那当前`m`就不能执行其他`g`，会阻塞当前`m`，等到当前`g`再次得到调度执行时，就会把`m`唤醒；如果没有绑定，就先看看`GC`是不是在等待执行。如果`GC`正等待执行，就去执行`GC`，回来再继续执行调度程序；
- 接下来，会检查有没有要执行的`timer`；
- 获取下一个要执行的`g`时，会先去本地`runq`中查找，没有的话就调用`findrunnable()`，这个函数直到获取到待执行的`g`才会返回：在`findrunnable()`处，也会**先判断是否要执行`GC`**，然后先**尝试从本地`runq`中获取**，没有的话就**从全局`runq`中获取一部分**，如果还没有，就先尝试**执行`netpoll`，恢复那些`I/O事件`已经就绪的`g`**，它们会被放回全局`runq`中，然后才会尝试从**其他`p`那里`steal`一些任务**；
- 当调度程序终于获得一个待执行的`g`后，还需要检查是否已经绑定某个`m`，如果已经绑定了某个`m`，还得把这个`g`送回去，而当前`m`不得不再次进行`schedule()`调度；如果没有绑定的`m`，就调用`execute()`在当前`m`上执行这个`g`；
- `execute()`会建立当前`m`与`g`的关联关系，并把`g`的状态从`_Grunnable`改为`_Grunning`，如果不继承上一个协程的时间片，就把`p`这里的调度计数`p.schedtick`+1；
- 最后，会调用`gogo()`函数，从`g.sched`这里恢复协程栈指针、指令指针等继续执行。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221221180655313.png" alt="image-20221221180655313" style="zoom:50%;" />

### 2.1.2 Mutex

`Mutex`的由来，应该是`mutual exclusion`的前缀组合，称为"互斥锁"。

#### 2.1.2.1 两种模式

Go语言`sync`包中`Mutex`的**数据结构**是这样的：

```go
type Mutex struct {
    state int32		// 存储互斥锁的状态
    sema  uint32	// 信号量, 用作等待队列
}
```

- `state`存储互斥锁的状态，加锁和解锁都是通过`atomic`包提供的函数原子性的操作该字段：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103165015315.png" alt="image-20230103165015315" style="zoom:50%;" />

- `sema`用作一个信号量，主要用作等待队列。

`Mutex`有**两种模式**：

- **正常模式：**在正常模式下，一个尝试加锁的`goroutine`会先自旋几次，尝试通过原子操作获得锁。若**几次自旋(自旋阶段)**后仍不能获得锁，则通过信号量排队等待，所有等待者会按照先入先出(`FIFO`)的顺序排队。

但是当锁被释放，第一个等待者被唤醒后并不会直接拥有锁，而是需要和后来者竞争，也就是那些处于**自旋阶段**，尚未排队等待的`goroutine`，这种情况下后来者更有优势：一方面，它们正在CPU运行，自然比刚唤醒的`goroutine`更有优势，另一方面处于自旋状态的`goroutine`可以有很多，而被唤醒的`goroutine`每次只有一个，所以被唤醒的`goroutine`有很大概率拿不到锁。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103165828162.png" alt="image-20230103165828162" style="zoom:33%;" />

这种情况下它会被重新插入到队列的头部，而不是尾部。而当一个`goroutine`本次加锁等待的时间超过`1ms`后，它会把当前`Mutex`从**正常模式**切换至**饥饿模式**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103170121363.png" alt="image-20230103170121363" style="zoom:50%;" />

- **饥饿模式：**在饥饿模式下，`Mutex`的所有权从执行`Unlock`的`goroutine`，直接传递给等待队列头部的`goroutine`。后来者不会自旋，也不会尝试获得锁，即使`Mutex`处于`Unlocked`状态。它们会直接从队列的尾部排队等待。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103170438409.png" alt="image-20230103170438409" style="zoom:50%;" />

当一个等待者获得锁后，它会在以下两种情况时将`Mutex`由**饥饿模式**切换回**正常模式**：

- 此轮获得锁的等待者的等待时间小于`1ms`，也就是它刚来不久；
- 它是最后一个等待者，等待队列已经空了，后面自然就没有饥饿的`goroutine`了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103170714988.png" alt="image-20230103170714988" style="zoom:33%;" />

**综上所述：**在**正常模式**下自旋和排队时同时存在的，执行`Lock`的`goroutine`会先一边自旋，尝试过几次后如果还没拿到锁，就需要去排队等待了。这种在排队之前先让大家来抢的模式，能够有更高的吞吐量，因为频繁的挂起、唤醒`goroutine`会带来较多的开销。但是又不能无限制的自旋，要把自旋的开销控制在较小的范围内。所以，在正常模式下，`Mutex`会有更好的性能，但是可能会出现队列尾端的`goroutine`迟迟抢不到锁的情况(尾端延迟)。

而**饥饿模式**下不再尝试自旋，所有`goroutine`都要排队，严格的先来后到，对于**防止出现尾端延迟**非常重要。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103171327094.png" alt="image-20230103171327094" style="zoom:33%;" />

#### 2.1.2.2 Lock和Unlock

首先看一下关于`Mutex.state`的几个**常量定义**：`state`的类型是`int32`。

- 第一位用作**锁状态标识**，置为`1`就表示已加锁，对应掩码常量为`mutexLocked`。
- 第二位用于**记录是否已有`goroutine`被唤醒**了，置为`1`表示已唤醒，对应掩码常量为`mutexWoken`。
- 第三位**标识`Mutex`的工作模式**，`0`代表正常模式，`1`代表饥饿模式，对应掩码常量为`mutexStarving`。
- 而常量`mutexWaiterShift`等于`3`，表示除了最低3位以外，`state`的其他位用来**记录有多少个等待者在排队**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103172843866.png" alt="image-20230103172843866" style="zoom:33%;" />

接下来看一下`Lock`和`Unlock`的代码，精简掉注释和部分`race`检测相关的代码。

两个方法中主要通过`atomic`函数实现了`Fast path`，相应的`Slow path`被单独放在了`lockSlow`和`unlockSlow`方法中，这样是为了便于编译器对`Fast path`进行内联优化。

`Lock`方法中的`Fast path`期望`Mutex`处于`Unlocked`状态，没有`goroutine`在排队，更不会饥饿。理想状况下，一个`CAS`操作就可以获得锁。但是如果`CAS`操作没能获得锁，就需要进入`Slow path`，也就是`lockSlow`方法。

`Unlock`方法同理，首先通过原子操作从`state`中减去`mutexLocked`，也就是释放锁。然后根据`state`的新值来判断是否需要执行`Slow path`。如果新值为`0`，也就意味着没有其他`goroutine`在排队，所以不需要执行额外操作；如果新值不为`0`，那就需要进入`Slow path`，看看是不是需要唤醒某个`goroutine`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103173816295.png" alt="image-20230103173816295" style="zoom:33%;" />

#### 2.1.2.3 Slow path

**问题：**当一个`goroutine`尝试给`mutex`加锁时，如果其他`goroutine`已经加了锁还没有释放，而且当前`mutex`工作在正常模式下，**是不是就要开始自旋了呢？**

**答：**不一定。因为如果当前是**单核**场景，或者 **`GOMAXPROCS=1`**，或者**当前没有其他`P`正在运行**。这些情况下**自旋是没有意义的**：

- 自旋的`goroutine`在等待持有锁的`goroutine`释放锁，而持有锁的`goroutine`在等待自旋的`goroutine`让出CPU。
- 除此之外，如果当前`P`的本地`runq`不为空，相较于自旋来说，切换到本地`goroutine`更有效率，所以为保障吞吐量**也不会自旋**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103174535064.png" alt="image-20230103174535064" style="zoom:33%;" />

> #### 问：Goroutine 自旋的条件？

最终，==只有在**多核**场景下，且`GOMAXPROCS > 1`，且至少有一个其他的`P`正在`running`，且当前`P`的本地`runq`为空的情况下，才可以自旋。==

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103174726515.png" alt="image-20230103174726515" style="zoom: 25%;" />



进入自旋的`goroutine`会先去争抢`mutex`的唤醒标识位，设置`mutexWoken`标识位的**目的**是在正常模式下，告知持有锁的`goroutine`在`Unlock`的时候不用再唤醒其他`goroutine`了，已经有`goroutine`在这里等待，以免唤醒太多的等待`goroutine`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103175319004.png" alt="image-20230103175319004" style="zoom: 33%;" />

`Mutex`中的自旋，底层是通过`procyield`循环执行`30`次`PAUSE`，**自旋次数上限为`4`**，而且每自旋一次都要重新判断是否可以继续自旋。如果锁被释放了，或者锁进入了饥饿模式，亦或者已经自旋了`4`次，都会**结束自旋**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103175301300.png" alt="image-20230103175301300" style="zoom:25%;" />

**结束自旋**或者**根本不用自旋**的`goroutine`就该尝试原子操作修改`mutex`的状态了。把此时`mutex.state`保存到`old`中，把要修改为的新`state`记为`new`：

- 如果`old`处于饥饿模式或加锁状态，`goroutine`就得去排队，所以这些情况下排队规模要加`1`。

- 如果是正常模式，就要尝试设置`lock`位，所以`new`中这一位要置为`1`；
- 如果当前`goroutine`等待的时间已经超过`1ms`，而且锁还没被释放，就要将`mutex`的状态切换为饥饿模式。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103180037499.png" alt="image-20230103180037499" style="zoom: 50%;" />

把排队规模和几个标识位都设置好后，在执行原子操作修改`state`前，若是当前`goroutine`**持有唤醒标识**的话，还需要将唤醒标识位**重置**，因为，接下来无论是去抢锁还是单纯去排队：

- 如果**原子操作成功**了，要么成功抢到了锁，要么是成功进到了等待队列，当前`goroutine`都不再是被唤醒的`goroutine`了，所以要释放唤醒标识。
- 而如果**原子操作失败**，也就意味着其他`goroutine`在我们保存`mutex.state`到`old`后又修改了`state`的值，当前`goroutine`就要回过头去继续从自旋检查这里开始再次尝试，所以也需要释放自己之前抢到的唤醒标识位，从头再来。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103181323794.png" alt="image-20230103181323794" style="zoom:33%;" />

##### Lock操作的`Slow path`

继续展开**原子操作成功**的分支：

- 如果是**抢锁操作成功**了，那么加锁的`Slow path`就可以宣告结束了；
- 如果是**排队规模设置成功**了，还要决定是排在等待队列头部还是尾部。如果当前`goroutine`已经排过队了，是在`Unlock`时从等待队列中唤醒的，那就要排到等待队列头部；如果是第一次排队，就得排到等待队列尾部，并且从第一次排队开始**记录当前`goroutine`的等待时间**。**接下来**就会让出，进到等待队列里，队列里的`goroutine`被唤醒时，要从**上次让出的地方**开始继续执行。**接下来**会判断，如果`mutex`处在正常模式，那就接着从自旋开始抢锁，如果唤醒后`mutex`处在饥饿模式，那就没有其他`goroutine`会和自己抢了，锁已经轮到自己这里，只需要把`mutex.state`中`lock`标识位设置为加锁，把等待队列规模减去1，再看看是不是要切换到正常模式，也就是自己的等待时间是不是小于`1ms`，或者等待队列已经空了，最后设置好`mutex.state`就一切ok了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103182744173.png" alt="image-20230103182744173" style="zoom:50%;" />

##### Unlock操作的Slow path

进到`Unlock`的`Slow path`说明除去`lock`标识位以外，剩下的位不全为`0`。

- 如果处在正常模式，若等待队列为空，或者已经有`goroutine`被唤醒或获得了锁，或者锁进入了饥饿模式，那就不需要唤醒某个`goroutine`，直接返回即可。**否则**就要尝试抢占`mutexWoken`标识位，获取唤醒一个`goroutine`的权利。抢占成功后，就会通过`runtime_Semrelease`函数唤醒一个`goroutine`；如果抢占不成功就进行循环尝试，直到等待队列为空，或者已经有一个`goroutine`被唤醒或获得了锁，或者锁进入了饥饿模式，则退出循环。

- 而在饥饿模式下，后来的`goroutine`不会争抢锁，而是直接排队，锁的所有权是直接从执行`Unlock`的`goroutine`传递给等待队列中首个等待者的，所以不用抢占`mutexWoken`标识位。第一个等待者唤醒后，会继承当前`goroutine`的时间片立刻开始运行，也就是继续`lockSlow`这里`goroutine`被唤醒以后的逻辑。 

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230103203721215.png" alt="image-20230103203721215" style="zoom: 50%;" />

### 2.1.3 信号量

上节主要讲解 sync.Mutex 的第一个字段 `state`，本节主要讲解 sync.Mutex 的第二个字段 `sema`。

> #### **问题：**协程等待一个锁时，要如何休眠、等待和唤醒呢？

**答：**这要靠`runtime.semaphore`来实现，这是可供协程使用的信号量。`runtime`内部会通过一个大小为`251`的`sematable`来管理所有`semaphore`，怎么通过这个大小固定的`table`来管理执行阶段数量不定的`semaphore`呢？**大致思路如下：**

这个`sematable`存储的是`251`课平衡树的根，平衡树中每个节点都是一个`sudog`类型的对象，要使用一个信号量时，需要提供一个记录信号量数值的变量，根据它的地址进行计算，映射到`sematable`中的一颗平衡树上，找到对应的节点就找到了该信号量的等待队列。

例如，我们常用的`sync.Mutex`中，有一个`sema`字段，用于记录信号量的数值，如果有协程想要等待这个`Mutex`，就会根据`sema`字段的地址计算映射到`sematable`中的某棵平衡树上，找到对应的节点，也就找到了这个`Mutex`的等待队列了。所以，`syne.Mutex`是通过信号量来实现排队的。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230104201053349.png" alt="image-20230104201053349" style="zoom:33%;" />

而`channel`需要有**读(发送)等待队列**以及**写(接收)等待队列**，还要支持缓冲区功能，所以并没有直接使用信号量来实现排队，而是自己实现了一套排队逻辑。

不过，无论是信号量还是`channel`，底层实现都离不开`runtime.mutex`，因为它们都需要保障在面临多线程并发时，不会出现同步问题。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230104201320363.png" alt="image-20230104201320363" style="zoom: 33%;" />

### 2.1.4 抢占式调度

#### 2.1.4.1 Go 1.13

Go 1.13 的抢占方式是依赖于**栈增长检测代码**的，并不算严格意义上的抢占式调度。

首先看一段代码：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230105151904406.png" alt="image-20230105151904406" style="zoom:25%;" />

按理来说，这段代码运行后应该会不断输出递增的数字，但是在 Go 1.14 之前，运行这段代码会发生阻塞。

运行环境是双核CPU，排查之后发现是在**执行`STW`时发生的阻塞**。

`GC` 开始前需要`STW`来进行开启写屏障等准备工作，所以`STW`就是要抢占所有的`P`，让`GC`得以正常工作。而示例程序中的`main goroutine`没能被抢占，它一直在执行，而`STW`一直在等待它让出，这样就陷入了僵局。

> #### **问题：**为什么会陷入这样的局面？

**答：**首先，梳理下`STW`的主要逻辑。`GC`需要抢占所有的`P`，但这不是说抢占就ok的，所以它会**记录下自己要等待多少个`P`让出**，当这个值减为`0`，目的就达到了。对于当前`P`、以及陷入系统调用的`P`(`_Psyscall`)、还有空闲状态的`P`，直接将其设置为`_Pgcstop`即可。对于还有`G`在运行的`P`，则会将对应的`g.stackguard0`设置为一个特殊标识(`runtime.stackPreempt`)，告诉它`GC`正在等待它让出。此外，还会设置一个`gcwaiting`标识(`sched.gcwaiting=1`)，接下来就通过这两个标识符的配合，来实现运行中的`P`的抢占。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230105155228882.png" alt="image-20230105155228882" style="zoom:33%;" />

**这是怎么实现的呢？**

`goroutine`创建之初，栈的大小时固定的，为了防止出现栈溢出的情况，编译器会在有明显栈消耗的函数头部插入一些检测代码。通过`g.stackguard0`来判断是否需要进行栈增长，但如果`g.stackguard0`被设置为特殊标识`runtime.stackPreempt`，便不会执行栈增长，而是去执行一次调度(`schedule()`)，在`schedule()`调度执行时，会检测`gcwaiting`标识，若发现`GC`在等待执行，便会让出当前`P`，将其置为`_Pgcstop`状态。

这样看来，示例`main goroutine`之所以没能让出，**是因为空的`for`循环并没有调用函数**，也就没有机会执行栈增长检测代码，所以它并不知道`GC`在等待它让出。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230105160033853.png" alt="image-20230105160033853" style="zoom:33%;" />

#### 2.1.4.2 Go 1.14 之后

**依赖栈增长检测代码的抢占方式，遇到没有函数调用的情况就会出现问题。**

在 Go 1.14 及之后，这一问题得到了解决。在`Linux`系统上，这种真正的抢占式调度是**基于信号**来实现的，所以也称为**异步抢占**。

函数`preemptone`用来抢占一个`P`，定位到该函数，对比 1.13 和 1.14 的实现有何不同。

##### 信号发送

1.13 中，`preemptone`函数主要负责设置`g.preempt=true`，并将`g.stackguard0`设置为特殊标识(`stackPreempt`)。

而在 1.14 中，增加了最后这个`if`语句块：

- 第一个判断用于确认当前硬件环境是否支持这种异步抢占，这个常量值(`preemptMSupported`)是在编译期间就确定的；
- 第二个判断(`debug.asyncpreemptoff`)用于检测用户是否允许开启异步抢占，默认情况下是允许的，但是用户可以通过`GODEBUG`环境变量来禁用异步抢占。
- 如果这两条验证都通过了，就将`p.preempt`字段置为`true`，实际的抢占操作会交由`preemptM`函数来完成。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230105170716733.png" alt="image-20230105170716733" style="zoom:33%;" />

定位到`preemptM`函数，它的主要逻辑是通过`runtime.signalM`函数，向指定`M`发送`sigPreempt`信号，**怎么发送的呢？**

`signalM`函数会通过调用操作系统中信号相关的系统调用，将指定信号发送给目标线程。信号发出去了，异步抢占的**前一半工作**就算是完成了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230105171254392.png" alt="image-20230105171254392" style="zoom: 33%;" />



##### 信号处理

后一半工作就要由接收到信号的工作线程来完成了。

线程接收到信号后，会调用对应的信号`handler`来处理，Go 语言中的信号交由`runtime.sighandler`来处理，`sighandler`在确定信号为`sigPreempt`以后，会调用`doSigPreempt`函数。**它会首先判断`runtime`是否要对指定的`G`进行异步抢占**，通过什么来判断呢？

首先，指定的`G`与其对应`P`的`preempt`字段都要为`true`，而且指定的`G`还要处在`_Grunning`状态，还要**确认在当前位置打断`G`并执行抢占是安全的**，那么怎么确保安全性呢？

- 指定的`G`可以挂起并安全的扫描它的栈和寄存器，并且当前被打断的位置并没有打断写屏障；
- 指定的`G`还有足够的栈空间来注入一个异步抢占函数调用(`asyncPreempt`)；
- 这里可以安全的和`runtime`进行交互，主要就是确定当前并没有持有`runtime`相关的锁，继而不会在后续尝试获得锁时发生死锁。

**确认了要抢占这个`G`，并且此时抢占是安全的**以后，就可以放心的通过`pushCall`向`G`的执行上下文中注入异步抢占函数调用了，被注入的异步抢占函数(`asyncPreempt`)是一个汇编函数，它会先把各个寄存器的值保存在栈上，也就是先保存现场到栈上，然后调用`runtime.asyncPreempt2`函数，这个函数最终会去执行`schedule()`，到这里，异步抢占就完成了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230105172607879.png" alt="image-20230105172607879" style="zoom: 33%;" />

再去执行上边的示例程序，发现可以正常运行。也就是说，即使空的`for循环`没有被插入栈增长检测代码，在 1.14 中，通过注入异步回调函数的方式，同样能实现抢占式调度。

### 2.1.5 Channel

#### 2.1.5.1 数据结构

```go
type hchan struct {
	qcount   uint
	dataqsiz uint
	buf      unsafe.Pointer
	elemsize uint16
	closed   uint32
	elemtype *_type
	sendx    uint
	recvx    uint
	recvq    waitq
	sendq    waitq

	lock mutex
}
```

> #### 问：`channel`底层数据结构是怎么设计的？

通过`make`创建一个缓冲区大小为`5`，元素类型为`int`的`channel`，`ch`是存在于函数栈帧上的一个指针，指向堆上的`hchan`数据结构。

- 因为`channel`支持协程间并发访问，所以要有一把锁`lock`来保护整个数据结构。
- 对于有缓冲来讲，需要知道缓冲区在哪，已经存储了多少个元素，最多存储多少个元素，每个元素占多大空间，所以实际上，缓冲区就是个数组`buf`、`elemsize`。
- 因为 Go 运行中，内存复制、垃圾回收等机制依赖数据的类型信息，所以`hchan`还要有一个指针，指向元素类型的类型元数据`elemtype`。
- 此外，`channel`支持交替的读(接收)写(发送)，需要分别记录读、写下标的位置`recvx`、`sendx`。
- 当读和写不能立即完成时，需要能够让当前协程在`channel`上等待，待到条件满足时，要能够立即唤醒等待的协程，所以要有两个等待队列，分别针对读(接收)和写(发送)`recvq`、`sendq`。
- 此外，`channel`能够`close`，所以还要记录它的关闭状态`closed`。

综上所述，`channel`底层就长这个样子：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230109162653598.png" alt="image-20230109162653598" style="zoom:50%;" />

初始状态下，`ch`的缓冲区为空，读、写下标都指向下标`0`的位置，等待队列也都为空。然后，一个协程`g1`向`ch`发送数据[1,5]，此时缓冲区已满，若还要继续发送数字6，`g1`就会进到`ch`的发送等待队列中。这是一个`sudog`类型的链表，里面会记录哪个协程在等待，等待哪个`channel`，等待发送的数据在哪儿等信息。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230109165507150.png" alt="image-20230109165507150" style="zoom:33%;" />

接下来协程`g2`从`ch`接收一个元素，`recvx`指向下一个位置，第0个位置就空出来了，所以会唤醒`sendq`中的`g1`，将这里的数据发送给`ch`，然后缓冲区再次满了，`sendq`队列为空。

这个过程中，`sendx`和`recvx`都会从`0`到`4`再到`0`，循环移动，所以`channel`的缓冲区也被称为**环形缓冲区**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230109170332532.png" alt="image-20230109170332532" style="zoom:33%;" />

#### 2.1.5.2 发送数据

##### 阻塞式发送

如果使用以下方式给`channel`发送数据：

```go
ch <- 10
```

不阻塞的情况：

- 缓冲区还有空闲位置；
- 有协程在等着接收数据；

阻塞的情况：

- `ch`为`nil`；
- `ch`没有缓冲区，而且也没有协程等着接收数据；
- `ch`有缓冲区，但缓冲区已用尽。

##### 非阻塞式发送

```go
select {
case ch <- 10:
    ...
default:
    ...
}
```

若采用这种写法，如果检测到`ch`可以发送数据，就会执行`case`分支；如果会发生阻塞就执行`default`分支。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230109210755580.png" alt="image-20230109210755580" style="zoom:33%;" />

#### 2.1.5.3 接收数据

##### 阻塞式接收

以下是接收数据的三种写法，都允许发生阻塞：

```go
<-ch			// 1. 丢弃结果
v := <-ch		// 2. 将结果赋值给v
v, ok := <-ch	// 3. comma ok风格的写法，ok为false表示ch已关闭，此时v是channel元素类型的零值
```

不阻塞的情况：

- 在缓冲区中有数据；
- 有协程等着发送数据；

阻塞的情况：

- `ch`为`nil`；
- `ch`无缓冲而且没有协程等着发送数据；
- `ch`有缓冲但缓冲区无数据；

##### 非阻塞式接收

```go
select {
case <-ch:
    ...
default:
    ...
}
```

若采用非阻塞式接收方式，如果检测到`ch`的`recv`操作不会阻塞时，就会执行`case`分支；如果会阻塞就会执行`default`分支。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230109210714175.png" alt="image-20230109210714175" style="zoom:50%;" />



---

#### 2.1.5.4 多路select

上面的`select`只是针对单个`channel`的操作。**多路`select`**是指存在**两个或更多**的`case`分支，每个分支可以是一个`channel`的`send`或`recv`操作。

例如一个协程通过多路`select`等待`ch1`和`ch2`，这里`default`分支是可选的，暂且把这个协程记为`g1`。

多路`select`会被编译器转换为对`runtime.selectgo`函数调用，首先看**参数**：

- 第一个参数`cas0`指向一个数组，数组里装的是`select`中所有的`case`分支，顺序是`send`在前`recv`在后；
- 第二个参数`order0`指向一个`uint16`类型的数组，数组大小等于`case`分支的2倍，实际上被用作两个数组：第一个数组用来对所有`channel`的轮询进行乱序，第二个数组用来对所有的`channel`的加锁操作进行排序。**轮询需要乱序才能保障公平性，而按照固定算法确定加锁顺序才能避免死锁**；
- 第三个参数`pc0`和`race`检测相关；
- `nsends`和`nrecvs`分别表示所有`case`中，执行`send`和`recv`操作的分支分别有多少个。
- `block`表示多路`select`是否要阻塞等待。对应到代码中，就是有`default`分支的不会被阻塞，没有的会阻塞。

再看**返回值**：

- 第一个返回值`int`类型，代表最终哪个`case`分支被执行了，对应到`cas0`指向的数组下标。但是如果进入到`default`分支，就会对应`-1`。
- 第二个返回值`bool`类型，用于在执行`recv`操作的`case`分支时，表明是实际接收到了一个值，还是因`channel`关闭而得到了零值。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230109213648426.png" alt="image-20230109213648426" style="zoom:33%;" />

- 多路`select`需要进行**轮询**来确定哪个`case`分支可操作了，但是轮询前需要先加锁，所以`selectgo`函数执行时，会先按照有序的**加锁**顺序，对所有的`channel`加锁；

- 然后按照**乱序的轮询**顺序检查所有`channel`的等待队列和缓冲区；

- 假如**检查**到`ch1`时，发现有数据可读，那就**直接拷贝数据，进入对应分支**；
- 假如所有的`channel`都不可操作，就**把当前协程添加到所有`channel`的`sendq`或`recvq`中**，对应这个例子，`g1`会被添加到`ch1`的`recvq`及`ch2`的`sendq`中，之后`g1`会被**挂起**，并**解锁**所有的`channel`；
- 假如接下来`ch1`有数据可读了，`g1`就会被**唤醒**，完成对应的分支操作后，会再次按照加锁顺序对所有`channel`**加锁**，然后从所有的`sendq`或`recvq`中将自己**移除**；
- 最后全部**解锁**后返回。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230109214411461.png" alt="image-20230109214411461" style="zoom:33%;" />

---

虽然，`channel`的读写操作写法众多，但事实上，`channel`**阻塞式的`send`操作**会被编译器转换为**对`runtime.chansend1()`的调用**，而它**内部只是调用了`runtime.chansend()`**；**非阻塞式的`send`操作**会被编译器转换为**对`runtime.selectnbsend()`的调用**，它也仅仅是**调用了`runtime.chansend()`**。所以，**`send`操作主要是通过`runtime.chansend()`函数实现的。**

同样的，**`channel`阻塞式的`recv`操作**会被编译器转换为**对`runtime.chanrecv1()`的调用**，而它**内部只是调用了`runtime.chanrecv()`**；**`comma ok`风格的写法**会被编译器转换为**对`runtime.chanrecv2()`的调用**，它的**内部也是调用`runtime.chanrecv()`**，**只不过比`chanrecv1()`多了一个返回值**；**非阻塞式的`recv`操作**会根据是否为`comma ok`风格，被编译器转换为**对`runtime.selectnbrecv()`或`selectnbrecv2()`的调用**，而它们两个也仅仅是**调用了`runtime.chanrecv()`**，所以，**`recv`操作主要是通过`runtime.chanrecv()`函数实现的。**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230109215407705.png" alt="image-20230109215407705" style="zoom:50%;" />

### 2.1.6 协程 

## 2.2 内存管理

### 2.2.1 内存分配器

程序中的数据和变量都会被分配到程序所在的虚拟内存中，内存空间包含两个重要区域：**栈区（Stack）和堆区（Heap）**。函数调用的参数、返回值以及局部变量大都会被分配到栈上，这部分内存会由编译器进行管理；不同编程语言使用不同的方法管理堆区的内存，C++ 等编程语言会由工程师主动申请和释放内存，Go 以及 Java 等编程语言会由工程师和编译器共同管理，堆中的对象由内存分配器分配并由垃圾收集器回收。

从进程虚拟空间地址来看：

- 程序要执行的指令在**代码段**；
- 全局变量、静态数据等都会分配在**数据段**；
- 函数的局部变量、参数和返回值都会分配在**函数栈帧**；

#### 2.2.1.1 设计原理

内存管理一般包含三个不同的组件，分别是**用户程序（Mutator）、分配器（Allocator）和收集器（Collector）**，当用户程序申请内存时，它会通过**内存分配器申请新内存**，而分配器会负责从堆中初始化相应的内存区域。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223151605383.png" alt="image-20221223151605383" style="zoom: 50%;" />

### 2.2.2 GC

从进程虚拟空间地址来看：

- 程序要执行的指令在**代码段**；
- 已初始化的全局变量、静态常量等都会分配在**==数据段==**；
- 未初始化的全局变量、静态常量等都会分配在**==BSS段==**；
- 函数的局部变量、参数和返回值都会分配在**==函数栈帧==**；

**==Point：==**

由于函数调用栈会在函数返回后销毁，因此，如果**不能在编译阶段确定数据对象的大小**，或者对象**生命周期会超过当前所在函数**，那么就**不适合分配在栈上**，而应该**分配到堆上**。

> #### **问题：为什么需要垃圾回收？**

**主要是为了释放==堆内存==**。随着程序运行，有些数据不会再被用到了，直接分配在**栈上的数据**，**会随着函数调用栈的销毁释放自身占用的内存**；但是分配在**堆上的数据**，它们占用的内存需要程序**主动释放**才可以重新使用，否则就会成为**垃圾**，而越积越多的垃圾会不断的消耗系统内存。

---

垃圾回收有几种常见方式：

- **手动垃圾回收：C、C++、Rust。**一旦释放早了，后续对该数据的访问便会出错，这就是所谓**"悬挂指针"**问题；而如果忘了释放，它又会一直占用内存，出现**"内存泄露"**；
- **自动垃圾回收：Python、Ruby、Java、Go**。由运行时识别不再有用的数据并释放他们所占的内存，内存何时被释放，被释放的内存如何处理，都不需要我们关心。
  - 跟踪式垃圾回收
  - 引用计数式垃圾回收

#### 2.2.2.1 设计原理

> #### **问题：自动垃圾回收怎么区分哪些数据是垃圾呢？**

可以确定，程序中用得到的数据，一定是从==**栈、数据段**==这些根节点追踪得到的数据。也就是说，**从这些根节点追踪不到的数据**一定是没用的数据，一定是**垃圾**。因此，目前主流的自动垃圾回收算法都是使用**"可达性"**近似等价**"存活性"**的。

##### 标记-清扫算法

标记清除（Mark-Sweep）算法是最常见的垃圾收集算法，标记清除收集器是**跟踪式垃圾收集器**，其执行过程可以分成**标记（Mark）**和**清除（Sweep）**两个阶段：

1. **标记阶段** — 从根对象出发查找并标记堆中所有存活的对象；
2. **清除阶段** — 遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222170145205.png" alt="image-20221222170145205" style="zoom:33%;" />

###### 三色抽象

**三色抽象**可以清晰的展现追踪式回收中**对象状态**的**变化过程**。

- 垃圾回收**开始时**，所有数据均为**白色**；
- 然后把直接追踪到的root节点都标记为**灰色**：灰色代表基于当前节点展开的追踪还未完成；
- 当基于某个节点的追踪任务完成后，便会把该节点标记为**黑色**：表示它是存活数据，而且无需基于它再次进行追踪了。
- 基于**黑色**节点找到的所有节点都被标记为**灰色**，表示还要基于它们进一步开始追踪。
- 当没有**灰色**节点时，就意味着标记工作可以结束了。此时，有用数据都为**黑色**，垃圾都为**白色**。接下来回收这些**白色**数据即可。

**标记-清扫算法的缺点：**

标记-清扫算法容易造成很多小内存，这一问题可以：

- 基于**BiBOP(Big Bag of Pages)**的思想，**把内存块划分为多种大小规格**，对相同规格的内存块进行统一管理。

- 还可以通过**紧凑**的方法减少碎片化内存，但是会带来多次移动的开销。

##### 复制回收算法

还有一种**复制式回收**算法，他会把堆划分为两个相等的空间From和To，程序执行时使用From空间，垃圾回收时会扫描From空间，把能追踪到的数据复制到To空间，当所有能追踪到的数据都复制到To空间后，把From和To空间角色对换，原来的To变为From，原来的From可以全部回收用作新的To。这种复制式不会产生碎片化问题，但是**会浪费一半的堆内存**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222171821098.png" alt="image-20221222171821098" style="zoom:33%;" />

为了提高堆内存利用率，通常会和其他垃圾回收算法搭配使用，**只在一部分堆内存中使用复制式回收**。比如**分代回收**。

###### 分代回收

**分代回收**主要基于**弱分代假说**：大部分对象都会在年轻时死亡。

- 新生代对象：新创建的对象；
- 老年代对象：经受住特定次数GC而依然存活的对象；

基于弱分代假说，大部分对象会在最初经历的GC中死亡，也就是说**新生代对象成为垃圾的概率高于老年代对象**。因此，可以将数据划分为新生代和老年代，**降低老年代执行垃圾回收的频率**，不用每次都处理所有数据，将明显提高垃圾回收执行的效率。而且，新生代和老年代还可以分别采用不同的回收策略，进一步提升回收效益并减少开销。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222173347821.png" alt="image-20221222173347821" style="zoom: 33%;" />

---

以上均为**跟踪式垃圾回收**，而引用计数式垃圾回收有很大的不同。

##### 引用计数式

**引用计数指的是一个数据对象被引用的次数。**程序执行过程中，会更新数据对象的引用计数，当对象的引用计数为0时，就表示这个对象不再被使用，可以回收它所占用的内存。因此，在引用计数法中垃圾识别的任务已经被分摊到每次对数据对象的操作中。虽然引用计数法可以及时回收无用内存，但是高频率的更新引用计数也会造成不小的开销，而且如果发生循环引用情况，那么将无法回收。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222173942715.png" alt="image-20221222173942715" style="zoom: 25%;" />

---

以上讨论均是在暂停用户程序，只专注于垃圾回收的前提下进行的，也即所谓**==STW==(Stop The World)**。但实际上用户程序无法接受长时间的暂停。

##### 增量式垃圾回收

**增量式垃圾回收：**将长时间的垃圾回收，分成若干小段，和用户程序交替执行，即缩短每次暂停的时间，但增加暂停的次数。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222174536321.png" alt="image-20221222174536321" style="zoom:33%;" />

**这会带来新的问题：**保不齐垃圾回收程序前脚刚标记一个黑色对象，用户程序后脚就修改了它(将它指向白色对象，此时原黑色对象应变为灰色，但并没有)，垃圾回收程序有可能误判将白色对象(应为灰色对象)回收。原因如下：

在三色抽象中，黑色对象处理完毕，不会被再次扫描，而灰色对象还会被回收器继续处理，所以**若出现黑色对象到白色对象的引用**，**同时没有任何灰色对象可以抵达这个白色对象**，它就会被判为**垃圾**，但实际上它仍是存活数据。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222180626713.png" alt="image-20221222180626713" style="zoom:33%;" />

##### 强三色不变式与弱三色不变式

如果能够做到**不出现黑色对象到白色对象的引用**，就必然不会出现这样的错误了。这被称为**"强三色不变式"**。

若把条件放宽一点，**允许出现黑色对象到白色对象的引用**，但是可以**保证通过灰色对象可以抵达该白色对象**，这样也可以避免这个错误，这被称为**"弱三色不变式"**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222180954270.png" alt="image-20221222180954270" style="zoom:33%;" />

##### 读写屏障

实现**"强/弱三色不变式"**通常的做法是建立**"读/写屏障"**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222212313757.png" alt="image-20221222212313757" style="zoom:33%;" />

- **写屏障：**会在写操作中插入指令，目的是把数据对象的修改通知到垃圾回收器，所以写屏障通常要有一个记录集，而记录集是采用顺序存储还是哈希表、记录精确到被修改的对象还是只记录其所在页等问题，就是写屏障具体实现要考虑的了。
- **"强三色不变式"**提醒我们关注白色指针指向黑色对象的写入操作，无论如何都**不允许出现黑色对象到白色对象的引用**。可以把白色指针变为灰色，也可以把黑色对象变为灰色。这些都属于**=="插入"写屏障==**。
  
- **"弱三色不变式"**则提醒我们关注对那些**到白色对象路径**的破坏行为。例如要删除灰色对象到白色对象的引用时，可以把白色对象变为灰色。这种写屏障属于**=="删除"写屏障==**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222211852448.png" alt="image-20221222211852448" style="zoom: 33%;" />

- **读屏障：**确保用户程序不会访问到已经存在副本的陈旧对象。（主要针对复制式回收）

##### 多核

- **并行垃圾回收：**多线程并行执行垃圾回收程序。
- **并发垃圾回收：用户程序与垃圾回收程序并发执行**，这有可能导致错误(有些线程开启了写屏障，有些还没开启，所以通常采用下边那种)

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222212720071.png" alt="image-20221222212720071" style="zoom:33%;" />

- 主体并发式垃圾回收：在某些阶段采取STW(确保大伙儿都开启了写屏障)，在其他阶段支持并发。
- 主体并发增量式垃圾回收：在"主体并发式垃圾回收"基础上支持增量式回收，

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222213057154.png" alt="image-20221222213057154" style="zoom:33%;" />

#### 2.2.2.2 Go

`Go`语言的垃圾回收采用**标记-清扫算法**，支持**==主体并发增量式回收==**，使用**插入与删除**两种写屏障结合的**混合写屏障**。

- `Go`语言的`GC`在准备阶段(`Mark Setup`)会为每个`p`创建一个`mark worker`协程，把对应的`g`指针存储到`p`中，这些后台`mark worker`创建后很快进入休眠，等到**标记阶段**得到调度执行。

- 接下来**第一次`STW`**，`GC`进入`_GCMark`阶段。全局变量`gcphase`记录`GC`阶段标识，全局变量`writeBarrier`记录是否开启写屏障，全局变量`gcBlackenEnabled`用于标识是否允许进行`GC`标记工作(此处置为`1`，标识允许)。

- 在`STW`的情况下**开启写屏障**。等所有准备工作做好以后，**`start the world`**，所有`p`都会知道写屏障已开启，然后这些后台`mark worker`可以得到调度执行，展开标记工作。

- 当没有标记任务时，**第二次`STW`**，`GC`进入`_GCMarkTermination`阶段，确认标记工作确实已经完成，然后停止标记工作，将`gcBlackenEnabled`置为`0`。

- 接下来，进入`_GCOff`阶段，关闭写屏障。**`start the world`，进入清扫阶段。**进入`_GCOff`阶段前，新分配的对象会直接标记为黑色，进入`_GCOff`阶段后，再新分配的对象就是白色的了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229151143665.png" alt="image-20221229151143665" style="zoom:33%;" />

执行清扫工作的协程由`runtime.main`在`gcenable`中创建，对应`g`指针存储在全局变量`sweep`中，到清扫阶段，这个后台的**`sweeper`**会被加入到`runq`中，它得到调度执行时会**执行清扫任务**，因为清扫工作也是增量进行的，所以每一轮`GC`开始前，还要先确保完成上一轮`GC`未完成的清扫工作。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229152005509.png" alt="image-20221229152005509" style="zoom: 33%;" />

这样看来似乎只需要两轮`STW`，**标记**与**清扫**工作并发增量执行的`GC`而已。

> #### **问题：**

1. **关于`GC`标记工作：**依照标记-清扫算法，**标记工作要从扫描`bss`段、数据段、以及协程栈上的这些`root`结点开始**，**追踪到堆上的节点**，那怎么确定这些数据对象是否是`GC`感兴趣的指针呢？

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229152754952.png" alt="image-20221229152754952" style="zoom:25%;" />



Go语言在编译阶段会生成`bss`段、数据段等对应的**元数据**，存储在可执行文件中，通过各模块对应的`moduledata`可以获得`gcdatamask`、`gcbssmask`等信息，它们会被**用于判断特定`root`节点是否为指针**。

协程栈也有对应的元数据，存储在`stackmap`中，扫描协程栈时，通过对应元数据，可以知道栈上的局部变量、参数、返回值等对象中那些是存活的指针。**确定了`root`节点是否是指针，还要进一步判断这些指针是否指向堆内存**，如果**指向堆内存**就得把他们加入到`GC`工作队列中进行进一步扫描。

堆上这些数据对象自然也有对应的**元数据**。`mheap`中每个`arena`对应一个`HeapArena`，记录`arena`的元数据信息。**其中有一个`bitmap`，**`bitmap`中一个`byte`可以标记`arena`中连续四个指针大小的内存，每个`word`对应的两个`bit`中，低位`bit`用于标记是否为指针(0为非指针，1为指针)，高位`bit`用于标记是否要继续扫描(1代表：扫描完当前`word`，并不能完成当前数据对象的扫描)，`bitmap`的信息在分配内存时设置，会用到对应类型元数据中的`gcdata`信息。`HeapArena`中还有一个`spans`字段，它是一个`*mspan`类型的数组，用于记录当前`arena`中每一页对应到哪个`span`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229160021183.png" alt="image-20221229160021183" style="zoom:33%;" />

基于`HeapArena`记录的**元数据信息**，我们只要知道一个对象的地址，就可以根据`bitmap`信息，扫描它内部是否含有指针；也可以根据对象地址，计算出它在哪一页，然后通过`spans`信息查到该对象存在哪一个`span`中，而每个`span`都有两个位图标记，记录在`mspan`中：`allocBits`中每一位用于标记一个对象存储单元是否已分配。`gcmarkBits`中每一位用于标记一个对象是否存活。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229155909739.png" alt="image-20221229155909739" style="zoom: 33%;" />

有了这些元数据和位图标记，就能知道哪些数据对象应该被标记为灰色，也就是把其对应的`gcmarkBits`标记为`1`，并加入到**工作队列**中。

2. **关于工作队列：全局变量`work`**中存储着全局工作队列缓存，同时，每个`p`都有一个**本地工作队列**(`p.gcw`)。本地工作队列中有两个`workbuf`，添加任务时总是往`wbuf1`添加，`wbuf1`满了就交换两个`workbuf`，交换后如果依然是满的，就把当前`wbuf1`的工作`flush`到全局缓存中去。后台`mark worker`执行标记工作**消耗工作队列**时，会处理本地工作队列和全局缓存中工作量均衡的问题，如果全局工作缓存为空，就把当前`p`的工作分一些到全局工作缓存中，**具体做法是**：如果`wbuf2`不为空，就把它整个`flush`到全局缓存中；如果为空，而`wbuf1`中元素个数大于`4`，就把`wbuf1`中一半的工作放到全局缓存中。而如果获取标记任务时发现，本地工作队列为空，也会从全局工作缓存中获取任务放到本地队列中。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229163955508.png" alt="image-20221229163955508" style="zoom:33%;" />

至于写屏障，每个`p`都有一个写屏障缓冲区，所以写屏障触发时，并不会直接操作工作队列，而是**把相关指针写入当前`p`的写屏障缓冲区**中，**当写屏障缓冲区已满或`mark worker`通过工作队列获取不到任务时**，会把写屏障缓冲内容`flush`到工作缓存中。通过区分本地工作队列与全局工作缓存，并**为每个`p`设置写屏障缓冲区，缓解了执行并发标记工作时操作工作队列的竞争问题。**

虽然`GC`初始化时为每个`p`都创建了一个`mark worker`，但是**调度时可以启动多少个也是个问题**。

3. **关于`GC`对`CPU`的使用率：**`GC`默认的`CPU`目标使用率为`25%`，在`GC`执行的初始化阶段，会根据当前`gomaxprocs`数值乘以`CPU`目标使用率，来计算需要启动的`mark worker`数量，为了应对计算结果不为整数的情况，会对该结果进行加`0.5`的`rounding`，但是这样的`rounding`会和原目标产生误差，误差超过`0.3`就比较显著了。例如`gomaxprocs=6`的情况，此时，如果调度时允许启用两个`mark worker`就有些多了。

   <img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229172058110.png" alt="image-20221229172058110" style="zoom:33%;" />

   **为此**，`GC`在`mark worker`中引入了不同的工作模式，记录在`p`中，作为对应`p`的`mark worker`的工作模式：
   
   - `Dedicated`模式的`worker`会执行标记任务直到被抢占；
   - `Fractional`模式的`worker`除了被抢占外，还可以在达到`fractional`部分的目标时主动让出。例如，如果`gomaxprocs=4`，就只需要启动一个`Dedicated`模式的`worker`即可；如果`gomaxprocs=6`，除了一个`Dedicated`模式的`worker`外，还允许再启用一个`Fractional`模式的`worker`，`Fractional`部分的目标为`0.5`，这会由所有`p`共同负责，所以这里对每个`p`而言，`fractionalUtilizationGoal=1/12`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229172521378.png" alt="image-20221229172521378" style="zoom:33%;" />

**全局变量`gcController`**中会记录可以启动多少个`Dedicated`模式的`worker`，还会记录`fractionalUtilizationGoal`，调度器执行`findRunnableGcWorker`，**想要恢复后台`mark worker`时**，需要设置`worker`运行的模式：

- 如果`Dedicated`模式的`worker`数目还没有达到上限，就设置为`Dedicated`模式；
- 如果`Dedicated`模式的`worker`数量达到上限后，就要看是否需要`Fractional`模式的`worker`辅助工作，需要的话就设置为`Fractional`模式，`p`会记录自己累计执行`Fractional worker`的时间与当前`worker`开始工作的时间，而`gcController`会记录本轮`GC`标记工作开始的时间，当前`p`执行`Fractional`模式的标记`worker`时，每完成一定量的工作，就会检查当前`p`累计在`Fractional`模式下的工作时间与本轮`GC`标记工作已执行的时间的比率，是否达到了`fractionalUtilizationGoal`：如果达到了，当前`worker`就可以让出了。

通过这样的方式，可以有效的控制`GC`的`CPU`使用率。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229173321726.png" alt="image-20221229173321726" style="zoom:33%;" />

4. **关于`GC`执行过程中的内存分配压力：**为了避免内存分配压力过大，Go语言实现了**`GC Assist`机制(辅助GC)**。

如果协程要分配内存，而**`GC`标记**工作尚未完成，它就要负责一部分标记工作，**要申请的内存越大，对应要负担的标记任务就越多(负债越多)**，这是一种借贷偿还机制：

- 当前`g`中`gcAssistBytes`若小于0，则当前`g`面临负债；
- 大于0，则表示有结余。

**有负债的`g`**在申请内存前，需要辅助`GC`完成一些标记工作来偿还债务，不过，后台`mark worker`每完成一定量的标记任务，就会在全局`gcController`存一笔信用，有债务需要偿还的`g`可以从`gcController`这里`steal`尽量多的信用，来抵消自己所欠的债务，不管是真正执行标记扫描任务，还是从`gcController`这里**`steal`信用**，如果这一次偿还了当前债务后还有结余，就可以用于抵消下次内存分配的债务了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229174239972.png" alt="image-20221229174239972" style="zoom:33%;" />

`GC`标记阶段每次内存分配，都会检查是否需要**`辅助标记`**；而到了`GC`清扫阶段，内存分配就可能会触发**`辅助清扫`**。

例如，直接从`mheap`分配大对象时，为了维持堆内存分配量与清扫页面数量的线性关系，可能需要执行一定量的清扫工作。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229174515178.png" alt="image-20221229174515178" style="zoom:33%;" />

再例如，从本地缓存中直接分配一个`span`时，若遇到尚未清扫的可用`span`，也需要先清扫这个`span`再分配使用。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229174635779.png" alt="image-20221229174635779" style="zoom:33%;" />

**`辅助标记`和`辅助清扫`可以避免出现并发垃圾回收中，因过大的内存分配压力导致`GC`来不及回收的情况。**

5. **关于混合写屏障：**

首先看下Go语言写屏障的伪代码，`slot`的原指针是`old`，如果要把`ptr`写入`slot`，那么对原指针可达路径的删除会触发**删除写屏障**，新指针到`slot`可达路径的增加会触发**插入写屏障**，Go语言整合了**插入与删除写屏障**，称之为**混合写屏障**。

> #### **问题：为什么要加入删除写屏障？**

**在引入混合写屏障前，只有插入写屏障**，但是这需要对所有堆、栈的写操作都开启写屏障，代价太大。为了改善这个问题，改为忽略协程栈上的写屏障，只在标记结束阶段，重新扫描那些被激活的栈帧，但是Go语言通常会有大量活跃的协程，这就导致第二次`STW`时，重新扫描协程栈的时间过长。**如果在忽略写屏障的前提下，能够保障写入栈上的数据对象不会被`hiding`**，就不用在第二次`STW`时重新扫描这些栈帧了。**而删除写屏障恰好可以解决这点**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229175748570.png" alt="image-20221229175748570" style="zoom:33%;" />

例如，当前`g`的栈帧中`A`已经完成扫描，然后`g`执行，把`old`写入栈上的本地变量`A`，栈上并没有插入写屏障，`old`并不会被标记，之后把新指针`ptr`写入`slot`，如果没有引入删除写屏障，此时抵达`old`的唯一路径被打断，`old`就不能被`GC`发现，所以需要在删除`old`的可达路径时，**通过删除写屏障把它标记为灰色**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229180135141.png" alt="image-20221229180135141" style="zoom: 33%;" />

如果`slot`已经标记为黑色，栈上的`C`还未被扫描，接下来`g`执行，把`ptr`写入`slot`，而当前`g`切断`C`到达`ptr`的可达路径时，并没有删除写屏障，不会标记`ptr`，为了避免将白色对象写入堆上的黑色对象，就要靠**插入写屏障，在写入`slot`时标记新指针**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221229180424615.png" alt="image-20221229180424615" style="zoom:33%;" />

至于判断当前栈是否为灰色，是因为如果当前已经是完成扫描的黑色栈，那么它指向的对象一定已经被标记了，插入写屏障就没必要再标记一次了。所以，**使用混合写屏障的意义在于：**既可以忽略当前栈帧的写屏障，也不用在第二次`STW`时，重新扫描所有活跃`g`的栈帧。只有**栈是灰色**的时候，才需要**插入写屏障**。

**==换句话说==，假如没有删除写屏障，每次操作栈上的指针都要经过插入写屏障，这将极大限制性能。**但实际上可以分析一下，当为栈上的指针赋值时，一共就四种来源：全新分配、栈上的指针、堆上的指针、全局数据段的指针。

- 全新分配的对象在标记阶段会自动被标记为**黑色**，因此不需要额外处理；
- 栈上指针间的赋值也不必用写屏障，因为不会隐藏其他指针；
- 堆和全局数据段，如果仅仅是从它们那里把指针复制到栈上，那也不会有啥问题，**问题出在**，若复制到栈上之后，堆或全局数据段中的旧指针被删掉了，且它指向的对象没有其他指针也指向它，那它指向的对象就会被"隐藏"了。**所以，问题只出在删除操作上。**

因此，**当堆或全局数据段删除对象时，需要应用删除写屏障**，灰化删除指针指向的对象。

==**这样，就不需要对栈使用写屏障了。**==

6. **关于`GC`触发方式：**

- **手动触发**。入口在`runtime.GC()`函数中。
- **分配内存**。分配内存时，有些情况需要检查是否需要触发`GC`，每次`GC`都会在标记结束后设置下一次触发`GC`的堆内存分配量，分配大对象或从`mcentral`获取空闲内存时，会**判断是否达到了这里设置的`gc_trigger`以决定是否要触发`GC`**。
- **`sysmon`**。**监控线程的一项任务就是强制执行`GC`**，在`runtime`包初始化时，会以`forcegchelper`为执行入口开启一个协程，只不过它被从创建后会很快进入休眠，**监控线程在检测到距离上次`GC`已经超过了指定时间**，**就会把`forcegchelper`协程添加到全局`runq`中，等它得到调度执行时就会开启新一轮的`GC`**。

### 2.2.3 堆内存分配

Go语言的`runtime`将堆地址空间划分成一个一个的`arena`，`arena`区域的起始地址被定义为常量`arenaBaseOffset`，在`amd64`架构下的`Linux`环境下，每个`arena`的大小是`64MB`，起始地址也对齐到`64MB`。每个`arena`包含`8192`个`page`，所以每个`page`大小为`8KB`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223153110980.png" alt="image-   20221223153110980" style="zoom:33%;" />

**问题：**因为程序运行起来所需分配的内存块有大有小，而分散的、大小不一的碎片化内存一方面可能降低内存利用率，另一方面可能会提高要找到大小合适的内存块的代价。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223153254645.png" alt="image-20221223153254645" style="zoom:33%;" />

**解决：**为降低碎片化内存给程序性能造成的不良影响，Go语言的堆分配采用了与**`tcmalloc`内存分配器**类似的算法。**简单来讲就是：**按照一组预置的大小规格把内存页划分成块，然后把不同规格的内存块放入对应的空闲链表中。程序申请内存时，分配器会先根据要申请的内存大小，找到最匹配的规格，然后从对应空闲链表中分配一个内存块。

Go 1.16 `runtime`包给出了67种预置的大小规格，最小`8B`，最大`32KB`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223154122579.png" alt="image-20221223154122579" style="zoom:33%;" />

所以，在划分的整整齐齐的`arena`里，又会按需划分出不同的`span`，每个`span`包含一组连续的`page`，并且按照特定规格划分成等大的内存块。**大小关系是：`arena` -> `span` -> `page` -> `内存块`**。这些就组成了堆内存。

#### 2.2.3.1 数据结构

在堆内存之外，有一大票用于管理堆内存的**数据结构**。

- `mheap`用于管理整个堆内存；
- 一个`arena`对应一个`heapArena`结构；
- 一个`span`对应一个`mspan`结构；
- `mheap`中，有一个全局的`mspan`管理中心，它是一个长度为`136`的数组，数组元素是一个**`mcentral`结构**+一个`padding`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223160941435.png" alt="image-20221223160941435" style="zoom:50%;" />

**问题：**`mcentral`怎么管理`span`呢？

**解答：**实际上，一个`mcentral`对应一种`mspan`规格类型，记录在`spanclass`中，`spanclass`高七位标记内存块大小规格编号，`runtime`提供的预置规格对应编号`1`到`67`，编号`0`留出来用作标记大于`32KB`的大块内存，也即一共`68`种。然后每种规格会按照是否不需要`GC`扫描，进一步区分开，用最低位进行标识。包含指针的需要`GC`扫描，归为`scannable`这一类，不含指针的归为`noscan`这一类。所以共分为136种。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223160954624.png" alt="image-20221223160954624" style="zoom: 33%;" />

每种`spanclass`的`mcentral`中，会进一步将已用尽与未用尽的`mspan`分别管理，每一种又会放到两个并发安全的`set`中：一个是已清扫的，一个是未清扫的。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223161158130.png" alt="image-20221223161158130" style="zoom:33%;" />

全局`mspan`管理中心`mcentral`方便取用各种类型的`mspan`，但是为了保障多个`p`之间并发安全，免不了频繁的加锁、解锁，**为降低多个`p`之间的竞争性**，Go语言的每个`p`都有一个**本地小对象缓存`p.mcache`**，从这里取用就不用再加锁了。`mcache`有一个长度为`136`的`*mspan`类型的数组，还有专门用于分配小于`16`字节的`noscan`类型的`tiny内存`，当前`p`需要用到特定规格类型的`mspan`时，先去本地缓存这里找对应的`mspan`，如果没有或者用完了，就去`mcentral`获取一个放到本地，把已用尽的归还到`mcentral`的`full set`中，

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223161653430.png" alt="image-20221223161653430" style="zoom:33%;" />

接下来看`heapArena`，这里存储着`arena`的元数据，里面有一群位图标记。

- 其中，`bitmap`位图，用**一位**标记这个`arena`中一个指针大小的内存单元到底是指针还是标量，再用一位来标记这块内存单元的后续单元是否包含指针，而且为了便于操作，`bitmap`中用**一字节**标记`arena`中4个指针大小的内存空间，低4位用于用于标记指针/标量，高4位用于标记扫描/终止。例如此处`slice`，`bitmap`第一字节的`0-3`位分别标记三个对应字段是指针还是标量，第`4-6`位分别标记三个对应字段是否需要继续扫描。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223172629595.png" alt="image-20221223172629595" style="zoom: 33%;" />

- `pageInUse`是个`uint8`类型的数组，长度为`1024`，所以一共`8192`位。这个位图只标记**处于使用状态(mSpanInUse)**的`span`的第一个`page`。例如，`arena`中第一个使用状态的`span`包括两个`page`，对应`pageInUse`中第`0`位标为`1`，第二个`span`也在使用中，它包括三个`page`，但只有第一个`page`对应的第`2`位会被标记为`1`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223173130960.png" alt="image-20221223173130960" style="zoom:33%;" />

- `pageMarks`的用法和`pageInUse`一样，只标记每个`span`的第一个`page`，在`GC`标记阶段会修改这个位图，标记哪些`span`中存在被标记的对象，在`GC`清扫阶段会根据这个位图，来释放不含标记对象的`span`，

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223173403051.png" alt="image-20221223173403051" style="zoom:33%;" />

- `spans`是个`*mspan`类型的数组，大小为`8192`，正好对应`arena`中的`8192`个`page`，所以用于定位一个`page`对应的`mspan`在哪儿。`mspan`管理着`span`中一组连续的`page`，同`mcentral`一样，将划分的内存块规格类型记录在`spanclass`中。
  - `nelem`记录着当前`span`共划分成多少个内存块，`freeIndex`记录着下个空闲内存块的索引。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223173926642.png" alt="image-20221223173926642" style="zoom:33%;" />

- 与`heapArena`不同，`mspan`这里的位图标记，面向的是划分好的内存块单元。

  - `allocBits`位图用来标记哪些内存块已经被分配了。

  - `gcmarkBits`是当前`span`的标记位图，在`GC`标记阶段会对这个位图进行标记，一个二进制位对应`span`中的一个内存块，到`GC`清扫阶段会释放掉旧的`allocBits`，然后把标记好的`gcmarkBits`用作新的`allocBits`，这样未被`GC`标记的内存块就能回收利用了。当然，还会重新分配一段清零的内存给`gcmarkBits`位图。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221223174343329.png" alt="image-20221223174343329" style="zoom:33%;" />

#### 2.2.3.2 分配策略

`mallocgc`是负责堆分配的关键函数，`runtime`中的`new`系列和`make`系列函数都依赖它。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221225164824873.png" alt="image-20221225164824873" style="zoom:33%;" />

它的主要逻辑可以分为四个部分：

- **第一部分：辅助GC。**如果程序**申请堆内存时**，正处于**GC标记阶段**，当下已分配的堆内存还没标记完，你这边又要分配新的内存，万一内存申请的速度超过了GC标记的速度，就可能会出现内存不够的情况。所以，申请一字节内存需要做多少扫描工作？或者说，完成一字节扫描工作后可以分配多大的内存空间？这都是根据GC扫描的进度更新计算的。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221225170457086.png" alt="image-20221225170457086" style="zoom:33%;" />

每次**执行辅助GC，最少要扫描`64KB`。**这是因为协程每次执行辅助GC，多出来的部分会作为信用存储到当前`g`中，就像信用卡的额度一样，后续再执行`mallocgc()`时，**只要信用额度用不完**，就**不用执行辅助GC**了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221225170844061.png" alt="image-20221225170844061" style="zoom:33%;" />

此外，还有一种方法可以逃避辅助GC：**窃取信用**。后台的`GC mark worker`执行扫描任务时，会在**全局`gcController`**这里**(bgScanCredit)积累信用**，如果能够窃取足够多的信用值来抵消当前协程背负的债务(**说明此时空闲内存足够大**)，那就不用执行辅助GC了。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221225171110124.png" alt="image-20221225171110124" style="zoom:33%;" />

- **第二部分：空间分配。**这里需要根据要分配的空间大小，以及是否为`noscan`型空间来选择不同的分配策略。

  - 如果是`noscan`类型且`大小<maxTinySize`，会使用`tiny allocator`；
  - `大小>maxSmallSize`的内存分配，包括`noscan`和`scanable`类型，都会采用大块内存分配器；
  - `maxSmallSzie>=大小>=maxTinySize`的`noscan`类型、以及`maxSmallSize>=大小`的`scanable`类型，会使用直接匹配预置大小规格来分配。

  <img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221225174042977.png" alt="image-20221225174042977" style="zoom:33%;" />

1. `大小>32KB`的大块内存额外处理，这是因为预置的内存规格最大才`32KB`，所以**会直接根据所需页面数，分配一个新的`span`**。

2. 而对于`<16B`的内存分配，也不直接匹配预置内存规格，主要是为了减少浪费：如果需要连续分配`16`次`1B`的内存，每次分配时匹配预置的内存规格为`8B`(这是最小的了)，那么每次就会浪费`7B`。而**`tiny allocator`能够将几个小块的内存分配请求合并**，所以例子中`16`次`1B`的内存分配请求可以合并到一个`16B`的内存块中。诸如此类，可以提高内存使用率。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221225174917460.png" alt="image-20221225174917460" style="zoom:33%;" />

**`tiny allcoator`分配内存的大致过程：**每个`p`的`mcache`有专门用于`tiny allocator`的内存(`mcache.tiny`)。这是一个`16B`的内存单元，`mcache.tinyoffset`记录这段内存已经用到哪里了，如果`tiny allocator`还够分配`size`大小的内存，就在`tiny`内存块中直接分配。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221225181609105.png" alt="image-20221225181609105" style="zoom:33%;" />

(接上)如果剩余的空间不够了，就从当前`p`的`mcache`中，找到对应的`mspan`，重新拿一个`16B`大小的内存块来用。如果本地缓存中相应规格的`mspan`也没有空间了，就会从`mcentral`中拿一个新的`mspan`过来，分配完以后，如果新拿来的内存块的剩余空间比旧内存的剩余空间还要大，那就用新的内存块把旧的`tiny`替换掉(旧的还在mcache，只不过不引用了)。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221225181807160.png" alt="image-20221225181807160" style="zoom:33%;" />

3. 对于最后一种，直接通过本地`mcache`与全局`mcentral`配合工作，找到匹配规格的`mspan`即可。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221225182136720.png" alt="image-20221225182136720" style="zoom:33%;" />

空间分配好了还没完，还要**记录下哪些内存已被分配，哪些数据需要GC扫描**，才能继续内存管理工作。所以接下来需要进行一系列**位图标记**。

#### 2.2.3.3 位图标记

> #### **问题：**通过一个堆内存地址，如何找到对应的`heapArena`和`mspan`？

- 已知：一个堆内存地址`p`，`arena`区域起始地址如图(arenaBaseOffset)，每个`arena`大小为`heapArenaBytes`。
- 求：`p`在第几个`arena`中？

- **答：**`arena编号` = `(p-arenaBaseOffset)`/`heapArenaBytes`

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227153807927.png" alt="image-20221227153807927" style="zoom: 33%;" />

- 已知：amd64架构的Linux环境下，一个`arena`大小和对齐边界都是`64M`(26位)，而虚拟地址空间中的线性地址有`48`位，那`48`位的线性地址可以寻址的虚拟空间就是`2^48`这么大。
- 求：这么大的空间可以划分成多少个`arena`？
- **答：**`2^48`/`64M`=`4M`

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227154547110.png" alt="image-20221227154547110" style="zoom:33%;" />

Go开发者把`heapArena`的地址存到了一个二维数组中，寻址`heapArena`时，也不能直接使用`arena`的编号，而是根据`arena`编号计算出一个`arenaIdx`，它本质上是一个`uint`，只不过分两部分，分别作为两个维度的索引。

- 在amd64位的Windows环境下，`arenas`数组第一维有`64`个元素，所以`arenaIdx`第一维度索引占`6`位，第二维数组长度为`1M`，所以`arenaIdx`中低`20`位用作第二维的索引。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227155429504.png" alt="image-20221227155429504" style="zoom: 50%;" />

- 但在amd64位的Linux环境下，这个`arenas`数组第一维只有一个元素，第二维有`4M`个元素，`arenaIdx`的低`22`位都用做第二维的索引，本质上和直接使用`arena`编号是一样的。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227155506501.png" alt="image-20221227155506501" style="zoom: 33%;" />

**到这里，总算能根据内存地址，找到对应的`heapArena`了，接下来就来找`mspan`。**

- 已知：`arena`中每个`page`大小为`pageSize`，每个`arena`中有`pagesPerArena`个`page`。
- 求：`p`在这个`arena`中第几个`page`？
- **答：**`page`编号 = `(p/pageSize)` % `pagePerArena`

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227160003822.png" alt="image-20221227160003822" style="zoom: 50%;" />

确定了`page`的索引，就能在`heapArena.spans`数组中，找到对应的`mspan`的地址了。不过标记完了也还不能结束，还要有**收尾工作**。

#### 2.2.3.4 收尾工作

如果当前处在`GC`标记阶段，就需要对新分配的对象进行`GC`标记，而且，如果此次内存分配达到了`GC`的触发条件，还会触发新一轮的`GC`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227160500434.png" alt="image-20221227160500434" style="zoom:33%;" />

### 2.2.4 栈内存分配

#### 2.2.4.1 栈内存分配过程

堆内存分配中的`arena`中的`span`除了用作堆内存分配外，也用于栈内存分配，只是用途不同的`span`对应的`mspan`状态不同，用作堆内存的`mspan`是`mSpanInUse`状态，用作栈内存的是`mSpanManual`状态。

为提高栈内存分配效率，调度器初始化时，会初始化**两个**用于栈内存分配的全局对象：

- **`stackpool`**面向**`32KB`以下**的栈分配，栈大小必须是`2`的幂，最小为`2KB`；

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227163813640.png" alt="image-20221227163813640" style="zoom:50%;" />

- **大于等于`32KB`**的栈，由**`stackLarge`**来分配这也是个`mspan`链表的数组，长度为25，`mspan`规格从`8KB`开始，之后每个链表的`mspan`规格都是前一个的`2`倍，`8KB`和`16KB`这两个链表实际上一直是空的，留着他们是方便使用`mspan`包含页面数的(以2为底)对数作为数组下标。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227163745608.png" alt="image-20221227163745608" style="zoom:50%;" />

初始化后，这些链表都还是空的，接下来它们会作为**全局栈缓存**来使用。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227165230972.png" alt="image-20221227165230972" style="zoom:50%;" />

同堆内存分配一样，每个`p`也有用于栈分配的本地缓存，这相当于是`stackpool`的本地缓存。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227165336958.png" alt="image-20221227165336958" style="zoom: 50%;" />

要分配栈内存时：

- **小于`32KB`的栈空间**，会优先使用当前`p`的本地缓存。**如果**本地缓存中对应规格的内存块链表为空，就从`stackpool`分配`16KB`的内存放到本地缓存(`stackcache`)中，然后继续从本地缓存分配；**如果**`stackpool`中对应的链表也为空，就从堆内存中直接分配一个`32KB`的`span`，划分成对应的内存块大小放到`stackpool`中；**不过有些情况下**，是无法使用本地缓存的，在不能使用本地缓存的情况下，就直接从`stackpool`分配；

1. 本地无可用缓存，从`stackpool`分配：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227165751844.png" alt="image-20221227165751844" style="zoom:33%;" />

2. `stackpool`也为空：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227170032933.png" alt="image-20221227170032933" style="zoom:33%;" />

3. 无法使用本地缓存：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227170251317.png" alt="image-20221227170251317" style="zoom:25%;" />

- **大于等于`32KB`的栈空间**，就计算需要的`page`数目，并以`2`为底求对数(`log2npage`)，将得到的结果作为`stackLarge`数组的下标，找到对应的空闲`span`链表。**若**链表不为空，就拿一个过来用；**若**链表为空，就直接从堆内存分配一个拥有这么多个页面的`span`，并把它整个用于分配栈内存，

1. 链表不为空：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227170450338.png" alt="image-20221227170450338" style="zoom:33%;" />

2. 链表为空：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221227170820284.png" alt="image-20221227170820284" style="zoom:33%;" />

---

#### 2.2.4.2 栈增长

栈内存初始分配发生在`goroutine`创建时，由于**初始栈大小都是`2KB`**，在实际业务中可能会不够用，所以需要实现一种**在运行阶段动态增长栈的机制**。

`goroutine`的栈增长，是通过编译器和`runtime`合作实现的。编译器会在函数的头部安插检测代码，检查当前剩余的栈空间是否够用。若不够用，就调用`runtime`中的相关函数来**增长栈空间(`runtime.morestack_noctxt`)**，栈空间是**成倍**增长的，需要增长时，就先把当前的栈空间大小`x2`，并把协程状态置为`_Gcopystack`，接下来调用`copystack`函数分配新的栈空间并拷贝旧栈上的数据，释放旧栈的空间，最后恢复协程运行`_Grunning`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221228155220366.png" alt="image-20221228155220366" style="zoom:33%;" />

#### 2.2.4.3 栈收缩

栈收缩可以减少**运行中的协程**对栈空间的浪费。

- 栈收缩不会缩到比`2KB`还小。

- **唯一**可以触发栈收缩的地方就是**`GC`**。

`GC`通过`scanstack`函数寻找标记`root`节点时，如果发现可以安全的收缩栈，就会执行栈收缩；不能马上执行时，就设置栈收缩标识(`g.preemptShrink=true`)，等到协程检测到抢占标识(`stackPreempt`)，在让出`CPU`前会检查这个栈收缩标识，为`true`时就会先进行栈收缩，再让出`CPU`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221228155813634.png" alt="image-20221228155813634" style="zoom: 50%;" />

#### 2.2.4.4 栈释放

但是**结束运行的协程**的栈空间该怎么回收利用？

常规`gorontine`结束时，会被放到调度器对象的空闲`g`队列(`sched.gFree`)中，这里的空闲协程分两种：

- 一种有协程栈(`sched.gFree.stack`)；
- 一种没有协程栈(`sched.gFree.noStack`)。

创建协程时，会先看看这里有没有空闲协程可以用，**优先使用有栈的协程**，其次使用无栈的协程。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221228160542263.png" alt="image-20221228160542263" style="zoom:50%;" />

不过，常规`goroutine`运行结束时，都有协程栈，应该进到哪个队列呢？

- 如果协程栈没有增长过(还是`2KB`)，就把这个协程放到**有栈的空闲`g`队列**中。而这些空闲协程的栈，也会在`GC`执行`markroot`时被释放，到时候有栈的空闲`g`也会加入到无栈的空闲`g`队列中。
- 如果协程栈有增长过，就把协程栈释放掉，再把协程栈放入**无栈的空闲`g`队列中**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221228161114623.png" alt="image-20221228161114623" style="zoom:50%;" />

**问题：**那么栈释放到哪里了呢？是放回到当前`p`的本地缓存？还是放回到全局栈缓存？抑或是直接还给堆内存？

**答：**其实都有可能，要视情况而定。

- **小于`32KB`的栈：**在释放时会先放回本地缓存中，如果本地缓存**对应链表**中栈空间总和大于`32KB`，就把一部分放回`stackpool`中，**本地这个链表只保留`16KB`**；如果本地缓存不可用，也会直接放回`stackpool`中。而且，如果发现这个`mspan`中，所有内存块都被释放了就会把它归还给堆内存。
- **大于等于`32KB`的栈：**如果当前处于`GC`清理阶段(`gcphase == _GCoff`)，就直接释放到堆内存；否则先把它放回到`stackLarge`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221228162025052.png" alt="image-20221228162025052" style="zoom:50%;" />

# 三、面试题

## 3.1 `Goroutine调度`

#### 题目：1

```go
func main() {
	runtime.GOMAXPROCS(1)

	var wg sync.WaitGroup
	wg.Add(3)

	go func(n int) {
		println(n)
		wg.Done()
	}(1)

	go func(n int) {
		println(n)
		wg.Done()
	}(2)

	go func(n int) {
		println(n)
		wg.Done()
	}(3)

	wg.Wait()
}
```

首先，第2行`runtime.GOMAXPROCS(1)`把`GMP`模型中**`P`的数量限制为`1`**，这也就限制了任一时刻只允许一个`M`执行`Go`代码。在这个前提下，再来分析这几个`goroutine`的执行顺序。

通过关键字`go`来创建新的`goroutine`，实际上会被编译器转化为对`runtime.newproc`的调用。**该函数的主要逻辑**就是先切换至系统栈，然后调用`newproc1`函数，分配并初始化一个新的`g`，再通过`runqput`把新的`g`添加到**当前`P`的本地`runq`**中。

听起来，最后的输出应该是`1 2 3`。**但实际上最后的输出是`3 1 2`。**

实际上，**`P`不仅有本地`runq`，还有一个`runnext`字段**，用来保存下次要运行的`g`，`newproc1`中调用`runqput`时会用到这个`runnext`。过程如下，我们将输出的数值作为对应`goroutine`的代号，：

- 首先，`1`号`goroutine`被记在`runnext`中，`2`号`goroutine`把`1`号`goroutine`挤走，`1`号`goroutine`进入到本地`runq`；
- 接下来，`3`号`goroutine`又会把`2`号`goroutine`挤走，`2`号`goroutine`会进入本地`runq`队列尾部；
- 调度`goroutine`执行时，通过`runqget`获取待执行的`g`。而`runqget`也会对`runnext`特殊处理，**优先调度`runnext`**这里记录的`g`，再按顺序调度本地`runq`中记录的`g`；
- 因此，`3`号`goroutine`优先执行，然后是本地`runq`队列，也就是`3 1 2`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230118225323923.png" alt="image-20230118225323923" style="zoom:33%;" />

**本题只有 3 个`goroutine`，那如果是 4、5、6、… 个呢？**又是什么情况？请看下题！

#### 题目：2

```go
func main() {
	runtime.GOMAXPROCS(1)
	var wg sync.WaitGroup
    N := 4	    // 可以不断增大 N，临界点在 257
	for i := 1; i <= N; i++ {	
		wg.Add(1)
		go func(n int) {
			println(n)
			wg.Done()
		}(i)
	}

	wg.Wait()
}
```

其中，`4`个`goroutine`的输出是`4 1 2 3`，`5`个`goroutine`的输出是`5 1 2 3 4`，直到`257`个`goroutine`的输出是`257 1 2 ... 256`，可以**看出`N <= 257`时，都是这个规律**。

**但是`N > 257`时又是什么情况呢？**不要忘记**全局`runq`**！！！

先解释一下`N <= 257`是为啥？`P`的`runnext`可以记录`1`个`g`，本地`runq`可以记录`256`个`g`，一共就是`257`个。所以若`N = 257`，就会出现如下情况：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230118230721962.png" alt="image-20230118230721962" style="zoom:33%;" />

接下来若又到达`258`号`g`，会把`257`号`g`挤走，但本地`runq`已满，所以**`257`号`g`会和本地`runq`中的前一半`g`**一起进到全局`runq`中(之所以取前一半是为了防止饥饿，之所以随机放到全局`runq`是为了避免多核`cpu`修改同一个缓存)。

**按照平时描述的调度逻辑**：先从本地`runq`获取待执行的`g`，没有的话再从全局`runq`获取，还没有的话就去别的`p`那里`steal`一部分。所以：

- 这里最先执行的是`runnext`中的`258`号`goroutine`；
- 接下来是本地`runq`中的`129`号-`256`号；
- 最后全局队列中的`g`才会被拿回本地`runq`。

==**但实际上并非如此**==。实际运行发现，在`129`号-`256`号之间，会发现`1`号和`2`号也穿插在这段区间内被执行了。这个问题与`runq`的排队逻辑无关，属于调度逻辑的范畴。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230119000143891.png" alt="image-20230119000143891" style="zoom:33%;" />

在介绍`runtime.schedule`时，介绍过**每隔`61`个`schedtick`就会优先从全局`runq`中获取`goroutine`**，这样是为了避免在每个`p`的本地`runq`都繁忙的时候，全局`runq`中的`goroutine`迟迟得不到调度的情况。

#### 问答题集合

> #### **问：GPM模型中，P引入的原因？**

**答：**

- 一开始所有的`g`都在一个全局`runq`中，用一个全局的`mutex`保护全局`runq`，多个`m`从全局`runq`中获取`g`时需要频繁的加解锁及等待；
- `g`的每次执行会被随机的分到不同的`m`，造成在不同`m`的频繁切换，破坏程序的局部性；(这点有点怪，有待确定)
- 每个`m`都会关联一个内存分配缓存，造成大量的内存开销，但实际上只有执行`g`的`m`才需要，那些阻塞在调度的`m`根本不需要；
- 引入`p`后，`m`就可以直接从`p`处获取待执行的`g`，不用每次都和众多`m`从一个全局队列中争抢任务，提高了并发性能。

> #### **问：多个线程可以属于同一个进程并共享内存空间。**
>
> 因为多线程不需要创建新的虚拟内存空间，所以它们也不需要内存管理单元处理上下文的切换，线程之间的通信也正是基于共享的内存进行的，与重量级的进程相比，线程显得比较轻量。这不是挺好的嘛，**为啥还要引入`goroutine`？**

**答：**

- 虽然线程比较轻量，但是在调度时也有比较大的额外开销；
- 每个线程都需要一个**用户栈**和**内核栈**，当需要使用系统资源的时候，会通过系统调用进入内核态。(为什么要分开？为了防止用户态代码访问内核数据)。当系统的线程达到一定规模，内核栈和用户栈会占用大量内存；并且，操作系统基于时间片的策略调度所有线程，若线程规模过大，为了降低延迟，线程每次获得的时间片会被压缩，从而导致线程切换频率变大。线程的频繁切换也会占用大量CPU资源。而高并发的场景需求就是要**频繁切换**。
- 引入协程，可以**节省内存空间**，降低**调度代价**。**协程的调度不需要操作系统参与，只需要用户态程序调度。**
- 每个线程都会占用 1M 以上的内存空间，在切换线程时不止会消耗较多的内存，恢复寄存器中的内容还需要向操作系统申请或者销毁资源，每一次线程上下文的切换都需要消耗 ~1us 左右的时间，但是 Go 调度器对 Goroutine 的上下文切换约为 ~0.2us，减少了 80% 的额外开销。

> #### **问：**谁负责触发`timer`注册的回调函数？

**答：**`timer`的触发分为**调度触发**和**监控线程触发**，两者主要是通过调用函数` checkTimers() `来实现。

其实每个`p`都有一个**最小堆`p.timers`**，用于管理自己的`timer`，**堆顶的`timer`就是接下来要触发的那一个**(老版本这个堆是全局的，就很慢！)。而**工作线程每次调度时(执行`schedule()`时)，都会调用`checkTimers()`函数**，检查并执行那些已经到时间的`timer`。**不过这不够稳妥**，万一所有的`m`都在忙，那么就不能及时触发调度了，可能会导致`timer`执行时间发生较大偏差。所以还会通过**监控线程**来增加一层保障。

**监控线程**是由`main goroutine`创建的，与`GPM`中的工作线程不同，并不需要依赖`p`，也不由`GPM`调度。监控线程有多个任务，其中一项便是**保障`timer`的正常执行**。监控线程检测到接下来有`timer`要执行时(遍历所有的`p`，找出下次最先执行(时间值最小)的时间和其所在的`p`)，若此时无空闲`m`，便会创建新的工作线程以保障`timer`可以顺利执行。

> #### **问：**如何**抢占**？具体说说？

**答：**监控线程要本着**公平调度**的原则，对运行时间过长的`p`实行**"抢占"**操作。就是告诉那些运行时间超过特定阈值(10ms)的`g`，该让出了！

Go 1.14之前依赖于栈增长。**展开：**当`runtime`希望某个协程让出CPU时，就会把他的`stackguard`赋值为`stackPreempt`，这是一个非常大的值，真正的栈指针不会指向这个位置，因此用作特殊标识。进而会跳转到`morestack`处，而`morestack`会调用`runtime.newstack()`函数，负责栈增长工作。不过他在进行栈增长工作前会先判断`stackguard0`是否等于`stackPreempt`，等于的话就不进行栈增长了，**而是执行一次协程调度**。这种方式的缺点是过度依赖栈增长代码，如果来个空的`for{}`循环，因为与栈增长无关，程序就会卡死在这个地方。

其实，为了充分利用CPU，监控线程还会抢占处在**系统调用中的`p`**，因为一个协程要执行系统调用，就要切换到`g0`栈，在系统调用没执行完之前，这个`m`和`g`其实绑定了，不能被分开，也就用不到`p`，所以在陷入系统调用前，当前`m`会让出`p`，解除`m.p`与当前`p`的强关联，只在`m.oldp`中记录这个`p`。`p`的数目毕竟有限，如果有其他协程正在等待执行，那就会把他关联到其他`m`。

不过如果当前`m`从系统调用中恢复，会先检测之前的`p`是否被占用，没有的话就继续使用，否则再去申请一个，没申请到的话，就把当前`g`放入全局`runq`中，然后当前线程就睡眠了。

> #### **问：**那抢占时，怎么知道某个`g`运行时间过长了呢？

**答：**`p`里面有一个`schedtick`字段，每当调度执行一个新的`g`，并且不继承上个`g`的时间片时，就会把`p.schedtick++`，而`sysmontick.schedwhen`记录上一次调度的时间。监控线程如果监测到`sysmontick.schedtick`与`p.schedtick`不相等，说明这个`p`发生了新的调度，就会同步`sysmontick.schedtick`的值，并更新调度时间`sysmontick.schedwhen`；但若二者相等，说明没发生新的调度，或者即使发生了新的调度，也沿用了之前的时间片，所以可以通过当前时间与`sysmontick.schedwhen`的差值来判断当前`p`上的`g`是否运行时间过长。

> #### **问：**如何**调度**？展开说说？

**答：已经抢占了`p`，得调度个的`g`过来执行吧？**这就用到了**`schedule()`**。

- 首先，要确定当前`m`是否和当前`g`绑定了。如果绑定了，那当前`m`就不能执行其他`g`，会阻塞当前`m`，等到当前`g`再次得到调度执行时，就会把`m`唤醒；如果没有绑定，就先看看`GC`是不是在等待执行。如果`GC`正等待执行，就去执行`GC`，回来再继续执行调度程序；
- 接下来，会执行`checkTimer()`检查有没有要执行的`timer`；然后会先去本地`runq`中查找，没有的话就调用`findrunnable()`，这个函数直到获取到待执行的`g`才会返回；
- 在`findrunnable()`处，也会判断是否要执行`GC`，然后先尝试从本地`runq`中获取 -> 没有的话就从全局`runq`中获取一部分 -> 如果还没有，就先尝试执行`netpoll`，恢复那些`I/O事件`已经就绪的`g`，它们会被放回全局`runq`中 -> 然后才会尝试从其他`p`那里`steal`一些任务。
- **当调度程序终于获得一个待执行的`g`后**，还需要检查是否已经绑定某个`m`，如果已经绑定了某个`m`，还得把这个`g`送回去，而当前`m`不得不再次进行`schedule()`调度；如果没有绑定的`m`，就调用`execute()`在当前`m`上执行这个`g`。
- `execute()`会建立当前`m`与`g`的关联关系，并把`g`的状态从`_Grunnable`改为`_Grunning`，如果不继承上一个协程的时间片，就把`p`这里的调度计数`p.schedtick`+1；
- 最后，会调用`gogo()`函数，从`g.sched`这里恢复协程栈指针、指令指针等继续执行。

> #### 问：

当你不知道一个协程什么时候停止时，你就不应该去创建这么个协程。也就是，不能滥用并发。

> #### 问：Goroutine 什么时候会发生泄漏？怎么发现？Goroutine 被占满了怎么办？:triangular_flag_on_post::star:

**Goroutine 泄露的原因大多集中在**：

- Goroutine 内正在进行 channel/mutex 等读写操作，但由于逻辑问题，某些情况下会**被一直阻塞**。比如：
  - 使用channel时，只有写没有读、只有读没有写，比如以下代码：

```go
// 1. 只写不读
func main() {
	fmt.Printf("goroutines: %d\n", runtime.NumGoroutine()) // 1，main  goroutine
	for i := 0; i < 4; i++ {
		queryAll()
		fmt.Printf("goroutines: %d\n", runtime.NumGoroutine()) // 每次 + 3
	}
}

func queryAll() {
	ch := make(chan int) // 无缓冲channel
	for i := 0; i < 3; i++ {
		go func() { ch <- 1 }() // 只有写，没有读，造成goroutine阻塞，也就是goroutine泄露
	}
}

// 2. 只读不写
func main() {
	defer func() {
		fmt.Println("goroutines: ", runtime.NumGoroutine())	// 2
	}()

	ch := make(chan int)
	go func() {
		<-ch	// 只读不写，也会阻塞，造成泄露
	}()
	time.Sleep(time.Second)
}
```

- Goroutine 内的业务逻辑**进入死循环**，资源一直无法释放
  - 比如说，用锁：

```go
func main() {
	total := 0
	defer func() {
		time.Sleep(time.Second)
		fmt.Println("total: ", total)                       // 0
		fmt.Println("goroutines: ", runtime.NumGoroutine()) // 11
	}()

	var mutex sync.Mutex
	mutex.Lock()
	for i := 0; i < 10; i++ {
		go func() {
			mutex.Lock() // 阻塞在此处
			total += 1
		}()
	}
}
```

- Goroutine 内的业务逻辑**进入长时间等待**，有不断新增的 Goroutine 进入等待。

如何避免？

- 创建带有缓冲区的 channel，这样就可以避免没有接收端导致的阻塞；
- 使用 select 尝试发送；

---

以上是，goroutine泄露的情况，如何进行排查？

- 通过 `runtime.NumGoroutine` 可以获取 goroutine 数量；
- 通过 `pprof`。

可继续学习 [pprof](https://golang2.eddycjy.com/posts/ch6/01-pprof-1/)。

> #### 问：如何控制协程的并发数量？

不同的应用程序，消耗的资源是不一样的。比较推荐的方式的是：应用程序来主动限制并发的协程数量。

1. **利用 channel 的缓冲区来实现**

```go
// main_chan.go
func main() {
	var wg sync.WaitGroup
	ch := make(chan struct{}, 3)
	for i := 0; i < 10; i++ {
		ch <- struct{}{}
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			log.Println(i)
			time.Sleep(time.Second)
			<-ch
		}(i)
	}
	wg.Wait()
}
```

最多有 3 个 goroutine。

2. **利用第三方包，构造协程池**

- [Jeffail/tunny](https://github.com/Jeffail/tunny)
- [panjf2000/ants](https://github.com/panjf2000/ants)

以 `tunny` 举例：

```go
package main

import (
	"log"
	"time"

	"github.com/Jeffail/tunny"
)

func main() {
	// 第一个参数是协程池的大小(poolSize)，第二个参数是协程运行的函数(worker)
	pool := tunny.NewFunc(3, func(i interface{}) interface{} {
		log.Println(i)
		time.Sleep(time.Second)
		return nil
	})
	defer pool.Close() // 关闭协程池

	for i := 0; i < 10; i++ {
		go pool.Process(i) // 将参数 i 传递给协程池定义好的 worker 处理
	}
	time.Sleep(time.Second * 4)
}
```

3. **调整资源的上限**

- ulimit

`ulimit -a` 可以看到系统当前的设置：

```zsh
❯ ulimit -a
-t: cpu time (seconds)              unlimited
-f: file size (blocks)              unlimited
-d: data seg size (kbytes)          unlimited
-s: stack size (kbytes)             2032
-c: core file size (blocks)         unlimited
-n: file descriptors                3200
-v: address space (kbytes)          unlimited
```

可通过上述参数，设置系统资源限制，控制协程数目。

## 3.2 组合式继承

> #### 问：如下代码输出什么？

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230501224820368.png" alt="image-20230501224820368" style="zoom:40%;" />

> #### 问：组合式继承中，编译器生成包装方法的规则？

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230501225344225.png" alt="image-20230501225344225" style="zoom: 40%;" />

## 3.3 闭包

```go
func main() {
	x()()
}

func x() (y func()) {
	y = func() {
		println("y")
	}
	return func() {
		println("z")
		y()
	}
}
```

会输出什么呢？

会不断的输出`z`。

先看`main`函数的栈帧：

`x`函数只有返回值没有参数，而`x`的返回值是一个函数。**函数作为参数、返回值或者被赋值给变量时，就被称为`Function Value`。**所以这里的`y`是一个`Function Value`。**`Function Value`本质上是一个指针，指向一个`runtime.funcval`结构体，这个结构体存储了对应函数的指令入口地址。**`x`返回一个匿名函数，而这个函数中捕获了`y`，所以这个返回值是一个**闭包对象**。**闭包对象就是一个有捕获列表的`Function Value`而已。**

也就是说在这个函数入口地址后面，会有该闭包函数捕获的变量，**不过这里捕获的究竟是变量的值还是它的地址？**其实，如果被捕获的变量在其赋初值后没有在被修改过，就会捕获变量的值；反之，就捕获变量的地址。

在函数`x`中，先给`y`赋初值，`return`又将新的返回值写到了`y`，所以闭包对象这里捕获的是变量的地址。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230131174816983.png" alt="image-20230131174816983" style="zoom:50%;" />

但是当`y`的地址被捕获了，当`x`执行结束，`main`函数调用返回的`y`时，这里的栈帧就不再为调用`x`服务了，所以被捕获的变量需要逃逸到堆上，返回值这里就要存储`y`在堆上的地址，但是，这样就改变了返回值的类型，所以让`y`逃逸到堆上是行不通的。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230131175045496.png" alt="image-20230131175045496" style="zoom:50%;" />

那么，编译器会怎样处理**返回值的地址被捕获的情况**呢？

实际上，编译器会在堆上分配一个`y`的副本，记为`y'`，同时为`x`生成一个局部变量，存储`y'`在堆上的地址，记为`py'`，然后在函数`x`和返回的闭包对象中都使用副本`y'`，这里捕获的是`y'`的地址，只在`x`返回前将`y'`的值拷贝到返回值空间，这样`y`和`y'`都指向堆上的同一个闭包对象。而在调用`x`的栈帧销毁后，这个闭包对象依然可以正常使用其捕获的变量。

当返回值`y`被调用，找到这里的闭包函数，输出第一个字母`z`，然后通过捕获列表找到`y'`，调用`y'`指向的函数-还是这个函数，因此会不停输出`z`。

## 3.4 GC

> #### **问：自动垃圾回收怎么区分哪些数据是垃圾呢？**

可以确定，程序中用得到的数据，一定是从栈、数据段这些根节点追踪得到的数据。也就是说，**从这些根节点追踪不到的数据**一定是没用的数据，一定是**垃圾**。因此，目前主流的自动垃圾回收算法都是使用**"可达性"**近似等价**"存活性"**的。

**==标记-清扫算法==**

标记清除（Mark-Sweep）算法是最常见的垃圾收集算法，标记清除收集器是**跟踪式垃圾收集器**，其执行过程可以分成**标记（Mark）**和**清除（Sweep）**两个阶段：

1. **标记阶段** — 从根对象出发查找并标记堆中所有存活的对象；
2. **清除阶段** — 遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20221222170145205.png" alt="image-20221222170145205" style="zoom:33%;" />

**==三色抽象==**

**三色抽象**可以清晰的展现**追踪式回收**中**对象状态**的**变化过程**。

- 垃圾回收**开始时**，所有数据均为**白色**；
- 然后把直接追踪到的root节点都标记为**灰色**：灰色代表基于当前节点展开的追踪还未完成；
- 当基于某个节点的追踪任务完成后，便会把该节点标记为**黑色**：表示它是存活数据，而且无需基于它再次进行追踪了。
- 基于**黑色**节点找到的所有节点都被标记为**灰色**，表示还要基于它们进一步开始追踪。
- 当没有**灰色**节点时，就意味着标记工作可以结束了。此时，有用数据都为**黑色**，垃圾都为**白色**。接下来回收这些**白色**数据即可。

## 3.5 Mutex

> #### 问：RWMutex 是读优先还是写优先？读优先的话，如果一直有读请求，那么写请求会饥饿吗？:star::triangular_flag_on_post:

`RWMutex` 是一个`读/写互斥锁`，在某一时刻只能由`任意数量`的 `reader` 持有 或者 `一个 writer` 持有。也就是说，要么放行任意数量的 reader，多个 reader 可以`并行读`；要么放行一个 writer，多个 writer 需要`串行写`。

一旦涉及到多个 reader 和 writer ，就需要考虑优先级问题，是 reader 优先还是 writer 优先：

- **读者优先（readers-preference）**：读者优先是读操作优先于写操作，即使写操作提出申请资源，但只要还有读者在读取操作，就还允许其他读者继续读取操作，直到所有读者结束读取，才开始写。读优先可以提供很高的并发处理性能，但是在频繁读取的系统中，会长时间写阻塞，**导致写饥饿**。
- **写者优先（writers-preference）**：写者优先是写操作优先于读操作，如果有写者提出申请资源，在申请之前已经开始读取操作的可以继续执行读取，但是如果再有读者申请读取操作，则不能够读取，只有在所有的写者写完之后才可以读取。写者优先解决了读者优先造成写饥饿的问题。但是若在频繁写入的系统中，会长时间读阻塞，**导致读饥饿**。

`RWMutex` 的数据结构：

```go
type RWMutex struct {
	w           Mutex  // 用于控制多个写锁，获得写锁首先要获取该锁，如果有一个写锁在进行，那么再到来的写锁将会阻塞于此
	writerSem   uint32 // 写阻塞等待的信号量，最后一个读者释放锁时会释放信号量
	readerSem   uint32 // 读阻塞的协程等待的信号量，持有写锁的协程释放锁后会释放信号量
	readerCount int32  // 记录读者个数
	readerWait  int32  // 记录写阻塞时读者个数
}
```

`RWMutex` 设计采用**写优先方法**。

问题要点：

1. **写操作是如何阻止写操作的？**

`RWMutex`包含一个互斥锁(Mutex)，**写锁定必须要先获取该互斥锁**。

2. **写操作是如何阻止读操作的？**

`RWMutex.readerCount`是个整型值，用于表示读者数量，不考虑写操作的情况下，每次读锁定将该值+1，每次解除读锁定将该值-1，所以readerCount取值为[0, N]，N为读者个数，实际上最大可支持2^30^个并发读者。

当写锁定进行时，会先将readerCount减去2^30^，从而readerCount变成了负值，此时再有读锁定到来时检测到readerCount为负值，便知道有写操作在进行，只好阻塞等待，同时还会对readerCount+1，这样等待的读操作个数并不会丢失，只需要将readerCount加上2^30^即可获得。

所以，**写操作将readerCount变成负值来阻止读操作**。

3. **读操作是如何阻止写操作的？**

写操作到来时，会把`RWMutex.readerCount`值拷贝到`RWMutex.readerWait`中，用于标记排在写操作前面的读者个数。前面的读操作结束后，除了会递减`RWMutex.readerCount`，还会递减`RWMutex.readerWait`值，当`RWMutex.readerWait`值变为0时唤醒写操作。

4. **为什么写锁定不会被饿死？**

写操作要等待读操作结束后才可以获得锁，写操作等待期间可能还有新的读操作持续到来，如果写操作等待所有读操作结束，很可能被饿死。然而，通过`RWMutex.readerWait`可完美解决这个问题。

写操作到来时，会把`RWMutex.readerCount`值拷贝到`RWMutex.readerWait`中，用于标记排在写操作前面的读者个数。

前面的读操作结束后，除了会递减`RWMutex.readerCount`，还会递减`RWMutex.readerWait`值，当`RWMutex.readerWait`值变为0时**唤醒写操作**。

> #### 问：Mutex的工作模式？正常模式和饥饿模式？

`Mutex `共有两种工作模式：正常模式和饥饿模式。

**正常模式**下，一个尝试加锁的`goroutine`会先自旋几次，尝试通过原子操作获得锁。若几次自旋之后不能获得锁，就会通过信号量进行排队等待。所有的等待者会按照先入先出的顺序排队，但是当锁被释放，被唤醒的等待者并不会直接获得锁，它需要和处于自旋阶段尚未排队的`goroutine`进行竞争。

这种情况后来者更有优势，首先是因为处于自旋状态的`goroutine`可能有多个，且后来者正在CPU上运行，显然比刚被唤醒的`goroutine`有优势。如果被唤醒的`goroutine`没有获得锁，它将再次进入排队队列，但是是直接在队头。

当一个`goroutine`本次等待加锁的时间超过`1ms`，它会把当前`Mutex`切换到**饥饿模式**，`Mutex`的所有权直接转让给队首`goroutine`，此时后来者也不再自旋，而是直接进入队列尾部开始排队。

当获得`Mutex`的`goroutine`是队列中最后一个时，或者它的等待时间小于`1ms`，它会把`Mutex`的状态改回**正常模式**。

正常模式需要大家争抢锁，从而获得更高的吞吐量；而饥饿模式对防止出现尾端延迟特别重要。

> #### 问：什么时候会发生自旋？

加锁时，如果当前 `Locked` 位为1，则说明当前该锁由其他协程持有，尝试加锁的协程并不是马上转入阻塞，而是会持续探测 `Locked` 位是否变为0，这个过程就是「**自旋**」。实际上就是执行了30次`PAUSE`。

- 多核场景。因为单核场景是没有意义的，一个占着CPU进行自旋等待锁，一个占着锁。这无意义。
- `gomaxprocs>1`。同上。
- 至少有一个其他的`p`正在`running` && 当前`p`的本地`runq`队列为空。比起调度这个`goroutine`进行自旋，不如调度别的`goroutine`。

自旋上限为4，每执行一次自旋都会重新判断是否可以继续自旋。如果锁被释放了，或者锁进入了饥饿模式，或者已经自旋了4次，都会结束自旋。

> #### 问：为什么自旋次数有上限？

自旋是为了避免频繁获得锁失败导致的协程切换，这样开销很大。若锁的持有时间很短，自旋几次就能获得，这样就能减少切换，但如果锁的持有时间很长，那就会无意义的自旋，反倒浪费了CPU资源，所以通常限制自旋次数，自旋次数内无法获得锁就让出CPU，以免长时间无意义的等待。

> #### 问：协程是如何进行排队的呢？

通过`sema`字段。`runtime`内部会通过一个大小为`251`的`sematable`来管理所有`semaphore`，这个`sematable`存储的是`251`课平衡树的根，平衡树中每个节点都是一个`sudog`类型的对象，要使用一个信号量时，需要提供一个记录信号量数值的变量，根据它的地址进行计算，映射到`sematable`中的一颗平衡树上，找到对应的节点就找到了**该信号量的等待队列**，该信号量的协程就在这个队列中进行等待唤醒。

## 3.6 方法集

> #### 问：为什么要限制 `T` 和 `*T` 不能声明同名方法？

首先，`T`和`*T`是两种类型，分别有着自己的类型元数据，而根据自定义类型的类型元数据，可以找到该类型关联的方法列表。

可以确定的是，`T`的方法集里，全部都是有明确定义的接收者为`T`类型的方法；而`*T`的方法集里，除了有明确定义的接收者为`*T`的方法以外，**还会有编译器生成的一些"包装方法"：**这些包装方法是对接收者为`T`类型的同名方法的"包装"。

如果给`T`和`*T`定义了同名方法，就有可能和编译器生成的包装方法**发生冲突**，所以 Go 干脆不允许为`T`和`*T`定义同名方法。

> #### 问：为什么编译器要为 `*T` 生成 `T` 的同名方法？

**这里首先要明确一点：**通过`*T`类型的变量直接调用`T`类型接收者的方法只是一种**语法糖**。经验证，这种调用方式，编译器会在调用端进行**指针解引用**，并**不会用到这里的包装方法**。

==**实际上，**编译器生成包装方法主要是**为了支持接口**。==

> #### 问：为什么是为了支持接口？

非空接口包含两个指针：一个和类型元数据相关，一个和接口装载的数据相关。虽然有数据指针，但是并不能像语法糖那样，**对指针进行解引用来调用值接收者的方法**。这是为啥呢？

**原因：**方法的接收者是方法调用时隐含的第一个参数，Go 中的函数参数通过栈进行传递，如果参数是指针类型，平台确定了，指针大小也就确定了。但**如果要解引用为值类型，就必须有明确的类型信息**，编译器才能知道这个参数**该在栈上分配多大的内存空间**。**对于接口来说**，**编译阶段并不能确定该接口会装载的数据类型**，也就不能进行指针解引用。所以选择为`*T`生成一套`T`的包装方法。

在链接阶段，不会用到的方法会被忽略。

## 3.7 抢占式调度

> #### 问：如何进行抢占式调度？

**Go 1.14 之前**，是借助于**栈增长检测**来实现抢占式调度的，若`goroutine`中没有导致栈增长的代码，就不会被抢占，所以这不算真正的抢占式调度。

以`GC`举例，`STW`阶段需要抢占所有的`P`，但不是说抢占就能抢占的，会先记录要等待多少个`P`，当这个值减为`0`，目的就达到了。

- 对于当前`P`、以及陷入系统调用的`P`(`_Psyscall`)、还有空闲状态的`P`，**直接将其设置为`_Pgcstop`即可**；
- 对于还有`G`在运行的`P`，则会将对应的`g.stackguard0`设置为一个特殊标识(`runtime.stackPreempt`)，告诉它`GC`正在等待它让出。此外，还会设置一个`gcwaiting`标识(设置`gcwaiting=1`)，接下来就通过这两个标识符的配合，来实现运行中的`P`的抢占。

`goroutine`创建之初，栈的大小是固定的，为了防止出现栈溢出的情况，**编译器会在有明显栈消耗的函数头部插入一些检测代码**，通过`g.stackguard0`来判断是否需要进行栈增长：如果`g.stackguard0`被设置为特殊标识`runtime.stackPreempt`，便不会执行栈增长，而是去执行一次调度(`schedule()`)；在`schedule()`调度执行时，会检测`gcwaiting`标识，若发现`GC`在等待执行，便会让出当前`P`，将其置为`_Pgcstop`状态。

**依赖栈增长检测代码的抢占方式，遇到没有函数调用的情况就会出现问题。**

**Go 1.14 之后**，`preemptone`函数会判断当前硬件环境是否支持异步抢占，还会判断用户是否允许开启异步抢占，默认情况下是允许的。如果这两条验证都通过了，就将`p.preempt`字段置为`true`，实际的抢占操作会交由`preemptM`函数来完成。`preemptM`函数会调用`signalM`函数，通过调用操作系统中信号相关的系统调用，**将指定信号发送给目标线程**。

**线程接收到信号后**，会调用对应的信号处理函数`sighandler`来处理，`sighandler`在确定信号为`sigPreempt`(抢占信号)以后，**它会首先判断`runtime`是否要对指定的`G`进行异步抢占**，通过什么来判断呢？

- 指定的`G`与其对应`P`的`preempt`字段都要为`true`，而且指定的`G`还要处在`_Grunning`状态；

- 还要**确认在当前位置打断`G`并执行抢占是安全的**

  - 指定的`G`**可以挂起并安全的扫描它的栈和寄存器**，并且当前被打断的位置并**没有打断写屏障**；

  - 指定的`G`还**有足够的栈空间**来注入一个异步抢占函数调用(`asyncPreempt`)；

  - 这里可以安全的和`runtime`进行交互，主要就是确定当前并**没有持有`runtime`相关的锁**，继而不会在后续尝试获得锁时发生死锁。


**确认了要抢占这个`G`，并且此时抢占是安全的**以后，就可以放心的通过`pushCall`向`G`中**注入异步抢占函数调用**了，被注入的异步抢占函数(`asyncPreempt`)最终会去执行`schedule()`，到这里，异步抢占就完成了。

## 3.8 Channel

> #### 问：channel 是否是并发安全的？

是滴

> #### 问：怎么通过 channel 实现协程间的通信？



> #### 问：向已关闭的 channel 写 会发生什么？从已关闭的 channel 读 会发生什么？:star:

```go
func testClose() {
	intChan := make(chan int, 3)
	intChan <- 2
	// 1. 关闭一个channel
	close(intChan)
	// 2. 向关闭的channel中写入数据
	//intChan <- 3 // 会报错 panic: send on closed channel
	// 3. 从关闭的channel读取数据
	num, ok := <-intChan
	fmt.Println(num, ok) // 2 true
	num, ok = <-intChan
	fmt.Println(num, ok) // 0 false
}
```

> #### 问：如何优雅关闭一个 Channel？:star::star:

**先说说为啥Channel关闭这么麻烦：**

- 不能无脑关闭，**如果一个channel已经关闭**，**重复关闭channel**会导致panic
- **往一个关闭的channel写数据**，也会导致panic

**channel的关闭原则：**:star:

- 不要在消费者端关闭channel
- 不要在有多个并行的生产者时关闭channel(应该只在唯一或者最后一个生产者协程中关闭channel)

==**优雅的关闭，其实要分情况的：**==

- **单个生产者，单个消费者**

​	直接让**生产者关闭**。

- **单个生产者，多个消费者**

  这种情况很简单，直接让**生产者关闭**即可。

- **多个生产者，单个消费者**

​	不能在消费端关闭，这违背了channel关闭原则。可以**让消费者标记一个`close`信号**，通知生产者不要继续写数据。

- **多个生产者，多个消费者**

​	可以设置一个中间调解者角色。

首先设置一个通道`toStop`，当生产者或消费者达到条件，就向`toStop`发送关闭信号，中间角色收到后关闭`stopCh`(仅用作通知发送者不要再向数据通道`dataCh`写数据了)，生产者收到`stopCh`的关闭信号后，不再向`dataCh`写数据。

请注意，信号通道`toStop`的容量必须至少为1。如果它的容量为0，则在中间调解者还未准备好的情况下就已经有某个协程向`toStop`发送信号时，此信号有可能被抛弃。

**参考文章**：

- https://gfw.go101.org/article/channel-closing.html

> #### 问：channel 的发送、接收、关闭？

**发送**：

- 发送操作会对 `hchan` 加锁；
- 当 `recvq` 中存在等待接收的 goroutine 时，若有 goroutine 要发送数据，就会调用 `memmove` 函数从发送 goroutine 的栈中，直接拷贝数据到接收 goroutine 的栈中，不经过 channel (无论 channel 是否有缓冲区)；
- 当 `recvq` 等待队列为空时，会判断 `hchan.buf` 是否可用。如果可用，则会将发送的数据拷贝至 `hchan.buf` 中；
- 如果 `hchan.buf`  已满，那么将当前发送 goroutine 置于 `sendq` 中排队，并在运行时中挂起；
- 向已经关闭的 channel 发送数据，会引发 panic；

对于无缓冲的 channel 来说，它天然就是 `hchan.buf` 已满的情况，因为它的 `hchan.buf` 的容量为 0。

**接收**：

- 接收操作会对 `hchan` 加锁。
- 当 `sendq` 中存在等待发送的 goroutine 时，意味着此时的 `hchan.buf` 已满（无缓存的天然已满），分两种情况：
  - 如果是有缓存的 `hchan`，那么先将缓冲区的数据拷贝给接收 goroutine，再将 `sendq` 的队头 `sudog` 出队，将出队的 `sudog` 上的元素拷贝至 `hchan` 的缓存区。
  - 如果是无缓存的 `hchan`，那么直接将出队的 `sudog` 上的元素拷贝给接收 goroutine。两种情况的最后都会唤醒出队的 `sudog` 上的发送 goroutine。
- 当 `sendq` 发送队列为空时，会判断 `hchan.buf` 是否可用。
  - 如果可用，则会将 `hchan.buf` 的数据拷贝给接收 goroutine。
  - 如果 `hchan.buf` 不可用，那么将当前接收 goroutine 置于 `recvq` 中排队，并在运行时中挂起。
- 与发送不同的是，当 `channel` 关闭时，goroutine 还能从 `channel` 中获取数据。如果 `recvq` 等待列表中有 goroutines，那么它们都会被唤醒接收数据。如果 `hchan.buf` 中还有未接收的数据，那么 goroutine 会接收缓冲区中的数据，否则 goroutine 会获取到元素的零值。

**关闭**：

- 如果关闭已关闭的 channel 会引发 painc。
- 关闭 channel 后，如果有阻塞的读取或发送 goroutines 将会被唤醒：
  - 读取 goroutine 会获取到 hchan 的已接收元素，如果没有，则获取到元素零值；
  - **发送 goroutine 的执行则会引发 painc**。

> #### 问：channel 非得关闭吗？:star::triangular_flag_on_post:

不用，channel 没有被任何协程用到后最终会被 GC 回收。

但，需要分别考虑两种情况：

- **channel 的`发送次数 == 接收次数`**：

发送者 goroutine 和接收者 goroutine 分别都会在发送或接收结束时结束各自的 goroutine (也即是channel中没有阻塞的goroutine)，此时，channel由于没有被使用，就会被垃圾收集器自动回收。这种情况下，**不关闭 channel，没有任何副作用**。

- **channel 的`发送次数 != 接收次数`**：

channel 的发送次数不等于接收次数时，可能会导致发送者或接收者阻塞在channel。因此channel由于一直被使用，导致无法被垃圾回收。阻塞的 goroutine 和未被回收的 channel 都造成了内存泄漏的问题。

> #### 问：如何判断channel已关闭？:thinking::question:

- 通过 `v, ok := <-chan`，若已关闭，ok 就是 false；

似乎得分有缓冲和无缓冲：

```go
// 1. 有缓冲时，只要缓冲区有数据，ok就是true，因此不能用这种方法判断
func main() {
	ch := make(chan int, 2)
	ch <- 1
	close(ch)				// 此处已关闭，但channel依然有数据未读完
	v1, ok1 := <-ch			// 此时channel依然有数据未读完
	fmt.Println(v1, ok1)	// 1 true
	v2, ok2 := <-ch			// 此时channel没数据可读
	fmt.Println(v2, ok2)	// 0 false
}

// 2. 无缓冲时，可以酱紫判断。
func main() {
	ch := make(chan int)
	//ch <- 1
	close(ch)            // 此处已关闭
	v1, ok1 := <-ch      
	fmt.Println(v1, ok1) // 0 false
	v2, ok2 := <-ch      // 此时channel没数据可读
	fmt.Println(v2, ok2) // 0 false
}
```

- 通过 `for range`，会自动判断 channel 是否结束，如果结束则自动退出 for 循环。

同上，也得分有无缓冲。

似乎，没有什么好办法？

> #### 问：Channel 有啥应用？

- 终止信号通知。例如，`main goroutine`等待`hello goroutine`执行完毕；

- 任务定时。与`timer`结合，实现**超时控制**。Etcd中很常见

  - ```go
    select {
        case <-time.After(100 * time.Millisecond):
        case <-s.stopc:
            return false
    }
    ```

- 生产者和消费者。生产者向`Channel`写数据，消费者从`Channel`读数据；

- 控制并发数；

  - ```go
    var limit = make(chan int, 3)
    
    func main() {
        // …………
        for _, w := range work {
            go func() {
                limit <- 1
                w()
                <-limit
            }()
        }
        // …………
    }
    ```

> #### 问：Channel 啥时候会导致内存泄漏？

泄漏的原因是 goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空的状态，一直得不到改变。

此时 `goroutine` 一直阻塞，得不到释放，就造成了内存泄露。其实并不是`channel`本身的内存泄漏？

> #### 问：`channel`的底层是怎样的？

用`make`创建`chan`时，函数栈帧上会存储`chan`的指针，指向堆上的`hchan`结构体。

- 因为`channel`支持协程间并发访问，所以要有一把**锁`lock`**来保护整个数据结构。
- 对于有缓冲来讲，需要知道**缓冲区**在哪，已经存储了多少个元素，最多存储多少个元素，每个元素占多大空间，所以实际上，缓冲区就是个数组`buf`、`elemsize`。
- 因为 Go 运行中，内存复制、垃圾回收等机制依赖数据的类型信息，所以`hchan`还要有一个指针，指向元素类型的**类型元数据**`elemtype`。
- 此外，`channel`支持交替的读(接收)写(发送)，需要分别记录**读、写下标的位置**`recvx`、`sendx`。
- 当读和写不能立即完成时，需要能够让当前协程在`channel`上等待，待到条件满足时，要能够立即唤醒等待的协程，所以要有两个**等待队列**，分别针对读(接收)和写(发送)`recvq`、`sendq`。
- 此外，`channel`能够`close`，所以还要记录它的**关闭状态`closed`**。

> #### 问：`channel`是如何进行读写的？

`channel`的读写是分为阻塞式和非阻塞式的。

1. **阻塞式写**：`ch <- 10`

- **有缓冲区的`channel`**

对于有缓冲区的`channel`来说，若缓冲区有空闲，就可以通过写指针将数据写入指定下标。若缓冲区已满，就会进入`channel`的发送阻塞队列。值得一提的是，`channel`的缓冲区是一个环形缓冲区，写指针和读指针都在不停的循环。

- **无缓冲区的`channel`**

对于无缓冲区的`channel`来说，若有其他协程正在读，就可以发送数据。若无其他协程正在读，就会进入`channel`的发送阻塞队列。

2. **非阻塞式写**：`select case default` 

如果发生阻塞的情况 ，就会执行`default`分支。

3. **阻塞式读**：`<-ch`

- **有缓冲区的`channel`**

对于有缓冲区的`channel`来说，若缓冲区中有数据，就可以通过读指针将数据读出。若缓冲区为空，就会进入`channel`的接收阻塞队列。

- **无缓冲区的`channel`**

对于无缓冲区的`channel`来说，若有其他协程正在写，就可以接收数据。若无其他协程发送数据时，就会进入`channel`的接收阻塞队列。

4. **非阻塞式读**：`select case default`

如果发生阻塞的情况，就会执行`default`分支。

> #### 问：`多路 select`？

**多路`select`**是指存在两个或更多的`case`分支，每个分支可以是一个`channel`的`send`或`recv`操作。

`select`会将所有的`channel`记录到链表，`send channel`在前，`recv channel`在后。当`select`开始执行时：

1. **按序加锁：**首先，会按照**有序的加锁顺序**对所有的`channel`进行加锁。这是为了保障`channel`的原子性，**有序是为了避免死锁**；
2. **乱序轮询：**然后会按照**乱序的轮询顺序**，检查所有`channel`的等待队列和缓冲区，来确定执行哪个`case`分支。**乱序是为了保证公平性**。
3. **挂起等待：**如果轮询完毕没有发现可以执行的`case`分支，那么会将`goroutine`添加到所有`channel`的阻塞队列中，然后挂起等待。
4. **按序解锁：**最后会按序将所有`channel`解锁。
5. **唤醒执行：**假如现在有数据可读/写了，`goroutine`就会被唤醒，执行相应`case`。
6. **离开队列：**`case`执行完毕后，会对所有`channel`加锁，并将`goroutine`从所有的`channel`的阻塞队列中删除。
7. **解锁返回：**然后解锁所有`channel`并返回。

## 3.9 内存

> #### 问：内存逃逸？:star::triangular_flag_on_post:

编译器会根据变量**是否被外部引用**来决定是否逃逸：

- 如果函数外部没有引用，则优先放到栈中；
- 如果函数外部存在引用，则必定放到堆中；
- 如果栈上放不下，则必定放到堆上；

**案例：**

- **指针逃逸：**函数返回值为局部变量的指针，函数虽然退出了，但是因为指针的存在，指向的内存不能随着函数结束而回收，因此只能分配在堆上。
- **栈空间不足：**当栈空间足够时，不会发生逃逸，但是当变量过大时，已经完全超过栈空间的大小时，将会发生逃逸到堆上分配内存。局部变量s占用内存过大，编译器会将其分配到堆上；
- **变量大小不确定：**编译期间无法确定slice的长度，这种情况为了保证内存的安全，编译器也会触发逃逸，在堆上进行分配内存；
- **动态类型：**动态类型就是编译期间不确定参数的类型、参数的长度也不确定的情况下就会发生逃逸；
- **闭包引用对象：**闭包函数中局部变量i在后续函数是继续使用的，编译器将其分配到堆上；
- **反射：**`ValueOf`。

> #### 问：make和new有啥区别？

- **两者的作用类型不同：**new给`int`、`string`、数组分配内存，make给`slice`、`map`、`channel`分配内存；
- **两者的返回值不同：**new的返回值类型为一个指向新分配好的内存空间的一个指定类型指针。而make的返回值类型为它本身。

```go
hash := make(map[int]bool, 10)	// hash 是一个指向 runtime.hmap 结构体的指针
ch := make(chan int, 5)			// ch 是一个指向 runtime.hchan 结构体的指针
```

- new分配的内存空间会被清零，make分配空间之后会被初始化。
- new分配的内存空间不一定会在堆上分配，比如说该指针就在本函数内使用。

**若用`ps := new([]string)`初始化，new 是不负责底层数组的分配的，仅仅返回slice的起始地址，此时这个slice还没有底层数组，如果对其进行赋值，就会出错。**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230405224743545.png" alt="image-20230405224743545" style="zoom:33%;" />

**需要通过Append进行分配底层数组。**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230405224814298.png" alt="image-20230405224814298" style="zoom:33%;" />

> #### 问：为什么要内存对齐？

首先，CPU 访问内存时，并不是逐个字节访问，而是以字长（word size）为单位访问。比如 32 位的 CPU ，字长为 4 字节，那么 **CPU 访问内存的单位也是 4 字节**。

**有两个目的**：

- **减少访存次数**；
- **便于原子性操作**。

**减少 CPU 访问内存的次数，加大 CPU 访问内存的吞吐量**。比如：

- 当内存对齐时，读取 b 只需要读取一次内存。
- 非内存对齐时，读取 b 需要两次访存([0, 3]，[4, 7]，然后拼接出 b)：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/memory_alignment.png" alt="memory alignment" style="zoom:50%;" />

**内存对齐对实现变量的原子性操作也是有好处的**，**每次内存访问是原子的**，如果变量的大小不超过字长，那么**内存对齐后，对该变量的访问就是原子的**，这个特性在并发场景下至关重要。

> #### 问：结构体是怎么进行内存对齐的？成员的顺序不同，结构体的大小会不同吗？

```go
type demo1 struct {
	a int8
	b int16
	c int32
}

type demo2 struct {
	a int8
	c int32
	b int16
}

func main() {
	fmt.Println(unsafe.Sizeof(demo1{})) // 8
	fmt.Println(unsafe.Sizeof(demo2{})) // 12
}
```

==**结构体内存对齐规则**==：

- 每个字段按照自身的对齐倍数来确定在内存中的偏移量，对齐倍数 = `min(自身的长度，机器字长)`；
- 排列完成后，需要对结构体整体再进行一次内存对齐。
- 成员的顺序不同，结构体的大小可能不同，因此结构体的内存对齐有技巧。

**分析上面的代码示例**(机器字长 32 位)，**首先是 demo1**：

- a 是第一个字段，默认是已经对齐的，从第 0 个位置开始占据 1 字节。
- b 是第二个字段，对齐倍数为 2，因此，必须空出 1 个字节，偏移量才是 2 的倍数，从第 2 个位置开始占据 2 字节。
- c 是第三个字段，对齐倍数为 4，此时，内存已经是对齐的，从第 4 个位置开始占据 4 字节即可。

因此 demo1 的内存占用为 **8 字节**。

**其次是 demo2**：

- a 是第一个字段，默认是已经对齐的，从第 0 个位置开始占据 1 字节。
- c 是第二个字段，对齐倍数为 4，因此，必须空出 3 个字节，偏移量才是 4 的倍数，从第 4 个位置开始占据 4 字节。
- b 是第三个字段，对齐倍数为 2，从第 8 个位置开始占据 2 字节。

**demo2 的对齐倍数由 c 的对齐倍数决定**，也是 4，因此，demo2 的内存占用为 **12 字节**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/memory_alignment_order.png" alt="memory alignment" style="zoom: 50%;" />

**==额外的问题==**：空 `struct{}` 的对齐

**空 `struct{}` 大小为 0**，**作为其他 struct 的字段时**，**一般不需要内存对齐**。但是**有一种情况除外**：即**当 `struct{}` 作为结构体最后一个字段时，需要内存对齐**。**因为如果有指针指向该字段，返回的地址将在结构体之外**，如果此指针一直存活不释放对应的内存，就会有内存泄露的问题（该内存不因结构体释放而释放）。

因此，当 `struct{}` 作为其他 struct 最后一个字段时，需要**填充额外的内存**保证安全。我们做个试验，验证下这种情况。

```go
type demo3 struct {
	c int32
	a struct{}
}

type demo4 struct {
	a struct{}
	c int32
}

func main() {
	fmt.Println(unsafe.Sizeof(demo3{})) // 8
	fmt.Println(unsafe.Sizeof(demo4{})) // 4
}
```

可以看到，`demo4{}` 的大小为 4 字节，与字段 c 占据空间一致，而 `demo3{}` 的大小为 8 字节，即额外填充了 4 字节的空间。

**另**，没有任何字段的空 struct{} 和没有任何元素的 array 占据的内存空间大小为 0，**不同的大小为 0 的变量可能指向同一块地址**。

## 3.10 数据结构

> #### 问：map 是并发安全的吗？那怎么让他并发安全？那加锁的 map 和 sync.map 有啥区别？sync.map 的适用场景？:star::triangular_flag_on_post:

普通的`map`不是并发安全的，`sync.map`是并发安全的。

---

如何让普通的`map`并发安全，一般的思路有两种：

- **加锁**，操作前先获取锁，操作完释放。缺点是：粒度太大，效率不高；
- **划分成几个小的 `map`**，只操作相应小的 `map`。缺点是：实现复杂，容易出错。

---

先看看`sync.map`：

```go
type Map struct {
    mu Mutex						// 保护 dirty 和 read
    read atomic.Value 				// readOnly，只读，所以是并发安全的
    dirty map[interface{}]*entry	// 一个非线程安全的原始 map
    misses int						// 计数作用。每次从read中读失败，则计数+1
}
```

**map+锁 和 sync.map 的对比**：

- 自己实现的加锁的Map，每次操作都需要先获得锁，这就造成极大的性能浪费；
- `sync.map`中有**只读数据`read`、锁`mu`、普通的Map`dirty`、计数器`misses`**。其中，执行增删查改前，都会先对`read`进行操作，因为这是只读的，支持原子操作，也就支持并发访问。这就会**省去很大一部分加锁解锁带来的开销**。

**性能测试结论**：

- **插入元素**：SyncMapStore < MapStore < RwMapStore；
- **查找元素**：MapLookup < RwMapLookup < SyncMapLookup；
- **删除元素**：RwMapDelete < MapDelete < SyncMapDelete

**测试如下**：

```go
❯ go test -bench=.
goos: windows
goarch: amd64
pkg: goLearn/sync/08-sync.map
cpu: 12th Gen Intel(R) Core(TM) i7-12700K
BenchmarkBuiltinMapStoreParalell-20             15114511                81.30 ns/op
BenchmarkBuiltinRwMapStoreParalell-20           16176103                83.39 ns/op
BenchmarkSyncMapStoreParalell-20                 8515656               171.0 ns/op

BenchmarkBuiltinMapLookupParalell-20            21025113                61.02 ns/op
BenchmarkBuiltinRwMapLookupParalell-20          26539101                45.61 ns/op
BenchmarkSyncMapLookupParalell-20               285766286                4.159 ns/op

BenchmarkBuiltinMapDeleteParalell-20            20315636                60.36 ns/op
BenchmarkBuiltinRwMapDeleteParalell-20          18308758                69.12 ns/op
BenchmarkSyncMapDeleteParalell-20               296166151                4.112 ns/op
PASS
ok      goLearn/sync/08-sync.map        13.145s
```

**场景分析**：

- `sync.map`的读和删，性能非常好，领先 map+锁；
- 写，性能非常差，落后于 map+锁。

**`sync.map`的适用场景：大量读，少量写**。这是因为，**sync.map**本质上是利用了**==读写分离==**，如果是**大量写**的场景，会导致`missess`一直增加，也就会**一直触发`dirty`晋升为`read`**，导致性能**反而不如普通的Map+锁**。

---

**思维发散：**

可以对大Map进一步划分，分成很多个小Map，单独对每部分加锁解锁，也就是细化锁的粒度。只是实现起来太麻烦了。

> #### 问：map中，key可以为nil吗？:star::triangular_flag_on_post:

需要分情况讨论。

- `nil`是`interface{}`、`chan`、`map`等类型的**零值**。因此，如果map的key的类型是`interface{}`，那么key可以为nil，就跟空接口效果一样；
- 如果是其他类型，nil是肯定不行的，但是对应类型的零值是ok的。

> #### 问：map中，func 可以做key吗？什么类型能做？为什么？:star::triangular_flag_on_post:

**不可以。**必须是可比较类型才能做key。那为什么要可比较的类型才能做key呢？不是直接通过 top 8 位就能拿到吗？

因为查找 key 的过程是这样的：当出现哈希冲突，就会到同一个桶中寻找，通过 top 8 位找到对应的 key'，用 key' 和 key 进行**对比**，如果一样，就可以拿到对应的 value。

因为还需要一次 **对比**！所以当然要可比较。。。

> #### 问：[]byte{} 和 string 怎么转换？性能？原理？各自的适用场景？:star::triangular_flag_on_post:

共两种方法：

- **标准转换**

```go
s1 := "hello"
b := []byte(s1)
s2 := string(b)
```

- **强制转换**

```go
func String2Bytes(s string) []byte {
	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
	bh := reflect.SliceHeader{
		Data: sh.Data,
		Len:  sh.Len,
		Cap:  sh.Len,
	}
	return *(*[]byte)(unsafe.Pointer(&bh))
}

func Bytes2String(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

**强转换方式的性能会明显优于标准转换。**

==**可以延伸如下问题**==：

- **为啥强转换性能会比标准转换好？**

对于标准转换，无论是从[]byte转string还是string转[]byte都会涉及底层数组的拷贝。而强转换是直接替换指针的指向，从而使得string和[]byte指向同一个底层数组。这样，当然后者的性能会更好。

- **为啥当x的数据较大时，标准转换方式会有一次分配内存的操作，从而导致其性能更差，而强转换方式却不受影响？**

标准转换时，当数据长度大于32个字节时，需要通过mallocgc申请新的内存，之后再进行数据拷贝工作。而强转换只是更改指针指向。所以，当转换数据较大时，两者性能差距会愈加明显。

- **既然强转换方式性能这么好，为啥go提供给我们使用的是标准转换方式？**

首先，我们需要知道Go是一门类型安全的语言，而安全的代价就是性能的妥协。但是，性能的对比是相对的，这点性能的妥协对于现在的机器而言微乎其微。另外强转换的方式，会给我们的程序带来极大的安全隐患。**如下代码会直接报错**：

```go
func main() {
	s := "hello"
	b := String2Bytes(s)
	fmt.Println(b)
	b[0] = 'l'
	fmt.Println(s)
}

unexpected fault address 0x6c47f8
fatal error: fault
[signal 0xc0000005 code=0x1 addr=0x6c47f8 pc=0x6ab5aa]
```

**s是string类型，是不可修改的**。通过强转换将s的底层数组赋给b，而b是一个[]byte类型，它的值是可以修改的，所以这时对底层数组的值进行修改，将会造成严重的错误（通过defer+recover也不能捕获）。

- **string为啥要设计成不可修改的？**

string不可修改，意味它是只读属性，这样的好处就是：在并发场景下，我们可以在不加锁的控制下，多次使用同一字符串，在保证高效共享的情况下而不用担心安全问题。

**因此，对于这两种方法的适用场景，有如下参考**：

1. 在你不确定安全隐患的条件下，尽量采用**标准方式进行数据转换**。
2. 当**程序对运行性能有高要求**，同时满足对数据**仅仅只有读操作**的条件，且**存在频繁转换（例如消息转发场景）**，可以使用**强转换**。

> #### 问：两种方法的转换原理？

```go
// slice 的底层数据结构
type slice struct {
	array unsafe.Pointer	// 指向的是某个数组的首地址
	len   int
	cap   int
}

// string 的底层数据结构
type stringStruct struct {
	str unsafe.Pointer	  // 指向的是某个数组的首地址
	len int
}
```

1. **标准转换**

```go
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte

func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
	}
	copy(b, s)
	return b
}

// rawbyteslice allocates a new byte slice. The byte slice is not zeroed.
func rawbyteslice(size int) (b []byte) {
	cap := roundupsize(uintptr(size))
	p := mallocgc(cap, nil, false)
	if cap != uintptr(size) {
		memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size))
	}

	*(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
	return
}
```

go 需要调用 mallocgc 分配一块新的内存（大小由s决定），然后进行拷贝。

2. **强制转换**

需要从两方面进行讨论：

- **万能的 `unsafe.Pointer` 指针**

在 go 中，**任何类型的指针 `*T` 都可以转换为 `unsafe.Pointer` 类型的指针**，它可以存储任何变量的地址。同时， `unsafe.Pointer` 类型的指针也可以转换回普通指针，而且可以不必和之前的类型 `*T` 相同。另外，`unsafe.Pointer` 类型还可以转换为 `uintptr` 类型，该类型保存了指针所指向地址的数值，从而可以使我们对地址进行数值计算。以上就是强转换方式的实现依据。

**而 `string` 和 `slice` 在 `reflect` 包中，对应的结构体是 `reflect.StringHeader` 和 `reflect.SliceHeader`**，它们是 `string` 和 `slice` 的运行时表达。

```go
type StringHeader struct {
	Data uintptr
	Len  int
}

type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

- **内存布局**

从 `string` 和 `slice` 的运行时表达可以看出，除了 `SilceHeader` 多了一个 `int` 类型的 `Cap` 字段，`Date` 和 `Len` 字段是一致的。所以，它们的**内存布局是可对齐的**，这说明我们就可以直接通过 `unsafe.Pointer` 进行转换。

## 3.11 结构体和接口

#### 基础

> #### 问：Go 如何实现 多态？原理是什么？:star::triangular_flag_on_post:

**如何实现**：

多态使内部结构不同的对象可以共享相同的外部接口。

Go 通过**接口**实现多态：

- **某个类型**若实现了接口的所有方法，就隐式的实现了该接口；
- **某个类型的对象**可以赋给它所实现的任意接口类型的变量；

(网上没找到啥好的讲解，自己尝试写个吧…)

**原理**：

首先要明确，每个类型在底层都会有一个类型元数据，存放着类型信息和方法元数据列表。

- 然后，**定义一个非空接口，并让某类型实现该接口**；

- 在多态的场景下，比如一个函数的入参是定义的非空接口类型，调用该函数时传参为**具体类型的对象**，其实就会把**该具体类型的对象赋值给该非空接口**。这里要了解，非空接口的底层结构体里共有两个字段，一个是`itab`，一个是`data`：
  - `itab`中会记录**接口的类型元数据(包含接口的方法列表)**、**动态类型元数据**、**动态类型实现的(接口所需要的)方法的地址列表`fun`**；
  - `data`为**动态值**，指向具体类型的对象。
- 因此，**赋值过程**就是把`data`指向具体类型的对象，修改`itab`中的**动态类型元数据为具体类型的类型元数据**，并从**该类型元数据**中的**方法元数据列表**拷贝方法地址到`fun`。

给出一段示例代码：

```go
type test interface {
	print()
}

type hello struct {
}

func (h hello) print() {
	fmt.Println("hello")
}

// 多态场景
func hi(t test) {
	t.print()
}

func main() {
	h := hello{}
	// 第一种调用方式
	var t test
	t = h
	t.print()
	// 第二种
	hi(h)
}
```

> #### 问：Go 如何实现 组合 和 继承？

 Go 语言中没有继承的概念，更提倡的是**组合**。**接口**和**结构体**都可以组合。

1. **接口的组合**，如下代码所示：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

//ReadWriter是Reader和Writer的组合
type ReadWriter interface {
    Reader
    Writer
}
```

ReadWriter 接口就是 Reader 和 Writer 的组合，组合后的 **ReadWriter 接口具有 Reader 和 Writer 中的所有方法**，这样新接口 ReadWriter 就不用定义自己的方法了，组合 Reader 和 Writer 的就可以了。

2. **结构体的组合**，如下代码所示：

```go
type person struct {
    name string
    age uint
    address	
}
```

**直接把结构体类型放进来，就是组合，不需要字段名。**组合后，被组合的 address 称为**内部类型**，person 称为**外部类型**：

- **外部类型**不仅可以使用**内部类型的字段**，也可以使用**内部类型的方法**，就像使用自己的方法一样；
- 如果外部类型定义了和内部类型同样的方法，那么外部类型的会覆盖内部类型，这就是**方法的覆写**。**方法覆写不会影响内部类型的方法实现**。

如下所示：

```go
p:=person{
        age:30,
        name:"飞雪无情",
        address:address{
            province: "北京",
            city:     "北京",
        },
    }
//像使用自己的字段一样，直接使用
fmt.Println(p.province)
```

因为 person 组合了 address，所以 address 的字段就像 person 自己的一样，可以**直接使用**。

#### 题目：1

```go
type People struct{}

func (p *People) ShowA() {
	fmt.Println("showA")
	p.ShowB()
}

func (p *People) ShowB() {
	fmt.Println("showB")
}

type Teacher struct {
	People
}

func (t *Teacher) ShowB() {
	fmt.Println("teacher showB")
}

func main() {
	t := Teacher{}
	t.ShowA()
}
```

这里定义了两个类型，类型`Teacher`中嵌入了类型`People`。**问题是：**通过`Teacher`类型的变量`t`调用方法`showA()`，输出结果是什么？

**答案：**

```go
showA
showB
```

**解释：**

首先要明确`T`和`*T`是两种类型，分别对应自己的类型元数据，有着各自的方法集，其中包含了自定义的方法以及编译器生成的方法。通过`go tool compile -l -p main main.go`可以生成`main.go`的`obj`文件，再通过`go tool nm main.o`就可以查看方法列表：

```go
❯ go tool nm main.o | grep 'T' | grep People
    301c T main.(*People).ShowA
    308a T main.(*People).ShowB
❯ go tool nm main.o | grep 'T' | grep Teacher
    36d9 T main.(*Teacher).ShowA
    30e0 T main.(*Teacher).ShowB
```

可以发现，`People`和`*People`的方法集同代码中定义的一致，但`Teacher`和`*Teacher`相关的方法列表中多了一个`ShowA()`方法，这就是编译器自动生成的了，所以编译器为`*Teacher`生成了这样一个包装方法(红字部分)：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230131160051014.png" alt="image-20230131160051014" style="zoom:50%;" />

所以，调用过程为：`t.ShowA()`会在语法糖的作用下转换为对`(*Teacher).ShowA()`方法的调用，而它又会取出`People`成员的地址作为接收者去执行`*People`这里的`ShowA()`方法，所以会有如上输出。

#### 题目：2

解释到这里，这道题已经解决了。但无法知道**为什么编译器只给`*Teacher`生成了包装方法？**为此探索一下**编译器生成包装方法的规则**。

```go
type A int

func (a A) Value() int {
	return int(a)
}

func (a *A) Set(n int) {
	*a = A(n)
}

type B struct {
	A
	b int
}

type C struct {
	*A
	c int
}
```

分析以上`B`和`C`会分别继承哪些方法。

首先，值接收者`A`有`Value()`方法，前面已经讲过，为了支持接口，编译器会为值接收者方法生成指针接收者的包装方法，所以`*A`会有`Value()`和`Set()`方法；`B`和`C`拥有的方法如下：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230131161145152.png" alt="image-20230131161145152" style="zoom:50%;" />

其中，只有`B`只继承了`Value()`方法，这是因为以`B`为接收者调用方法时，方法操作的已经是`B`的副本，无法获取嵌入的`A`的原始地址；而`*A`的方法从语义上来讲需要操作原始变量，也就是说，对于`B`而言它继承`*A`的方法是没有意义的，所以编译器并没有给`B`生成`Set()`方法。

**结论就是：**无论是嵌入值还是嵌入指针，值接收者方法始终能够被继承；而只有在能够拿到嵌入对象的地址时，才能继承指针接收者方法。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230131162321168.png" alt="image-20230131162321168" style="zoom:50%;" />

## 3.12 对比 Java

> #### 问：Go 和 Java 的区别？

1. **性能**：无论在串行还是并发的的业务下，Java的性能都比Go差；

- goroutine 默认占用内存远比 Java 、C 的线程少(goroutine：2KB，线程：8MB)，占用资源更少；
- goroutine 可以避免内核态和用户态的切换导致的成本。

2. **部署编译**：

- Java通过虚拟机编译，使用JVM跨平台编译；
- Go语言针对不同的平台，编译对应的机器码。

3. **访问权限**：

- Java使用public、protected、private、默认等几种修饰符来控制访问权限；
- Go语言通过大小写控制包外可访问还是不可访问。Go语言中根据首字母的大小写来确定可以访问的权限。无论是方法名、常量、变量名还是结构体的名称，如果首字母大写，则可以被其他的包访问；如果首字母小写，则只能在本包中使用。可以简单的理解成，首字母大写是公有的，首字母小写是私有的。

4. **接口**

- Java等面向对象编程的接口是侵入式接口，需要明确声明自己实现了某个接口；
- Go语言的非侵入式接口不需要通过任何关键字声明类型与接口之间的实现关系，只要一个类型实现了接口的所有方法，那么这个类型就是这个接口的实现类型。

5. **异常处理**

- Java中错误（Error）和异常(Exception)被分类管理，二者的区别是：
  - Error（错误）：程序在执行过程中所遇到的硬件或操作系统的错误。错误对程序而言是致命的，将导致程序无法运行。常见的错误有内存溢出，jvm虚拟机自身的非正常运行，calss文件没有主方法。程序本生是不能处理错误的，只能依靠外界干预。Error是系统内部的错误，由jvm抛出，交给系统来处理；
  - EXCEPTION（异常）：是程序正常运行中，可以预料的意外情况。比如数据库连接中断，空指针，数组下标越界。异常出现可以导致程序非正常终止，也可以预先检测，被捕获处理掉，使程序继续运行。
- Go语言中只有error，一旦发生错误逐层返回，直到被处理。Golang中引入两个内置函数panic和recover来触发和终止异常处理流程，同时引入关键字defer来延迟执行defer后面的函数。golang弱化了异常，只有错误，在意料之外的panic发生时，在defer中通过recover捕获这个恐慌，转化为错误以code,message的形式返回给方法调用者，调用者去处理，这也是go极简的精髓。

6. **继承**

- Java的继承通过extends关键字完成，不支持多继承；
- Go语言的继承通过匿名组合完成：基类以Struct的方式定义，子类只需要把基类作为成员放在子类的定义中，支持多继承。

7. **多态**

- Java的多态，必须满足继承，重写，向上转型；任何用户定义的类型都可以实现任何接口，所以通过不同实体类型对接口值方法的调用就是多态。

- 在Go语言中通过接口实现多态，对接口的实现只需要某个类型T实现了接口中的方法，就相当于实现了该接口。

8. **指针**

- Java中不存在显式的指针，而Golang中存在显式的指针操作，使用 * 来定义和声明指针，通过&来取得对象的指针。注意，Java和Golang都是只存在值传递。

9. **并发**

- 在Java中，通常借助于共享内存（全局变量）作为线程间通信的媒介，通常会有线程不安全问题，使用了加锁（同步化）、使用原子类、使用volatile提升可见性等解决；
- 但在Go语言中使用的是通道（Channel）作为协程间通信的媒介，这也是Go语言中强调的：**不要通过共享内存通信，而通过通信来共享内存**。

10. **垃圾回收和内存管理机制**

- Java基于JVM虚拟机的分代收集算法完成GC，Go语言内存释放是语言层面，对不再使用的内存资源进行自动回收，使用三色标记算法；

