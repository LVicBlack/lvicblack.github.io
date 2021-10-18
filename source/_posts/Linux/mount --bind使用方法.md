---
title: mount --bind使用方法
date: 2021-10-18 11:06:00
categories: 
- Linux
- 基础概念
tags:
- Linux
---

我们可以通过mount --bind命令来将两个目录连接起来，mount --bind命令是将前一个目录挂载到后一个目录上，所有对后一个目录的访问其实都是对前一个目录的访问，如下所示：

```
## test1 test2为两个不同的目录
linux-UMLhEm:/home/test/linux # ls test1
11.test  1.test
linux-UMLhEm:/home/test/linux # ls test2
22.test  2.test
linux-UMLhEm:/home/test/linux # ls -lid test1
1441802 drwx------ 2 root root 4096 Feb 13 09:50 test1
linux-UMLhEm:/home/test/linux # ls -lid test2
1441803 drwx------ 2 root root 4096 Feb 13 09:51 test2

## 执行mount --bind 将test1挂载到test2上，inode号都变为test1的inode
linux-UMLhEm:/home/test/linux # mount --bind test1 test2
linux-UMLhEm:/home/test/linux # ls -lid test1
1441802 drwx------ 2 root root 4096 Feb 13 09:50 test1
linux-UMLhEm:/home/test/linux # ls -lid test2
1441802 drwx------ 2 root root 4096 Feb 13 09:50 test2
linux-UMLhEm:/home/test/linux # ls test2
11.test  1.test

## 对test2的访问或修改实际上是改动test1目录
linux-UMLhEm:/home/test/linux # cd test2
linux-UMLhEm:/home/test/linux/test2 # touch 3.test
linux-UMLhEm:/home/test/linux/test2 # ls
11.test  1.test  3.test
linux-UMLhEm:/home/test/linux/test2 # cd ..
linux-UMLhEm:/home/test/linux # ls test1
11.test  1.test  3.test

## 解挂载后，test1目录保持修改，test2保持不变
linux-UMLhEm:/home/test/linux # umount test2
linux-UMLhEm:/home/test/linux # ls test1
11.test  1.test  3.test
linux-UMLhEm:/home/test/linux # ls test2
22.test  2.test

## 将test2挂载到test1上
linux-UMLhEm:/home/test/linux # ls -lid test2
1441803 drwx------ 2 root root 4096 Feb 13 09:51 test2
linux-UMLhEm:/home/test/linux # mount --bind test2 test1
linux-UMLhEm:/home/test/linux # ls -lid test1
1441803 drwx------ 2 root root 4096 Feb 13 09:51 test1
linux-UMLhEm:/home/test/linux # ls -lid test2
1441803 drwx------ 2 root root 4096 Feb 13 09:51 test2
linux-UMLhEm:/home/test/linux # ls test1
22.test  2.test
```

mount --bind test1 test2为例，当mount --bind命令执行后，Linux将会把被挂载目录的目录项（也就是该目录文件的block，记录了下级目录的信息）屏蔽，即test2的下级路径被隐藏起来了（注意，只是隐藏不是删除，数据都没有改变，只是访问不到了）。同时，内核将挂载目录（test1）的目录项记录在内存里的一个s_root对象里，在mount命令执行时，VFS会创建一个vfsmount对象，这个对象里包含了整个文件系统所有的mount信息，其中也会包括本次mount中的信息，这个对象是一个HASH值对应表（HASH值通过对路径字符串的计算得来），表里就有 /test1 到 /test2 两个目录的HASH值对应关系。
    
命令执行完后，当访问 /test2下的文件时，系统会告知 /test2 的目录项被屏蔽掉了，自动转到内存里找VFS，通过vfsmount了解到 /test2 和 /test1 的对应关系，从而读取到 /test1 的inode，这样在 /test2 下读到的全是 /test1 目录下的文件。

1. mount --bind连接的两个目录的inode号码并不一样，只是被挂载目录的block被屏蔽掉，inode被重定向到挂载目录的inode（被挂载目录的inode和block依然没变）。

2. 两个目录的对应关系存在于内存里，一旦重启挂载关系就不存在了。

在固件开发过程中常常遇到这样的情况：为测试某个新功能，必需修改某个系统文件。而这个文件在只读文件系统上（总不能为一个小小的测试就重刷固件吧），或者是虽然文件可写，但是自己对这个改动没有把握，不愿意直接修改。这时候mount --bind就是你的好帮手。 
假设我们要改的文件是/etc/hosts，可按下面的步骤操作： 
1. 把新的hosts文件放在/tmp下。当然也可放在硬盘或U盘上。 
2. mount --bind /tmp/hosts /etc/hosts       此时的/etc目录是可写的，所做修改不会应用到原来的/etc目录，可以放心测试。测试完成了执行 umount /etc/hosts 断开绑定。 