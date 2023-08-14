# 

[toc]

参考文章：

- [开源消息引擎系统 Kafka 3 新特性](https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649770210&idx=1&sn=5fe3cf70e9860583e37e9c93c7c69963&chksm=beccdb9989bb528f49915f3a64dbb1ad824a04718c309116f30af67c4766afa09189d91d725b&scene=126&sessionid=1680509849&subscene=7&key=b163269b5f582364f3e15a545b4ce6cfcb5c44e9f2aadf5f494f7d0fd092019c72b42c8f9926ab1473a00d31c649359aadd5a9a48220e32b9b42c75a092dff47fc2e54d9f23c4cd3e2103e160ea676c903dd458eec46b759bf9d74bc91658285115dcbb458a1956473110e2a8b4d96918534f1407caafd357b71b7071b60d749&ascene=0&uin=MjY0NTQ5ODYwNg%3D%3D&devicetype=Windows+11+x64&version=63090214&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQYil1W%2FOINidN4PqmG9UvrRLgAQIE97dBBAEAAAAAAF%2FII8SfzIMAAAAOpnltbLcz9gKNyK89dVj0%2BvKrP7jXAcVFe99neHAs6KtUSyhV%2FMrrdfAQCY7u4E93DIJ6x75evvfUMzs30266YMeCTCeAdb3sAwicFoanWUEObOTanc%2FuJXFPyt1RmEm0qKzTKnZG6KAK3YV%2BGYFcLUAjVCECi2mJjzqTzn1159vqcD3AvBTDEo9lAHr5cbISXaPUELXK55NaIhYjZLuQSvvopJPeO%2FVBTAppfE2ur1XMbbD5eLf4W9uG%2FSlNzh0lUyeZMhPMugJo&acctmode=0&pass_ticket=k%2FzWru6X4JY95hsUybySkk2FQsPLl2jEUYqy15D2X6d7cOLFDrwJiPuQ%2FaPaAP9mjFnmPKQ8KpXLjMQsDWJAZQ%3D%3D&wx_header=1)
- https://juejin.cn/post/7176576097205616700
- https://mp.weixin.qq.com/s/vzvmOXGcsX7rwY4J_--onw

- [20道经典的Kafka面试题详解](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Kafka%E6%A0%B8%E5%BF%83%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/%E7%89%B9%E5%88%AB%E6%94%BE%E9%80%81%EF%BC%88%E5%9B%9B%EF%BC%8920%E9%81%93%E7%BB%8F%E5%85%B8%E7%9A%84Kafka%E9%9D%A2%E8%AF%95%E9%A2%98%E8%AF%A6%E8%A7%A3.md)

