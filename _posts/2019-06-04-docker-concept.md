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

> 原因：容器运行于容器引擎之上，且容器引擎直接运行于Host OS之上，而虚拟机则运行于Hypervisor之上，Hypervisor直接管理虚拟化后的硬件资源（大部分Hypervisor如此），这就造成了容器的隔离实际上是要比虚拟机的隔离程度差一些的。但是有利也有弊，虚拟机启动时需要启动所有它自己的系统服务，而容器则不需要，因为它直接使用宿主机的内核所以不需要启动任何服务，所以虚拟机所消耗的资源比容器消耗的要多得多，导致容器启动十分快速，而虚拟机相对慢很多。

#### 为什么容器可以实现隔离
Linux Namespaces：让每个进程只看到系统的“个人视角”，包括文件系统，网络接口，主机名等等。

Linux Control Groups（cgroups）：限制了一个进程可以消耗的系统资源，包括CPU，内存，网络带宽等等。

#### Docker的组成
Docker Engine的组成如下所示:
- 一个后台运行的守护进程
- 一个REST API接口用于接收客户端发来的REST请求
- 一个CLI命令行工具

![avatar](/static/img/engine-components-flow.png)
<br>
<br>
下面这张图片则展示了docker的总体架构:

![avatar](/static/img/architecture.svg)

docker使用C/S结构, docker client与docker daemon交互, docker daemon负责构建、运行和分发Docker容器。Docker客户机和守护进程可以运行在同一个系统上，也可以将Docker客户机连接到远程Docker守护进程。Docker客户机和守护进程使用REST API通过UNIX套接字或网络接口进行通信。

---

## 为什么使用Docker
其实根据上面的介绍, 已经可以提炼出docker的大部分优点了, 接下来做一个总结:

- 更快的启动速度
- 更高效的利用系统资源
- 一致的运行环境
- 更轻松的迁移
- 自动伸缩
- ...

---

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

> 注意: `Dockerfile`中的每个`COPY`, `ADD`, `RUN`命令都会在原先的镜像上新加`layer（层）`，所以尽量减少这类型的命令数量可以压缩镜像的大小，便于分发，下面介绍的Multistage-build的方式可以有效解决这种问题

#### Dockerfile基本命令参考
| 命令   | 说明                         | 例子                               |
|--------|------------------------------|------------------------------------|
| FROM   | Dockerfile的第一个指令       | `FROM ubuntu`                      |
| COPY   | 拷贝文件到容器内部的指定路径 | `COPY .bash_profile /home`         |
| ENV    | 给容器设置环境变量           | `ENV HOSTNAME=host`                |
| RUN    | 执行一个命令                 | `RUN apt-get update`               |
| CMD    | 容器的默认执行命令           | `CMD ["/bin/echo", "hello world"]` |
| EXPOSE | 指示这个容器监听的端口       | `EXPOSE 8080`                      |

#### 使用Dockerfile构建Java应用镜像
使用如下命令创建一个Java工程：
```
mvn archetype:generate -DgroupId=org.examples.java -DartifactId=helloworld -DinteractiveMode=false
```
使用`mvn package`build工程。

然后在工程目录下创建Dockerfile，内容如下：
```Dockerfile
FROM openjdk:latest

COPY target/helloworld-1.0-SNAPSHOT.jar /usr/src/helloworld-1.0-SNAPSHOT.jar

CMD java -cp /usr/src/helloworld-1.0-SNAPSHOT.jar org.examples.java.App
```
然后执行`docker build`命令创建镜像：
```
docker image build . -t hello-java:latest
```

build成功之后执行`docker run`命令运行该镜像：
```
docker container run hello-java:latest
```

控制台会输出：
```
Hello World!
```

#### 使用maven插件创建Java应用镜像

