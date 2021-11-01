---
title: JAVA JVM不同版本对比和整理
date: 2021-11-01 10:17:03
categories: 
- Java
- JVM
tags:
- Java
- JVM
---

## JDK1.6、JDK1.7、JDK1.8 内存模型演变
JDK1.6、JDK1.7、JDK1.8 内存模型演变图:

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101164823.png)

这个内存模型就是 JVM 运行时数据区依照JVM虚拟机规范的具体实现过程。
各个版本的迭代都是为了更好的适应CPU性能提升，最大限度提升的JVM运行效率。这些版本的JVM内存模型主要有以下差异：
- JDK 1.6：有永久代，静态变量存放在永久代上。
- JDK 1.7：有永久代，但已经把字符串常量池、静态变量，存放在堆上。逐渐的减少永久代的使用。
- JDK 1.8：无永久代，运行时常量池、类常量池，都保存在元数据区，也就是常说的元空间。但字符串常量池仍然存放在堆上。

JDK1.8内存模型图
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101165656.png)

其中：
- 方法区(Method Area)和Java堆（Java Heap）是**各个线程共享的内存区域**
- 程序计数器（Program Counter Register）、虚拟机栈（JVM Stacks）、本地方法栈（Native Method Stacks）是**线程隔离的数据区**

## 内存模型各区域介绍

### 1. 程序计数器（Program Counter Register）

- 较小的内存空间、线程私有，记录当前线程所执行的字节码行号。
- 如果线程执行的是非native方法，则程序计数器中保存的是当前需要执行的指令的地址；如果线程执行的是native方法，则程序计数器中的值是undefined。
- **此内存区域是唯一 一个在Java虚拟机规范中没有规定OutOfMemoryError情况的区域。**

举个栗子：计算圆形的周长。
```
public static float circumference(float r){
        float pi = 3.14f;
        float area = 2 * pi * r;
        return area;
}
```
这段代码的在虚拟机中的执行过程，左侧是它的程序计数器对应的行号：

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101172027.png)

- 这些行号每一个都会对应一条需要执行的字节码指令，是压栈还是弹出或是执行计算。
- 之所以说是线程私有的，因为如果不是私有的，那么整个计算过程最终的结果也将错误。

### 2. Java虚拟机栈（JVM Stacks）

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101172438.png)

- 每个方法在执行的同时都会创建一个栈帧（Stack Frame）用于存储**局部变量表、操作数栈、动态链接、方法出口**等信息。
- 方法从调用到执行完成，都对应着栈帧从虚拟机中入栈和出栈的过程。
- 最终，栈帧会随着方法的创建到结束而销毁。


**虚拟机栈可能会抛出两种异常：**
- 如果线程请求的栈深度大于虚拟机所规定的栈深度，则会抛出StackOverFlowError即栈溢出
- 如果虚拟机的栈容量可以动态扩展，那么当虚拟机栈申请不到内存时会抛出OutOfMemoryError即OOM内存溢出

经常有人把Java内存区分为堆内存（Heap）和栈内存（Stack），其中所指的“堆”就是Java堆，而所指的“栈”就是现在所讲的虚拟机栈，或者说是虚拟机栈中局部变量表部分。

局部变量表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，它不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关位置）和returnAddress类型（指向了一条字节码指令的地址）。

其中64为长度的long和double类型的数据会占用2个局部变量空间（Slot），其余的数据类型只占用1个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，**这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。**

### 3. 本地方法栈（Native Method Stacks）

- 本地方法栈与Java虚拟机栈作用类似，唯一不同的就是本地方法栈执行的是Native方法，而虚拟机栈是为JVM执行Java方法服务的。
- 另外，与 Java 虚拟机栈一样，本地方法栈也会抛出 StackOverflowError 和 OutOfMemoryError 异常。
- JDK1.8 HotSpot虚拟机直接就把本地方法栈和虚拟机栈合二为一。

#### 参数设置
- -Xss 设置每个线程的栈大小。JDK1.5+ 每个线程栈大小为1M，一般来说如果栈不是很深的话，1M是绝对够用的。

#### 参数含义解析
- 以-X开头的参数是和实现有关的，第一个s表示stack，第二个s表示size；

>注意：
>在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。

### 4. Java堆（Java Heap）

对于大多数应用来说，Java堆（Java Heap）是**Java虚拟机所管理的内存中最大的一块。**

