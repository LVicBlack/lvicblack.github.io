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

- 本地方法栈与Java虚拟机栈作用类似，唯一不同的就是本地方法栈执行的是**Native方法**，而虚拟机栈是为JVM执行Java方法服务的。
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

堆和元空间模型：
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101231125.png)

- 用于存储已被虚拟机加载的**类信息、常量、静态变量**、即时编译器**编译后的代码**等数据，即存放静态文件，如Java类、方法等。

>方法区在java8以前是放在JVM内存中的，由永久代（PermGen）实现，受JVM内存大小参数的限制，在java8中移除了永久代的内容，方法区由元空间(Meta Space)实现，并直接放到了本地内存中，不受JVM参数的限制（当然，如果物理内存被占满了，方法区也会报OOM），并且**将原来放在方法区的字符串常量池和静态变量都转移到了Java堆中。**

方法区在编译期间和类加载完成后的内容有少许不同，不过总的来说分为这两部分：
- 类常量池（Class Constant Pool）
	- 类元信息在类编译期间放入方法区，里面放置了类的基本信息，包括类的版本、字段、方法、接口以及常量池表（Constant Pool Table）
	- 常量池表（Constant Pool Table）存储了类在编译期间生成的字面量、符号引用(什么是字面量？什么是符号引用？)，这些信息在类加载完后会被解析到运行时常量池中
- 运行时常量池（Runtime Constant Pool）
	- 运行时常量池主要存放在类加载后被解析的字面量与符号引用，但不止这些
	- 运行时常量池具备动态性，可以添加数据，比较多的使用就是String类的intern()方法

#### 永久代（PermGen）

永久代是hotspot 的jdk1.8以前的实现，使用jdk1.7的老司机肯定以前经常遇到过“java.lang.OutOfMemoryError: PremGen space”异常。这里的“PermGen space”其实指的就是方法区。

（**方法区和永久代的关系**）不过方法区和“PermGen space”又有着本质的区别。前者是JVM的规范，而后者则是JVM规范的一种实现，并且只有HotSpot才有“PermGen space”。

jdk1.8以前版本的 class和JAR包数据存储在 PermGen下面 ，PermGen 大小是固定的，而且项目之间无法共用，公有的 class，所以比较容易出现OOM异常。

#### 为什么要移除持久代

HotSpot团队选择移除持久代，有内因和外因两部分，从外因来说，我们看一下JEP 122的Motivation（动机）部分：
>This is part of the JRockit and Hotspot convergence effort. JRockit customers do not need to configure the permanent generation (since JRockit does not have a permanent generation) and are accustomed to not configuring the permanent generation.
>
>大致就是说移除持久代也是为了和JRockit进行融合而做的努力，JRockit用户并不需要配置持久代（因为JRockit就没有持久代）。

从内因来说，持久代大小受到-XX：PermSize和-XX：MaxPermSize两个参数的限制，而这两个参数又受到JVM设定的内存大小限制，这就导致**在使用中可能会出现持久代内存溢出的问题**，因此在Java 8及之后的版本中彻底移除了持久代而使用Metaspace来进行替代。

#### 元空间（Metaspace）

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101175926.png)

Metaspace是方法区在HotSpot中的实现，它**与持久代最大的区别**在于：Metaspace并不在虚拟机内存中而是使用本地内存，被存储在叫做Metaspace native memory。因此Metaspace具体大小理论上取决于32位/64位系统可用内存的大小

其实，移除永久代的工作从JDK 1.7就开始了。JDK 1.7中，存储在永久代的部分数据就已经转移到Java Heap或者Native Heap。但永久代仍存在于JDK 1.7中，并没有完全移除，譬如符号引用(Symbols)转移到了native heap；字面量(interned strings)转移到了Java heap；类的静态变量(class statics)转移到了Java heap。

Metaspace存放了以下信息：
- 虚拟机加载的类信息
- 常量池
- 静态变量
- 即时编译后的代码

对于僵死的类及类加载器的垃圾回收将在元数据使用达到“MaxMetaspaceSize”参数的设定值时进行。
适时地监控和调整元空间对于减小垃圾回收频率和减少延时是很有必要的。持续的元空间垃圾回收说明，可能存在类、类加载器导致的内存泄漏或是大小设置不合适

#### 使用元空间的优点
- 字符串常量池迁移到堆中，避免溢出
- 永久代会为 GC 带来不必要的复杂度，并且回收效率偏低。
- 通过类加载器来控制垃圾回收。

