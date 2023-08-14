# 

Protobuf 的编码是基于变种的 Base 128。因此，按照 `Base 64 --> Base 128 --> Protobuf` 的顺序来学习比较好~

[原文链接](https://mp.weixin.qq.com/s/enDUynhZ1Pnzg_4xEjR21A)

## 一、Base 64

计算机之间传输数据时，数据本质上是一串字节流。由于不同机器采用的字符集不同等原因，我们并不能保证目标机器能够正确地“理解”字节流。

先看看 Base 64 如何工作，假设这里有 **4 个字节**，代表要传输的二进制数据：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230405220001528.png" alt="image-20230405220001528" style="zoom:50%;" />

1. 首先将这个字节流按**每 6 个 bit 为一组**进行分组，**剩下少于 6 bits 的低位补 0**；

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230405220053643.png" alt="image-20230405220053643" style="zoom: 50%;" />

2. 然后在**每一组 6 bits 的高位补两个 0**；

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230405220134940.png" alt="image-20230405220134940" style="zoom:50%;" />

3. 对照 Base 64 table，字节流可以用 `ognC0w` 来表示。另外，Base 64 编码是按照 6 bits 为一组进行编码，每 3 个字节的原始数据要用 4 个字节来储存，**编码后的长度要为 4 的整数倍**，**不足 4 字节的部分要使用 pad 补齐**，所以最终的编码结果为`ognC0w==`。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230405220203286.png" alt="image-20230405220203286" style="zoom: 40%;" />

Base 64 编码之后所有字节**均可以用数字、字母、+、/、= 进行表示**，这些都是可以被**正常显示**的 ascii 字符，即**“安全”的字节**。绝大部分的计算机和操作系统都对 ascii 有着良好的支持，**保证了编码之后的字节流能被正确地复制、传播、解析**。

**Base 64 存在的问题就是**：编码后的每一个字节的**最高两位总是 0**，在不考虑 pad 的情况下，**有效 bit 只占 bit 总数的 75%**，造成大量的空间浪费。

## 二、Base 128

因此，**Base 128** 的大致实现思路是：**将字节流按 7 bits 进行分组，然后低位补 0**。

但问题来了，**Base 64 实际上用了 2^6^+1 个 ascii 字符**，按照这个思路 **Base 128 需要使用 2^7^+1 个 ascii 个字符**，但是 **ascii 字符一共只有 128 个**。另外，即使不考虑 pad，ascii 中包含了一些**不可以正常打印的控制字符**，编码之后的字符还可能包含会被不同操作系统转换的换行符号(10 和 13)。因此，**Base 64 至今依然没有被 Base 128 替代**。

## 三、Base 128 Varints

**Protocol Buffers** 所用的编码方式就是 Base 128 Varints。(按照小端模式讨论)

#### 编码/解码

对于编码后的每个字节，**低 7 位用于储存数据**，**最高位**用来标识当前字节是否是当前整数的最后一个字节，称为**最高有效位**（most significant bit, msb）。msb 为 1 时，代表着后面还有数据；msb 为 0 时代表着当前字节是当前整数的最后一个字节。

1. 如何将使用 Base 128 Varints 对**整数**进行**编码**：

- 将数据按每 7 bits 一组拆分
- 逆序每一个组(因为小端)
- 添加 msb

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230413213832575.png" alt="image-20230413213832575" style="zoom:33%;" />



2. 如何将使用 Base 128 Varints 对**整数**进行解码：

- 去除 msb
- 将字节流逆序(msb 为 0 的字节储存原始数据的高位部分，小端模式)
- 最后拼接所有的 bits。

`300 (0b 10 0101100)`编码：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230413214205518.png" alt="image-20230413214205518" style="zoom:33%;" />

解码：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230413214133131.png" alt="image-20230413214133131" style="zoom:33%;" />



protobuf 的 varints **最多可以编码 8 字节的数据**，这是因为绝大部分的现代计算机最高支持处理 64 位的整型。

## 四、Protobuf

**Protocol Buffers** 所用的编码方式就是 Base 128 Varints。(按照小端模式讨论)

#### 数据类型

protobuf 支持的数据类型(**`wire type`**)：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230413214619993.png" alt="image-20230413214619993" style="zoom: 67%;" />

当实际使用 protobuf 进行编码时，经过了两步处理：

- 将 **编程语言的数据结构** 转化为 **`wire type`**。
- 根据不同的 `wire type` 使用对应的方法编码。前文所提到的 Base 128 Varints 用来编码 varint 类型的数据，其他 `wire type` 则使用其他编码方式。

部分数据类型到 `wire type` 的转换规则：

- ##### 有符号整型

采用 ZigZag 编码来将 sint32 和 sint64 转换为 wire type 0。下面是 ZigZag 编码的规则（注意是算术位移）：

```c
 n * 2    	// when n >= 0
-n * 2 - 1 	// when n < 0
```

一些例子：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230413215613221.png" alt="image-20230413215613221" style="zoom: 50%;" />

- ##### 定长数据(64-bit)

直接采用小端模式储存，不作转换。

- ##### 字符串

以字符串`"testing"`为例：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230413215946810.png" alt="image-20230413215946810" style="zoom:33%;" />

编码后的 value 分为两部分：

- 蓝色，表示字符串采用 UTF-8 编码后字节流的长度(bytes)，采用 Base 128 Varints 进行编码。
- 白色，字符串用 UTF-8 编码后的字节流。

#### 消息结构

Protobuf 采用 proto3 作为 DSL 来描述其支持的消息结构。

```protobuf
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

设想一下这样一个场景：数据的发送方在业务迭代之后需要在消息内携带**更多的字段**，而有的接收方**并没有更新**自己的 proto 文件。要保持较好的兼容性，**接收方需要分辨出哪些字段是自己可以识别的，哪些是不能识别的新增字段**。要做到这一点，发送方在编码消息时还必须**附带每个字段的 key**，**客户端读取到未知的 key 时，可以直接跳过对应的 value**。

proto3 中每一个字段后面都有一个 `= x`，比如：

```protobuf
  string query = 1;
```

这里的等号并不是用于赋值，而是给每一个字段指定一个 `ID`，称为 **`field number`**。**消息内同一层次字段的 `field number` 必须各不相同**。

上面所说的 key，在 protobuf 源码中被称为 `tag`。**`tag` 由 `field number` 和 `type` 两部分组成**：

- `field number` 左移 3 bits
- 在最低 3 bits 写入 `wire type`

一个生成 `tag` 的例子：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230413220725664.png" alt="image-20230413220725664" style="zoom:50%;" />

Go 版本 Protobuf 中生成 tag 的源码：

```go
func EncodeTag(num Number, typ Type) uint64 {
    return uint64(num)<<3 | uint64(typ&7)
}
```

源码中生成的 `tag` 是 uint64，代表着 `field number` 可以使用 61 个 bit 吗？**并非如此。**事实上，**==`tag` 的长度不能超过 32 bits==**，意味着 `field number` 的最大取值为 2^29^-1 (536870911)。而且在这个范围内，**有一些数是不能被使用的**：

- `0`：protobuf 规定 `field number` 必须为正整数。
- `19000 到 19999`： protobuf 仅供内部使用的保留位。

理解了生成 tag 的规则之后，不难得出以下结论：

- `field number` 不必从 1 开始，可以从合法范围内的任意数字开始。
- 不同字段间的 `field number` 不必连续，只要合法且不同即可。

**当修改 proto 文件时，需要注意**:star:：

- **`field number` 一旦被分配了就不应该被更改**，除非你能保证所有的接收方都能更新到最新的 proto 文件；
- **由于 `tag` 中不携带 `field name` 信息，更改 `field name` 并不会改变消息的结构**。发送方认为的 apple 到接受方可能会被识别成 pear。双方把字段读取成哪个名字完全由双方自己的 proto 文件决定，**只要字段的 `wire type` 和 `field number` 相同即可**。

---

**最后再来个复杂例子(能看明白就算会了~)**：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230413221828128.png" alt="image-20230413221828128" style="zoom:50%;" />

#### 嵌套消息

`wire type 2`不仅支持 string，也支持 embedded messages。

对于嵌套消息：

- 首先要**将被嵌套的消息进行编码成字节流**；
- 然后就可以像处理 UTF-8 编码的字符串一样处理这些字节流：在字节流前面加入使用 Base 128 Varints 编码的长度即可。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230413222243688.png" alt="image-20230413222243688" style="zoom:50%;" />

能看明白就是胜利！:wink:

#### 字段顺序

Proto 文件中定义字段的顺序与最终编码结果的字段顺序无关，两者有可能相同也可能不同。

**任何 Protobuf 的实现都应该保证字段以任意顺序编码的结果都能被解码。**

#### 安全性

由于 Protobuf 序列化后就是一堆字节流，需要有原 Proto 声明文件才能反序列化，因此也具备一定的保密性。

#### 对比JSON、XML

- XML、JSON 更注重数据结构化，关注人类可读性和语义表达能力；
- ProtoBuf 更注重数据序列化，关注效率、空间、速度，人类可读性差，语义表达能力不足（为保证极致的效率，会舍弃一部分元信息）

ProtoBuf 的应用场景更为明确，XML、JSON 的应用场景更为丰富。

