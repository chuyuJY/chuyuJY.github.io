# 

[toc]

## 一、简介

### 1.1 部署方式的演变

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230416193626038.png" alt="image-20230416193626038" style="zoom:33%;" />

- **传统部署时代**：直接将应用程序部署在物理机上，不能为应用程序定义资源使用边界，很难合理地分配计算资源，所以**程序之间容易产生影响**；
- **虚拟化部署时代**：可以在一台物理机上运行多个虚拟机(比如 VM)，每个虚拟机都是独立的一个环境。缺点是**增加了操作系统**，浪费了部分资源；
- **容器化部署时代**：类似于虚拟化，被认为是**轻量级**的，具有自己的文件系统、CPU 、内存、进程空间等，运行应用程序所需要的资源都被容器包装，**部署方便**。

> ## 容器化部署的优势：
>
> - `敏捷性`：敏捷应用程序的创建和部署。和使用 VM 镜像相比，提高了容器镜像创建的简便性和效率；
>
> - `解耦性`：关注开发和运维的分离。在构建、发布时创建应用程序的容器镜像，而不是在部署的时候，从而将应用程序和基础架构分离；
> - `跨平台`：跨开发、测试和生产的环境一致性。在便捷式的计算机上和在云上相同的运行；
> - `大分布式`：松散耦合、分布式、弹性、解放的微服务。应用程序被分解成较小的独立部分，并且可以动态的部署和管理。

> ##### 容器化部署带来的问题：
>
> - 故障转移：一个容器故障停机了，怎么样让另外一个容器立刻启动去替补停机的容器；
> - 负载均衡：当并发访问量变大的时候，怎么样做到横向扩展容器数量。

这些问题统称为**容器编排**问题，Kubernetes 是最主流的容器编排软件。

### 1.2 为什么使用 Kubernetes (k8s)

Kubernetes 的本质是**一组服务器集群**，它可以在集群的每个节点上运行特定的程序，来对节点中的容器进行管理。目的是实现资源管理的自动化，主要提供了如下的主要功能：

- **自我修复**：一旦某一个容器崩溃，能够在1秒中左右迅速启动新的容器；
- **弹性伸缩**：可以根据需要，自动对集群中正在运行的容器数量进行调整；
- **服务发现**：服务可以通过自动发现的形式找到它所依赖的服务；
- **负载均衡**：如果一个服务启动了多个容器，能够自动实现请求的负载均衡；
- **版本回退**：如果发现新发布的程序版本有问题，可以立即回退到原来的版本；
- **存储编排**：可以根据容器自身的需求自动创建存储卷。

### 1.3 k8s 的组件

一个 k8s 集群主要是由**控制节点(master)**、**工作节点(node)**构成，每个节点上都会安装不同的组件：

**master：集群的控制平面，负责集群的决策 ( 管理 )**

- **ApiServer**: 资源操作的唯一入口，接收用户输入的命令，提供认证、授权、API注册和发现等机制；
- **Scheduler**: 负责集群资源调度，按照预定的调度策略将Pod调度到相应的node节点上；
- **ControllerManager**: 负责维护集群的状态，比如程序部署安排、故障检测、自动扩展、滚动更新等；
- **Etcd**：负责存储集群中各种资源对象的信息。

**node：集群的数据平面，负责为容器提供运行环境 ( 干活 )**

- **Kubelet**: 负责维护容器的生命周期，即通过控制docker，来创建、更新、销毁容器；
- **KubeProxy**: 负责提供集群内部的服务发现和负载均衡；
- **Docker**: 负责节点上容器的各种操作。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20200406184656917.png" alt="img" style="zoom:50%;" />

> ## 如何让 k8s 部署一个 nginx 服务？
>
> 1. kubernetes环境启动之后，master和node都会将自身的信息存储到etcd数据库中；
>
> 2. 一个nginx服务的安装请求会首先被发送到master节点的apiServer组件；
>
> 3. apiServer组件会调用scheduler组件来决定到底应该把这个服务安装到哪个node节点上；
>
>    在此时，它会从etcd中读取各个node节点的信息，然后按照一定的算法进行选择，并将结果告知apiServer；
>
> 4. apiServer调用controller-manager去调度Node节点安装nginx服务；
>
> 5. kubelet接收到指令后，会通知docker，然后由docker来启动一个nginx的pod；
>
>    pod是kubernetes的最小操作单元，容器必须跑在pod中；
>
> 6. 至此，一个nginx服务就运行了，如果需要访问nginx，就需要通过kube-proxy来对pod产生访问的代理；
>
> 7. 外界用户就可以访问集群中的nginx服务了。