#### Metaspace参数设置
- -XX:MetaspaceSize	初始化的Metaspace大小，控制Metaspace发生GC的阈值。GC后，动态增加或者降低MetaspaceSize，默认情况下，这个值大小根据不同的平台在12M到20M之间浮动
- -XX:MaxMetaspaceSize	限制Metaspace增长上限，防止因为某些情况导致Metaspace无限使用本地内存，影响到其他程序，默认为无上限
- -XX:MinMetaspaceFreeRatio	当进行过Metaspace GC之后，会计算当前Metaspace的空闲空间比，如果空闲比小于这个参数，那么虚拟机增长Metaspace的大小，默认为40，即70%
- -XX:MaxMetaspaceFreeRatio	当进行过Metaspace GC之后，会计算当前Metaspace的空闲空间比，如果空闲比大于这个参数，那么虚拟机会释放部分Metaspace空间，默认为70，即70%
- -XX:MaxMetaspaceExpanison	Metaspace增长时的最大幅度，默认值为5M
- -XX:MinMetaspaceExpanison	Metaspace增长时的最小幅度，默认为330KB

#### 查看元空间
- jps 查看pid
- jinfo -flag MetaspaceSize pid 查看元空间大小，默认MetaspaceSize大小21807104（约20M）
- jinfo -flag MaxMetaspaceSize pid 查看元空间最大大小

#### 直接内存（DirectBuffer）

直接内存位于本地内存，不属于JVM内存，但是也会在物理内存耗尽的时候报OOM。

直接内存（Direct Memory）的容量大小可通过-XX：MaxDirectMemorySize参数来指定，如果不去指定，则默认与Java堆最大值（由-Xmx指定）一致。

>一般直接内存溢出的原因：
>
>在jdk1.4中加入了NIO（New Input/Putput）类，引入了一种基于通道（channel）与缓冲区（buffer）的新IO方式，它可以使用native函数直接分配堆外内存，然后通过存储在java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，这样可以在一些场景下大大提高IO性能，避免了在java堆和native堆来回复制数据。
>
>- ByteBuffer.allocate(capability) 第一种方式是分配VM堆内存，属于GC管辖范围，由于需要拷贝所以速度相对较慢。
>- ByteBuffer.allocateDirect(capability) 第二种方式是分配OS本地内存，不属于GC管辖范围，由于不需要内存拷贝所以速度相对较快。
>
>如果不断分配本地内存，堆内存很少使用，那么JVM就不需要执行GC，DirectByteBuffer对象们就不会被回收，这时候堆内存充足，但本地内存可能已经使用光了，再次尝试分配本地内存就会出现OutOfMemoryError，那程序就直接崩溃了。

### ex. 常量池
// TODO 



## 常见问题

### 什么是Native方法？

由于java是一门高级语言，离硬件底层比较远，有时候无法操作底层的资源，于是，java添加了native关键字，被native关键字修饰的方法可以用其他语言重写，这样，我们就可以写一个本地方法，然后用C语言重写，这样来操作底层资源。当然，使用了native方法会导致系统的可移植性不高，这是需要注意的。

### 成员变量、局部变量、类变量分别存储在内存的什么地方？

- 类变量
	- 类变量是用static修饰符修饰，定义在方法外的变量，随着java进程产生和销毁
	- 在java8之前把静态变量存放于方法区，在java8时存放在堆中

- 成员变量
	- 成员变量是定义在类中，但是没有static修饰符修饰的变量，随着类的实例产生和销毁，是类实例的一部分
	- 由于是实例的一部分，在类初始化的时候，从运行时常量池取出直接引用或者值，与初始化的对象一起放入堆中

- 局部变量
	- 局部变量是定义在类的方法中的变量
	- 在所在方法被调用时放入虚拟机栈的栈帧中，方法执行结束后从虚拟机栈中弹出，所以存放在虚拟机栈中

### 类常量池、运行时常量池、字符串常量池有什么关系？有什么区别？

- 类常量池与运行时常量池都存储在方法区，而字符串常量池在jdk7时就已经从方法区迁移到了java堆中。

- 在类编译过程中，会把类元信息放到方法区，类元信息的其中一部分便是类常量池，主要存放字面量和符号引用，而字面量的一部分便是文本字符，在类加载时将字面量和符号引用解析为直接引用存储在运行时常量池；

- 对于文本字符来说，它们会在解析时查找字符串常量池，查出这个文本字符对应的字符串对象的直接引用，将直接引用存储在运行时常量池；字符串常量池存储的是字符串对象的引用，而不是字符串本身。

### 什么是字面量？什么是符号引用？

- 字面量
java代码在编译过程中是无法构建引用的，字面量就是在编译时对于数据的一种表示:
```
int a=1;//这个1便是字面量
String b="i love u";//i love u便是字面量
```

- 符号引用
由于在编译过程中并不知道每个类的地址，因为可能这个类还没有加载，所以如果你在一个类中引用了另一个类，那么你完全无法知道他的内存地址，那怎么办，我们只能用他的类名作为符号引用，在类加载完后用这个符号引用去获取他的内存地址。

例子：我在com.demo.Solution类中引用了com.test.Quest，那么我会把com.test.Quest作为符号引用存到类常量池，等类加载完后，拿着这个引用去方法区找这个类的内存地址。






