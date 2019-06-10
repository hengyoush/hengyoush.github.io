---
layout: post
title:  "Kubernetes简介以及基本使用"
date:   2019-06-09 13:47:00 +0700
categories: [k8s]
---

## 背景和k8s的出现
---
现如今, 单体应用正逐渐分解成为小的独立的组件, 我们称为微服务. 微服务之间彼此解耦, 它们之间可以独立开发,测试,部署. 随着服务个数逐渐增多, 配置管理系统的正常运行变得越来越困难.

(比如说我们想要获得足够高的资源利用率, 把服务部署在什么地方需要人为的操作, 包括监视, 故障处理, 自动调度等, Kubernetes给了我们一套解决方案)

Kubernetes抽象了硬件基础设施, 使得对外暴露的是一个资源池, 这让我们在部署应用时不用关注底层的服务器. 使用Kubernetes部署应用时, 它会为每个组件选择一个合适的服务器, 而且通过内置的服务发现机制可以使每个组件互相发现并且实现通信.

简单来说, 容器技术给了我们在发布和更新应用方面上的方便和快捷, 而Kubernetes则帮助我们决定这些容器在哪里运行何时运行并且帮助它们找到所需的资源.

## K@的组成
---

## Pod
---
Pod是Kubernetes中最为重要的核心概念, K@的其他对象仅仅是在管理,暴露Pod或者被Pod使用.

一个Pod是你能创建部署的最小最简单的K@对象, 一个Pod封装了一组容器, 一个唯一的IP, Docker是在K@中应用最广泛的*容器运行时*, 但K@同时也支持其他容器运行时

![avatar](/static/img/Pod-1.png)

如上所示, Pod是一组容器(当然也可以只有单个容器), 而且一个Pod总是部署在一个工作节点上.
一个Pod就像是一个逻辑主机, 运行在Pod中的每一个进程就像是运行在物理机上一样, 只是每个进程都封装在容器之中.

那么Pod如何管理多个容器的呢?
举个例子，假设我们有一个Web服务，一个是Web服务器，用来接收用户请求，显示页面，而这个页面由另一个进程加载，将html页面放在共享volume中，这时我们可以在一个
Pod中放置这两个容器，因为它们是紧耦合的。

![avatar](/static/img/Pod-2.svg)


#### Pod中的网络
---
每一个Pod都会被分配一个IP地址，同一个Pod内的容器可以通过localhost相互通信，而Pod间的通信则需要知道IP地址。



#### Pod中的存储
---
每一个Pod中都可以指定多个volume，每个volume都可以被Pod内的所有容器共享，Volume也可以在Pod重启时持久化数据。


#### 管理Pod
---
一般情况下，不需要手动管理Pod，K@中有一种类型的资源叫做*Replication Controller*，由RC进行管理Pod的创建，删除，调度等，当然如果你想的话
也可以手动建立Pod，不过手动建立的Pod在停止时不会自动重启，也失去了很多K@提供的特性，所以尽量不要手动管理Pod。