Heap的组成：
- Java 堆，由年轻代和年老代组成，分别占据1/3和2/3。
- 而年轻代又分为三部分，Eden、From Survivor、To Survivor，占据比例为8:1:1，可调。

存放：
- 对象实例
	- 类初始化生成的对象
	- 基本数据类型的数组也是对象实例
- 线程分配缓冲区（Thread Local Allocation Buffer）
	- 线程私有，但是不影响java堆的共性
	- 增加线程分配缓冲区是为了提升对象分配时的效率

根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像我们的磁盘空间一样。
在实现时，既可以实现成固定大小的，也可以是扩展的，不过当前主流的虚拟机都是按照可扩展来实现的（通过-Xmx和-Xms控制）。
如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。

#### 参数设置
- -Xms 设置堆的最小空间大小；通常为操作系统可用内存的1/64大小即可。
- -Xmx 设置堆的最大空间大小；通常为操作系统可用内存的1/4大小。
- -Xmn 设置新生代大小，是对-XX：newSize、-XX：MaxnewSize两个参数的同时配置，这个参数是在JDK1.4版本以后出现的；通常为Xmx的1/3或1/4。新生代 = Eden + 2个Survivor空间。实际可用空间 = Eden + 1个Survivor，即90%。
- -XX：NewSize 设置新生代最小空间大小；
- -XX：MaxNewSize 设置新生代最大空间大小；
- -XX：NewRatio 新生代与老年代的比例，如-XX：NewRatio=2，则新生代占整个堆空间的1/3，老年代占2/3。
- -XX：SurvivorRatio 新生代中 Eden 与 Survivor的比值。默认值为 8 。即Eden占新生代空间的8/10，另外两个Survivor各占1/10。

#### 参数含义解析
- 以-X开头的参数是和实现有关的，并不是适用于所有的参数；
- 最开始只有 -Xms的参数，表示‘初始’ memory size，m表示memory，s表示size；
- 紧接是参数 -Xmx，为了对齐三字符，压缩了其表示形式，采用计算机中约定表示方式：用 x 表示“”大“ （可以联想到衣服的号码大小，S、M、L、XL、XXL），因此 -Xmx中的m应当还是memory。既然有了最大内存的概念，那么一开始的 -Xms所表示的”初始“内存也就有了一个”最小“内存的概念（其实常用的做法中初始内存采用的也就是最小内存）。如果不对齐参数长度的话，其表示应当是-Xmsx。

>注意：
>	开发过程中，通常会将-Xms与-Xmx两个参数的配置相同的值，其目的是为了能够在Java垃圾回收机制清理完堆区后不需要重新分隔计算堆区的大小而浪费资源。

### 5. 方法区(Method Area) / 非堆（Non-Heap）

- 用于存储已被虚拟机加载的**类信息、常量、静态变量**、即时编译器**编译后的代码**等数据，即存放静态文件，如Java类、方法等。

方法区在java8以前是放在JVM内存中的，由永久代（PermGen）实现，受JVM内存大小参数的限制，在java8中移除了永久代的内容，方法区由元空间(Meta Space)实现，并直接放到了本地内存中，不受JVM参数的限制（当然，如果物理内存被占满了，方法区也会报OOM），并且**将原来放在方法区的字符串常量池和静态变量都转移到了Java堆中。**

jdk1.8以前版本的 class和JAR包数据存储在 PermGen下面 ，PermGen 大小是固定的，而且项目之间无法共用，公有的 class，所以比较容易出现OOM异常。

方法区在编译期间和类加载完成后的内容有少许不同，不过总的来说分为这两部分：
- 类常量池（Class Constant Pool）
	- 类元信息在类编译期间放入方法区，里面放置了类的基本信息，包括类的版本、字段、方法、接口以及常量池表（Constant Pool Table）
	- 常量池表（Constant Pool Table）存储了类在编译期间生成的字面量、符号引用(什么是字面量？什么是符号引用？)，这些信息在类加载完后会被解析到运行时常量池中
- 运行时常量池（Runtime Constant Pool）
	- 运行时常量池主要存放在类加载后被解析的字面量与符号引用，但不止这些
	- 运行时常量池具备动态性，可以添加数据，比较多的使用就是String类的intern()方法

#### 元空间

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101175926.png)



#### 元空间查看
- jps 查看pid
- jinfo -flag MetaspaceSize pid 查看元空间大小，默认MetaspaceSize大小21807104（约20M）
- jinfo -flag MaxMetaspaceSize pid 查看元空间最大大小













































参考：

