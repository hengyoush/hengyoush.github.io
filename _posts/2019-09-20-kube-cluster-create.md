---
layout: post
title:  "K8s集群搭建步骤: 单master"
date:   2019-09-20 20:02:00 +0700
categories: [k8s]
---

K8s集群搭建步骤
## 安装必要软件
K8s的master和worker节点都需要安装如下软件：docker，kubeadm，kubelet，kubectl
脚本如下所示：
卸载旧版本
```
yum remove -y docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-selinux \
docker-engine-selinux \
docker-engine
```

## 设置 yum repository
```
yum install -y yum-utils \
device-mapper-persistent-data \
lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

## 安装并启动 docker
```
yum install -y docker-ce-18.09.7 docker-ce-cli-18.09.7 containerd.io
systemctl enable docker
systemctl start docker
```

## 安装 nfs-utils
必须先安装 nfs-utils 才能挂载 nfs 网络存储
```
yum install -y nfs-utils
```

## 关闭 防火墙
```
systemctl stop firewalld
systemctl disable firewalld
```

## 关闭 SeLinux
```
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

## 关闭 swap
```
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab
```

## 修改 /etc/sysctl.conf
如果有配置，则修改
```
sed -i "s#^net.ipv4.ip_forward.*#net.ipv4.ip_forward=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-ip6tables.*#net.bridge.bridge-nf-call-ip6tables=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-iptables.*#net.bridge.bridge-nf-call-iptables=1#g"  /etc/sysctl.conf
```
可能没有，追加
```
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
```
执行命令以应用
```
sysctl -p
```

## 配置K8S的yum源
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

## 卸载旧版本
yum remove -y kubelet kubeadm kubectl

## 安装kubelet、kubeadm、kubectl
```
yum install -y kubelet-1.15.3 kubeadm-1.15.3 kubectl-1.15.3

# 修改docker Cgroup Driver为systemd
# # 将/usr/lib/systemd/system/docker.service文件中的这一行 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
# # 修改为 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd
# 如果不修改，在添加 worker 节点时可能会碰到如下错误
# [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". 
# Please follow the guide at https://kubernetes.io/docs/setup/cri/
sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service

# 设置 docker 镜像，提高 docker 镜像下载速度和稳定性
# 如果您访问 https://hub.docker.io 速度非常稳定，亦可以跳过这个步骤
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io

# 重启 docker，并启动 kubelet
systemctl daemon-reload
systemctl restart docker
systemctl enable kubelet && systemctl start kubelet

docker version
2. 使用kubeadm在master机器上初始化集群
使用kubeadm初始化集群  
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.15.3
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "172.31.129.129:6443"
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.244.0.0/16"
  dnsDomain: "cluster.local"
其中要注意：

controlPlaneEndpoint填写master的ip。
podSubnet不能和master节点/worker节点重叠。（podSubnet之后还要使用）
初始化结束后，在master上执行如下命令，让kubectl可用：
配置kubectl  
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
配置好kubectl后，执行：
kubectl get pod -n kube-system
查看master上pod的状态，正常应该是全部为Running。
使用kubectl get node查看机器节点的状况：
 
目前只有一个节点即master节点。
3. 配置Pod网络
我们使用calico部署k8s集群网络。
先获取calico.yaml: wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
执行命令： sed -i "s#192\.168\.0\.0/16#${POD_SUBNET}#" calico.yaml
其中${POD_SUBNET}替换为上面配置master时的podSubnet。
然后执行：kubectl apply -f calico.yaml

4. 将worker节点加入集群
worker节点安装好必要的软件之后，我们就可以开始加入到集群中了。
首先获取加入到集群中的命令，在master上执行：kubeadm token create --print-join-command
得到类似如下的命令：
配置kubectl  
kubeadm join 172.31.129.129:6443 --token 5h5stg.rmjosu2a3wbhz2ql     --discovery-token-ca-cert-hash sha256:ad066deabfd7c55482f8d02f3bc35bdaf95060fca021ec6b68f551009195a582
在worker上执行即可。
然后在master上执行kubectl get node命令观察节点的状态是否都是Ready。
之后再执行get pod命令观察calico是否是Running状态，如果出现CrashBackoff状态那么可能是calico获取的网络接口不对，
整calicao 网络插件的网卡发现机制，修改IP_AUTODETECTION_METHOD对应的value值。官方提供的yaml文件中，ip识别策略（IPDETECTMETHOD）没有配置，即默认为first-found，这会导致一个网络异常的ip作为nodeIP被注册，从而影响node-to-node mesh。我们可以修改成can-reach或者interface的策略，尝试连接某一个Ready的node的IP，以此选择出正确的IP。
配置kubectl  
// calico.yaml 文件添加以下二行
- name: IP_AUTODETECTION_METHOD
value: "interface=eth.*"  #  根据实际网卡开头配置
 
 // 配置如下             
- name: CLUSTER_TYPE
  value: "k8s,bgp"
# Auto-detect the BGP IP address.
- name: IP
  value: "autodetect"
- name: IP_AUTODETECTION_METHOD
  value: "interface=ens.*"
# Enable IPIP
- name: CALICO_IPV4POOL_IPIP
  value: "Always"
Optional可选步骤：
安装dashboard
首先获取dashboard的yaml文件，然后对其进行改动使其可以由外部根据端口直接访问，否则还需要启动一个kube-proxy进行代理比较麻烦。
改动如下：
配置kubectl  
ports:
- containerPort: 8443
protocol: TCP
name: https
args:
- --auto-generate-certificates
- --token-ttl=43200


...


kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
      name: https
  type: NodePort
  selector:
    k8s-app: kubernetes-dashboard
```

如上，我们将dashboard的服务类型改为NodePort，这样它就可以直接由外部的port访问。
我们配置了--token-ttl=43200设置token的失效时间。
登录在浏览器中输入https://master_ip:30001即可访问，但是需要输入登录凭证，这里我们使用token登录，token配置如下。

配置token
接着我们需要配置一个用户来获取它的token，配置文件如下：
配置kubectl  
```
apiVersion: v1
kind: ServiceAccount
metadata:
name: admin-user
namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
name: admin-user
roleRef:
apiGroup: rbac.authorization.k8s.io
kind: ClusterRole
name: cluster-admin
subjects:
- kind: ServiceAccount
name: admin-user
namespace: kube-system
```
使用apply -f创建一个名为admin-user的ServiceAccount和角色之间的绑定关系。
然后我们使用如下命令获取该用户的token：
`kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep admin-user) | grep token`
获取token后在dashboard的登录界面输入即可登录。

配置reset
当集群配置过程出现问题时中（特别是master），可以使用kubeadm reset命令将集群重置。
注意此命令比较危险，请不要随意尝试！



