---
title:      "kubernetes overview"
date:       2018-06-06
tags:
    - 架构
    - k8s
---

# kubernetes的组件

## master上的组件

1. kube-apiserver

    组件暴露了Kubernetes的API接口。它是k8s控制平面的前端。它被设计成水平扩展的，也就是可以通过部署多个实例来实现。

2. etcd

    存储所有的集群数据。

3. kube-scheduler

    监控新创建的并且还没有被分配到节点上的pods，为这些pods选择一个节点让他们运行。
    
    调度决策需要考虑的因素包括个人的和公共的资源需求，硬件/软件/策略约束，数据本地化，工作负载干扰以及运行期限。

4. kube-controller-manager

    在master上运行的控制器(通过apiserver来监控集群的状态)。逻辑上，每个控制器都是独立的进程，但为了降低复杂度，他们都运行在同一个进程内。这些控制器包括：

    - Node Controller: 当节点下线时负责通知并作出响应
    - Replication Controller：为系统中的每个复制控制器对象维护正确数量的pods
    - Endpoints Controller: 填充endpoints对象(添加services和pods)
    - Service Account & Token Controllers: 为新的namespace创建默认账号和API访问token

## node上的组件

1. kubelet

    在集群的每个节点上运行的代理。它确保容器运行在pod中。

    kubelet通过一套PodSpecs提供多种机制并确保在PodSpecs中的容器正常运行。kubelet不会管理那些不是由k8s创建的容器。

2. kube-proxy

    kube-proxy通过维护主机上的网络规则和执行链接转发使k8s的服务抽象成为可能。

3. container runtime

    支持Docker，rkt等容器技术。

# kubernetes Master-Node 通信

## Cluster -> Master

从cluster到master的所有通信路径都在apiserver终止(master上的其他组件都没有被暴露远程服务)。apiserver监听远程连接(https:443)并且开启了一种或多种形式的客户端验证。

节点也应该提供公共根证书，这样他们可以安全的链接到apiserver。

pods想要链接到apiserver也可以通过服务账号安全的做到，当pod被实例化时，kubernetes会自动把证书和token注入其中。kubernetes服务配置了一个虚拟IP地址，该IP地址被重定向（通过kube-proxy）到apiserver上的HTTPS端点。

## Master -> Cluster

从master(apiserver)到cluster主要有两条通信线路。第一种是从apiserver到kubelet，第二种是从apiserver到任意的node、pod或service，通过apiserver的代理功能。

1. apiserver -> kubelet

    链接从apiserver到kubelet用于：

    - 获取pods的日志
    - 通过kubectl连接正在运行的pods
    - 提供kubelet的端口转发功能

    这些连接最终到达kubelet的HTTPS endpoints。默认情况下，apiserver不会验证kubelet的证书,如果运行在公网上，这可能会受到中间人攻击。

2. apiserver -> nodes, pods, services

    从apiserver到node, pod和service默认是HTTP通信的，因此不需要认证和加密。

# kubernetes的对象

kubernetes的对象在系统中是持久化的实体。系统使用这些实体来展示集群的状态。它可以描述的有：

- 哪些集装箱的应用在哪个节点上运行
- 应用程序可使用的资源
- 应用程序如何运行的策略，例如重启策略，升级和失败容忍

k8s对象是一个“意图的记录”，一旦创建了对象，系统将不断的工作以确保对象的存在。通过创建对象，有效的告诉了系统集群的理想状态。

## 对象说明书和状态

每个对象包含两个嵌套的对象字段来管理对象的配置：对象说明书和对象状态。你必须提供说明书以描述对象的理想状态-希望对象拥有的特征。状态描述了对象真实的状态。

例如，Deployment是个可以对象，它可以显示一个应用运行在集群中。当创建一个Deployment，需要设置说明书来指定你想要3个应用副本同时运行。k8s系统读取Deployment说明书并且启动3个实例。如果任意一个实例失败(状态改变)，系统通过进行修正来响应说明书和状态之间的差异，在这个例子中，启动一个replacement实例。

## 描述对象

当使用kubectl创建对象时，需要提供一个.yaml格式的文件用于描述对象信息。必须的字段有：
- apiVersion: 使用哪个API版本来创建对象
- kind: 对象的类型
- metadata: 帮助唯一识别对象的数据，包括name，UID和namespace
- spec: 每个对象有不同的格式，具体参见API文档

一个Deployment对象的yaml描述：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

# Pods

Pod是k8s的基本组成部分——创建和部署k8s对象的最小最简单的单元。一个Pod表示集群中运行的一个进程。

一个Pod封装了一个或多个应用层容器，存储资源，唯一IP地址，以及管理容器如何运行的选项。

一个Pod代表一个部署单元：一个单独的应用实例，它可能由一个或几个紧密耦合和共享资源的容器组成。

Docker是Pod中最常见的容器运行环境，但是Pod还支持其他的container runtimes。

Pods设计成支持多个合作进程(containers)从而形成有凝聚力的服务单元。集群里在相同物理机或虚拟机中的Pod里的容器会自动的共同协作和共同调度。这些容器能共享资源和依赖关系,相互通信，协调何时以及怎样终止他们。

## Pod Lifecycle

1. Pod phase

    pod包含以下几个阶段：

    - Pending: Pod已经被系统接收，但容器还没有创建。这包括在调度之前的时间以及下载镜像的时间，这可能需要花一些时间
    - Running：Pod已经被分配到了一个节点上，并且所有容器已创建。至少有一个容器在运行或正在启动或重启
    - Succeeded：Pod中所有容器已经成功终止并且不再重启
    - Failed：所有容器已经终止，并且至少有一个容器是异常终止的
    - Unknown：Pod的状态无法获取，通常由于通信问题导致的

2. Pod conditions

    Conditions用于描述Pod更详细的状态:

    - PodScheduled: pod正处于调度中，但还没有被分配到具体node上，ip也没有绑定
    - Initaized：指定的init container已创建完毕
    - Ready：容器创建成功，可提供服务
    - Running：所有容器正常运行