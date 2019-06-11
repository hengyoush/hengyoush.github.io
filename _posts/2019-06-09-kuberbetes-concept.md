---
layout: post
title:  "Kubernetes概念简介"
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
使用K@是非常直观的，你只需要使用K@ APIl来描述集群的期望状态，希望运什么应用程序，使用什么镜像，有多少个副本，希望什么网络和磁盘资源等类似声明式的方式来操控K@集群。

一旦你提供了所需的状态，*Kubernetes Control Plane*会自动的执行各种任务来让集群状态与你描述的一致，这包括：启动或重启容器，自动伸缩副本数量等，*Kubernetes Control Plane*由以下几个进程组成：
- **主节点-Kubernetes Master**：运行在集群中单个节点上的进程集合，包括：kube-apiserver，kube-controller-manager 和 kube-scheduler.
- 每个非主节点上运行着两个进程：
  - **kubelet**：与主节点通信。
  - **kube-proxy**：将整个K@网络映射在节点上。

![avatar](/static/img/K8sOverview-1.png)

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
探针是由Kubelet定时调度的诊断程序，Kubelet会调用一个由容器实现的Handler， 有三种Handler如下：

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

![avatar](/static/img/ReplicationController-1.png)

那么这是如何做到的呢?

首先来看一下ReplicaSet是如何定义的.
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # modify replicas according to your case
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

一个ReplicaSet的定义由以下部分组成:
- 一个*标签选择器(label selector)*: 它的主要作用是指定哪些Pod是属于它“管辖”的.
- 一个Pod数量: 即ReplicaSet所要保持的Pod的数量
- 一个Pod Template: 定义了这个ReplicaSet所创建Pod的定义.

通过标签选择器查看当前Pod的数量是否满足要求, 如果当前Pod多了, 那么会删除多出来的Pod, 如果少了那么会利用Pod Template创建新的Pod.


### Replication Controller
---
Replication Controller的作用与ReplicaSet相同, 只不过Replication Controller在标签选择上面的功能相对于ReplicaSet较弱, 具体体现在以下几个方面:

1. 假如有一个标签为`env = production` 和标签为 `env = dev` 的两个Pod, Replication Controller无法同时匹配这两个标签, 但是ReplicaSet可以.
2. Replication Controller无法仅基于标签的存在来匹配Pod, 而ReplicaSet可以.

所以在实际使用时, 不要使用Replication Controller, 使用ReplicaSet更加合适.

一个Replication Controller的定义示例如下所示：
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
可以看到，ReplicationController的定义除了selector的标签处理方式以外与ReplicaSet的定义几乎一样.

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

如下图所示：

![avatar](/static/img/Service-1.png)

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
在这里, 我们定义了一个Service：

所有标签`app = MyApp`的Pod将会是这个Service的一部分;使用的协议为TCP,并且所有发到Service的80端口的流量将会导入至Pod的9376端口.

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
举个例子, 有一个服务“Redis-Master“, ClusterIP是10.0.0.11, 端口暴露在6379, 那么之后创建的Pod中将会有如下环境变量:
```properties
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```
> 注意:当你使用环境变量这种方式发现服务时， 必须保证Service启动完成之后，Pod再启动，因为只会在Pod启动的时候将环境变量注入到Pod中。

#### DNS
可以看到，使用环境变量的方式非常不灵活，幸好K@为我们提供了另一种方式：使用一个提供DNS服务的附加组件。
DNS服务可以监视K@ API， 如果有新的Service建立，那么会在DNS record中新增一条记录，这样K@集群中的所有Pod都可以使用DNS Name来连接到Service。

例如，如果你有一个Service叫做“My-Service”，它在“my-ns”命名空间中，在“my-ns”命名空间中的Pod可以通过`"my-service"`这个DNS名称解析到IP地址，当然也可以使用`"my-service.my-ns"`。如果是其他命名空间的Pod则需要使用`"my-service.my-ns"`这个名称。

