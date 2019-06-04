---
layout: post
title:  "docker简介以及基本使用"
date:   2019-06-04 13:47:00 +0700
categories: [docker]
---

## What is docker？
下面是官方的定义：
> Docker is a platform for developers and sysadmins to develop, deploy, and run applications with containers.

> Docker是一个供开发者和系统管理员使用容器（container）开发、部署和运行应用的平台

#### 什么是容器？
说到容器我们就不得不提到**镜像（image）**，因为没有镜像就没有容器。

**镜像**是一个可执行的包，并且包含了一个应用以及其所需要的所有运行时环境，假如你的应用是Java，那么它就包含了一个jre，它的依赖，以及所需要的环境变量。

一个**容器**就是一个镜像的运行实例。每个容器是一个单独的进程，且它们之间相互隔离，只能访问授予它们的资源。

docker则是一个开源容器运行引擎。

#### 容器与虚拟机的相同与不同之处
容器和虚拟机都提供了一定程度的资源隔离，环境隔离功能，但是它们的区别仍然十分明显，比如：
1. 容器是轻量级的，而虚拟机则相对“笨重”。
2. 一台主机上的容器共享OS内核，而虚拟机则使用完全独立的OS内核。
3. 基于docker的镜像是分层的，可以重用。

*原因：容器运行于容器引擎之上，且容器引擎直接运行于Host OS之上，而虚拟机则运行于Hypervisor之上，Hypervisor直接管理虚拟化后的硬件资源（大部分Hypervisor如此），这就造成了容器的隔离实际上是要比虚拟机的隔离程度差一些的。但是有利也有弊，虚拟机启动时需要启动所有它自己的系统服务，而容器则不需要，因为它直接使用宿主机的内核所以不需要启动任何服务，所以虚拟机所消耗的资源比容器消耗的要多得多，导致容器启动十分快速，而虚拟机相对慢很多。*

#### 为什么容器可以实现隔离
Linux Namespaces：让每个进程只看到系统的“个人视角”，包括文件系统，网络接口，主机名等等。

Linux Control Groups（cgroups）：限制了一个进程可以消耗的系统资源，包括CPU，内存，网络带宽等等。

## 为什么使用Docker
TODO

## 创建镜像
Docker通过从*Dockerfile*的文件中读取指令构建镜像。下面来介绍如何使用Dockerfile构建镜像。

#### Dockerfile
首先创建一个helloworld镜像，在任意目录下新建`Dockerfile`，然后在`Dockerfile`内写入以下内容：
```Dockerfile
FROM ubuntu:latest

CMD ["/bin/echo", "hello world"]
```
上面这个例子的第一行`FROM`说明新镜像的基础镜像是`ubuntu`，
然后设置了一个默认执行命令即`/bin/echo` `hello world`。

在当前目录下执行以下命令：
```
dokcer image build . -t helloworld 
```
`.`指明了这次构建镜像的`context`是当前目录，`-t`为这个镜像打上了一个tag(`name:tag`或`tag`，不指定`tag`默认是`latest`)。

输出如下：
```
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM ubuntu:latest
latest: Pulling from library/ubuntu
9fb6c798fa41: Pull complete
3b61febd4aef: Pull complete
9d99b9777eb0: Pull complete
d010c8cf75d7: Pull complete
7fac07fb303e: Pull complete
Digest: sha256:31371c117d65387be2640b8254464102c36c4e23d2abe1f6f4667e47716483f1
Status: Downloaded newer image for ubuntu:latest
 ---> 2d696327ab2e
Step 2/2 : CMD /bin/echo hello world
 ---> Running in 9356a508590c
 ---> e61f88f3a0f7
Removing intermediate container 9356a508590c
Successfully built e61f88f3a0f7
Successfully tagged helloworld:latest
```

接着我们使用`docker image ls`命令查看我们刚刚创建的镜像如下：
```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
helloworld          latest              e61f88f3a0f7        3 minutes ago       122MB
ubuntu              latest              2d696327ab2e        4 days ago          122MB
```
可以看到镜像构建完成了，那么接下来我们运行这个镜像：
```
docker run helloworld
```
输出：
```
hello world
```

#### Dockerfile基本命令参考
命令 | 说明 | 例子
-|-|-
FROM | Dockerfile的第一个指令 | `FROM ubuntu`
COPY | 拷贝文件到容器内部的指定路径 | `COPY .bash_profile /home`
ENV | 给容器设置环境变量 | `ENV HOSTNAME=host`
RUN | 执行一个命令 | `RUN apt-get update`
CMD | 容器的默认执行命令 | `CMD ["/bin/echo", "hello world"]`
EXPOSE | 指示这个容器监听的端口 | `EXPOSE 8080`

#### 使用maven插件创建Java应用镜像

##### docker-maven-plugin

##### Jib


##### multi-stage build

## 核心概念

#### 存储

#### 网络

