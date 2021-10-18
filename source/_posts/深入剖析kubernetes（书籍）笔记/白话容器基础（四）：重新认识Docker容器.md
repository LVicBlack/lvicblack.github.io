---
title: 白话容器基础（四）：重新认识Docker容器
date: 2021-10-11 10:06:00
categories: 
- 深入剖析kubernetes（书籍）笔记
- 容器技术概念入门
tags:
- kubernetes
---

## 重新认识Docker容器

### Docker总结

#### 容器与虚拟机对比
**容器的本质是一个进程，只不过这个进程加上了视图上的隔离和资源上的限制。 **
就容器本质而言，它并没有在宿主机上启动一个“容器进程”，它启动的还是用户原来要启动的应用程序，只不过这个应用程序上加了视图隔离和资源限制。
虚拟机也能实现视图隔离和资源限制，但它底层的技术实现与容器不同，在宿主机上你能看到这样一个“虚拟机进程”。
因此，与容器相比，虚拟机会带来更多的性能损耗，主要在：

1. 虚拟机本身的进程需要占用一部分资源 
2. 与底层硬件交互的时候（网络、磁盘I/O）都需要经过虚拟化软件拦截，会有损耗。但是**它比容器有更好的隔离性和安全性。**

#### 容器使用的底层技术
1. 视图隔离：Namespace 
2. 资源限制：cgropus

#### Docker核心原理
>为待创建的用户进程：
>1. 启用 Linux Namespace 配置；**视图隔离**
>2. 设置指定的 Cgroups 参数；**资源限制**
>3. 切换进程的根目录（Change Root）。**挂载。容器镜像生效，以实现环境一致性。所谓容器镜像，本质就是容器的根文件系统(rootfs)。**

Docker 项目在最后一步的切换上会优先使用 pivot_root 系统调用，如果系统不支持，才会使用 chroot。

>pivot_root与chroot的区别：
>- chroot是只改变即将运行的 某进程的根目录。
>- pviot_root主要是把整个系统切换到一个新的root目录，然后去掉对之前rootfs的依赖，以便于可以umount 之前的文件系统（pivot_root需要root权限）

Docker容器的增量rootfs：即下层已经生成的永远不会改变，所有的修改都通过在上层叠加。比如，删除A文件，就是在上层添加一个“白障”，让系统无法读取到下层A文件。修改则是先copy一个备份到新的层（新老的文件可能都在可读写层），然后读取的时候直接读取新的层。

Docker通过Volume来实现将宿主机的文件挂载到容器中

### 制作容器镜像（实际案例）

本文会通过一个实际案例，对“白话容器基础”系列的所有内容做一次深入的总结和扩展。

#### 应用代码 
app.py。功能是：如果当前环境中有“NAME”这个环境变量，就把它打印在“Hello”之后，否则就打印“Hello world”，最后再打印出当前环境的 hostname。
```
from flask import Flask
import socket
import os

app = Flask(__name__)

@app.route('/')
def hello():
    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>"           
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname())
    
if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```
#### Dockerfile

```
# 使用官方提供的Python开发镜像作为基础镜像
FROM python:2.7-slim

# 将工作目录切换为/app
WORKDIR /app

# 将当前目录下的所有内容复制到/app下
ADD . /app

# 使用pip命令安装这个应用所需要的依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# 允许外界访问容器的80端口
EXPOSE 80

# 设置环境变量
ENV NAME World

# 设置容器进程为：python app.py，即：这个Python应用的启动命令
CMD ["python", "app.py"]
```
**Dockerfile 的设计思想，是使用一些标准的原语（即大写高亮的词语），描述我们所要构建的 Docker 镜像。并且这些原语，都是按顺序处理的。**