### 发布服务
---
以上介绍的都是在集群内部发现服务的方式，我们也会需要将服务暴露给集群外部，这时候可以使用`ServiceType`指定Service的类型，以上介绍的是默认的Service类型：`ClusterIP`，还有其他类型如下：
- **`ClusterIP`**: 在集群内部暴露一个IP，这个IP只能由集群内部来访问，这是默认的ServiceType。
- **`NodePort`**: 在集群中的每个工作节点Node上暴露一个静态端口--`NodePort`, 这样所有外部请求到地址：`<NodeIP>:<NodePort>`的请求将会自动转发到一个自动创建的`ClusterIP`类型的Service。
- **`LoadBalancer`**：由云服务提供商提供的负载均衡器暴露服务。

> Port、TargetPort、NodePort的区别：Service中定义的port指的是Service的虚拟端口，而TargetPort指的是容器接收请求的端口，NodePort指的是Node上开放的静态端口，每个发到该端口上的请求将会被路由至一个自动创建的ClusterIP类型的Service上。

### 服务发现相关总结
一个Service的背后是一群Pod，每个Pod提供相同的功能，发往Service上的请求会被负载均衡至每个后端的Pod。实际上Service并不直接与Pod打交道，它不断的使用标签选择器持续计算Pod的结果并且将结果POST至一个EndPoint对象（这个ENdPoint对象的名字与Service的名字相同），当一个Pod挂掉，它自动从EndPoint中移除，当一个新的Pod加入，Service的标签选择器会发现它并且将它加入endpoint中。

### Ingress--统一管理外部访问
外部访问的解决方案已经有了，比如NodePort和LoadBalance，但是这些方案都有一个缺点那就是对于每一群Pod都要创建一个Service，那么当Service的数量增长，如何进行有效的管理？比如说使用一个IP地址和不同的路径就可以访问所有Service？

K@为我们提供了一种方式：Ingress。
Ingress将外网与service之间的HTTP路由暴露， Ingress会根据请求的主机名和路径决定请求到达的服务, 还可以提供会话亲和性等功能. 

Ingress等定义实例如下:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: kubia.example.com
http: paths:
      - path: /
        backend:
          serviceName: kubia-nodeport
          servicePort: 80
```
 根据上图的定义, Ingress将`kubia.example.com`映射到我们的服务, 并且将所有请求发送至端口80上的`kubia-nodeport`服务.

 根据上图的文件创建好Ingress资源后, 可以通过`kubectl get ingresses`获取Ingress的IP地址.然后可以在你的host文件中将IP地址和你文件中定义的host名称对应起来.

> 下图为Ingress的工作流程, 客户端首先对kubia.example.com进行DNS查找, DNS服务器返回了Ingress Controller的IP, 客户端然后通过这个IP对Ingress Controller发送HTTP请求, Controller通过客户端请求的路径匹配到了后端的一个服务, 然后通过与该服务相关联的EndPoint对象查看pod IP, 并将客户端的请求转发给其中一个pod.
![avatar](/static/img/Ingress-1.png)

你也可以通过Ingress暴露不同的服务, 着这当然需要你配置不同的主机或路径,如下所示:
```yaml
spec: rules:
  - host: foo.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: foo
          servicePort: 80
  - host: bar.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: bar
          servicePort: 80