##### docker-maven-plugin
我们复用上一节创建的pom文件，在pom文件中加入以下配置：
```xml
<profiles>
        <profile>
            <id>docker</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>io.fabric8</groupId>
                        <artifactId>docker-maven-plugin</artifactId>
                        <version>0.20.1</version>
                        <configuration>
                            <images>
                                <image>
                                    <name>hellojava</name>
                                    <build>
                                        <from>adoptopenjdk/openjdk8</from>
                                        <assembly>
                                            <descriptorRef>artifact</descriptorRef>
                                        </assembly>
                                        <cmd>java -jar maven/${project.name}-${project.version}.jar</cmd>
                                    </build>
                                    <run>
                                        <wait>
                                            <log>Hello World!</log>
                                        </wait>
                                    </run>
                                </image>
                            </images>
                        </configuration>
                        <executions>
                            <execution>
                                <id>docker:build</id>
                                <phase>package</phase>
                                <goals>
                                    <goal>build</goal>
                                </goals>
                            </execution>
                            <execution>
                                <id>docker:start</id>
                                <phase>install</phase>
                                <goals>
                                    <goal>run</goal>
                                    <goal>logs</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
```
然后在工程目录下执行：
```
mvn -f pom.xml pakcage -Pdocker
```

输出如下：
```
[INFO] DOCKER> [hellojava:latest]: Created docker-build.tar in 189 milliseconds
[INFO] DOCKER> [hellojava:latest]: Built image sha256:6215c
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
```

然后使用`docker images`查看构建的镜像：
```
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
hellojava               latest              6215ccdcb0d8        2 minutes ago       305MB
```

##### Jib
Jib是google开源的基于Maven插件的docker容器构建工具, 和上面的docker-maven-plugin一样, 只要在pom文件中做相应的配置即可
完成镜像的创建、推送等操作.

首先创建一个Hello World工程:
```
mvn artchetype:generate
```

然后修改其pom文件,增加如下内容:
```xml
 <plugin> 
  <groupId>com.google.cloud.tools</groupId>  
  <artifactId>jib-maven-plugin</artifactId>  
  <version>1.2.0</version>  
  <configuration> 
    <from> 
      <image>openjdk:alpine</image>  
      <auth> 
        <username>hengyoush</username>  
        <password>1179332922</password> 
      </auth> 
    </from>  
    <to> 
      <image>docker.io/hengyoush/myimage</image>  
      <auth> 
        <username>hengyoush</username>  
        <password>1179332922</password> 
      </auth> 
    </to>  
    <container> 
      <jvmFlags> 
        <jvmFlag>-Xms512m</jvmFlag>  
        <jvmFlag>-Xdebug</jvmFlag> 
      </jvmFlags>  
      <mainClass>com.example.App</mainClass>  
      <args> 
        <arg>some</arg>  
        <arg>args</arg> 
      </args>  
      <ports> 
        <port>1000</port>  
        <port>2000-2003/udp</port> 
      </ports> 
    </container> 
  </configuration> 
</plugin>
```

下面来逐个解释xml中对应的标签意义:
标签名 | 标签类型 | 描述
-|-|- 
to | to | 通过应用构建的目标镜像配置,包括该镜像的仓库,名字以及tag等
from | from | 配置一个基础镜像
container | container | 容器的相关配置, 包括环境变量, jvm参数, 监听端口等
image | string | 基础镜像, 比如openjdk:alpine
auth | auth | 在from和to中配置, 用于拉取和推送镜像时的身份校验
appRoot | string | 应用的内容在容器中存放的目录位置, 默认为“/app”
jvmFlags | list | 运行应用时的JVM参数
mainClass | string | 运行应用的main类
args | list | 运行java应用的参数
ports | list | 容器暴露的端口

然后执行如下命令可以将镜像构建至docker daemon:
```
mvn compile jib:dockerBuild
```

然后执行`docker run hengyoush/myimage`命令运行镜像, 验证是否成功:
输出如下:
```
Hello World!
```
执行`docker images`你也可以看到本地的镜像.

## 常用操作

查看docker版本号
```
docker version
```