- [《深入理解Kafka：核心设计与实践原理》](https://book.douban.com/subject/30437872/)

## 一、消息队列

这里的消息队列指的是各个服务以及系统内部各个组件/模块之前的通信，属于一种中间件。使用消息队列可以降低系统耦合性、实现任务异步、有效地进行流量削峰，是分布式和微服务系统中重要的组件之一。

> #### 问：为啥要使用消息队列？消息队列有啥用？

六个字总结：**解耦、异步、削峰**

- **解耦**

**传统模式**下系统间的耦合性太强。举个例子：系统 A 通过接口调用发送数据到 B、C、D 三个系统，如果将来 E 系统接入或者 B 系统不需要接入了，那么系统 A 还需要修改代码，非常麻烦。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/20210303224922.png" alt="img" style="zoom:50%;" />

如果系统 A 产生了一条比较关键的数据，那么它就要时时刻刻考虑 B、C、D、E 四个系统如果挂了该咋办？这条数据它们是否都收到了？**显然，系统 A 跟其它系统严重耦合。**

**而如果我们将数据（消息）写入消息队列，需要消息的系统直接自己从消息队列中消费。**这样下来，系统 A 就不需要去考虑要给谁发送数据，不需要去维护这个代码，也不需要考虑其他系统是否调用成功、失败超时等情况，反正我只负责生产，别的我不管。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/20210303225222.png" alt="img" style="zoom:50%;" />

- **异步**

将用户的请求数据存储到消息队列之后就立即返回结果，消息发送者不需要等待消息接收者的响应，可以继续执行其他任务。随后，系统再对消息进行消费。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/Asynchronous-message-queue.png" alt="通过异步处理提高系统性能" style="zoom: 80%;" />

因为用户请求数据写入消息队列之后就立即返回给用户了，但是请求数据在后续的业务校验、写数据库等操作中可能失败。因此，**使用消息队列进行异步处理之后，需要适当修改业务流程进行配合**，比如用户在提交订单之后，订单数据写入消息队列，不能立即返回用户订单提交成功，需要在消息队列的订单消费者进程真正处理完该订单之后，甚至出库后，再通过电子邮件或短信通知用户订单成功，以免交易纠纷。这就类似我们平时手机订火车票和电影票。

- **削峰**

**先**将短时间**高并发产生的事务消息**存储在**消息队列**中，然后**后端服务(mysql)**再**慢慢**根据自己的能力去消费这些消息，这样就避免直接把后端服务打垮掉。

> #### 问：引入消息队列会带来哪些问题？

- **系统可用性降低：**在加入 MQ 之前，你**不用考虑消息丢失**或者说 **MQ 挂掉**等等的情况；
- **系统复杂性提高：**加入 MQ 之后，你需要**保证消息没有被重复消费**、**处理消息丢失**的情况、**保证消息传递的顺序性**等等问题！
- **一致性问题：**消息队列可以实现**异步**，消息队列带来的异步确实可以提高系统响应速度。但是，万一消息的真正消费者**并没有正确消费消息**怎么办？这样就会导致数据不一致的情况了!

> #### 问：队列模型是啥？存在啥问题？

使用队列（Queue）作为消息通信载体，**满足生产者与消费者模式**，**一条消息只能被一个消费者使用**，未被消费的消息在队列中保留直到被消费或超时。

**存在的问题：**

假如我们存在这样一种情况：我们需要将生产者产生的消息分发给多个消费者，并且每个消费者都能接收到完整的消息内容。队列模型就不好解决了。

> #### 问：发布-订阅模型是啥？

**发布-订阅模型**主要是为了解决队列模型存在的问题。

**发布-订阅模型(Pub-Sub)**使用**主题(Topic)**作为消息通信载体，类似于**广播模式**：发布者发布一条消息，该消息通过主题传递给所有的订阅者，**在一条消息广播之后才订阅的用户则是收不到该条消息的**。

在发布-订阅模型中，**如果只有一个订阅者**，那它和队列模型就基本是一样的了。所以说，发布-订阅模型在功能层面上是可以**兼容队列模型**的。

## 二、Kafka

Kafka 是一个**分布式流处理平台**。

### 2.1 Kafka 应用生态

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230419144125809.png" alt="image-20230419144125809" style="zoom: 33%;" />

==**Kafka 应用架构：**==

- 这张图中居于核心地位的是 **Kafka Core 的集群**，也就是常用的**消息引擎**的这部分功能，是本篇的重点；
- 核心的左右两端分两层，**下层**分别是 Producer 和 Consumer 的基础 API，提供基础事件流消息的**推送和消费**；**上层**提供了更加高级的 Connect API，能够实现 Kafka 和其他数据系统的连接，比如消息数据自动推送到 MySQL 数据库或者将 RPC 请求转换为事件流消息进行推送；
- 最上面的是，Kafka 基于消息引擎打造了一个**流式计算平台 Streams**，提供流式的计算和存储一体化的服务。

==**具有三个关键功能：**==

1. **消息队列**：**发布和订阅消息流**，这个功能类似于消息队列，这也是 Kafka 也被归类为消息队列的原因。
2. **存储事件流**：Kafka 会把**消息持久化**到磁盘，有效避免了消息丢失的风险。
3. **处理事件流**：在消息发布的时候进行处理，Kafka 提供了一个完整的**流式处理类库**。

==**Kafka 主要有两大应用场景：**==

1. **数据收集：**可以将**不同来源的数据流**通过Kafka进行**集中收集和处理**；
1. **消息队列**：建立实时流数据管道，以可靠地在系统或应用程序之间获取数据。
2. **流式处理**：构建实时的流数据处理程序来转换或处理数据流。

### 2.2 Kafka Core 架构

#### 2.2.1 消息模型

> #### 问：Kafka 使用啥消息模型？

Kafka 使用**发布-订阅模型**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230419151229088.png" alt="image-20230419151229088" style="zoom:50%;" />

- **消费端**：消费者通过消费者组进行划分，一条消息只能由消费者组中的一个消费者消费，消费从队列的头部开始，在 HW 前结束；
- **消息队列**：消息存储是队列的数据结构，只允许消息追加写入，此外，Kafka 的队列还设计了高水位机制，避免未被从副本完成同步的消息被消费者感知并消费；
- **生产端**：生产端的 Producer 持续发送消息到队列中，消息追加到队列尾部，通过指定分区算法可以决定消息发往 Topic 下的哪个分区。

Kafka 将生产者发布的消息发送到 **Topic **中，需要这些消息的消费者可以订阅这些 **Topic **，如下图所示：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230502175221214.png" alt="image-20230502175221214" style="zoom: 50%;" />

> #### 问：Kafka消息模型中有哪些角色和实体？

消息模型的图中也为我们引出了，Kafka 比较重要的几个概念：

1. **Producer(生产者)**：将消息**发送**到指定 Topic 的 Partition 中，写入到 LEO 的位置；

2. **Consumer(消费者)**：从指定的 Topic 和 Partition 中**拉取或推送**消息，并且可以指定消费组(Consumer Group)和消费位移(Consumer Offset)等参数；

3. **Consumer Group(消费者组)**：**消费者组**由一个或者多个**消费者**组成，同一个组中的消费者对于同一条消息只消费一次。每个分区只能由**同一个消费者组内的一个消费者**(consumer)来消费(**避免竞争**)，可以由**不同的消费者组**来消费；

4. **Controller(管理者)**：整个 Kafka 集群的管理者角色，任何集群范围内的状态变更都需要通过 Controller 进行，在整个集群中是个单点的服务，可以通过选举协议进行故障转移；

   - Topic的新建和删除；

   - Partition的新建、重新分配；

   - Broker 的加入、退出；

   - 触发分区 Leader 选举。

5. **Broker(代理)**：Broker是Kafka中**负责存储和转发消息**的**服务器节点**，它可以组成一个集群，并且通过Zookeeper进行协调和管理。

   - **Topic(主题)**：一个Topic代表**一类消息**，是消息的逻辑分类。**每个Topic可以被分为多个分区**(Partition)，每个分区可以存储一定数量的消息，并且有一个唯一的ID。Producer 将消息发送到特定的主题，Consumer 通过订阅特定的 Topic(主题)来消费消息；

   - **Partition(分区)**：一个Partition是Topic的物理划分，是消息的存储单位。每个Partition中的消息是有序的，并且按照先进先出（FIFO）的顺序消费；

   - **Replica(副本)**：每个Partition可以有多个副本，其中一个副本是**Leader**，**负责处理读写请求**，其他副本是**Follower**，**负责同步Leader的数据**。


​		主题下的每条消息只会保存在某一个分区中，而不会在多个分区中被保存多份。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230321152056228.png" alt="image-20230321152056228" style="zoom: 33%;" />

> #### 问：Kafka 是通过 Push 还是 Pull？

Kafka中，消费端是通过主动 pull 消息的方式来消费的。直觉上会觉得本来就应该这样，但其实不是。Kafka 的文档里有讨论这点，**主要围绕**：消息消费的**流控策略**应该放在 **Broker 端**还是 **Consumer 端**。

#### 2.2.2 集群架构

一个具有代表性的 Kafka 集群通常具备：

- 1 个独立的 ZK 集群；
- 3 个部署在不同节点的 Broker 实例。

就以一个这样的典型集群的为例来介绍 Kafka 的整体架构：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230419155026348.png" alt="image-20230419155026348" style="zoom:50%;" />

#### 2.2.3 Zookeeper 的作用



---

参考文章：

- [深入解读基于 Kafka 和 ZooKeeper 的分布式消息队列原理](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E5%88%86%E5%B8%83%E5%BC%8F%E4%B8%AD%E9%97%B4%E4%BB%B6%E5%AE%9E%E8%B7%B5%E4%B9%8B%E8%B7%AF%EF%BC%88%E5%AE%8C%EF%BC%89/13%20%E6%B7%B1%E5%85%A5%E8%A7%A3%E8%AF%BB%E5%9F%BA%E4%BA%8E%20Kafka%20%E5%92%8C%20ZooKeeper%20%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97%E5%8E%9F%E7%90%86.md)

### 2.3 核心机制

#### 2.3.1 水位机制

> #### 问：水位机制的作用？

水位机制共有两个作用：

- 辅助从副本完成异步同步；
- 定义消息可见性，即标识哪些消息可消费。

> #### 问：Partition 中的水位机制？:star:

每个 Partition 是一个独立的消息队列：

- **`LSO`(Log Stable Offset) 是起点**，此偏移会随着消息过期时间等的影响，逐渐向右移动；
- **`HW` 是已提交消息的可见性的边界**，仅在此偏移之下的消息对外是可见的(注意，不含 `HW `本身):
  - **`Leader HW = min(LEO)`**；
  - **`Follower HW = min(Follower本地 LEO, Leader HW)`**。


- **`LEO`(Log End Offset) 是消息队列的终点**，下一条消息将在这个地方写入。

其中，`[LSO, HW)` 就是**消息的可见范围(已提交)**。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230419153859379.png" alt="image-20230419153859379" style="zoom: 33%;" />

> #### 问：水位更新？:star:

(这个感觉有点问题。)

主要是高水位是如何更新的，这里用一个 3 副本的场景描述高水位是如何更新到 5 的：

- 阶段1：此时 ISR 中最小 LEO 为 4。副本 1 发出同步请求，获取 Offset = 4 的数据；
- 阶段2：同时，HW 更新为 4；
- 阶段3：当副本 1 收到 Offset = 4 的数据，更新本地 LEO 为 5。此时 ISR 中最小 LEO 更新为 5，于是 HW 更新到 5。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230502200706595.png" alt="image-20230502200706595" style="zoom:50%;" />

---

参考文章：

- https://juejin.cn/post/7070319066325450765

#### 2.3.2 Partition 和 Replica

Kafka 中通过分区的多副本策略解决消息备份问题，有如下概念：

- **AR**: **所有副本(replicas)**统称为assigned replicas，即 AR；
- **ISR**: **leader 副本** 以及**与 leader 副本保持一定同步的 follower 副本**，叫 In Sync Replica；
- **OSR**: 与 leader 副本同步数据有一些延迟的 follower 节点。

> #### 问：Kafka 的多分区(Partition)以及多副本(Replica)机制有什么好处呢？:star:

**多分区(Partition)的好处**：

- **解决伸缩性问题，实现负载均衡，提高并发性能：**每个Topic可以被划分为多个Partition，各个Partition可以分布在不同的Broker上，每个Partition可以被不同的Producer和Consumer并行读写，从而实现**负载均衡**。

**多副本(Replica)的好处**：

- **提高容错能力：**每个Partition可以有多个Replica，其中一个Replica是Leader，负责处理读写请求，其他Replica是Follower，负责同步Leader的数据。当Leader出现故障时，Zookeeper会自动选举一个Follower作为新的Leader，从而**保证服务可用**。

> #### 问：Kafka 分区的分配策略有哪些？:star:

**消费者与主题之间的分区分配策略**：用来解决到底由哪个consumer来消费哪个partition的数据。Kafka 提供了默认的分区策略，同时支持自定义分区策略。

- **Range(平均)**：将每个主题的分区**平均分配**给消费者。这种策略适合每个主题的分区数相同，且**消费者数量稳定**的场景。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230321153750135.png" alt="image-20230321153750135" style="zoom:33%;" />

- **Round Robin(轮询):star:**：轮流将每个分区分配给消费者。这种策略适合每个主题的分区数不同，且需要实现**负载均衡**的场景。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230321153914615.png" alt="image-20230321153914615" style="zoom: 33%;" />

- **Sticky(粘性)**：在保持之前分配结果不变的前提下，**尽量减少再平衡时重新分配的分区数量**。这种策略适合需要**避免频繁再平衡**导致性能下降的场景。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230321154453985.png" alt="image-20230321154453985" style="zoom: 50%;" />

> #### 问：Leader Replica 和 Follower Replica 的同步机制？:star:

==// TODO==

> #### 问：Partition 的并发性？

**一句话总结**：同一个 Topic 不同 Partition 之间是支持并发写入消息的，同一个 Partition 不支持并发写入消息。

这很好理解，**单个 Partition 是临界资源**，需要用**锁**来进行冲突检测保证同一时间只有一批消息在写入避免出现消息乱序或者写入被覆盖的情况。

> #### 问：Kafka 如何保证消息的消费顺序？:star:

**Kafka 是不能保证全局消息顺序的，只能保证单个 Partition 下的顺序**

Kafka 中 Partition(分区) 是真正保存消息的地方，每次添加消息到 Partition(分区) 的时候都会采用**尾加法**，即 **Kafka 只能为我们保证 单个Partition(分区) 中的消息有序**。因此，有如下两种做法：

- 一个 `Topic` 只拥有一个 `Partition`；
- 在需要保证顺序的场景可以**使用 Key-Ordering 策略将同一个用户的消息发送到同一分区，即可保证顺序**。例如订单ID或用户ID等，这样同一业务字段的消息会发送到同一分区，并且消费者也按照业务字段进行消费。

#### 2.3.3 持久化

Kafka 通过：

- 使用**日志(Log)**来保存数据，一个日志就是磁盘上一个只能追加写(Append-only)消息的物理文件；
- 日志越来越大，必然要定期地删除日志以回收磁盘，这是通过**日志段(Log Segment)机制**。

Kafka 的日志架构：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230502204000910.png" alt="image-20230502204000910" style="zoom:50%;" />

一般情况下，一个 Topic 有很多 Partition，每个 Partition 就对应一个 Log 对象，在物理磁盘上则对应于一个子目录。比如创建了一个双 Partition 的 Topic:test-topic，那么，Kafka 在磁盘上会创建两个子目录：test-topic-0 和 test-topic-1。而在服务器端，这就是两个 Log 对象，**每个子目录下存在多组日志段**，也就是多组.log、.index、.timeindex文件组合，只不过文件名不同，因为每个日志段的起始位移不同。

#### 2.3.4 消息丢失与重复消费

##### 消息丢失

**一句话概括，Kafka 只对“已提交”的消息(committed message)做有限度的持久化保证。**

- 第一个核心要素是**已提交的消息**。

什么是已提交的消息？当 Kafka 的若干个 Broker 成功地接收到一条消息并写入到日志文件后，它们会告诉生产者程序这条消息**已成功提交**。此时，这条消息在 Kafka 看来就正式变为“已提交”消息了。

那为什么是若干个 Broker 呢？这取决于你对“已提交”的定义。你可以选择只要有一个 Broker 成功保存该消息就算是已提交，也可以是令所有 Broker 都成功保存该消息才算是已提交。不论哪种情况，Kafka 只对已提交的消息做持久化保证这件事情是不变的。

**==总结：==**当达到设定数量的Broker成功接收消息并写入保存之后，该消息就认为是**已提交的**。

- 第二个核心要素就是**有限度的持久化保证**。

也就是说 Kafka 不可能保证在任何情况下都做到不丢失消息。

有限度其实就是说 Kafka 不丢消息是有前提条件的。假如你的消息保存在 N 个 Kafka Broker 上，那么这个前提条件就是这 N 个 Broker 中至少有 1 个存活。只要这个条件成立，Kafka 就能保证你的这条消息永远不会丢失。

> #### 问：Kafka 如何避免消息丢失？

在使用 MQ 的时候最大的问题就是消息丢失，常见的丢失情况如下：

- 1）Producer 端丢失
- 2）Broker 端丢失
- 3）Consumer 端丢失

一条消息从生产到消费一共要经过以下 3 个流程：

- 1）Producer 发送到 Broker
- 2）Broker 保存消息(持久化)
- 3）Consumer 消费消息

3 个步骤分别对应了上述的 3 种消息丢失场景：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230505134539208.png" alt="image-20230505134539208" style="zoom: 25%;" />

**==实际情况分析：==**

1. **Producer 端丢失**

**原因：**由于网络或者Broker异常造成的。

**解决方案**：**消息重传**。

**具体实现**：Producer 不使用`fire and forget`的发送方式，**永远要使用带有回调通知的发送 API，也就是说不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)**。通过回调，如果Producer发送消息后没有收到Broker的确认回复，就会认为发送失败，进而**重传**。如果重试，可能会导致消息重复；如果放弃，可能会导致消息丢失。

