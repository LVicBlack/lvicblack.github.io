## 基本概念

### 哪些属性属于pod层级

#### 调度、网络、存储，以及安全相关的属性

这些属性的共同特征是，它们描述的是“机器”这个整体，而不是里面运行的“程序”。
- 配置这个“机器”的网卡（即：Pod 的网络定义）
配置这个“机器”的磁盘（即：Pod 的存储定义）
配置这个“机器”的防火墙（即：Pod 的安全定义）
这台“机器”运行在哪个服务器之上（即：Pod 的调度）

##### NodeSelector

```
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   disktype: ssd
```
Pod 永远只能运行在携带了“disktype: ssd”标签（Label）的节点上；否则，它将调度失败。

##### NodeName

pod.spec.nodeName将Pod直接调度到指定的Node节点上，会【跳过Scheduler的调度策略】，该匹配规则是【强制】匹配。可以越过Taints污点进行调度。

一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字。

所以，这个字段一般由调度器负责设置，也可以在测试或者调试的时候设置它来“骗过”调度器。

##### HostAliases

>定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容
>需要指出的是，在 Kubernetes 项目中，如果要设置 hosts 文件里的内容，一定要通过这种方法。否则，如果直接修改了 hosts 文件的话，在 Pod 被删除重建之后，kubelet 会自动覆盖掉被修改的内容。

```
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

在这个 Pod 的 YAML 文件中，我设置了一组 IP 和 hostname 的数据。这样，这个 Pod 启动后，/etc/hosts 文件的内容将如下所示：

```
cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote
```
最下面两行记录，就是通过 HostAliases 字段为 Pod 设置的。


#### 跟容器的 Linux Namespace 相关的属性

Pod 的设计，就是要让它里面的容器尽可能多地共享 Linux Namespace，仅保留必要的隔离和限制能力。这样，Pod 模拟出的效果，就跟虚拟机里程序间的关系非常类似了。

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

>shareProcessNamespace: true
>共享pid，意味着可以在各自容器看到其它容器的进程id，包括infra容器

YAML 文件中，还定义了两个容器：一个是 nginx 容器，一个是开启了 tty 和 stdin 的 shell 容器。

>tty 和 stdin
>等同于设置了 docker run 里的 -it（-i 即 stdin，-t 即 tty）参数。
>
>- tty: Linux 给用户提供的一个常驻小程序，用于接收用户的标准输入，返回操作系统的标准输出。
>- stdin：标准输入流，为了能够在 tty 中输入信息。

```
1. 创建Pod

$ kubectl create -f nginx.yaml

2. 连接到 shell 容器的 tty 上

$ kubectl attach -it nginx -c shell
/ # ps ax
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   14 101       0:00 nginx: worker process
   15 root      0:00 sh
   21 root      0:00 ps ax
```

>kubectl attach (POD | TYPE/NAME) -c CONTAINER 
>连接到 pod 上的 container 中

不仅可以看到它本身的 ps ax 指令，还可以看到 nginx 容器的进程，以及 Infra 容器的 /pause 进程。这就意味着，整个 Pod 里的每个容器的进程，对于所有容器来说都是可见的：它们共享了同一个 PID Namespace。


#### Pod 中的容器要共享宿主机的 Namespace

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  hostIPC: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```



- metadata.labels: 设置Pod的标签
- spec.template.metadata.labels: 设置Pod下容器的标签
- spec.selector.matchLabels: Pod的标签
- 在定义pod模板时，spec.template.metadata.lables必须和spec.selector.matchLabels的键值一致。







































