# 微服务基础知识 


## 一、微服务

### 1.1 微服务技术栈

先给出微服务的架构：

![image-20230420215011808](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230420215011808.png)

分类：

![image-20230420215144450](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230420215144450.png)

### 1.2 认识微服务

**单体架构**：将业务的所有功能集中在一个项目中开发，打包成一个部署。

- 优点：架构简单，部署成本低；
- 缺点：耦合度太高，不适合大型项目。

**微服务架构**的特征：

- **单一职责**：微服务拆分的粒度更小，**每一个服务都对应唯一的业务能力**，**做到单一职责**，避免重复业务开发；
- **面向服务**：微服务对外暴露业务接口，允许远程调用；
- **隔离性强**：各服务做好隔离、容错、降级等，避免服务挂了对其他服务产生影响。

## 二、gRPC

### 概述

gRPC 是谷歌推出的一个开源、高性能的 RPC 框架。默认情况下**使用 protoBuf 进行序列化和反序列化**，并**基于 HTTP/2 传输**报文，带来诸如多请求复用一个 TCP 连接(所谓的多路复用)、双向流、流控、头部压缩等特性。

在 gRPC 中，**开发者可以像调用本地方法一样，通过 gRPC 的客户端调用远程机器上 gRPC 服务的方法**。gRPC 客户端封装了 HTTP/2 协议数据帧的打包、以及网络层的通信细节，把复杂留给框架自己，把便捷提供给用户。

gRPC 基于这样的一个设计理念：

- 定义一个服务，及其被远程调用的方法(方法名称、入参、出参)，在 gRPC 服务端实现这个方法的业务逻辑，并在 gRPC 服务端处理来自远程客户端对这个 RPC 方法的调用；
- 在 gRPC 客户端也拥有这个 RPC 方法的存根(stub)。gRPC 的客户端和服务端都可以用任何支持 gRPC 的语言来实现，例如一个 gRPC 服务端可以是 C++ 语言编写的，以供 Ruby 语言的 gRPC 客户端和 JAVA 语言的 gRPC 客户端调用，如下图所示：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230416144445687.png" alt="image-20230416144445687" style="zoom: 33%;" />

gRPC 默认**使用 ProtoBuf 对请求/响应进行序列化和反序列化**，这使得传输的请求体和响应体比 JSON 等序列化方式包体更小、更轻量。

gRPC 基于 **HTTP/2 协议传输报文**，HTTP/2 具有多路复用、头部压缩等特性，基于 HTTP/2 的帧设计，实现了多个请求复用一个 TCP 连接，基本解决了 HTTP/1.1 的队头阻塞问题，相对 HTTP/1.1 带来了巨大的性能提升。

### 协议格式

gRPC 基于 HTTP/2/协议进行通信，使用 ProtoBuf 序列化与反序列化，gRPC 的协议格式如下图：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230405213840874.png" alt="image-20230405213840874" style="zoom:50%;" />

gRPC 协议在 HTTP 协议的基础上，对 HTTP/2 的帧的**有效包体(Frame Payload)**做了进一步编码：**gRPC 包头(5 字节)+gRPC 变长包头**，其中：

1. **5 字节**的 gRPC 包头：**1 字节的压缩标志(compress flag)** 和 **4 字节的 gRPC 包头长度**；
2. **gRPC 包体长度是变长的**，是一串二进制流：**使用指定序列化方式(通常是 ProtoBuf)序列化成字节流**，再使用指定的压缩算法**对序列化的字节流压缩**而成的。如果对序列化字节流进行了压缩，gRPC 包头的压缩标志为 1。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230405214533622.png" alt="image-20230405214533622" style="zoom:50%;" />


