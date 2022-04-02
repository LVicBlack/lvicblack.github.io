---
title: ubuntu20.4安装k8s1.20（kubeadm）
date: 2022-03-13 22:06:00
categories: 
- kubernetes
- Kubernetes集群搭建
tags:
- kubernetes
---

###  准备工作

虚拟机配置：
0. 3台
1. Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-90-generic x86_64)
2. 2c4g
3. 40GB磁盘
4. 内网互通
5. 外网访问权限不受限制



| ip            | name       |
| ------------- | ---------- |
| 10.168.56.101 | k8smaster1 |
| 10.168.56.103 | k8snode1   |
| 10.168.56.102 | k8snode2   |



### 前置

#### 同步时间

```
apt install ntpdate
ntpdate ntp.aliyun.com

crontab -e
0 */1 * * * /usr/sbin/ntpdate ntp.aliyun.com
```

#### 禁用  swapoff
```shell
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
#永久关闭  vim /etc/fstab  注释掉最后一行的swap
```


### 安装并启用 Docker

卸载旧版本Docker
```shell
#卸载旧版本docker
sudo apt-get remove docker docker-engine docker-ce docker.io	

#清空旧版docker占用的内存
sudo apt-get remove --auto-remove docker

#更新系统源
sudo apt-get update
```

```shell
apt-get update
apt-get -y install apt-transport-https ca-certificates curl software-properties-common

# 添加阿里云的docker GPG密钥
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

# 添加阿里镜像源
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

# 查看有哪些版本
apt-cache madison docker-ce
#安装最新版（需要安装20版本以下）
sudo apt-get install -y docker-ce
#安装5:19.03.15~3-0~ubuntu-focal版
sudo apt-get install -y docker-ce=5:19.03.15~3-0~ubuntu-focal docker-ce-cli=5:19.03.15~3-0~ubuntu-focal

docker --version
Docker version 20.10.7, build 20.10.7-0ubuntu5~20.04.2

# 配置docker镜像加速
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["阿里云加速器地址"]
}
EOF

# kubelet需要让docker容器引擎使用systemd作为CGroup的驱动，其默认值为cgroupfs，因而，我们还需要编辑docker的配置文件/etc/docker/daemon.json，添加如下内容。
"exec-opts": ["native.cgroupdriver=systemd"]

# 重新加载配置文件，然后重启服务，并且设置为开机启动。
sudo systemctl daemon-reload
sudo systemctl restart docker
systemctl enable docker
```

### 在各主机上生成kubelet和kubeadm等相关程序包的仓库，这里使用阿里云镜像源

```shell
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
sudo apt-add-repository "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"
apt update
```

### 安装kubelet、kubeadm、kubectl，并将kubelet设置为开机启动

```shell
apt install kubelet=1.20.13-00 kubeadm=1.20.13-00 kubectl=1.20.13-00 -y
systemctl enable kubelet
```

> 安装完成后，要确保kubeadm等程序文件的版本，这将也是后面初始化Kubernetes集群时需要明确指定的版本号。

### 通过配置文件方式部署 Kubernetes 的 Master 节点

```
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
# 这里的地址即为初始化的控制平面第一个节点的IP地址；
localAPIEndpoint:
  advertiseAddress: "10.168.56.101"
  bindPort: 6443
nodeRegistration:
  criSocket: "/var/run/dockershim.sock"
# 第一个控制平面节点的主机名称；
  name: k8smaster1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
# 使用自定义资源（Custom Metrics）进行自动水平扩展
controllerManager:
  extraArgs:
    horizontal-pod-autoscaler-use-rest-clients: "true"
    horizontal-pod-autoscaler-sync-period: "10s"
    node-monitor-grace-period: "10s"
# 版本号要与部署的目标版本保持一致；
kubernetesVersion: "v1.20.13"
imageRepository: "registry.aliyuncs.com/google_containers"
apiServer:
  timeoutForControlPlane: 4m0s
# 控制平面的接入端点，我们这里选择适配到10.168.56.101这一IP上；
controlPlaneEndpoint: "10.168.56.101:6443"
certificatesDir: "/etc/kubernetes/pki"
clusterName: "kubernetes"
dns:
  type: "CoreDNS"
etcd:
  local:
    dataDir: "/var/lib/etcd"
networking:
# 要使用的域名，默认为cluster.local
  dnsDomain: "cluster.local"
# Cluster或Service的网络地址；
  serviceSubnet: "10.96.0.0/12"
# Pod的网络地址，10.244.0.0/16用于适配Calico网络插件的IP值；
  podSubnet: "10.244.0.0/16"
scheduler: {}
```