2. **Broker 端丢失**

**原因：**Kafka 为了减少磁盘 I/O，采用**异步批量的刷盘策略**，也就是按照一定的消息量和间隔时间进行刷盘。Kafka 收到消息后会先存储在也缓存中(Page Cache)中，之后由操作系统根据自己的策略进行刷盘或者通过 fsync 命令强制刷盘。**如果系统挂掉，在 PageCache 中的数据就会丢失。**

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230505135601561.png" alt="image-20230505135601561" style="zoom:25%;" />

**解决方案**：考虑以**集群方式部署 Kafka 服务**，通过部署**多个副本**备份数据，保证消息**尽量**不丢失。

**具体实现**：Kafka 集群中有一个 Leader 负责消息的写入和消费，可以有多个 Follower 负责数据的备份。Follower 中有一个特殊的集合叫做 ISR，当 Leader 故障时，会从 ISR 中新选举出来 Leader。Leader 的数据会异步地复制给 Follower，这样在 Leader 发生掉电或者宕机时，Kafka 会从 Follower 中消费消息，减少消息丢失的可能。

由于消息是异步地从 Leader 复制到 Follower 的，所以一旦 Leader 宕机，那些还没有来得及复制到 Follower 的消息还是会丢失。**为了解决这个问题**，Kafka 为生产者提供一个选项叫做 `acks`，当这个选项被设置为 `all` 时，**生产者**发送的每一条消息除了发给 Leader 外还会发给所有的 ISR，并且**必须得到 Leader 和所有 ISR 的确认后才被认为发送成功**。这样，只有 Leader 和所有的 ISR 都挂了，消息才会丢失。