### 1.4 k8s 概念

- **Master**：集群控制节点，每个集群需要至少一个master节点负责集群的管控；

- **Node**：工作负载节点，由master分配容器到这些node工作节点上，然后node节点上的docker负责容器的运行；

- **Pod**：kubernetes的最小控制单元，容器都是运行在pod中的，一个pod中可以有1个或者多个容器；

- **Controller**：控制器，通过它来实现对pod的管理，比如启动pod、停止pod、伸缩pod的数量等等；

- **Service**：pod对外服务的统一入口，下面可以维护者同一类的多个pod；

- **Label**：标签，用于对pod进行分类，同一类pod会拥有相同的标签；

- **NameSpace**：命名空间，用来隔离pod的运行环境。

## 二、集群环境搭建

### 2.1 集群分类

Kubernetes 集群大致分为两类：

- - **一主多从**：一个 Master 节点和多台 Node 节点，搭建简单，但是有单机故障风险，适合用于测试环境。

- - **多主多从**(高可用)：多台 Master 节点和多台 Node 节点，搭建麻烦，安全性高，适合用于生产环境。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20200404094800622.png" alt="image-20200404094800622" style="zoom:50%;" />

### 2.2 安装

#### 2.2.1 安装方式

目前生产部署Kubernetes 集群主要有两种方式：

- **Kubeadm**：Kubeadm 是一个K8s 部署工具，提供kubeadm init 和 kubeadm join，用于快速部署Kubernetes 集群。

