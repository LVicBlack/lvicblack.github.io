---
title: 从容器到容器云：谈谈Kubernetes的本质
date: 2021-10-18 15:06:00
categories: 
- 深入剖析kubernetes（书籍）笔记
- 容器技术概念入门
tags:
- kubernetes
---

### 容器

一个正在运行的 Linux 容器，其实可以被“一分为二”地看待：
1. 一组联合挂载在 /var/lib/docker/aufs/mnt 上的 rootfs，这一部分我们称为“容器镜像”（Container Image），是容器的静态视图；
2. 一个由 Namespace+Cgroups 构成的隔离环境，这一部分我们称为“容器运行时”（Container Runtime），是容器的动态视图。

>更进一步地说，作为一名开发者，我并不关心容器运行时的差异。因为，在整个“开发 - 测试 - 发布”的流程中，真正承载着容器信息进行传递的，是容器镜像，而不是容器运行时。

更重要的是，只要从我这个承载点向 Docker 镜像制作者和使用者方向回溯，整条路径上的各个服务节点，比如 CI/CD、监控、安全、网络、存储等等，都有我可以发挥和盈利的余地。这个逻辑，正是所有云计算提供商如此热衷于容器技术的重要原因：通过容器镜像，它们可以和潜在用户（即，开发者）直接关联起来。

这样，**容器就从一个开发者手里的小工具，一跃成为了云计算领域的绝对主角；而能够定义容器组织和管理规范的“容器编排”技术，则当仁不让地坐上了容器技术领域的“头把交椅”。**

### Kubernetes
#### 起源
跟很多基础设施领域先有工程实践、后有方法论的发展路线不同，Kubernetes 项目的理论基础则要比工程实践走得靠前得多，这当然要归功于 Google 公司在 2015 年 4 月发布的 Borg 论文了。

相比于 Spanner、BigTable 等相对上层的项目，Borg 要承担的责任，是承载 Google 公司整个基础设施的核心依赖。在 Google 公司已经公开发表的基础设施体系论文中，Borg 项目当仁不让地位居整个基础设施技术栈的最底层。

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/Google%E5%9F%BA%E7%A1%80%E8%AE%BE%E6%96%BD%E6%A0%88.png)

上面这幅图，来自于 Google Omega 论文的第一作者的博士毕业论文。它描绘了当时 Google 已经公开发表的整个基础设施栈。在这个图里，你既可以找到 MapReduce、BigTable 等知名项目，也能看到 Borg 和它的继任者 Omega 位于整个技术栈的最底层。
正是由于这样的定位，Borg 可以说是 Google 最不可能开源的一个项目。而幸运的是，得益于 Docker 项目和容器技术的风靡，它却终于得以以另一种方式与开源社区见面，这个方式就是 Kubernetes 项目。

#### Kubernetes 项目要解决的问题
实际上，这个问题到目前为止都没有固定的答案。因为在不同的发展阶段，Kubernetes 需要着重解决的问题是不同的。

但是，对于大多数用户来说，他们希望 Kubernetes 项目带来的体验是确定的：现在我有了应用的容器镜像，请帮我在一个给定的集群上把这个应用运行起来。
更进一步地说，我还希望 Kubernetes 能给我提供路由网关、水平扩展、监控、备份、灾难恢复等一系列运维能力。
如果 Kubernetes 项目只是停留在拉取用户镜像、运行容器，以及提供常见的运维功能的话，那么别说跟“原生”的 Docker Swarm 项目竞争了，哪怕跟经典的 PaaS 项目相比也难有什么优势可言。

而实际上，在定义核心功能的过程中，Kubernetes 项目正是依托着 Borg 项目的理论优势，才在短短几个月内迅速站稳了脚跟，进而确定了一个如下图所示的全局架构：
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/Kubernetes%E5%85%A8%E5%B1%80%E6%9E%B6%E6%9E%84.png)

我们可以看到，Kubernetes 项目的架构，跟它的原型项目 Borg 非常类似，都由 Master 和 Node 两种节点组成，而这两种角色分别对应着控制节点和计算节点。

- Master节点 
	即控制节点，由三个紧密协作的独立组件组合而成，它们分别是：
	- kube-apiserver 负责 API 服务
	- kube-scheduler 负责调度
	- kube-controller-manager 负责容器编排
	整个集群的持久化数据，则由 kube-apiserver 处理后保存在 Etcd 中。