<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230505135948401.png" alt="image-20230505135948401" style="zoom:25%;" />

**建议是：**

- 如果你需要确保消息一条都不能丢失，那么建议不要开启消息队列的同步刷盘，而是需要使用集群的方式来解决，可以**配置当所有 ISR Follower 都接收到消息才返回成功**。

- 如果对消息的丢失有一定的容忍度，那么建议不部署集群，即使以集群方式部署，也建议配置**只发送给一个 Follower** 就可以返回成功了。

3. **Consumer 端丢失**

**原因：**一个消费者消费消息的进度是记录在消息队列集群中的，而消费的过程分为三步：接收消息、处理消息、**更新消费进度**。如果消费者在消费完消息后没有及时提交offset，就可能在下次启动时重新消费已经消费过的消息；如果消费者**在消费完消息前就提交了offset**，就可能在发生异常时漏掉一些实际上未消费的消息。

偏移量(offset)表示消费者在分区中已经消费的消息的位置。

**解决方案**：**一定要等到消息接收和处理完成后才能更新消费进度**，比如取消自动提交，改为手动提交。确定消费成功后，再手动提交偏移量。**但这样容易重复消费**。

##### 重复消费

为了避免消息丢失，需要付出两方面的代价：一方面是性能的损耗；一方面可能造成消息重复消费。性能损耗还能接受，但消息重复消费可能就会造成严重错误，那么如何避免呢？

