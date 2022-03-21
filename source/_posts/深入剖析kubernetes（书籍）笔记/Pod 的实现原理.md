---
title: Pod 的实现原理
date: 2022-03-20 10:00:00
categories: 
- 深入剖析kubernetes（书籍）笔记
- Pod
tags:
- kubernetes
- Pod
---

> Pod，是 Kubernetes 项目中最小的 API 对象
> 或者说，Pod，是 Kubernetes 项目的原子调度单位。

### 类比Linux系统

展示当前系统中正在运行的进程的树状结构```pstree -g```

```

systemd(1)-+-accounts-daemon(1984)-+-{gdbus}(1984)
           | `-{gmain}(1984)
           |-acpid(2044)
          ...      
           |-rsyslogd(1632)-+-{in:imklog}(1632)
           |  |-{in:imuxsock) S 1(1632)
           | `-{rs:main Q:Reg}(1632)
```
在一个真正的操作系统里，进程是以进程组的方式，“有原则地”组织在一起。
例如rsyslogd 的程序，它负责的是 Linux 操作系统里的日志处理。可以看到，rsyslogd 的主程序 main，和它要用到的内核日志模块 imklog 等，同属于 1632 进程组。这些进程相互协作，共同完成 rsyslogd 程序的职责。

- Kubernetes 项目将“进程组”的概念映射到了容器技术中，并使其成为了这个云计算“操作系统”里的“一等公民”。

- 在 Borg 项目的开发和实践过程中，Google 公司的工程师们发现，他们部署的应用，往往都存在着类似于“进程和进程组”的关系。更具体地说，**就是这些应用之间有着密切的协作关系，使得它们必须部署在同一台机器上。**

#### 容器的“单进程模型”

把 rsyslogd 这个应用给容器化，由于受限于容器的“单进程模型”，这三个模块必须被分别制作成三个不同的容器。

>再次强调一下：**容器的“单进程模型”，并不是指容器里只能运行“一个”进程，而是指容器没有管理多个进程的能力。**这是因为容器里 PID=1 的进程就是应用本身，其他的进程都是这个 PID=1 进程的子进程。可是，用户编写的应用，并不能够像正常操作系统里的 init 进程或者 systemd 那样拥有进程管理的功能。比如，你的应用是一个 Java Web 程序（PID=1），然后你执行 docker exec 在后台启动了一个 Nginx 进程（PID=3）。可是，当这个 Nginx 进程异常退出的时候，你该怎么知道呢？这个进程退出后的垃圾收集工作，又应该由谁去做呢？

### Kubernetes 的项目调度

- 常规部署存在的问题：
- 资源囤积：
	- 方案：在所有设置了 Affinity 约束的任务都达到时，才开始对它们统一进行调度。
	- 问题：带来了不可避免的调度效率损失和死锁的可能性
- 乐观调度：
	- 方案：先不管这些冲突，而是通过精心设计的回滚机制在出现了冲突之后解决问题。
	- 问题：复杂程度，不是常规技术团队所能驾驭的。


- Kubernetes：Pod 是 Kubernetes 里的原子调度单位。这就意味着，Kubernetes 项目的调度器，是统一按照 Pod 而非容器的资源需求进行计算的。

例如：像 imklog、imuxsock 和 main 函数主进程这样的三个容器，正是一个典型的由三个容器组成的 Pod。Kubernetes 项目在调度时，自然就会去选择可用内存等于 3 GB 的 node-1 节点进行绑定，而根本不会考虑 node-2（内存不足3GB）。

#### “超亲密关系”

像这样容器间的紧密协作，我们可以称为“超亲密关系”。这些具有“超亲密关系”容器的典型特征包括但不限于：互相之间会发生直接的文件交换、使用 localhost 或者 Socket 文件进行本地通信、会发生非常频繁的远程调用、需要共享某些 Linux Namespace（比如，一个容器要加入另一个容器的 Network Namespace）等等。

### 容器设计模式

#### Pod 的概念

>Pod只是一个逻辑概念，Kubernetes 真正处理的，还是宿主机操作系统上 Linux 容器的 Namespace 和 Cgroups，而并不存在一个所谓的 Pod 的边界或者隔离环境。

- Pod的“创建”其实是一组共享了某些资源的容器。
**Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。**

例如，一个有 A、B 两个容器的 Pod，等同于一个容器（容器 A）共享另外一个容器（容器 B）的网络和 Volume 的玩儿法

使用docker实现：
```
$ docker run --net=B --volumes-from=B --name=A image-A ...
```
但是，如果这样做的话，容器 B 就必须比容器 A 先启动，这样一个 Pod 里的多个容器就**不是对等关系，而是拓扑关系了。**

#### Pod 的实现原理

在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 **Infra 容器**。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。这样的组织关系，可以用下面这样一个示意图来表达：

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20220321143123.png)