Dockerfile指令
- [菜鸟教程](https://www.runoob.com/docker/docker-dockerfile.html)
- [掘金](https://juejin.cn/post/6844904081966759943)

CMD ["python", "app.py"]等价于"docker run  python app.py"。
ENTRYPOINT和 CMD 都是 Docker 容器进程启动所必需的参数，完整执行格式是：“ENTRYPOINT CMD”。但是，默认情况下，Docker 会为你提供一个隐含的 ENTRYPOINT，即：/bin/sh -c。所以，在不指定 ENTRYPOINT 时，比如在我们这个例子里，实际上运行在容器里的完整进程是：/bin/sh -c "python app.py"，即 CMD 的内容就是 ENTRYPOINT 的参数。

>备注：基于以上原因，**我们后面会统一称 Docker 容器的启动进程为 ENTRYPOINT，而不是 CMD。**

#### 制作镜像

当前目录
```
$ ls
Dockerfile  app.py   requirements.txt
```

执行
```
$ docker build -t helloworld .
```

-t 的作用是给这个镜像加一个 Tag，即：起一个好听的名字。
docker build 会自动加载当前目录下的 Dockerfile 文件，然后按照顺序，执行文件中的原语。
这个过程，实际上可以等同于 Docker 使用基础镜像启动了一个容器，然后在容器中依次执行 Dockerfile 中的原语。

**需要注意的是，Dockerfile 中的每个原语执行后，都会生成一个对应的镜像层**即使原语本身并没有明显地修改文件的操作（比如，ENV 原语），它对应的层也会存在。只不过在外界看来，这个层是空的。

docker build 操作完成后，我可以通过 docker images 命令查看结果：
```
$ docker image ls

REPOSITORY            TAG                 IMAGE ID
helloworld         latest              653287cdf998
```

#### 启动容器

```
$ docker run -p 4000:80 helloworld
```

在这一句命令中，镜像名 helloworld 后面，我什么都不用写，因为在 Dockerfile 中已经指定了 CMD。否则，我就得把进程的启动命令加在后面：

```
$ docker run -p 4000:80 helloworld python app.py
```

容器启动之后，我可以使用 docker ps 命令看到：

```

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED
4ddf4638572d        helloworld       "python app.py"     10 seconds ago
```

通过 -p 4000:80 告诉了 Docker，请把容器内的 80 端口映射在宿主机的 4000 端口上。

验证：

```
$ curl http://localhost:4000
<h3>Hello World!</h3><b>Hostname:</b> 4ddf4638572d<br/>
```

### 镜像上传

首先需要注册一个 Docker Hub 账号，然后使用 docker login 命令登录。

用 docker tag 命令给容器镜像起一个完整的名字：

```
$ docker tag helloworld geektime/helloworld:v1
```

> "geektime"替换成你自己的 Docker Hub 账户名称

执行docker push进行上传

```
$ docker push geektime/helloworld:v1
```

还可以使用 docker commit 指令，把一个正在运行的容器，直接提交为一个镜像。
这么操作原因是：这个容器运行起来后，我又在里面做了一些操作，并且要把操作结果保存到镜像里，比如：

```
$ docker exec -it 4ddf4638572d /bin/sh
# 在容器内部新建了一个文件
root@4ddf4638572d:/app# touch test.txt
root@4ddf4638572d:/app# exit

#将这个新建的文件提交到镜像中保存
$ docker commit 4ddf4638572d geektime/helloworld:v2
```

而由于使用了联合文件系统，你在容器里对镜像 rootfs 所做的任何修改，都会被操作系统先复制到这个可读写层，然后再修改。这就是所谓的：Copy-on-Write。

而正如前所说，Init 层的存在，就是为了避免你执行 docker commit 时，把 Docker 自己对 /etc/hosts 等文件做的修改，也一起提交掉。

#### 镜像上传系统

统一存放镜像的系统，就叫作 Docker Registry。
可以查看[Docker 的官方文档](https://docs.docker.com/registry/)，以及[VMware 的 Harbor 项目](https://github.com/goharbor/harbor)。

### docker exec 是怎么做到进入容器里的

一个进程的 Namespace 信息在宿主机上是以一个文件的方式存在。

查看当前正在运行的 Docker 容器的进程号
```
$ docker inspect --format '{{ .State.Pid }}'  4ddf4638572d
25686
```

查看宿主机的 proc 文件，看到这个 25686 进程的所有 Namespace 对应的文件：
```
$ ls -l  /proc/25686/ns
total 0
lrwxrwxrwx 1 root root 0 Aug 13 14:05 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 ipc -> ipc:[4026532278]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 mnt -> mnt:[4026532276]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 net -> net:[4026532281]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 pid -> pid:[4026532279]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 pid_for_children -> pid:[4026532279]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 uts -> uts:[4026532277]
```

可以看到，一个进程的每种 Linux Namespace，都在它对应的 /proc/[进程号]/ns 下有一个对应的虚拟文件，并且链接到一个真实的 Namespace 文件上。
我们就可以对 Namespace 做一些很有意义事情了，比如：加入到一个已经存在的 Namespace 当中。
> **这也就意味着：一个进程，可以选择加入到某个进程已有的 Namespace 当中，从而达到“进入”这个进程所在容器的目的，这正是 docker exec 的实现原理。**

#### Linux 系统调用 -- setnx()

```
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE);} while (0)

int main(int argc, char *argv[]) {
    int fd;
    
    fd = open(argv[1], O_RDONLY);
    if (setns(fd, 0) == -1) {
        errExit("setns");
    }
    execvp(argv[2], &argv[2]); 
    errExit("execvp");
}
```
- 第一个参数是 argv[1]，即当前进程要加入的 Namespace 文件的路径，比如 /proc/25686/ns/net；
- 而第二个参数，则是你要在这个 Namespace 里运行的进程，比如 /bin/bash。

**这段代码的核心操作，则是通过 open() 系统调用打开了指定的 Namespace 文件，并把这个文件的描述符 fd 交给 setns() 使用。在 setns() 执行后，当前进程就加入了这个文件对应的 Linux Namespace 当中了。**

现在，你可以编译执行一下这个程序，加入到容器进程（PID=25686）的 Network Namespace 中：

```
$ gcc -o set_ns set_ns.c 
$ ./set_ns /proc/25686/ns/net /bin/bash 
$ ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:02  
          inet addr:172.17.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:12 errors:0 dropped:0 overruns:0 frame:0
          TX packets:10 errors:0 dropped:0 overruns:0 carrier:0
     collisions:0 txqueuelen:0 
          RX bytes:976 (976.0 B)  TX bytes:796 (796.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
    collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

实际上，在 setns() 之后我看到的这两个网卡，正是我在前面启动的 Docker 容器里的网卡。也就是说，我新创建的这个 /bin/bash 进程，由于加入了该容器进程（PID=25686）的 Network Namepace，**它看到的网络设备与这个容器里是一样的**，即：/bin/bash 进程的网络设备视图，也被修改了。

而一旦一个进程加入到了另一个 Namespace 当中，在宿主机的 Namespace 文件上，也会有所体现。

找到这个 set_ns 程序执行的 /bin/bash 进程，其真实的 PID 是 28499：
```
# 在宿主机上
ps aux | grep /bin/bash
root     28499  0.0  0.0 19944  3612 pts/0    S    14:15   0:00 /bin/bash
```

查看一下这个 PID=28499 的进程的 Namespace，你就会发现这样一个事实：

```
$ ls -l /proc/28499/ns/net
lrwxrwxrwx 1 root root 0 Aug 13 14:18 /proc/28499/ns/net -> net:[4026532281]

$ ls -l  /proc/25686/ns/net
lrwxrwxrwx 1 root root 0 Aug 13 14:05 /proc/25686/ns/net -> net:[4026532281]
```

在 /proc/[PID]/ns/net 目录下，这个 PID=28499 进程，与我们前面的 Docker 容器进程（PID=25686）指向的 Network Namespace 文件完全一样。这说明这两个进程，共享了这个名叫 net:[4026532281]的 Network Namespace。

Docker 还专门提供了一个参数，可以让你**启动一个容器并“加入”到另一个容器的 Network Namespace 里**，这个参数就是 -net，比如:

```
$ docker run -it --net container:4ddf4638572d busybox ifconfig
```

如果我指定**–net=host**，就意味着这个容器不会为进程启用 Network Namespace。这就意味着，这个容器拆除了 Network Namespace 的“隔离墙”，所以，它会和宿主机上的其他普通进程一样，**直接共享宿主机的网络栈。**这就为容器直接操作和使用宿主机网络提供了一个渠道。

### Volume（数据卷）

容器技术使用了 rootfs 机制和 Mount Namespace，构建出了一个同宿主机完全隔离开的文件系统环境。这时候，我们就需要考虑这样两个问题：
- 容器里进程新建的文件，怎么才能让宿主机获取到？
- 宿主机上的文件和目录，怎么才能让容器里的进程访问到？

>**Volume 机制，允许你将宿主机上指定的目录或者文件，挂载到容器里面进行读取和修改操作。**

#### Volume 声明方式

在 Docker 项目里，它支持**两种** Volume 声明方式，可以把宿主机目录挂载进容器的 /test 目录当中：

```
$ docker run -v /test ...
$ docker run -v /home:/test ...
```
而这两种声明方式的本质，实际上是相同的：都是把一个宿主机的目录挂载进了容器的 /test 目录。
- 在第一种情况下，由于你并没有显示声明宿主机目录，那么 Docker 就会默认在宿主机上创建一个临时目录 /var/lib/docker/volumes/[VOLUME_ID]/_data，然后把它挂载到容器的 /test 目录上。
- 而在第二种情况下，Docker 就直接把宿主机的 /home 目录挂载到容器的 /test 目录上。

#### 挂载机制

当容器进程被创建之后，尽管开启了 Mount Namespace，但是在它执行 chroot（或者 pivot_root）之前，容器进程一直可以看到宿主机上的整个文件系统。

而宿主机上的文件系统，也自然包括了我们要使用的容器镜像。这个镜像的各个层，保存在 /var/lib/docker/aufs/diff 目录下，在容器进程启动后，它们会被联合挂载在 /var/lib/docker/aufs/mnt/ 目录中，这样容器所需的 rootfs 就准备好了。

**所以，我们只需要在 rootfs 准备好之后，在执行 chroot 之前，把 Volume 指定的宿主机目录（比如 /home 目录），挂载到指定的容器目录（比如 /test 目录）在宿主机上对应的目录（即 /var/lib/docker/aufs/mnt/[可读写层 ID]/test）上，这个 Volume 的挂载工作就完成了。**

更重要的是，由于执行这个挂载操作时，“容器进程”已经创建了，也就意味着此时 Mount Namespace 已经开启了。所以，这个挂载事件只在这个容器里可见。你在宿主机上，是看不见容器内部的这个挂载点的。**这就保证了容器的隔离性不会被 Volume 打破。**

>注意：这里提到的"容器进程"，是 Docker 创建的一个容器初始化进程 (dockerinit)，而不是应用进程 (ENTRYPOINT + CMD)。dockerinit 会负责完成根目录的准备、挂载设备和目录、配置 hostname 等一系列需要在容器内进行的初始化操作。最后，它通过 execv() 系统调用，让应用进程取代自己，成为容器里的 PID=1 的进程。

#### 绑定挂载（bind mount）
而这里要使用到的挂载技术，就是 Linux 的**绑定挂载（bind mount）**机制。它的主要作用就是，允许你将一个目录或者文件，而不是整个设备，挂载到一个指定的目录上。并且，这时你在该挂载点上进行的任何操作，只是发生在被挂载的目录或者文件上，而原挂载点的内容则会被隐藏起来且不受影响。

理解绑定挂载（bind mount）,参考：{% post_link 'Linux/mount --bind使用方法' %}

其实，如果你了解 Linux 内核的话，就会明白，绑定挂载实际上是一个 inode 替换的过程。在 Linux 操作系统中，inode 可以理解为存放文件内容的“对象”，而 dentry，也叫目录项，就是访问这个 inode 所使用的“指针”。

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/mount-bind-sc.png)

正如上图所示，mount --bind /home /test，会将 /home 挂载到 /test 上。其实相当于将 /test 的 dentry，重定向到了 /home 的 inode。这样当我们修改 /test 目录时，实际修改的是 /home 目录的 inode。这也就是为何，一旦执行 umount 命令，/test 目录原先的内容就会恢复：因为修改真正发生在的，是 /home 目录里。

**所以，在一个正确的时机，进行一次绑定挂载，Docker 就可以成功地将一个宿主机上的目录或文件，不动声色地挂载到容器中。**

这样，进程在容器里对这个 /test 目录进行的所有操作，都实际发生在宿主机的对应目录（比如，/home，或者 /var/lib/docker/volumes/[VOLUME_ID]/_data）里，而不会影响容器镜像的内容。

#### docker commit是否会提交挂载目录里的内容

不会

原因：容器的镜像操作，比如 docker commit，都是发生在宿主机空间的。而由于 Mount Namespace 的隔离作用，宿主机并不知道这个绑定挂载的存在。所以，在宿主机看来，容器中可读写层的 /test 目录（/var/lib/docker/aufs/mnt/[可读写层 ID]/test），**始终是空的。**

不过，由于 Docker 一开始还是要创建 /test 这个目录作为挂载点，所以执行了 docker commit 之后，你会发现新产生的镜像里，会多出来一个空的 /test 目录。毕竟，新建目录操作，又不是挂载操作，Mount Namespace 对它可起不到“障眼法”的作用。

#### 验证

启动一个 helloworld 容器，给它声明一个 Volume，挂载在容器里的 /test 目录上：
```
$ docker run -d -v /test helloworld
cf53b766fa6f
```

容器启动之后，我们来查看一下这个 Volume 的 ID：
```
$ docker volume ls
DRIVER              VOLUME NAME
local               cb1c2f7221fa9b0971cc35f68aa1034824755ac44a034c0c0a1dd318838d3a6d
```

然后，使用这个 ID，可以找到它在 Docker 工作目录下的 volumes 路径：
```
$ ls /var/lib/docker/volumes/cb1c2f7221fa/_data/
```

这个 _data 文件夹，就是这个容器的 Volume 在宿主机上对应的临时目录了。接下来，我们在容器的 Volume 里，添加一个文件 text.txt：
```
$ docker exec -it cf53b766fa6f /bin/sh
cd test/
touch text.txt
```

这时，我们再回到宿主机，就会发现 text.txt 已经出现在了宿主机上对应的临时目录里：
```
$ ls /var/lib/docker/volumes/cb1c2f7221fa/_data/
text.txt
```

可是，如果你在宿主机上查看该容器的可读写层，虽然可以看到这个 /test 目录，但其内容是空的（关于如何找到这个 AuFS 文件系统的路径，请参考我上一次分享的内容）：
```
$ ls /var/lib/docker/aufs/mnt/6780d0778b8a/test
```

可以确认，容器 Volume 里的信息，并不会被 docker commit 提交掉；但这个挂载点目录 /test 本身，则会出现在新的镜像当中。

### 总结

着重介绍了如何使用 Linux Namespace、Cgroups，以及 rootfs 的知识，对容器进行了一次庖丁解牛似的解读。

借助这种思考问题的方法，最后的 Docker 容器，我们实际上就可以用下面这个“全景图”描述出来：

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/container-panorama.png)

- 这个容器进程“python app.py”，运行在由 Linux Namespace 和 Cgroups 构成的隔离环境里；而它运行所需要的各种文件，比如 python，app.py，以及整个操作系统文件，则由多个联合挂载在一起的 rootfs 层提供。
- 这些 rootfs 层的最下层，是来自 Docker 镜像的只读层。
- 在只读层之上，是 Docker 自己添加的 Init 层，用来存放被临时修改过的 /etc/hosts 等文件。
- 而 rootfs 的最上层是一个可读写层，它以 Copy-on-Write 的方式存放任何对只读层的修改，容器声明的 Volume 的挂载点，也出现在这一层。

### 思考题

1. 你在查看 Docker 容器的 Namespace 时，是否注意到有一个叫 cgroup 的 Namespace？它是 Linux 4.6 之后新增加的一个 Namespace，你知道它的作用吗？
参考：{% post_link 'Linux/Cgroup namespace' %}
Cgroup namespace带来以下一些好处：

	1. 可以限制容器的cgroup filesytem视图，使得在容器中也可以安全的使用cgroup；

	2. 此外，会使容器迁移更加容易；在迁移时，/proc/self/cgroup需要复制到目标机器，这要求容器的cgroup路径是唯一的，否则可能会与目标机器冲突。有了cgroupns，每个容器都有自己的cgroup filesystem视图，不用担心这种冲突。

2. 如果你执行 docker run -v /home:/test 的时候，容器镜像里的 /test 目录下本来就有内容的话，你会发现，在宿主机的 /home 目录下，也会出现这些内容。这是怎么回事？为什么它们没有被绑定挂载隐藏起来呢？（提示：Docker 的“copyData”功能）
Docker的copyData功能可以使挂载点文件和挂载目录下的文件同时存在

3. 请尝试给这个 Python 应用加上 CPU 和 Memory 限制，然后启动它。根据我们前面介绍的 Cgroups 的知识，请你查看一下这个容器的 Cgroups 文件系统的设置，是不是跟我前面的讲解一致。
参考：[Docker容器CPU、memory资源限制](https://www.cnblogs.com/zhuochong/p/9728383.html)