**Kafka 出现消息重复消费的原因：**

- **根本原因**：已经消费的数据没有成功提交 offset。
- Kafka 侧由于服务端处理业务时间长或者网络链接等等原因让 Kafka 认为服务假死，触发了分区 rebalance。

想要完全的避免消息重复的发生是很难做到的，因为网络的抖动、机器的宕机和处理的异常都是比较难以避免的，在工业上并没有成熟的方法，因此我们会把要求放宽，**只要保证即使消费到了重复的消息，从消费的最终结果来看和只消费一次是等同的就好了**，也就是保证在消息的生产和消费的过程是**幂等**的。

注：**`幂等`**指只执行一次操作和多次执行同一个操作，最终得到的结果是相同的

> #### 问：Kafka 如何保证消息不被重复消费？/如何实现幂等性，设计去重机制？

**生产端**：

- **思路**：Kafka 支持将 Producer 升级为**幂等性 Producer**，保证消息虽然可能在生产端重复生产，但是最终**在消息队列存储时只会存储一份**。

- **具体做法**：给每一个生产者一个唯一的 ID，并且为生产的每一条消息赋予一个唯一 ID，消息队列的服务端会存储 `< 生产者 ID，最后一条消息 ID>` 的映射。当某一个生产者产生新的消息时，消息队列服务端会比对消息 ID 是否与存储的最后一条 ID 一致，如果一致，就认为是重复的消息，服务端会自动丢弃。

**消费端**：