#### Pod Template
---
所有K@内的资源都以json或者yaml的形式表示，Pod也不例外，下面是一个Pod yaml的示例：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```
RC等Controller就是根据Pod Template进行Pod的创建，对Pod Template的任何修改都不会影响已经创建的Pod，只是后续创建的Pod会根据新Template创建。



#### Pod的状态
---
Pod的状态可以通过如下命令查看：
```
kubectl get pods
```

| Value | 描述 |
| ------ | ------ |
| `Pending` | Pod正在创建中，但是Pod中的某些镜像还没有下载 |
| `Running` | Pod已经被调度而且所有Pod内部的容器已经被创建 |
| `Succeeded` | Pod内的容器已经成功终止并且不会被重新启动 |
| `Failed` | Pod内的容器存在终止失败的 |
| `Unknown` | 无法获取Pod的状态 |



#### 容器探针
---
K@通过容器探针来检测容器的状态从而判断是否需要重启Pod。

<br>
探针是由*Kubelet*定时调度的诊断程序，Kubelet会调用一个由容器实现的*Handler*， 有三种Handler如下：

- ExecAcion: 在容器内部执行一个指定的命令，如果命令执行成功那么探针诊断的结果则判断为成功。<br>
- TcpSocketAction： 针对容器的Ip和特定的端口进行一次TCP检查，如果端口开放则认为诊断成功。 <br>
- HTTPGetAction ：针对容器的Ip和特定的端口和路径进行一次Http Get请求，如果返回状态码返回大于等于200小于400则认为诊断成功，否则认为诊断失败。<br>

对于探针的类型，有如下两种：
1. `livenessProbe`：用于确定容器是否处于`Running`状态，如果诊断失败，那么kubelet会停止该容器并且根据`Restart Policy`进行下一步的操作。
2. `readinessProbe`：用于确定该容器是否准备好接收请求，如果诊断失败，那么`endpoinrs controller`会将所有匹配该Pod的Service的Endpoint中移除该Pod的Ip地址。

*RestartPolicy*

RestartPolicy用于确定容器因为存活探针诊断失败而被kill之后的处理方式，有如下三种：
1. Always 默认选项，适用于*ReplicationController, ReplicaSet, 或者 Deployment*这种会自动创建Pod的。
2. Never
3. OnFailure



## Controller
---
上面说了， 我们一般不会直接管理Pod， 而是通过*Controller*这种资源进行管理， Controller主要有三种：*ReplicaSet，Replication Controller，Deployment*

### ReplicaSet
---
ReplicaSet的主要作用是维持Pod集合的稳定运行, 即通常情况下保持一个稳定的数量运行.

那么这是如何做到的呢?

首先来看一下ReplicaSet是如何定义的.

#### ReplicaSet的组成
一个ReplicaSet的定义由以下部分组成:
- 一个*标签选择器(label selector)*: 它的主要作用是指定哪些Pod是属于它“管辖”的.
- 一个Pod数量: 即ReplicaSet所要保持的Pod的数量
- 一个Pod Template: 定义了这个ReplicaSet所创建Pod的定义.

通过标签选择器查看当前Pod的数量是否满足要求, 如果当前Pod多了, 那么会删除多出来的Pod, 如果少了那么会利用Pod Template创建新的Pod.


### Replication Controller
---
Replication Controller的作用与ReplicaSet相同, 只不过Replication Controller在标签选择上面的功能相对于ReplicaSet较弱, 具体体现在以下几个方面:

1. 假如有一个标签为`env = production` 和标签为 `env = dev` 的两个Pod, Replication Controller无法同时匹配这两个标签, 但是ReplicaSet可以.
2. Replication Controller无法仅基于标签的存在来匹配Pod, 而ReplicaSet可以.

所以在实际使用时, 不要使用Replication Controller, 使用ReplicaSet更加合适.

### Deployment
---
Deployment是一种高阶资源, 可以更加容易的更新应用程序,用于部署应用程序而且以声明的方式升级应用.
当创建一个Deployment时, 一个ReplicaSet会被自动创建.它们与Pod之间的关系如下图所示:

![avatar](/static/img/Deployment-1.png)

可以看到, Deployment并不是直接管理Pod, 这看上去有些不直观, 为什么我们还要增加“复杂度”, 在ReplicaSet上再加一层? 原因主要是由于Deployment要支持滚动升级, K@的方式(之前的方式)是通过再创建一个ReplicationController并协调两个Controller使它们根据彼此不断修改, 然而这个过程可能会造成干扰所以需要另外一个资源来协调.

#### Deployment的定义
如下是一个Deployment的定义:
```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
```
可以看到除了apiVersion、kind、metadata等必须的选项之外, 在spec中定义了一个Pod Template, 包括Pod的数量以及其镜像.

### DaemonSet
---
当你希望在K@的所有工作节点上都运行一个Pod实例, 那么你可以使用DaemonSet.
下图是ReplicaSet与DaemonSet的对比图.

![avatar](/static/img/DaemonSet-1.png)

当然也可以使用DaemonSet只在特定的节点上运行Pod,这个功能需要结合标签选择器来实现.

## Service
---
K@的Pod的生命是短暂的, 每一个Pod都有自己的IP地址, 但是可能一个Deployment中当前运行的Pod和下一秒运行的Pod是不同的, 那么这会导致一个问题: 如果一些服务(比如说一些前端服务)依赖于另一些服务(比如说后端服务), 那么那些前台服务如何发现并且跟踪后端服务IP地址的变化呢?

这就需要一个服务发现机制, 我们不需要自己手动引进一个服务发现工具, K@已经给了我们开箱即用的解决方案.这就是**Services**.

有了Service, 可以达到服务(指应用服务)之间的解耦, 比如说一个无状态的数据处理后端，它使用3个副本运行,前端服务并不关心它们使用的是哪个后端,虽然组成后端集的实际pod可能会发生变化，但前端客户端不应该知道这一点，也不应该自己跟踪后端集。

### Service的定义
---
Service的定义实例如下所示:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```
在这里, 我们定义了一个Service, 代理标签`app = MyApp`的Pod, 使用的协议为TCP, 并且所有发到Service的80端口的流量将会导入至Pod的9376端口.

当然也可以定义不带selector的Service, 如下所示:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

此时需要为这个Service指定一个关联的EndPoint对象, 定义如下所示:
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```

如上所示, 所有发往Service的流量将会被路由至`192.0.2.42:9376(TCP)`.


所有Service被创建之后将会具有一个ClusterIP, 这个IP是虚构的, 所有被发往这个IP的流量, 都会被路由至这个Service所定义的Pod中去.另外还需要注意的是Service仅在集群内部访问使用.

### 服务发现
---
Service创建之后我们就可以通过一个单一的IP地址访问Pod, 即使后端的Pod改变了, 但是通过Service的IP我们仍然可以访问到服务.但是我们如何知道Service的IP和端口? K@为我们提供了发现Service的IP和端口的方式.

#### 通过环境变量发现服务
pod开始运行的时候, K@会初始化一系列的环境变量指向现在存在的服务, 如果我们创建的服务早于pod的创建, 那么pod的环境变量可以根据环境变量获得服务的IP地址和端口号.

可以执行如下命令获取pod内部的环境变量:
```
kubectl exec ${pod名} env
```




