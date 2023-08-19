# Go-Channel


[toc]

参考文章：

- [Golang Channel 实现原理](https://mp.weixin.qq.com/s?__biz=MzkxMjQzMjA0OQ==&mid=2247483770&idx=1&sn=fa999e22d5de4624544488562d6f799d&chksm=c10c4fa4f67bc6b2f381ea7669dfd3322a3ced0ce0836f528185cf85f52d8414659afda0557f&scene=126&sessionid=1682837915#rd)

## 一、核心数据结构

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230430221634641.png" alt="image-20230430221634641" style="zoom: 67%;" />

### 1.1 hchan

```go
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    elemtype *_type 		// element type
    sendx    uint   		// send index
    recvx    uint   		// receive index
    recvq    waitq  		// list of recv waiters
    sendq    waitq  		// list of send waiters
    
    lock mutex
}
```

- qcount：当前 channel 中存在多少个元素；
- dataqsize: 当前 channel 能存放的元素容量；
- buf：channel 中用于存放元素的环形缓冲区；
- elemsize：channel 元素类型的大小；
- closed：标识 channel 是否关闭；
- elemtype：channel 元素类型；
- sendx：发送元素进入环形缓冲区的 index；
- recvx：接收元素所处的环形缓冲区的 index；
- recvq：因接收而陷入阻塞的协程队列；
- sendq：因发送而陷入阻塞的协程队列；

### 1.2 waitq

```go
type waitq struct {
    first *sudog
    last  *sudog
}
```

waitq：阻塞的协程队列

- first：队列头部
- last：队列尾部

### 1.3 sudog

```go
type sudog struct {
    g *g

    next *sudog
    prev *sudog
    elem unsafe.Pointer // data element (may point to stack)

    isSelect bool

    c        *hchan 
}
```

sudog：用于包装协程的节点

- g：goroutine，协程；
- next：队列中的下一个节点；
- prev：队列中的前一个节点；
- elem: 读取/写入 channel 的数据的容器;
- isSelect：标识当前协程是否处在 select 多路复用的流程中；
- c：标识与当前 sudog 交互的 chan.

## 二、





## 面试题

> #### 问：要求实现一个 map：
>
> 1. 支持高并发；
> 2. 只存在插入和查询操作 O(1)；
> 3. 查询时，若 key 存在，直接返回 val；若 key 不存在，阻塞直到 key-val 被放入后，获取 val 返回；等待指定时长仍未放入，就返回超时错误；
> 4. 不能有死锁或者 panic 风险。