```
如上所示, 根据请求中的Host头, Controller接收到的请求将会被转发到foo或bar服务, 取决于其请求的host. DNS需要将`foo.example.com`和`bar.example.com`都映射到Ingress Controller的IP地址.

## 存储
---
### Volume

由于Pod从K@的概念上来说不会一直存在，因为某种原因挂掉的话，kubelet将会重启它但是容器内部的文件会丢失，另外当pod内部的容器如果需要共享文件的话也需要K@的存储功能的支持。K@的`Volume`解决了上述问题。

Volume的生命周期与Pod中的容器无关，而与Pod本身相关，当Pod被删除时，Volume中的内容才会丢失。

### 持久化存储
下面介绍一种存储，它是独立于Pod的生命周期之外的存储方式：

**`PersistentVolume`**(PV)是集群中的存储资源，由管理员或者由Storage Class动态分配，它和node工作节点一样是集群中的资源。

**`PersistentVolumeClaim`**(PVC):由用户发起的存储资源请求，就和Pod消费节点资源一样，PVC消耗的是PV资源。

如下是PV与PVC与用户管理员之间的关系:

![avatar](/static/img/PVPVC-1.png)

首先由集群管理员创建存储,然后通过向K@ API传递PV声明创建PV.
然后需要使用存储的用户只需要创建一个PVC, 然后K@会找到一个具有足够容量的PV将其置于访问模式, 并将PVC绑定到PV.然后用户创建一个pod并通过volume配置引用PVC.

bar.example.com
## 配置
---
### 资源分配

为了实现资源（这里主要指CPU和内存）被有效利用， K@采用Requests和Limit两种限制类型来对资源进行分配。

- Request：容器使用的最小资源需求，只有当工作节点的可用资源量>=Pod请求资源量时才允许将该Pod调度到该节点。
- Limit：Pod使用资源最大值，设为0表示无上限。

Request能够保证Pod有足够的资源来运行，而Limit则是防止某个Pod无限制地使用资源，导致其他Pod崩溃。两者之间必须满足关系: 0<=Request<=Limit<=Infinity (如果Limit为0表示不对资源进行限制，这时可以小于Request)

### 资源抢占

![avatar](/static/img/Resource-1.png)

如上图，在一个4U4G的Node上，部署了四个Pod，每个Pod的Request为1U1G，Limit为2U2G，很有可能会出现Pod占用的资源大于1U，那么此时会出现资源抢占，对于资源抢占，K@根据资源能不能进行伸缩进行分类，分为可压缩资源和不可以压缩资源。CPU资源--是现在支持的一种可压缩资源。内存资源和磁盘资源为现在支持的不可压缩资源。

假设四个Pod同时负载变高，CPU使用量超过1U，这个时候每个Pod将会按照各自的Request设置按比例分占CPU调度的时间片。在示例中，由于4个Pod设置的Request都为1U，发生资源抢占时，每个Pod分到的CPU时间片为1U/(1U×4)，实际占用的CPU核数为1U。在抢占发生时，Limit的值对CPU时间片的分配为影响，在本例中如果条件容器Limit值的设置，抢占情况下CPU分配的比例保持不变。

对于不可压缩的资源，则按照优先级的不同进行Pod的驱逐（Evict）。

对于不可压缩资源，如果发生资源抢占，则会按照优先级的高低进行Pod的驱逐。驱逐的策略为： 优先驱逐Request=Limit=0的Pod，其次驱逐0<Request<Limit<Infinity (Limit为0的情况也包括在内)。 0<Request==Limit的Pod的会被保留，除非出现删除其他Pod后，节点上剩余资源仍然没有达到Kubernetes需要的剩余资源的需求。

由于对于不可压缩资源，发生抢占的情况会出Pod被意外Kill掉的情况，所以建议对于不可以压缩资源(Memory，Disk)的设置成0<Request==Limit。


### 利用ConfigMap解耦配置
ConfigMap本质上是一个键值对映射, 值可以是简单的字面量也可以是一个配置文件.
此外应用对于ConfigMap是无感知的, 映射通过环境变量或者volume文件的形式传递给容器, 可以通过`${ENV_VAR}`的语法引用.

![avatar](/static/img/ConfigMap-1.png)

对于不同环境下, Pod的定义相同但是配置不相同的情况, ConfigMap也非常方便, 只要在不同的命名空间下定义不同的ConfigMap即可.

![avatar](/static/img/ConfigMap-2.png)
