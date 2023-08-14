# 

[toc]

## 一、Introduction

### Lab

课程总共有 4 个 Lab：

- [ ] Lab 1：MapReduce
- [ ] Lab 2：Raft
- [ ] Lab 3：K/V Server
- [ ] Lab 4：Shared K/V Service

### 分布式系统特性

- 可扩展性
  - 两台计算机构成的系统如果有两倍性能或者吞吐量，就具备很好的可扩展性；
- 可用性
  - 容错性、可恢复性
  - 两种解决方法：持久化、复制
- 一致性
  - 主要是从成本考虑，强一致性太昂贵了，可能需要做很多的通信，比如访问所有的节点，找到最新的信息。
  - 强一致性
  - 弱一致性

### 实验环境

需要在Linux下：

```bash
# 安装 Go 环境
wget -qO- https://go.dev/dl/go1.18.linux-amd64.tar.gz | sudo tar xz -C /usr/local

# 添加环境变量
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
export GOPATH=$HOME/Project/Go

go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

尝试运行示例：

```bash
cd 6.824
cd src/main

# 构建 MR APP 的动态链接库
go build -race -buildmode=plugin ../mrapps/wc.go

# 运行 MR
rm mr-out*
go run -race mrsequential.go wc.so pg*.txt

# 查看结果
more mr-out-0
```



## 二、MapReduce

### 相关背景

在 20 世纪初，包括本文作者在内的 Google 的很多程序员，为了处理海量的原始数据，已经实现了数以百计的、专用的计算方法。这些计算方法用来处理大量的原始数据，比如，文档抓取（类似网络爬虫的程序）、Web 请求日志等等；也为了计算处理各种类型的衍生数据，比如倒排索引、Web 文档的图结构的各种表示形势、每台主机上网络爬虫抓取的页面数量的汇总、每天被请求的最多的查询的集合等等。

### 要解决的问题

大多数以上提到的数据处理运算在概念上很容易理解。然而由于输入的数据量巨大，因此要想在可接受的时间内完成运算，只有将这些计算分布在成百上千的主机上。如何处理并行计算、如何分发数据、如何处理错误？所有这些问题综合在一起，需要大量的代码处理，因此也使得原本简单的运算变得难以处理。

### 解决方法

#### 模型

为了解决上述复杂的问题，本文设计一个新的抽象模型，使用这个抽象模型，用户只要表述想要执行的简单运算即可，而不必关心并行计算、容错、数据分布、负载均衡等复杂的细节，这些问题都被封装在了一个库里面：利用一个输入 key/value pair 集合来产生一个输出的 key/value pair 集合。

MapReduce 库的用户可以用**两个函数表达这个计算：Map 和 Reduce**。

- 用户自定义的 Map 函数接受一个输入的 key/value pair 值，然后产生一个**中间 key/value pair 值的集合**。MapReduce 库把所有具有相同中间 key 值 I 的中间 value 值集合在一起后传递给 reduce 函数。
- 用户自定义的 Reduce 函数接受一个中间 key 的值 I 和相关的一个 value 值的集合。Reduce 函数合并这些 value 值，形成一个较小的 value 值的集合。一般的，每次 Reduce 函数调用只产生 0 或 1 个输出 value 值。通常 Map 通过一个迭代器把中间 value 值提供给 Reduce 函数，这样 Reduce Worker 就可以处理无法全部放入内存中的大量的 value 值的集合。

在概念上，用户定义的 Map 和 Reduce 函数都有相关联的类型：

```
map(k1, v1) -> list(k2, v2)
reduce(k2, list(v2)) -> list(v2)
```


比如，输入的 key 和 value 值与输出的 key 和 value 值在类型上推导的域不同。此外，中间 key 和 value 值与输出 key 和 value 值在类型上推导的域相同。

#### 执行流程

通过将 Map 调用的输入数据自动分割为 M 个数据片段的集合，Map 调用被分布到多台机器上执行。输入的数据片段能够在不同的机器上并行处理。使用分区函数将 Map 调用产生的中间 key 值分成 R 个不同分区（例如，hash(key) mod R），Reduce 调用也被分布到多台机器上执行。分区数量（R）和分区函数由用户来指定。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230515175400311.png" alt="image-20230515175400311" style="zoom:33%;" />

上图展示了 MapReduce 实现中操作的全部流程。当用户调用 MapReduce 函数时，将发生下面的一系列动作：

1. 用户程序首先调用的 MapReduce 库将输入文件分成 M 个数据片度，每个数据片段的大小一般从 16MB 到 64MB（可以通过可选的参数来控制每个数据片段的大小）。然后用户程序在机群中创建大量的程序副本。
2. 这些程序副本中的有一个特殊的程序–master。副本中其它的程序都是 worker 程序，由 master 分配任务。有 M 个 Map 任务和 R 个 Reduce 任务将被分配，master 将一个 Map 任务或 Reduce 任务分配给一个空闲的 worker。
3. 被分配了 map 任务的 worker 程序读取相关的输入数据片段，从输入的数据片段中解析出 key/value pair，然后把 key/value pair 传递给用户自定义的 Map 函数，由 Map 函数生成并输出的中间 key/value pair，并缓存在内存中。
4. 缓存中的 key/value pair 通过分区函数分成 R 个区域，之后周期性的写入到本地磁盘上。缓存的 key/value pair 在本地磁盘上的存储位置将被回传给 master，由 master 负责把这些存储位置再传送给 Reduce worker
5. 当 Reduce worker 程序接收到 master 程序发来的数据存储位置信息后，使用 RPC 从 Map worker 所在主机的磁盘上读取这些缓存数据。当 Reduce worker 读取了所有的中间数据后，通过对 key 进行排序后使得具有相同 key 值的数据聚合在一起。由于许多不同的 key 值会映射到相同的 Reduce 任务上，因此必须进行排序。如果中间数据太大无法在内存中完成排序，那么就要在外部进行排序。
6. Reduce worker 程序遍历排序后的中间数据，对于每一个唯一的中间 key 值，Reduce worker 程序将这个 key 值和它相关的中间 value 值的集合传递给用户自定义的 Reduce 函数。Reduce 函数的输出被追加到所属分区的输出文件。
7. 当所有的 Map 和 Reduce 任务都完成之后，master 唤醒用户程序。在这个时候，在用户程序里的对 MapReduce 调用才返回。



### 任务分析

在 main/mrsequential.go 中我们可以找到初始代码预先提供的 **单进程串行** 的 MapReduce 参考实现，而我们的任务是实现一个 **单机多进程并行** 的版本。

通过阅读 Lab 文档 http://nil.csail.mit.edu/6.824/2021/labs/lab-mr.html 以及初始代码，可知信息如下：

- 整个 MR 框架由一个 Coordinator 进程及若干个 Worker 进程构成
- Coordinator 进程与 Worker 进程间通过本地 Socket 进行 [Golang RPC](https://golang.org/pkg/net/rpc/) 通信
- 由 Coordinator 协调整个 MR 计算的推进，并分配 Task 到 Worker 上运行
- 在启动 Coordinator 进程时指定 输入文件名 及 Reduce Task 数量
- 在启动 Worker 进程时指定所用的 MR APP 动态链接库文件
- Coordinator 需要留意 Worker 可能无法在合理时间内完成收到的任务（Worker 卡死或宕机），在遇到此类问题时需要重新派发任务
- Coordinator 进程的入口文件为 [main/mrcoordinator.go](https://github.com/Mr-Dai/MIT-6.824/blob/master/src/main/mrcoordinator.go)
- Worker 进程的入口文件为 [main/mrworker.go](https://github.com/Mr-Dai/MIT-6.824/blob/master/src/main/mrworker.go)
- 我们需要补充实现 mr/coordinator.go、mr/worker.go、mr/rpc.go 这三个文件

基于此，我们不难设计出，Coordinator 需要有以下功能：

- 在启动时根据指定的输入文件数及 Reduce Task 数，生成 Map Task 及 Reduce Task
- 响应 Worker 的 Task 申请 RPC 请求，分配可用的 Task 给到 Worker 处理
- 追踪 Task 的完成情况，在所有 Map Task 完成后进入 Reduce 阶段，开始派发 Reduce Task；在所有 Reduce Task 完成后标记作业已完成并退出

