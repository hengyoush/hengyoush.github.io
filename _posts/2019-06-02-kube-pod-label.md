---
layout: post
title:  "Kubernetes学习: 使用label管理pod"
date:   2019-06-02 15:39:00 +0700
categories: [k8s]
---

## 困境
当pod的数量增多(我们会存在众多的服务), 并且每个服务会部署不同的版本时, 如果没有一个方便管理某一类pod的方法那将是运维地狱, 接下来我们将介绍一种通过标签--label管理pod的方法.*(实际上不仅仅是pod, 其他类型的资源也可以有label)*

## label的基本使用

#### label介绍
一个pod可以有多个不同的标签, 而且不仅仅是pod, 其他类型的资源也可以有label.
针对pod我们可以设置两个标签, 分别是 *app* 和 *rel*, 代表它是哪个应用和其版本.

#### 如何创建label
我们首先创建一个上一篇文章中yaml文件的升级版, 文件名为`kubia-manual-with-labels.yaml`.
内容如下:
```yaml
apiVersion: v1 # 遵循v1版本的Kubernetes API
kind: Pod # 这个yaml描述的是一个Pod
metadata:
    name: kubia-manual-v2 # Pod的名称
    labels: 
      creation_method: manual
      env: prod
spec:
    containers:
    - image: docker.io/hengyoush/kubia # 创建容器使用的镜像
      name: kubia #容器的名称
      ports:
      - containerPort: 8080 # 应用监听的端口, 指定端口是展示性的
        protocol: TCP
```
在这里, 我们给这个pod创建了两个label, 一个是`creation_method`, 还有一个是`env`.
现在来创建pod: 
```
kubectl create -f kubia-manual-with-labels.yaml
```

创建完成后,使用如下命令查看pod以及其label的信息:
```
kubectl get po --show-labels
```

输出如下:
```
NAME              READY   STATUS    RESTARTS   AGE    LABELS
kubia-manual-v2   1/1     Running   0          47s    creation_method=manual,env=prod
```

如果只对env这个标签感兴趣, 那么输入如下命令即可:
```
kubectl get po -L env
```

输出如下:
```
NAME              READY   STATUS    RESTARTS   AGE    ENV 
kubia-manual-v2   1/1     Running   0          52s    prod
```

#### 修改标签
首先让我们为为一个pod添加label.
输入如下命令:
```
kubectl label po kubia-manual creation_method=manual
```
这样我们就为`kubia-manual`这个pod添加了一个label.

现在让我把`kubia-manual-v2`的`env`标签修改为`debug`.
输入如下命令:
```
kubectl label po kubia-manual-v2 env=debug --overwrite
```

现在让我们查看label修改的效果:
```
NAME              READY   STATUS    RESTARTS   AGE     LABELS
kubia-d58dn       1/1     Running   0          146m    run=kubia
kubia-manual      1/1     Running   0          43m     creation_method=manual
kubia-manual-v2   1/1     Running   0          7m13s   creation_method=manual,env=debug
```

## 标签选择器--label selector

#### 基本用法示例
列出所有creation_method=manual的pod:
```
kubectl get po -l creation_method=manual
```

列出所有包含env标签的pod:
```
kubectl get po -l env
```

列出所有不包含env的pod(注意有''号)
```
kubectl get po -l '!env'
```

列出creation_method不为manual的pod:
```
kubectl get po -l 'creation_method!=manual'
```

列出所有env为prod和debug的pod:
```
kubectl get po -l 'env in (prod,debug)'
```

列出所有env不为prod和debug的pod:
```
kubectl get po -l 'env notin (prod,debug)'
```

使用多个条件时用“,”分割:
```
kubectl get po -l 'env notin (prod,debug)', reation_method=manual
```

#### 使用标签分类工作节点
不仅仅是pod, 工作节点也可以加label.
命令如下:
```
kubectl label node minikube gpu=true
```
这样就给minikube这个节点加上了一个label为`gpu=true`的标签.

我们现在想将pod只调度到含有该节点的node上, 可以在创建pod的yaml文件中做如下改动:
```yaml
apiVersion: v1 # 遵循v1版本的Kubernetes API
kind: Pod # 这个yaml描述的是一个Pod
metadata:
    name: kubia-manual # Pod的名称
spec:
    nodeSelector:
      gpu: "true"
    containers:
    - image: docker.io/hengyoush/kubia # 创建容器使用的镜像
      name: kubia #容器的名称
```
现在我们加上了一个`nodeSelector`字段, 这样它只会被调度到含有该标签值的节点上去了.