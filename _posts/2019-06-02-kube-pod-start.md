---
layout: post
title:  "Kubernetes学习: pod入门"
date:   2019-06-02 14:47:00 +0700
categories: [k8s]
---
#### 使用YAML创建一个pod描述文件
创建一个名称为`kubia-manual.yaml`的文件, 内容如下:
```yaml
apiVersion: v1 # 遵循v1版本的Kubernetes API
kind: Pod # 这个yaml描述的是一个Pod
metadata:
    name: kubia-manual # Pod的名称
spec:
    containers:
    - image: docker.io/hengyoush/kubia # 创建容器使用的镜像
      name: kubia #容器的名称
      ports:
      - containerPort: 8080 # 应用监听的端口, 指定端口是展示性的
        protocol: TCP
```

然后输入如下命令, 创建pod:
```
kubectl create -f kubia-manual.yaml
```

输出如下:
*`pod/kubia-manual created`*

输入`kubectl get pods`命令,查看刚刚创建的pod如下:
```
NAME           READY   STATUS              RESTARTS   AGE
kubia-manual   0/1     ContainerCreating   0          11s
...(略去其他pod)
```
#### 得到pod的完整定义
输入如下命令, 获得刚刚创建的pod的完整定义:
```
kubectl get po kubia-manual -o yaml
```

#### 查看应用程序的日志
一般来说我们可以登陆到pod正在运行的节点(使用minikube的话是虚拟机), 并且使用`docker logs`命令查看日志.但Kubernetes提供了一种更为简便的方法.
只需要如下命令:
```
kubectl logs kubia-manual
```

输出 *`Kubia server starting...`*.

如果pod中存在多个容器, 那么可以使用如下命令:
```
kubectl logs kubia-manual -c kubia
```

#### 向pod中发送请求
如何在不使用service的情况下访问pod呢?
可以使用本地端口转发, 命令如下:
```
kubectl port-forward kubia-manual 8888:8080
```

输出如下: 
```
Forwarding from 127.0.0.1:8888 -> 8080
Forwarding from [::1]:8888 -> 8080
```

此时打开另一个终端,访问本地的8888端口, `curl localhost:8888`,
此时得到响应: *`You've hit kubia-manual`*.