- **思路**：可分为**通用层**和**业务层**。
- **具体做法**：
  - **通用层**：可以在消息被生产的时候，使用发号器给它**生成一个全局唯一的消息 ID**，消息被处理之后，**把这个 ID 存储在数据库中**，在处理下一条消息之前，先从数据库里面查询这个全局 ID 是否被消费过，如果被消费过就放弃消费；
  - **业务层**：利用乐观锁的方式来实现。这个机制是**在消息中添加一个版本号**，在生产消息时先查询数据的版本号，并且将版本号连同消息一起发送给消息队列。消费端在拿到消息和版本号后，执行时也带上版本号。比如在消费第一条消息时，version 值为 1，SQL 可以执行成功，并且同时把 version 值改为了 2；在执行第二条相同的消息时，由于 version 值不再是 1，所以这条 SQL 不能执行成功，也就保证了消息的幂等性。

#### 2.3.5 零拷贝

**零拷贝是一种思想**：指将数据在内核空间直接从磁盘文件复制到网卡中，而不需要经由用户态的应用程序。这样既可以提高数据读取的性能，也能减少核心态和用户态之间的上下文切换，提高数据传输效率。

**零拷贝基于 DMA 技术**，DMA 传输将数据从一个地址空间复制到另外一个地址空间。当CPU 初始化这个传输动作，传输动作本身是由 DMA 控制器来实行和完成。因此**通过DMA，硬件则可以绕过CPU，自己去直接访问系统主内存**。很多硬件都支持DMA，其中就包括网卡、声卡、磁盘驱动控制器等。

##### 传统数据文件拷贝

- 操作系统将数据从**磁盘**拷贝到**内核缓冲区**；
- 应用程序通过系统调用将**内核缓存区**的数据拷贝到**用户缓冲区**；
- 应用程序将**用户缓冲区**的数据拷贝到**内核的 Socket 缓冲区**中；
- 操作系统将 **Socket 缓冲区**的数据拷贝到**网卡缓冲区**中，通过网卡发送给数据接收方。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230502215933425.png" alt="image-20230502215933425" style="zoom:40%;" />

**总结**：涉及 1 次内核态到用户态的数据拷贝和 1 次 用户态到内核态的数据拷贝。

##### 零拷贝

操作系统提供了 `Sendfile` 函数，可以减少数据被拷贝的次数。使用了 `Sendfile` 之后，在内核缓冲区的数据不会被拷贝到用户缓冲区，而是直接被拷贝到 Socket 缓冲区，节省了一次拷贝的过程，提升了消息发送的性能。

- 操作系统将数据从**磁盘**拷贝到**内核缓冲区**；
- 系统调用 `Sendfile` 将**数据的文件描述符**直接被拷贝到 **Socket 缓冲区**(仅仅会拷贝一个描述符过去，不会拷贝数据到 Socket 缓存)；
- 操作系统将 **Socket 缓冲区**的数据拷贝到**网卡缓冲区**中，通过网卡发送给数据接收方。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230502220341173.png" alt="image-20230502220341173" style="zoom:40%;" />

**总结**：省略了两次不必要的数据拷贝：

- 从内核空间拷贝到用户空间；
- 从用户空间再次拷贝到内核空间。

> #### 问：零拷贝(Zero-Copy)在 Kafka 中的应用？优点？:star:

在Kafka中，体现Zero Copy使用场景的地方有两处：**基于mmap的索引**和**日志文件读写所用的TransportLayer**：

- 索引都是基于MappedByteBuffer的，也就是让用户态和内核态共享内核态的数据缓冲区，此时，数据不需要复制到用户态空间。
- TransportLayer是Kafka传输层的接口，它使用了FileChannel的transferTo方法，该方法底层使用**sendfile**实现了Zero Copy。对Kafka而言，如果I/O通道使用普通的PLAINTEXT，那么，Kafka就可以利用Zero Copy特性，直接将页缓存中的数据发送到网卡的Buffer中，避免中间的多次拷贝。相反，如果I/O通道启用了SSL，那么，Kafka便无法利用Zero Copy特性了。

## 三、常见面试题

- 什么是 Kafka，它有哪些特点和优势？
- Kafka 的架构是怎样的，它包含哪些组件和角色？
- Kafka 是如何实现高吞吐量和高可用性的？
- Kafka 是如何保证消息的顺序性和一致性的？
- Kafka 是如何实现分区和副本的，它们有什么作用？
- Kafka 是如何实现生产者和消费者之间的通信的，它们有哪些参数和策略可以配置？
- Kafka 是如何维护消费者的消费状态和偏移量的，它们存储在哪里？
  - Kafka使用消费者组来维护消费者的消费状态和偏移量。消费者组中的每个消费者都会读取一个或多个分区中的数据，并将其偏移量存储在内存中。Kafka使用一个名为__consumer_offsets的内部主题来存储消费者组的偏移量信息。每个消费者组都有一个对应的__consumer_offsets主题，其中包含每个分区的偏移量信息。消费者组中的每个消费者都会定期将其偏移量提交到__consumer_offsets主题中，以便其他消费者可以知道哪些数据已经被消费，哪些数据还没有被消费。
- Kafka 有哪些常见的使用场景，它与其他消息系统有什么区别和联系？
- Zookeeper 对于 Kafka 的作用是什么，如果 Zookeeper 宕机了会怎样？
- 如何监控和度量 Kafka 的运行状态和性能指标？
- 如何优化 Kafka 的生产者和消费者的性能，提高吞吐量和降低延迟？
- 如何保证 Kafka 的数据安全性，避免数据丢失或重复消费？
- 如何处理 Kafka 的异常情况，比如分区不平衡、主从切换、消息堆积等？