Infra 容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作：k8s.gcr.io/pause。这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器，解压后的大小也只有 100~200 KB 左右。

在 Infra 容器“Hold 住”Network Namespace 后，用户容器就可以加入到 Infra 容器的 Network Namespace 当中了。所以，如果你查看这些容器在宿主机上的 Namespace 文件，它们指向的值一定是完全一样的。

这也就意味着，对于 Pod 里的容器 A 和容器 B 来说：
- 它们可以直接使用 localhost 进行通信；
- 它们看到的网络设备跟 Infra 容器看到的完全一样；
- 一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址；
- 当然，其他的所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享；
- Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关。

对于同一个 Pod 里面的所有用户容器来说，它们的进出流量，可以认为都是通过 Infra 容器完成的。**如果为 Kubernetes 开发一个网络插件时，应该重点考虑的是如何配置这个 Pod 的 Network Namespace，而不是每一个用户容器如何使用你的网络配置，这是没有意义的。

**Kubernetes 项目只要把所有 Volume 的定义都设计在 Pod 层级即可。**
这样，一个 Volume 对应的宿主机目录对于 Pod 来说就只有一个，Pod 里的容器只要声明挂载这个 Volume，就一定可以共享这个 Volume 对应的宿主机目录。比如下面这个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:      
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

debian-container 和 nginx-container 都声明挂载了 shared-data 这个 Volume。对应在宿主机上的目录是：/data。而这个目录，其实就被同时绑定挂载进了上述两个容器当中。

这也是nginx-container 可以从它的 /usr/share/nginx/html 目录中，读取到 debian-container 生成的 index.html 文件的原因。

Pod 这种“超亲密关系”容器的设计思想，实际上就是希望，当用户想在一个容器里跑多个功能并不相关的应用时，应该优先考虑它们是不是更应该被描述成一个 Pod 里的多个容器。

### 例1：WAR 包与 Web 服务器

把 WAR 包和 Tomcat 分别做成镜像，然后把它们作为一个 Pod 里的两个容器“组合”在一起。

```
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}
```

> Init Container
> 在 Pod 中，所有 Init Container 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。

pod启动顺序：
1. geektime/sample:v2 容器的/app 目录和Tomcat 容器webapps 目录同时挂载在 app-volume 的 Volume。
2. 启动Init Container，执行"cp /sample.war /app"，把应用的 WAR 包拷贝到 /app 目录下，然后退出。
3.启动Tomcat 容器，webapps 目录下存在 sample.war 文件，运行war包

#### sidecar 边车模式

上面的“组合”操作，正是容器设计模式里最常用的一种模式，边车模式。

>sidecar 指的就是我们可以在一个 Pod 中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。

比如，在我们的这个应用 Pod 中，Tomcat 容器是我们要使用的主容器，而 WAR 包容器的存在，只是为了给它提供一个 WAR 包而已。所以，我们用 Init Container 的方式优先运行 WAR 包容器，扮演了一个 sidecar 的角色。

### 例2：容器的日志收集

比如，我现在有一个应用，需要不断地把日志文件输出到容器的 /var/log 目录中。

这时，我就可以把一个 Pod 里的 Volume 挂载到应用容器的 /var/log 目录上。

然后，我在这个 Pod 里同时运行一个 sidecar 容器，它也声明挂载同一个 Volume 到自己的 /var/log 目录上。

这样，接下来 sidecar 容器就只需要做一件事儿，那就是不断地从自己的 /var/log 目录里读取日志文件，转发到 MongoDB 或者 Elasticsearch 中存储起来。这样，一个最基本的日志收集工作就完成了。

跟第一个例子一样，这个例子中的 sidecar 的主要工作也是使用共享的 Volume 来完成对文件的操作。

Pod 的另一个重要特性是，它的所有容器都共享同一个 Network Namespace。这就使得很多与 Pod 网络相关的配置和管理，也都可以交给 sidecar 完成，而完全无须干涉用户容器。这里最典型的例子莫过于 Istio 这个微服务治理项目了。

>备注：Kubernetes 社区曾经把“容器设计模式”这个理论，整理成了一篇[小论文](https://www.usenix.org/conference/hotcloud16/workshop-program/presentation/burns)

### 除了 Network Namespace 外，Pod 里的容器还可以共享哪些 Namespace

Linux 支持7种namespace:
1. cgroup用于隔离cgroup根目录;
2. IPC用于隔离系统消息队列;
3. Network隔离网络;
4. Mount隔离挂载点;
5. PID隔离进程;
6. User隔离用户和用户组;
7. UTS隔离主机名nis域名。

pod里网络和文件系统是用得最多的

### pod和容器的区别

- pod是k8s的最小单元，容器包含在pod中。
- 一个pod中有一个pause容器和若干个业务容器，而容器就是单独的一个容器。


