# 

[toc]

## 一、简介

### 1.1 概念

一句话，Docker 是解决了**运行环境和配置问题**的软件容器，方便做**持续集成**并有助于**整体发布**的容器。

虚拟化技术演历路径可分为三个时代：

1. **物理机时代**，多个应用程序可能跑在一台物理机器上

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230428160837655.png" alt="image-20230428160837655" style="zoom:33%;" />

2. **虚拟机时代**，一台物理机器启动多个虚拟机实例，一个虚拟机跑多个应用程序

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230428160903729.png" alt="image-20230428160903729" style="zoom: 40%;" />

3. **容器化时代**，一台物理机上启动多个容器实例，一个容器跑多个应用程序

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230428160929323.png" alt="image-20230428160929323" style="zoom:40%;" />

**Docker、虚拟机对比**：

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230428161824525.png" alt="image-20230428161824525" style="zoom: 50%;" />

- 一次构建、随处运行，省去重复配置环境、设置的过程，就很方便；
- 仅包含业务运行所需的 runtime 环境，无需虚拟化整个操作系统；

### 1.2 安装

官方链接：https://docs.docker.com/get-docker/

### 1.3 组成

docker 的基本组成：

- 镜像(image)：相当于一份模板，是跨平台、可移植的程序+环境包；
- 容器(container)：相当于镜像是实例，可以创建出很多个实例；
- 仓库(repository)：镜像的存储位置，有云端仓库和本地仓库之分，官方镜像仓库地址