​	官方地址：[https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/](https://gitee.com/link?target=https%3A%2F%2Fkubernetes.io%2Fdocs%2Freference%2Fsetup-tools%2Fkubeadm%2Fkubeadm%2F)

- **二进制包**：从github 下载发行版的二进制包，手动部署每个组件，组成 Kubernetes 集群。

方便起见，还是用 Kubeadm 吧hhh，以后再说用二进制包的方式。

#### 2.2.2 准备环境

在开始之前，部署 Kubernetes 集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS7.x-86_x64
- 硬件配置：2GB 或更多RAM，2 个CPU 或更多CPU，硬盘30GB 或更多
- 集群中所有机器之间网络互通
- 可以访问外网，需要拉取镜像
- 禁止swap 分区

**准备 3 个 CentOS7.x-86_x64 虚拟机**：

- 在所有节点上安装Docker 和kubeadm
- 部署Kubernetes Master
- 部署容器网络插件
- 部署Kubernetes Node，将节点加入Kubernetes 集群中
- 部署Dashboard Web 页面，可视化查看Kubernetes 资源

| 角色     | IP地址      | 组件                              |
| -------- | ----------- | --------------------------------- |
| master01 | 192.168.5.3 | docker，kubectl，kubeadm，kubelet |
| node01   | 192.168.5.4 | docker，kubectl，kubeadm，kubelet |
| node02   | 192.168.5.5 | docker，kubectl，kubeadm，kubelet |

#### 2.2.3 环境初始化

##### 2.2.3.1 检查操作系统的版本

```
# 此方式下安装kubernetes集群要求Centos版本要在7.5或之上
[root@master ~]# cat /etc/redhat-release
Centos Linux 7.5.1804 (Core)
```

##### 2.2.3.2 主机名解析

为了方便集群节点间的直接调用，在这个配置一下主机名解析，企业中推荐使用内部DNS服务器

```
# 主机名成解析 编辑三台服务器的/etc/hosts文件，添加下面内容
192.168.90.100 master
192.168.90.106 node1
192.168.90.107 node2
```

##### 2.2.3.3 时间同步

kubernetes要求集群中的节点时间必须精确一直，这里使用chronyd服务从网络同步时间

企业中建议配置内部的会见同步服务器

```
# 启动chronyd服务
[root@master ~]# systemctl start chronyd
[root@master ~]# systemctl enable chronyd
[root@master ~]# date
```

##### 2.2.3.4 禁用iptable和firewalld服务

kubernetes和docker 在运行的中会产生大量的iptables规则，为了不让系统规则跟它们混淆，直接关闭系统的规则

```
# 1 关闭firewalld服务
[root@master ~]# systemctl stop firewalld
[root@master ~]# systemctl disable firewalld
# 2 关闭iptables服务
[root@master ~]# systemctl stop iptables
[root@master ~]# systemctl disable iptables
```

##### 2.2.3.5 禁用selinux

selinux是linux系统下的一个安全服务，如果不关闭它，在安装集群中会产生各种各样的奇葩问题

```
# 编辑 /etc/selinux/config 文件，修改SELINUX的值为disable
# 注意修改完毕之后需要重启linux服务
SELINUX=disabled
```

##### 2.2.3.6 禁用swap分区

swap分区指的是虚拟内存分区，它的作用是物理内存使用完，之后将磁盘空间虚拟成内存来使用，启用swap设备会对系统的性能产生非常负面的影响，因此kubernetes要求每个节点都要禁用swap设备，但是如果因为某些原因确实不能关闭swap分区，就需要在集群安装过程中通过明确的参数进行配置说明

```
# 编辑分区配置文件/etc/fstab，注释掉swap分区一行
# 注意修改完毕之后需要重启linux服务
vim /etc/fstab
注释掉 /dev/mapper/centos-swap swap
# /dev/mapper/centos-swap swap
```

##### 2.2.3.7 修改linux的内核参数

```
# 修改linux的内核采纳数，添加网桥过滤和地址转发功能
# 编辑/etc/sysctl.d/kubernetes.conf文件，添加如下配置：
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1

# 重新加载配置
[root@master ~]# sysctl -p
# 加载网桥过滤模块
[root@master ~]# modprobe br_netfilter
# 查看网桥过滤模块是否加载成功
[root@master ~]# lsmod | grep br_netfilter
```

##### 2.2.3.8 配置ipvs功能

在Kubernetes中Service有两种带来模型，一种是基于iptables的，一种是基于ipvs的两者比较的话，ipvs的性能明显要高一些，但是如果要使用它，需要手动载入ipvs模块

```
# 1.安装ipset和ipvsadm
[root@master ~]# yum install ipset ipvsadm -y
# 2.添加需要加载的模块写入脚本文件
[root@master ~]# cat <<EOF> /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
# 3.为脚本添加执行权限
[root@master ~]# chmod +x /etc/sysconfig/modules/ipvs.modules
# 4.执行脚本文件
[root@master ~]# /bin/bash /etc/sysconfig/modules/ipvs.modules
# 5.查看对应的模块是否加载成功
[root@master ~]# lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

##### 2.2.3.9 安装docker

```
# 1、切换镜像源
[root@master ~]# wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

# 2、查看当前镜像源中支持的docker版本
[root@master ~]# yum list docker-ce --showduplicates

# 3、安装特定版本的docker-ce
# 必须制定--setopt=obsoletes=0，否则yum会自动安装更高版本
[root@master ~]# yum install --setopt=obsoletes=0 docker-ce-18.06.3.ce-3.el7 -y

# 4、添加一个配置文件
#Docker 在默认情况下使用Vgroup Driver为cgroupfs，而Kubernetes推荐使用systemd来替代cgroupfs
[root@master ~]# mkdir /etc/docker
[root@master ~]# cat <<EOF> /etc/docker/daemon.json
{
	"exec-opts": ["native.cgroupdriver=systemd"],
	"registry-mirrors": ["https://kn0t2bca.mirror.aliyuncs.com"]
}
EOF

# 5、启动dokcer
[root@master ~]# systemctl restart docker
[root@master ~]# systemctl enable docker
```

##### 2.2.3.10 安装Kubernetes组件

```
# 1、由于kubernetes的镜像在国外，速度比较慢，这里切换成国内的镜像源
# 2、编辑/etc/yum.repos.d/kubernetes.repo,添加下面的配置
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgchech=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
			http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

# 3、安装kubeadm、kubelet和kubectl
[root@master ~]# yum install --setopt=obsoletes=0 kubeadm-1.17.4-0 kubelet-1.17.4-0 kubectl-1.17.4-0 -y

# 4、配置kubelet的cgroup
#编辑/etc/sysconfig/kubelet, 添加下面的配置
KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"

# 5、设置kubelet开机自启
[root@master ~]# systemctl enable kubelet
```

##### 2.2.3.11 准备集群镜像

```
# 在安装kubernetes集群之前，必须要提前准备好集群需要的镜像，所需镜像可以通过下面命令查看
[root@master ~]# kubeadm config images list

# 下载镜像
# 此镜像kubernetes的仓库中，由于网络原因，无法连接，下面提供了一种替换方案
images=(
	kube-apiserver:v1.17.4
	kube-controller-manager:v1.17.4
	kube-scheduler:v1.17.4
	kube-proxy:v1.17.4
	pause:3.1
	etcd:3.4.3-0
	coredns:1.6.5
)

for imageName in ${images[@]};do
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
	docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName 
done
```

##### 2.2.3.11 集群初始化

> 下面的操作只需要在master节点上执行即可

```
# 创建集群
[root@master ~]# kubeadm init \
	--apiserver-advertise-address=192.168.90.100 \
	--image-repository registry.aliyuncs.com/google_containers \
	--kubernetes-version=v1.17.4 \
	--service-cidr=10.96.0.0/12 \
	--pod-network-cidr=10.244.0.0/16
# 创建必要文件
[root@master ~]# mkdir -p $HOME/.kube
[root@master ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@master ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> 下面的操作只需要在node节点上执行即可

```
kubeadm join 192.168.0.100:6443 --token awk15p.t6bamck54w69u4s8 \
    --discovery-token-ca-cert-hash sha256:a94fa09562466d32d29523ab6cff122186f1127599fa4dcd5fa0152694f17117 
```

在master上查看节点信息

```
[root@master ~]# kubectl get nodes
NAME    STATUS   ROLES     AGE   VERSION
master  NotReady  master   6m    v1.17.4
node1   NotReady   <none>  22s   v1.17.4
node2   NotReady   <none>  19s   v1.17.4
```

##### 2.2.3.13 安装网络插件，只在master节点操作即可

```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

由于外网不好访问，如果出现无法访问的情况，可以直接用下面的 记得文件名是kube-flannel.yml，位置：/root/kube-flannel.yml内容：

```
https://github.com/flannel-io/flannel/tree/master/Documentation/kube-flannel.yml
```

也可手动拉取指定版本 docker pull quay.io/coreos/flannel:v0.14.0 #拉取flannel网络，三台主机 docker images #查看仓库是否拉去下来

`个人笔记` 若是集群状态一直是 notready,用下面语句查看原因， journalctl -f -u kubelet.service 若原因是： cni.go:237] Unable to update cni config: no networks found in /etc/cni/net.d mkdir -p /etc/cni/net.d #创建目录给flannel做配置文件 vim /etc/cni/net.d/10-flannel.conf #编写配置文件

```
{
 "name":"cbr0",
 "cniVersion":"0.3.1",
 "type":"flannel",
 "deledate":{
    "hairpinMode":true,
    "isDefaultGateway":true
  }

}
```

##### 2.2.3.14 使用kubeadm reset重置集群

```
#在master节点之外的节点进行操作
kubeadm reset
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
##重启kubelet
systemctl restart kubelet
##重启docker
systemctl restart docker
```

##### 2.2.3.15 重启kubelet和docker

```
# 重启kubelet
systemctl restart kubelet
# 重启docker
systemctl restart docker
```

使用配置文件启动fannel

```
kubectl apply -f kube-flannel.yml
```

等待它安装完毕 发现已经是 集群的状态已经是Ready

![img](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/2232696-20210621233106024-1676033717.png)

##### 2.2.3.16 kubeadm中的命令

```
# 生成 新的token
[root@master ~]# kubeadm token create --print-join-command
```

#### 2.2.4 集群测试

##### 2.2.4.1 创建一个nginx服务

```
kubectl create deployment nginx  --image=nginx:1.14-alpine
```

##### 2.2.4.2 暴露端口

```
kubectl expose deploy nginx  --port=80 --target-port=80  --type=NodePort
```

##### 2.2.4.3 查看服务

```
kubectl get pod,svc
```

##### 2.2.4.4 查看pod

![img](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/2232696-20210621233130477-111035427.png)

浏览器测试结果：

![img](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/2232696-20210621233157075-1117518703.png)

## 三、资源管理

### 3.1 简介

在kubernetes中，所有的内容都抽象为资源，用户需要通过操作资源来管理kubernetes。

- kubernetes的本质上就是一个集群系统，用户可以在集群中部署各种服务，所谓的部署服务，其实就是在kubernetes集群中运行一个个的**`容器`**，并将指定的程序跑在容器中。

- kubernetes的**最小管理单元是pod**而不是容器，所以只能将容器放在`Pod`中，而kubernetes一般也不会直接管理Pod，而是通过`Pod控制器`来管理Pod的。

- Pod可以提供服务之后，就要考虑如何访问Pod中服务。kubernetes提供了**`Service`**实现这个功能。

- 当然，如果Pod中程序的数据需要持久化，kubernetes还提供了各种**`存储`**系统。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20200406225334627.png" alt="image-20200406225334627" style="zoom:50%;" />

> ##### 学习kubernetes的核心，就是学习如何对集群上的`Pod、Pod控制器、Service、存储`等各种资源进行操作。

### 3.2 YAML

**YAML 常用来作为配置文件**。

YAML的语法比较简单，主要有下面几个：

- 大小写敏感
- 使用缩进表示层级关系
- 缩进不允许使用tab，只允许空格( 低版本限制 )
- 缩进的空格数不重要，只要相同层级的元素左对齐即可
- '#'表示注释

YAML支持以下几种数据类型：

- **纯量：单个的、不可再分的值**

```yaml
# 纯量, 就是指的一个简单的值，字符串、布尔值、整数、浮点数、Null、时间、日期
# 1 布尔类型
c1: true (或者True)
# 2 整型
c2: 234
# 3 浮点型
c3: 3.14
# 4 null类型 
c4: ~  # 使用~表示null
# 5 日期类型
c5: 2018-02-17    # 日期必须使用ISO 8601格式，即yyyy-MM-dd
# 6 时间类型
c6: 2018-02-17T15:02:31+08:00  # 时间使用ISO 8601格式，时间和日期之间使用T连接，最后使用+代表时区
# 7 字符串类型
c7: heima     # 简单写法，直接写值 , 如果字符串中间有特殊字符，必须使用双引号或者单引号包裹 
c8: line1
    line2     # 字符串过多的情况可以拆成多行，每一行会被转化成一个空格
```

- **对象：键值对的集合**

```yaml
# 对象
# 形式一(推荐):
heima:
  age: 15
  address: Beijing
# 形式二(了解):
heima: {age: 15,address: Beijing}
```

- **数组：列表(list)**

```yaml
# 数组
# 形式一(推荐):
address:
  - 顺义
  - 昌平  
# 形式二(了解):
address: [顺义,昌平]
```

> ### 小提示：
>
> 1. 书写yaml切记在 **`:`** 后面要加一个空格
>
> 2. 如果需要将多段yaml配置放在一个文件中，中间要使用 **`---`** 分隔
>
> 3. [一个yaml转json的网站](https://www.json2yaml.com/convert-yaml-to-json)，可以通过它验证yaml是否书写正确

### 3.3 资源管理方式

- **命令式对象管理**：直接使用命令去操作kubernetes资源

  ```bash
  kubectl run nginx-pod --image=nginx:1.17.1 --port=80 # 很少直接用，顶多测试的时候用
  ```

- **命令式对象配置**：通过命令配置和配置文件去操作kubernetes资源

  ```bash
  kubectl create/patch -f nginx-pod.yaml				 # 开发环境用
  ```

- **声明式对象配置**：通过apply命令和配置文件去操作kubernetes资源

  ```bash
  kubectl apply -f nginx-pod.yaml						 # 支持目录操作，能直接应用整个目录下的YAML
  ```

| 类型           | 操作对象 | 适用环境 | 优点           | 缺点                             |
| -------------- | -------- | -------- | -------------- | -------------------------------- |
| 命令式对象管理 | 对象     | 测试     | 简单           | 只能操作活动对象，无法审计、跟踪 |
| 命令式对象配置 | 文件     | 开发     | 可以审计、跟踪 | 项目大时，配置文件多，操作麻烦   |
| 声明式对象配置 | 目录     | 开发     | 支持目录操作   | 意外情况下难以调试               |

// TODO：https://gitee.com/yooome/golang/blob/main/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B-%E8%B0%83%E6%95%B4%E7%89%88/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B.md#3-%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86