将上面的内容保存到文件中，例如kubernetes-init.yaml，然后执行以下命令：
```
kubeadm init --config kubeadm-init.yaml
```
如果执行出错，可以执行 `kubeadm reset`后重新执行`kubeadm init`

操作成功后返回
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20220313212927.png)

上图返回的信息中我们需要注意的地方：
```
# 下面是成功完成第一个控制平面节点初始化的提示信息及后续需要完成的步骤
Your Kubernetes control-plane has initialized successfully!

# 在开始使用集群之前，你需要额外手动完成几个必要的步骤，使用的命令如下
To start using your cluster, you need to run the following as a regular user:

# 第1个步骤提示，Kubernetes集群管理员认证到Kubernetes集群时使用的kubeconfig配置文件；
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 我们也可以不做上述设定，而使用环境变量KUBECONFIG为kubelet等指定默认使用的kubeconfig；
Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

# 第2个步骤提示，管理员需要使用网络插件为Kubernetes集群部署Pod网络，具体选用的插件取决于管理员；
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/


# 第3个步骤提示，向Kubernetes集群添加工作节点
You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 10.168.56.101:6443 --token cqjhu6.kytbhv1pfd6flol4 \
    --discovery-token-ca-cert-hash sha256:da3dfbaf8801475fb2c248154a0b923b5aa19c083be597cb96f2990d34f9b858 \
    --control-plane

Then you can join any number of worker nodes by running the following on each as root:

# 在部署好kubeadm等程序包的各工作节点上以root用户运行类似如下命令
kubeadm join 10.168.56.101:6443 --token cqjhu6.kytbhv1pfd6flol4 \
    --discovery-token-ca-cert-hash sha256:da3dfbaf8801475fb2c248154a0b923b5aa19c083be597cb96f2990d34f9b858

```

### 设定kubectl

kubectl是kube-apiserver的命令行客户端程序，实现了除系统部署之外的几乎全部的管理操作，是kubernetes管理员使用最多的命令之一。kubectl需经由API server认证及授权后方能执行相应的管理操作，kubeadm部署的集群为其生成了一个具有管理员权限的认证配置文件/etc/kubernetes/admin.conf，它可由kubectl通过默认的"$HOME/.kube/config"的路径进行加载。当然，用户也可在kubectl命令上使用–kubeconfig选项指定一个别的位置。

下面复制认证为Kubernetes系统管理员的配置文件至目标用户（例如当前用户root）的家目录下，用于kubectl连接Kubernetes集群：

```
# mkdir -p $HOME/.kube
# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# chown $(id -u):$(id -g) $HOME/.kube/config
```

### 部署网络插件(calico)

- 下载配置文件
```
# wget https://docs.projectcalico.org/manifests/calico.yaml
```

- 修改配置
下载完后还需要修改里面定义Pod网络（CALICO_IPV4POOL_CIDR），与前面kubeadm init命令中使用的--pod-network-cidr选项指定的一样
```
4222             - name: CALICO_IPV4POOL_CIDR
4223               value: "10.244.0.0/16"
```

-  应用配置清单并部署网络插件
```
# kubectl apply -f calico.yaml
```

-  确认部署网络插件的Pod正常运行状态
```
kubectl get pod -n kube-system -l k8s-app=calico-node
NAME                READY   STATUS            RESTARTS   AGE
calico-node-m64mn   1/1     Running           0          17m
calico-node-m7286   1/1     Running           0          11m
calico-node-z74zk   0/1     PodInitializing   0          11m
```

-  验证master节点已经就绪
```
kubectl get nodes
NAME         STATUS     ROLES                  AGE   VERSION
k8smaster1   Ready      control-plane,master   37m   v1.20.13
```

-  添加Node节点到集群中
下面的步骤，需要在集群中的所有Node节点上执行
```
kubeadm join 10.168.56.101:6443 --token cqjhu6.kytbhv1pfd6flol4 \
    --discovery-token-ca-cert-hash sha256:da3dfbaf8801475fb2c248154a0b923b5aa19c083be597cb96f2990d34f9b858
```