> #### 问：Kafka是什么？

Kafka 是一款**分布式流处理框架**，用于实时构建流处理应用。它有一个核心的功能广为人知，即作为企业级的**消息引擎**被广泛使用。

> #### 问：什么是消费者组？:star:

**标准答案**：关于它的定义，**官网**上的介绍言简意赅，**即消费者组是 Kafka 提供的可扩展且具有容错性的消费者机制。**切记，一定要加上前面那句，以显示你对官网很熟悉。

另外，最好再介绍下消费者组的原理：在 Kafka 中，**消费者组是一个由多个消费者实例构成的组**。**多个实例共同订阅若干个主题，实现共同消费**。同一个组下的每个实例都配置有相同的组ID，被**分配不同的订阅分区**。**当某个实例挂掉的时候，其他实例会自动地承担起它负责消费的分区**。

> 此时，又有一个小技巧给到你：消费者组的题目，能够帮你在某种程度上掌控下面的面试方向。
>
> - 如果你擅长**位移值原理**，就不妨再提一下消费者组的位移提交机制；
> - 如果你擅长Kafka **Broker**，可以提一下消费者组与Broker之间的交互；
> - 如果你擅长与消费者组完全不相关的**Producer**，那么就可以这么说：“消费者组要消费的数据完全来自于Producer端生产的消息，我对Producer还是比较熟悉的。”
>
> 使用这个策略的话，面试官可能会被你的话术所影响，偏离他之前想问的知识路径。当然了，如果你什么都不擅长，那就继续往下看题目吧。

> #### 问：在 Kafka 中，ZooKeeper 的作用是什么？:triangular_flag_on_post::star:

这是一道能够帮助你脱颖而出的题目。碰到这个题目，请在心中暗笑三声。

**标准答案**：目前，**Kafka 使用 ZooKeeper 存放集群元数据、成员管理、Controller 选举，以及其他一些管理类任务**。KIP-500 提案之后，**Kafka 就不再依赖于 ZooKeeper**。

- **存放元数据**：是指主题分区的所有数据都保存在ZooKeeper中，且以它保存的数据为权威，其他“人”都要与它保持对齐；
- **成员管理**：是指Broker节点的注册、注销以及属性变更；
- **Controller选举**：是指选举集群Controller，而其他管理类任务包括但不限于主题删除、参数配置等。

- **KIP-500**：是使用社区自研的基于Raft的共识算法，替代ZooKeeper，实现Controller自选举。

> #### 问：解释下 Kafka 中位移(offset)的作用？

在 Kafka 中，**每个 Partition 下的每条消息**都被赋予了一个**唯一的 ID 数值**，用于**标识它在分区中的位置**。这个ID数值，就被称为位移，或者叫**偏移量**。一旦消息被写入到分区日志，它的位移值将不能被修改。

主要是用来：

- 记录当前消费到哪里了？
- 记录当前日志提交到哪里了？引出水位机制。

> #### 问：阐述下 Kafka 中的主副本(Leader Replica)和从副本(Follower Replica)的区别？

可以这么回答：**Kafka 副本分为 Leader 副本和 Replica 副本**。**只有 Leader 副本才能对外提供读写服务**，响应 Clients 端的请求。**Follower 副本只是采用拉(PULL)的方式，被动地同步Leader副本中的数据**，并且在 Leader 副本所在的 Broker 宕机后，随时准备**选举成 Leader 副本**，实现高可用性。

通常来说，回答到这个程度，其实才只说了60%，因此，我建议你再回答两个额外的加分项。

- **强调 Follower 副本也能对外提供读服务**。自 Kafka 2.4 版本开始，社区通过引入新的 Broker 端参数，**允许 Follower 副本有限度地提供读服务**；
- **强调 Leader 和 Follower 的消息序列在实际场景中不一致**。很多原因都可能造成Leader和Follower保存的消息序列不一致，比如程序Bug、网络问题等。这是很严重的错误，必须要完全规避。你可以补充下，之前确保一致性的主要手段是高水位机制，但高水位值无法保证Leader连续变更场景下的数据一致性，因此，**社区引入了Leader Epoch机制**，来修复高水位值的弊端。关于“Leader Epoch机制”，国内的资料不是很多，它的普及度远不如高水位，不妨大胆地把这个概念秀出来，力求惊艳一把。上一季专栏的[第27节课]讲的就是Leader Epoch机制的原理，推荐你赶紧去学习下。

> #### 问：LEO、LSO、AR、ISR、HW 都表示什么含义？