搜索可用镜像
```
docker search ${镜像名}
```

下载镜像
```
docker pull ${镜像名}
```

将当前容器提交生成新的镜像
```
docker commit -a "作者" -m "评论" {容器} {新的镜像命名}
```

将镜像保存为文件
```
docker save -o 文件名 镜像名
```

将文件导入为镜像
```
docker load -I 文件名
```

给容器打tag
```
docker tag ubuntu:latest ubuntu:1.9
```

运行镜像相关命令参数, 首先看一个较为完整的例子:
```
docker run -d \
-it \
--mount source=${pwd}/target target=/app \
--name mycontainer \
-p 8080:80
myimage
```

如上所示. `docker run`运行了一个名为`myimage`的镜像,选项含义如下: 
- `-d`: 指其作为后台模式运行.
*(此时所有I/O数据只能通过网络资源或者共享卷组来进行交互。因为container不再监听你执行docker run的这个终端命令行窗口。但你可以通过执行docker attach 来重新挂载这个container里面。需要注意的时，如果你选择执行-d使container进入后台模式，那么将无法配合"--rm"参数。)*
- `--name`: 指定容器的名字
- `-it`: `i`保证标准输入流开启, `t`虚拟出一个TTY窗口. 这两个选项组合在一起就可以申请一个控制台同容器内部进行交互.
- `--mount`: (参见*存储*一节)
- `-p`: 映射容器端口与主机端口, 例子中的是主机的8080端口映射到容器的80端口

还有其他常用的选项:
- `--rm`: 指定在容器运行完成时清理其产生的数据(volume等存储)
- `--network`: 可配置不同的网络, 可参见*网络*一节.


---

## 存储
默认情况下, 任何在容器中的文件修改操作都会在现有镜像层上的*writable container laye*上进行, 如果容器不存在了那么所有修改的数据都会丢失, 而且如果其他进程想要处理这些数据将会十分困难, 而且相对于直接写入宿主机的文件, 容器中的文件操作性能上将会消耗更多.

docker有两种方式处理这种情况:
1. *Volumes*
2. *bind mount*

这两种方式的差别如下所示:

![avatar](/static/img/types-of-mounts.png)

可以看到, 这两种方式对于容器来说都是一样的, 只是文件在宿主机上存在的方式不同.
- **Volumes**是Docker管理在宿主机的`/var/lib/docker/volumes/`目录下管理的, Volumes是存储持久化文件的最佳方式.
- **bind mount**不由Docker管理, 其文件可存在于宿主机文件系统上的任意位置.

#### Volume
相对于bind mount, volumes又如下主要优势:
- Volumes更容易备份以及迁移
- 可以通过Docker的命令行或者API管理Volumes.
- Volumes可以被存储在远程主机上
- ...

综上所述, Volumes应该被优先考虑使用.

接下来看如何创建管理Volume
使用
```
docker volume create my-vol
```
如上创建了一个名为`my-vol`的volume.
使用`docker volume ls`查看我们刚刚创建的volume如下:
```
DRIVER              VOLUME NAME
local               my-vol
```
接下来使用`docker inspect`查看创建的volume的详细信息:
```
docker volume inspect my-vol
```
输出如下:
```json
[
    {
        "CreatedAt": "2019-06-07T14:50:49Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```
接下来可以使用`docker volume rm my-vol`来移除该volume(先别着急删).


好了, 创建了volume, 接下来需要让一个容器挂载该volume,
一般使用`--mount`选项挂载Volume到容器中(这个选项不仅可以挂载Volume, 还可以挂载bind mount和tmpfs)
```
docker run -d \
  --name devtest \
  --mount source=my-vol,target=/app \
  nginx:latest
```
我们来解释下`--mount`各个参数的意思:
- source: 表示挂载到volume的名称
- target: 表示容器内部的挂载路径
- readonly: 容器对volume只有读权限 