-  验证Node节点添加结果
在每个工作节点添加完成后，即可通过kubectl验证添加结果。下面的命令及其输出是在k8s-node01和k8s-node02均添加完成后运行的，其输出结果表明两个Node已经准备就绪。

```
kubectl get nodes
NAME         STATUS     ROLES                  AGE   VERSION
k8smaster1   Ready      control-plane,master   37m   v1.20.13
k8snode1     NotReady   <none>                 43s   v1.20.13
k8snode2     Ready      <none>                 79s   v1.20.13
```

-  配置kubectl命令自动补全功能
```
# echo "source <(kubectl completion bash)" >> ~/.profile
```

###  测试Kubernetes集群

#### 验证Pod工作

在Kubernetes集群中创建1个Pod，验证是否正常运行
```
root@k8smaster1:/opt/kubeadm# kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
root@k8smaster1:/opt/kubeadm# kubectl get pod
NAME                     READY   STATUS              RESTARTS   AGE
nginx-6799fc88d8-l9857   0/1     ContainerCreating   0          19s
```
从上面的返回结果中来看，新建的Pod转态为"Running"，并且就绪。表示运行正常。

#### 验证Pod网络通信
为Deployment资源对象nginx创建NodePort类型的service，将上面创建的Pod应用发布到集群外部。
```
root@k8smaster1:/opt/kubeadm# kubectl get deployments.apps
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           3m48s
root@k8smaster1:/opt/kubeadm# kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort
service/nginx exposed
root@k8smaster1:/opt/kubeadm# kubectl get pod,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-6799fc88d8-l9857   1/1     Running   0          5m5s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        80m
service/nginx        NodePort    10.102.12.108   <none>        80:31725/TCP   13s
root@k8smaster1:/opt/kubeadm# kubectl get ep nginx
NAME    ENDPOINTS           AGE
nginx   10.244.185.193:80   26s
```
如上，已经成功的创建了service资源，并且关联到了后端的pod，现在打开浏览器，输入任意节点的IP+32082端口即可访问pod提供的应用， http://NodeIP:NodePort

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20220313224539.png)

#### 验证DNS解析
使用busybox:1.28.4镜像创建一个pod，用于dns解析上面创建的service名称

```
root@k8smaster1:/opt/kubeadm# kubectl run -it --rm --image=busybox:1.28.4 sh
If you don't see a command prompt, try pressing enter.
/ # nslookup nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx
Address 1: 10.102.12.108 nginx.default.svc.cluster.local
```
从上面返回的结果信息中，可以验证Kubernetes集群中的DNS服务工作正常。

至此，1个Master，并附带2个Node的kubernetes集群基础设置已经部署完成，并且其核心功能可以正常使用。

### ~~部署 Kubernetes 的 Worker 节点~~

### ~~通过 Taint/Toleration 调整 Master 执行 Pod 的策略~~

### 部署 Dashboard 可视化插件

#### 下载配置清单文件

```
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```

#### 修改相关内容
默认情况下，Service类型为ClusterIP，只能用于集群内部访问，这里指定使用"NodePort"类型，并使用固定端口30001，将其发布集群外部访问

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20220313231112.png)

#### 应用配置清单文件，创建相关资源

```
kubectl apply -f recommended.yaml
```

#### 验证pod是否正常运行

```
root@k8smaster1:/opt/kubeadm# kubectl get pod -n kubernetes-dashboard -l k8s-app=kubernetes-dashboard
NAME                                    READY   STATUS              RESTARTS   AGE
kubernetes-dashboard-5dbf55bd9d-zm4nl   0/1     ContainerCreating   0          7s
```
从上面的反馈结果来看，pod运行正常，并且已经就绪。

#### 访问Dashboard
打开浏览器，输入集群中任意节点的IP以https://NodeIP:30001格式访问
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20220313231306.png)

#### 生成token
创建service account并绑定默认cluster-admin管理员集群角色：

```
# 创建用户
# kubectl create serviceaccount dashboard-admin -n kube-system

# 用户授权
# kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin

# 获取用户token
# kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
Name:         dashboard-admin-token-7mq2d
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: 1b028e19-4cf9-4b31-9060-4c86f5a6dbc1

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  11 bytes
token:      ********
```

#### 使用token登录Dashboard
复制上面生成的token到浏览器，以登录Dashboard

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20220313231646.png)

### ~~部署容器存储插件~~