- **LEO**：Log End Offset。日志末端位移值或末端偏移量，表示**日志下一条待插入消息的位移值**。举个例子，如果日志有10条消息，位移值从0开始，那么，第10条消息的位移值就是9。此时，LEO = 10；
- **LSO**：Log Stable Offset。这是Kafka事务的概念。如果你没有使用到事务，那么这个值不存在（其实也不是不存在，只是设置成一个无意义的值）。**该值控制了事务型消费者能够看到的消息范围**。它经常与Log Start Offset，即日志起始位移值相混淆，因为有些人将后者缩写成LSO，这是不对的。在Kafka中，LSO就是指代Log Stable Offset；
- **HW**：高水位值（High watermark）。这是**控制消费者可读取消息范围的重要字段**。一个普通消费者只能“看到”Leader副本上介于Log Start Offset和HW（不含）之间的所有消息。水位以上的消息是对消费者不可见的；
- **AR**：Assigned Replicas。AR是主题被创建后，**所有副本**统称为assigned replicas，即 AR；
- **ISR**：In-Sync Replicas。**指代的是AR中那些与Leader保持同步的副本集合**。在AR中的副本可能不在ISR中，但Leader副本天然就包含在ISR中。关于ISR，还有一个常见的面试题目是**如何判断副本是否应该属于ISR**。目前的判断依据是：**Follower LEO落后Leader LEO的时间，是否超过了Broker端参数replica.lag.time.max.ms值**。如果超过了，副本就会被从ISR中移除。

> #### 问：Leader Partition 的选举策略有几种？/在哪些场景下需要执行 Leader 选举？

Partition 的 Leader 选举由 Controller 负责。当前，Kafka有 4 种 Leader Partition 选举策略：

- **OfflinePartition Leader选举**：**每当有分区上线时，就需要执行 Leader 选举**。所谓的分区上线，可能是创建了新分区，也可能是之前的下线分区重新上线。这是最常见的分区 Leader 选举场景。
- **ReassignPartition Leader选举**：当你手动运行kafka-reassign-partitions命令，或者是调用Admin的alterPartitionReassignments方法执行分区副本重分配时，可能触发此类选举。假设原来的AR是[1，2，3]，Leader是1，当执行**副本重分配**后，副本集合AR被设置成[4，5，6]，显然，Leader必须要变更，此时会发生Reassign Partition Leader选举。
- **PreferredReplicaPartition Leader选举**：当你手动运行kafka-preferred-replica-election命令，或自动触发了Preferred Leader选举时，该类策略被激活。所谓的Preferred Leader，指的是AR中的第一个副本。比如AR是[3，2，1]，那么，Preferred Leader就是3。
- **ControlledShutdownPartition Leader选举**：当Broker正常关闭时，该Broker上的所有Leader副本都会下线，因此，需要为受影响的分区执行相应的Leader选举。

这 4 类选举策略的大致思想是类似的，即**从 AR 中挑选首个在 ISR 中的副本**，作为新 Leader。

> #### 问：Kafka 的哪些场景中使用了零拷贝(Zero Copy)？:star:

Zero Copy是特别容易被问到的高阶题目。Kafka的数据是持久化到每个Partition下的.log文件中的，因此当需要消费已经持久化的消息时，势必需要从磁盘中将数据读取到内存中，并通过网卡发送给消费者。而Kafka的高性能主要就依赖于**零拷贝技术**。

在Kafka中，体现Zero Copy使用场景的地方有两处：**基于mmap的索引**和**日志文件读写所用的TransportLayer**：

- 索引都是基于MappedByteBuffer的，也就是让用户态和内核态共享内核态的数据缓冲区，此时，数据不需要复制到用户态空间。
- TransportLayer是Kafka传输层的接口，它使用了FileChannel的transferTo方法，该方法底层使用**sendfile**实现了Zero Copy。对Kafka而言，如果I/O通道使用普通的PLAINTEXT，那么，Kafka就可以利用Zero Copy特性，直接将页缓存中的数据发送到网卡的Buffer中，避免中间的多次拷贝。相反，如果I/O通道启用了SSL，那么，Kafka便无法利用Zero Copy特性了。

> #### 问：Kafka 为什么不支持读写分离？:star:

Kafka 2.4 之前，**不支持**；2.4 之后，**Kafka 提供了有限度的读写分离，也就是说，Follower 副本能够对外提供读服务**。

说完这些之后，你可以再给出之前的版本不支持读写分离的理由。

- **场景不适用**。读写分离适用于那种**读负载很大，而写操作相对不频繁**的场景，可Kafka不属于这样的场景。
- **同步机制**。Kafka采用PULL方式实现Follower的同步，因此，Follower与Leader存在不一致性窗口。如果允许读Follower副本，就势必要处理消息滞后（Lagging）的问题。

> #### 问：Controller 发生网络分区（Network Partitioning）时，Kafka 会怎么样？

一旦发生Controller网络分区，那么，第一要务就是查看集群是否出现“脑裂”，即同时出现两个甚至是多个Controller组件。这可以根据Broker端监控指标ActiveControllerCount来判断。

> #### 问：简述 Follower 副本消息同步的完整流程？:star:

- 首先，Follower 发送 FETCH 请求给 Leader；
- 接着，Leader 会读取底层日志文件中的消息数据，再更新它内存中的 Follower 副本的 LEO 值，更新为 FETCH 请求中的 fetchOffset 值。最后，尝试更新分区高水位值；
- Follower 接收到 FETCH 响应之后，会把消息写入到底层日志，接着更新 LEO 和 HW 值。

Leader 和 Follower 的 HW 值更新时机是不同的，Follower 的 HW 更新永远落后于 Leader 的 HW。这种时间上的错配是造成各种不一致的原因。

==// TODO==

