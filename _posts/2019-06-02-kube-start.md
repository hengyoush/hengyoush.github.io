---
layout: post
title:  "使用minikube在Kubernetes上运行第一个应用"
date:   2019-06-02 13:47:00 +0700
categories: [k8s, docker]
---

## 准备工作

### 安装docker
docker可以从官网下载, 链接地址: https://hub.docker.com/?overlay=onboarding

如果是linux可以使用apt-get或者yum直接安装.

### 安装VirtualVM
下载地址: https://download.virtualbox.org/virtualbox/5.2.18/VirtualBox-5.2.18-124319-OSX.dmg

### 安装kubectl
执行以下命令:
```
brew install kubernetes-cli
```
安装成功执行:
```
kubectl version
```

### 安装minikube
minikube是一个可以构建单节点Kubernetes集群的工具,对于测试Kubernetes和开发应用都非常有帮助.

(由于使用minikube构建集群时需要下载一些镜像,但是由于国内的网络环境问题,我们可以使用阿里改造后的minikube.)

在终端执行以下命令:
```
curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v0.28.1/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

下载完成即可.

## 正式开始

#### 准备一个应用

### 启动minikube
使用以下命令启动minikube:
```
minikube start --registry-mirror=https://registry.docker-cn.com
```
出现如下命令行输出可认为启动成功:
```
😄  minikube v1.1.0 on darwin (amd64)
✅  using image repository registry.cn-hangzhou.aliyuncs.com/google_containers
🔥  Creating virtualbox VM (CPUs=2, Memory=2048MB, Disk=20000MB) ...
🐳  Configuring environment for Kubernetes v1.14.2 on Docker 18.09.6
🚜  Pulling images ...
🚀  Launching Kubernetes ... 
⌛  Verifying: apiserver proxy etcd scheduler controller dns
🏄  Done! kubectl is now configured to use "minikube"
```

现在我们还没有在这个集群中运行任何容器,接下来让我们部署应用.

### 部署应用
这里我们采用一种简单的方式在Kubernetes中部署应用.
在命令行中输入以下命令:
```
kubectl run kubia --image=docker.io/hengyoush/kubia --port=8080 --generator=run/v1
```

可以看到如下输出:<br/>
*replicationcontroller "kubia" created*

这里我们使用kubectl run命令, --image指定我们需要使用的容器镜像, --port告诉k8s容器正在监听8080端口, --generator让k8s创建一个*Replication Controller*.

接下来输入以下命令查看容器是否成功部署:
```
kubectl get pods
```

如果输出如下所示,那么恭喜你,部署成功了:
```
NAME          READY     STATUS    RESTARTS   AGE
kubia-d58dn   1/1       Running   0          29s
```

如果status那一行是`Pending`, 那么稍等一会再看一下,因为镜像的下载需要相应时间.

### 从外部访问应用
每个pod都有自己的IP地址, 但是这个地址是集群内部的, 无法从集群外部访问, 要让其可以访问, 我们需要一个*LoadBalancer*类型的服务. 通过创建*LoadBalancer*类型的服务, 让它可以从外部访问.

运行如下命令:
```
kubectl expose rc kubia --type=LoadBalancer --name kubia-http
```
如果输出如下,说明创建成功:<br/>
*`service "kubia-http" exposed`*

接下来运行如下命令,查看刚刚创建的服务暴露出的ip以及端口:
```
kubectl get svc
```
输出如下:
```
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP      10.96.0.1     <none>        443/TCP          18h
kubia-http   LoadBalancer   10.105.25.8   <pending>     8080:32055/TCP   3m
```

(实际上Minikube不支持LoadBalancer类型的服务, 可以运行`minikube service kubia-http`获取可以访问的ip和端口)

### 水平伸缩应用
我们现在只有一个pod副本, 我们希望把副本数扩大到三个.

输入如下命令:
```
kubectl scale rc kubia --replicas=3
```

输出如下:
*`replicationcontroller "kubia" scaled`*

接下来我们看一下pod的情况.

运行`kubectl get pods`命令输出如下:
```
NAME          READY     STATUS              RESTARTS   AGE
kubia-7rvr6   0/1       ContainerCreating   0          23s
kubia-d58dn   1/1       Running             0          22m
kubia-tkqbm   0/1       ContainerCreating   0          23s
```

可见kubia的pod副本增加到了三个,其中两个还在启动.

接下来我们看一下水平伸缩后负载均衡的效果:
```
➜  ~ curl http://192.168.99.104:32055/
You've hit kubia-7rvr6
➜  ~ curl http://192.168.99.104:32055/
You've hit kubia-tkqbm
➜  ~ curl http://192.168.99.104:32055/
You've hit kubia-tkqbm
➜  ~ curl http://192.168.99.104:32055/
You've hit kubia-d58dn
```

可以看到,请求被随机的分发到了不同的pod.

### dashboard

输入如下命令, dashboard将在默认浏览器中打开:
```
minikube dashboard
```