- Node节点
	即计算节点，最核心的部分，则是一个叫作 kubelet 的组件。

### kubelet
**在 Kubernetes 项目中，kubelet 主要负责同容器运行时（比如 Docker 项目）打交道。**

#### CRI（Container Runtime Interface）
而这个交互所依赖的，是一个称作 CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。

Kubernetes 项目并不关心你部署的是什么容器运行时、使用的什么技术实现，只要你的这个容器运行时能够运行标准的容器镜像，它就可以通过实现 CRI 接入到 Kubernetes 项目当中。

而具体的容器运行时，比如 **Docker 项目**，则一般通过 **OCI** 这个容器运行时规范同底层的 Linux 操作系统进行交互，即：把 CRI 请求翻译成对 Linux 操作系统的调用（操作 Linux Namespace 和 Cgroups 等）。

#### Device Plugin
此外，**kubelet 还通过 gRPC 协议同一个叫作 Device Plugin 的插件进行交互。**这个插件，是 Kubernetes 项目用来管理 GPU 等宿主机物理设备的主要组件，也是基于 Kubernetes 项目进行机器学习训练、高性能作业支持等工作必须关注的功能。

#### CNI（Container Networking Interface）
调用网络插件为容器配置网络

#### CSI（Container Storage Interface）
调用存储插件为容器持久化存储

#### kubelet与Borglet
实际上，kubelet 这个奇怪的名字，来自于 Borg 项目里的同源组件 Borglet。不过，如果你浏览过 Borg 论文的话，就会发现，这个命名方式可能是 kubelet 组件与 Borglet 组件的唯一相似之处。因为 Borg 项目，并不支持我们这里所讲的容器技术，而只是简单地使用了 Linux Cgroups 对进程进行限制。
这就意味着，像 Docker 这样的“容器镜像”在 Borg 中是不存在的，Borglet 组件也自然不需要像 kubelet 这样考虑如何同 Docker 进行交互、如何对容器镜像进行管理的问题，也不需要支持 CRI、CNI、CSI 等诸多容器技术接口。
可以说，**kubelet 完全就是为了实现 Kubernetes 项目对容器的管理能力而重新实现的一个组件，与 Borg 之间并没有直接的传承关系。**

>备注：虽然不使用 Docker，但 Google 内部确实在使用一个包管理工具，名叫 Midas Package Manager (MPM)，其实它可以部分取代 Docker 镜像的角色。

### Borg 对于 Kubernetes 项目的指导作用

>Master 节点

虽然在 Master 节点的实现细节上 Borg 项目与 Kubernetes 项目不尽相同，但它们的出发点却高度一致，即：**如何编排、管理、调度用户提交的作业？**

所以，Borg 项目完全可以把 Docker 镜像看作一种新的应用打包方式。这样，Borg 团队过去在大规模作业管理与编排上的经验就可以直接“套”在 Kubernetes 项目上了。

这些经验最主要的表现就是，**从一开始，Kubernetes 项目就没有像同时期的各种“容器云”项目那样，把 Docker 作为整个架构的核心，而仅仅把它作为最底层的一个容器运行时实现。**

而 Kubernetes 项目要着重解决的问题，则来自于 Borg 的研究人员在论文中提到的一个非常重要的观点：

>**运行在大规模集群中的各种任务之间，实际上存在着各种各样的关系。这些关系的处理，才是作业编排和管理系统最困难的地方。**

但容器技术出现以后，你就不难发现，在“功能单位”的划分上，容器有着独一无二的“细粒度”优势：毕竟容器的本质，只是一个进程而已。

也就是说，只要你愿意，那些原先拥挤在同一个虚拟机里的各个应用、组件、守护进程，都可以被分别做成镜像，然后运行在一个个专属的容器中。它们之间互不干涉，拥有各自的资源配额，可以被调度在整个集群里的任何一台机器上。而这，正是一个 PaaS 系统最理想的工作状态，也是所谓“微服务”思想得以落地的先决条件。

>**Kubernetes 项目最主要的设计思想是，从更宏观的角度，以统一的方式来定义任务之间的各种关系，并且为将来支持更多种类的关系留有余地。**




