好了, 容器运行完成, 接下来我们使用`inspect`窥探一下容器的内部,
输出如下:
```json
"Mounts": [
            {
                "Type": "volume",
                "Name": "my-vol",
                "Source": "/var/lib/docker/volumes/my-vol/_data",
                "Destination": "/app",
                "Driver": "local",
                "Mode": "z",
                "RW": true,
                "Propagation": ""
            }
        ]
```
可以看到展示了宿主机的路径和容器内的路径以及读写设置.

移除volume, 执行如下命令:
```
docker volume prune
```
这样就移除了所有不使用的volume.

#### Bind mount
创建Bind mount:
```
docker run -d \
  -it \
  --name devtest \
  --mount type=bind,source="$(pwd)"/target,target=/app \
  nginx:latest
```
如上命令创建了一个Bind mount, 将主机上当前路径下的target目录映射到了容器内部的/app目录下, 除此之外还可以设置
readonly选项, 用于控制容器的读写权限.

使用`docker inspect`命令查看容器的挂载情况, 输出如下:
```json
"Mounts": [
    {
        "Type": "bind",
        "Source": "/tmp/source/target",
        "Destination": "/app",
        "Mode": "",
        "RW": true,
        "Propagation": "rprivate"
    }
],
```

---

## 网络

docker中的网络类型:
1. bridge: 桥接类型, 默认的网络类型
2. host: 容器与主机之间没有任何网络隔离
3. overlay: 用于与多个docker daemon通信, 常用于`docker swarm`中.
4. macvlan: 通过物理网络进行通信, 会分配一个MAC地址给容器, 通常用于必须进行物理网络通信的遗留系统.
5. none: 通常用于自定义网络类型.

使用如下命令查看docker网络情况:
```
docker network ls
```

显示如下:
```
NETWORK ID          NAME                DRIVER              SCOPE
bdd2e4a8ca3e        bridge              bridge              local
49097dd448bb        host                host                local
80225cb43502        none                null                local
```
可以看到docker内置了三种类型的网络, 我们看一下默认存在的bridge(default-bridge network)信息:
```
docker network inspect bridge
```

输出如下:
```json
[
    {
        "Name": "bridge",
        "Id": "bdd2e4a8ca3e690d122c216ca7a5cf7f27468aaf3d3b112d5a0f3e1f3cf3937d",
        "Created": "2019-06-01T11:08:04.581530358Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Options": {
            ...
        },
        "Labels": {}
    }
]
```
可以看到网关和子网掩码以及各种信息.

我们现在随便运行一个镜像然后再次执行`docker network inspect bridge`命令,输出如下(部分输出):
```json
Containers: {
            "75d4c6a88bf9842f3b00abaf334af1fb18d810ddb35995138acd7c1f0dc18c8e": {
                "Name": "recursing_keller",
                "EndpointID": "d8e12d5778a6815bc455fa153242f47feae303368f2df8ed9c46aab4c0f79651",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        }
```
可以看到Containers出现了一条记录, 包含了我们刚刚运行的容器的ip地址.


一般我们在生产环境中不推荐使用默认的`default-bridge`网络, 我们最好使用自定义bridge网络(user-define bridge network),
又如下好处:
1. 自定义bridge网络不仅可以通过ip地址也可以通过容器名称通信(必须在同一个自定义bridge网络下), 这种能力叫做**自动服务发现**.
2. 同一个自定义bridge网络下, 不同容器之间自动暴露所有端口, 而且对于外界网络没有暴露任何端口(使用`--publish`选项可暴露端口).

运行如下命令创建自定义桥接网络:
```
docker network create --driver bridge alpine-net
```
我们创建了一个名为`alpine-net`的网络.

接下来我们运行一个容器连接上这个网络如下:
```
docker run -dit --name alpine1 --network alpine-net alpine ash
```
我们使用`--network`选项指定网络`alpine-net`用于连接.

*这里有详细教程: https://docs.docker.com/network/network-tutorial-standalone/*