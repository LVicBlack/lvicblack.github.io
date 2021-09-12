---
title: JVM
date: 2020-03-17 06:17:03
categories: 
- Java
tags:
- Java
- 面试
---

## 1.类的加载过程：

![687474703a2f2f7374617469632e7a7962756c756f2e636f6d2f686f6d6973732f737936663436726165787437676c6179673832776274786e2f696d6167655f31626c39743167656731356c6a35387034316b643135316b3639392e706e67.png](image/687474703a2f2f7374617469632e7a7962756c756f2e636f6d2f686f6d6973732f737936663436726165787437676c6179673832776274786e2f696d6167655f31626c39743167656731356c6a35387034316b643135316b3639392e706e67.png)

    

加载：加载类的字节码文件，在堆内存生成对应的class文件

    连接：分为三部分

        验证：验证类文件格式，字节码是否正确，符号引用是否正确

        准备：加载类的静态变量，赋予初始默认值、

        解析：类的符号引用转为直接引用。类或接口的解析，2.字段解析，3.类方法解析，4.接口方法解析。

    初始化：将静态变量赋予初始值。

    使用：new对象

    卸载：垃圾回收

### 1.1类加载器：

![687474703a2f2f7374617469632e7a7962756c756f2e636f6d2f686f6d6973732f613761776e367471346d7a67766a3739726c7272787038712f696d6167655f31626c3974343737656931723165306b3139336d707536317132696d2e706e67.png](image/687474703a2f2f7374617469632e7a7962756c756f2e636f6d2f686f6d6973732f613761776e367471346d7a67766a3739726c7272787038712f696d6167655f31626c3974343737656931723165306b3139336d707536317132696d2e706e67.png)**

    Bootstrap ClassLoader：启动类加载器，c++实现，负责lib包和rt.jar

    Extension ClassLoader：扩展类加载器，java实现，lib\\ext文件夹或者java.ext.dirs系统变量指定的路径中

    Application ClassLoader：自定义加载器，负责加载ClassPath路径中指定的类

### 1.2双亲委派模型：

    当一个类加载器收到加载请求，会层层向上找到根加载器，即启动类加载器，若没有则向下去扩展类加载器中进行寻找，还未找到报错ClassNotFoundException,接着会在自定义加载器中进行加载。

    作用: 避免代码被错误引用

## 2.JVM的内存结构

![a6efce1b9d16fdfa1c6aa223708d4b5095ee7b84.jpg](image/a6efce1b9d16fdfa1c6aa223708d4b5095ee7b84.jpg)

    栈：存放局部变量

    堆：存放所有new出来的东西

        Eden: 这是对象最初诞生的区域，并且对大多数对象来说，这里是它们唯一存在过的区域。

        Survivor: 从伊甸园幸存下来的对象会被挪到这里。

        Tenured:这是足够老的幸存对象的归宿。年轻代收集（Minor-GC）过程是不会触及这个地方的。当年轻代收集不能把对象放进终身颐养园时，就会触发一次完全收集（Major-GC），这里可能还会牵扯到压缩，以便为大对象腾出足够的空间

    方法区：被虚拟机加载的类信息、常量、静态常量等。

    程序计数器(和系统相关)

    本地方法栈

## 3.GC(包括GC算法，垃圾回收器)

### 3.1 GC算法

#### 3.1.1、标记清除算法

标记-清除算法分为标记和清除两个阶段。该算法首先从根集合进行<span style="background-color: #ffaaaa">扫描</span>，对存活的对象对象标记，标记完毕后，再<span style="background-color: #ffaaaa">扫描</span>整个空间中未被标记的对象并进p行回收，如下图所示。

%\!(EXTRA markdown.ResourceType=, string=, string=)

标记-清除算法的主要不足有两个：

效率问题：标记和清除两个过程的效率都不高;

空间问题：标记-清除算法不需要进行对象的移动，并且仅对不存活的对象进行处理，因此标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

![9300974-5be4e9a0f4e81054.webp](image/9300974-5be4e9a0f4e81054.webp)

#### 3.1.2、复制算法(新生代)

　　<span style="background-color: #ffaaaa">复制算法将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉</span>。这种算法适用于对象存活率低的场景，比如新生代。这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。该算法示意图如下所示：

![9300974-991bdc703b8d87e1.webp](image/9300974-991bdc703b8d87e1.webp)

事实上，现在商用的虚拟机都采用这种算法来回收新生代。因为研究发现，新生代中的对象每次回收都基本上只有10%左右的对象存活，所以需要复制的对象很少，效率还不错。实践中会将<span style="background-color: #ffaaaa">新生代内存分为一块较大的Eden空间和两块较小的Survivor空间</span> (如下图所示)，<span style="background-color: #ffaaaa">每次使用Eden和其中一块Survivor</span>。当回收时，将Eden和Survivor中还存活着的对象一次地复制到另外一块Survivor空间上，最后清理掉Eden和刚才用过的Survivor空间。HotSpot虚拟机默认Eden和Survivor的大小比例是 8:1，也就是每次新生代中可用内存空间为整个新生代容量的90% ( 80%+10% )，只有10% 的内存会被“浪费”。

![9300974-74a6eb9f1c1efd5d.webp](image/9300974-74a6eb9f1c1efd5d.webp)

#### 3.1.3、标记整理算法(老年代)

[GC算法之三 标记-压缩算法](evernote:///view/18347854/s13/70e5efa6-d0ee-4277-bff6-9b23beda0c0d/70e5efa6-d0ee-4277-bff6-9b23beda0c0d/)

　　复制收集算法在对象存活率较高时就要进行较多的复制操作，效率将会变低。更关键的是，如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况，所以在老年代一般不能直接选用这种算法。标记整理算法的标记过程类似标记清除算法，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存，类似于磁盘整理的过程，该垃圾回收算法适用于对象存活率高的场景（老年代），其作用原理如下图所示。

![9300974-edc0ffe66b66b877.webp](image/9300974-edc0ffe66b66b877.webp)

标记整理算法与标记清除算法最显著的区别是：标记清除算法不进行对象的移动，并且仅对不存活的对象进行处理；而标记整理算法会将所有的存活对象移动到一端，并对不存活对象进行处理，因此其不会产生内存碎片。标记整理算法的作用示意图如下：

![9300974-95f10ab387f047bb.webp](image/9300974-95f10ab387f047bb.webp)

#### 3.1.4、分代收集算法

　　对于一个大型的系统，当创建的对象和方法变量比较多时，堆内存中的对象也会比较多，如果逐一分析对象是否该回收，那么势必造成效率低下。分代收集算法是基于这样一个事实：不同的对象的生命周期(存活情况)是不一样的，而不同生命周期的对象位于堆中不同的区域，因此对堆内存不同区域采用不同的策略进行回收可以提高 JVM 的执行效率。当代商用虚拟机使用的都是分代收集算法：新生代对象存活率低，就采用复制算法；老年代存活率高，就用标记清除算法或者标记整理算法。Java堆内存一般可以分为新生代、老年代和永久代三个模块，如下图所示：

![9300974-d7eb4be835fde8df.webp](image/9300974-d7eb4be835fde8df.webp)

1).新生代（Young Generation）

　　新生代的目标就是尽可能快速的收集掉那些生命周期短的对象，一般情况下，所有新生成的对象首先都是放在新生代的。新生代内存按照<span style="background-color: #ffaaaa">8:1:1</span>的比例分为一个eden区和两个survivor(survivor0，survivor1)区，大部分对象在Eden区中生成。在进行垃圾回收时，先将eden区存活对象复制到survivor0区，然后清空eden区，当这个survivor0区也满了时，则将eden区和survivor0区存活对象复制到survivor1区，然后清空eden和这个survivor0区，此时survivor0区是空的，然后交换survivor0区和survivor1区的角色（即下次垃圾回收时会扫描Eden区和survivor1区），即保持survivor0区为空，如此往复。特别地，当survivor1区也不足以存放eden区和survivor0区的存活对象时，就将存活对象直接存放到老年代。如果老年代也满了，就会触发一次FullGC，也就是新生代、老年代都进行回收。注意，新生代发生的GC也叫做MinorGC，MinorGC发生频率比较高，不一定等 Eden区满了才触发。

2).老年代（Old Generation）

　　老年代存放的都是一些生命周期较长的对象，就像上面所叙述的那样，在新生代中经历了N次垃圾回收后仍然存活的对象就会被放到老年代中。此外，老年代的内存也比新生代大很多(大概比例是<span style="background-color: #ffaaaa">1:2</span>)，当老年代满时会触发Major GC(Full GC)，老年代对象存活时间比较长，因此FullGC发生的频率比较低。

3).永久代（Permanent Generation）

　　永久代主要用于存放静态文件，如Java类、方法等。永久代对垃圾回收没有显著影响，但是有些应用可能动态生成或者调用一些class，例如使用反射、动态代理、CGLib等bytecode框架时，在这种时候需要设置一个比较大的永久代空间来存放这些运行过程中新增的类。

#### 3.1.5、小结

![9300974-e8f60285abb799b2.webp](image/9300974-e8f60285abb799b2.webp)

### 3.2 垃圾回收器

**HotSpot虚拟机所包含的收集器：**

![1326194-20181017145352803-1499680295.png](image/1326194-20181017145352803-1499680295.png)

图中展示了7种作用于不同分代的收集器，如果两个收集器之间存在连线，则说明它们可以搭配使用。虚拟机所处的区域则表示它是属于新生代还是老年代收集器。

**新生代收集器**：Serial、ParNew、Parallel Scavenge

**老年代收集器**：CMS、Serial Old、Parallel Old

**整堆收集器**： G1

### **几个相关概念：**

**并行收集**：指多条垃圾收集线程并行工作，但<span style="background-color: #ffaaaa">此时用户线程仍处于等待状态</span>。

**并发收集**：指用户线程与垃圾收集线程同时工作（不一定是并行的可能会交替执行）。用户程序在继续运行，而垃圾收集程序运行在另一个CPU上。

**吞吐量**：即CPU用于运行用户代码的时间与CPU总消耗时间的比值（<span style="background-color: #ffaaaa">吞吐量 = 运行用户代码时间 / ( 运行用户代码时间 + 垃圾收集时间 )</span>）。例如：虚拟机共运行100分钟，垃圾收集器花掉1分钟，那么吞吐量就是99%

#### 3.2.1：Serial 收集器

Serial收集器是最基本的、发展历史最悠久的收集器。

**特点：**<span style="background-color: #ffaaaa">单线程、简单高效</span>（与其他收集器的单线程相比），对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程手机效率。收集器进行垃圾回收时，必须暂停其他所有的工作线程，直到它结束（Stop The World）。

**应用场景**：适用于Client模式下的虚拟机。

**Serial / Serial Old收集器运行示意图**

![1326194-20181017164151556-1187071653.png](image/1326194-20181017164151556-1187071653.png)

 

#### 3.2.2：ParNew收集器

ParNew收集器其实就是<span style="background-color: #ffaaaa">Serial收集器的多线程版本。</span>

除了使用多线程外其余行为均和Serial收集器一模一样（参数控制、收集算法、Stop The World、对象分配规则、回收策略等）。

**特点**：多线程、ParNew收集器默认开启的收集线程数与CPU的数量相同，在CPU非常多的环境中，可以使用<span style="background-color: #ffaaaa">-XX:ParallelGCThreads参数来限制垃圾收集的线程数</span>。

　　　和Serial收集器一样存在Stop The World问题

**应用场景**：ParNew收集器是许多运行在Server模式下的虚拟机中首选的新生代收集器，因为它是<span style="background-color: #ffaaaa">除了Serial收集器外，唯一一个能与CMS收集器配合工作的</span><span style="background-color: #ffaaaa">。</span>

**ParNew/Serial Old组合收集器运行示意图如下：**

 ![1326194-20181017170542230-1929674942.png](image/1326194-20181017170542230-1929674942.png)

 

#### 3.2.3：Parallel Scavenge 收集器

与吞吐量关系密切，故也称为吞吐量优先收集器。

**特点**：属于<span style="background-color: #ffaaaa">新生代收集器</span>也是采用<span style="background-color: #ffaaaa">复制算法</span>的收集器，又是<span style="background-color: #ffaaaa"><span style="background-color: #ffaaaa">并行</span></span><span style="background-color: #ffaaaa">的多线程</span><span style="background-color: #ffaaaa">收集器</span>（与ParNew收集器类似）。

该收集器的目标是达到一个可控制的吞吐量。还有一个值得关注的点是：GC自适应调节策略（与ParNew收集器最重要的一个区别）

**GC自适应调节策略**：Parallel Scavenge收集器可设置<span style="background-color: #ffaaaa">-XX:+UseAdptiveSizePolicy</span>参数。当开关打开时不需要手动指定新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX:SurvivorRation）、晋升老年代的对象年龄（-XX:PretenureSizeThreshold）等，虚拟机会根据系统的运行状况收集性能监控信息，<span style="background-color: #ffaaaa">动态设置这些参数以提供最优的停顿时间和最高的吞吐量</span>，这种调节方式称为GC的自适应调节策略。

Parallel Scavenge收集器使用两个参数控制吞吐量：

* <span style="background-color: #ffaaaa">XX:MaxGCPauseMillis 控制最大的垃圾收集停顿时间</span>
* <span style="background-color: #ffaaaa">XX:GCRatio 直接设置吞吐量的大小。</span>

#### 3.2.4：Serial Old 收集器

Serial Old是Serial收集器的老年代版本。

**特点**：同样是<span style="background-color: #ffaaaa">单线程收集器</span>，采用<span style="background-color: #ffaaaa"><span style="background-color: #ffaaaa">标记-整理</span><span style="background-color: #ffaaaa">算法。</span></span>

**应用场景**：主要也是使用在Client模式下的虚拟机中。也可在Server模式下使用。

Server模式下主要的两大用途（在后续中详细讲解···）：

1. 在JDK1.5以及以前的版本中与Parallel Scavenge收集器搭配使用。
2. 作为CMS收集器的后备方案，在并发收集Concurent Mode Failure时使用。

Serial / Serial Old收集器工作过程图（Serial收集器图示相同）：

![1326194-20181017164151556-1187071653.png](image/1326194-20181017164151556-1187071653.png)

#### 3.2.5：Parallel Old 收集器

是Parallel Scavenge收集器的老年代版本。

**特点**：<span style="background-color: #ffaaaa">多线程</span>，采用<span style="background-color: #ffaaaa">标记-整理算法</span>。

**应用场景**：注重高吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge+Parallel Old 收集器。

**Parallel Scavenge/Parallel Old收集器工作过程图：**

![1326194-20181017215237165-1960446438.png](image/1326194-20181017215237165-1960446438.png)

#### 3.2.6：CMS收集器

一种以获取最短回收停顿时间为目标的收集器。

**特点**：基于<span style="background-color: #ffaaaa">标记-清除</span><span style="background-color: #ffaaaa">算法</span>实现。<span style="background-color: #ffaaaa">并发收集、低停顿。</span>

**应用场景**：适用于<span style="background-color: #ffaaaa">注重服务的响应速度</span>，希望系统停顿时间最短，给用户带来更好的体验等场景下。如web程序、b/s服务。

**CMS收集器的运行过程分为下列4步：**

**初始标记**：标记GC Roots能直接到的对象。速度很快但是仍存在Stop The World问题。

**并发标记**：进行GC Roots Tracing 的过程，找出存活对象且用户线程可并发执行。

**重新标记**：为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录。仍然存在Stop The World问题。

**并发清除**：对标记的对象进行清除回收。

 CMS收集器的内存回收过程是与用户线程一起并发执行的。

 CMS收集器的工作过程图：

![1326194-20181017221500926-2071899824.png](image/1326194-20181017221500926-2071899824.png)

CMS收集器的缺点：

* 对CPU资源非常敏感。
* <span style="background-color: #ffaaaa">无法处理浮动垃圾</span>，可能出现Concurrent Model Failure失败而导致另一次Full GC的产生。
* 因为采用标记-清除算法所以会存在

#### 3.2.7：G1收集器

一款面向服务端应用的垃圾收集器。

**特点如下：**

并行与并发：G1能充分利用多CPU、多核环境下的硬件优势，使用多个CPU来缩短Stop-The-World停顿时间。部分收集器原本需要停顿Java线程来执行GC动作，G1收集器仍然可以通过并发的方式让Java程序继续运行。

分代收集：G1能够独自管理整个Java堆，并且采用不同的方式去处理新创建的对象和已经存活了一段时间、熬过多次GC的旧对象以获取更好的收集效果。

空间整合：G1运作期间不会产生空间碎片，收集后能提供规整的可用内存。

可预测的停顿：G1除了追求低停顿外，还能建立可预测的停顿时间模型。能让使用者明确指定在一个长度为M毫秒的时间段内，消耗在垃圾收集上的时间不得超过N毫秒。

**G1为什么能建立可预测的停顿时间模型？**

因为它有计划的避免在整个Java堆中进行全区域的垃圾收集。G1跟踪各个Region里面的垃圾堆积的大小，在后台维护一个优先列表，<span style="background-color: #ffaaaa">每次根据允许的收集时间，优先回收价值最大的Region</span>。这样就保证了在有限的时间内可以获取尽可能高的收集效率。

**G1与其他收集器的区别**：

其他收集器的工作范围是整个新生代或者老年代、G1收集器的工作范围是<span style="background-color: #ffaaaa">整个Java堆</span>。在使用G1收集器时，它将整个Java堆划分为多个大小相等的独立区域（Region）。虽然也保留了新生代、老年代的概念，但新生代和老年代不再是相互隔离的，他们都是一部分Region（不需要连续）的集合。

**G1收集器存在的问题：**

Region不可能是孤立的，分配在Region中的对象可以与Java堆中的任意对象发生引用关系。在采用可达性分析算法来判断对象是否存活时，<span style="background-color: #ffaaaa">得扫描整个Java堆才能保证准确性</span>。其他收集器也存在这种问题（G1更加突出而已）。会导致Minor GC效率下降。

**G1收集器是如何解决上述问题的？**

采用Remembered Set来避免整堆扫描。G1中每个Region都有一个与之对应的Remembered Set，虚拟机发现程序在对Reference类型进行写操作时，会产生一个Write Barrier暂时中断写操作，检查Reference引用对象是否处于多个Region中（即检查老年代中是否引用了新生代中的对象），如果是，便通过CardTable把相关引用信息记录到被引用对象所属的Region的Remembered Set中。当进行内存回收时，在GC根节点的枚举范围中加入Remembered Set即可保证不对全堆进行扫描也不会有遗漏。

**如果不计算维护 Remembered Set 的操作，G1收集器大致可分为如下步骤：**

**初始标记**：仅标记GC Roots能直接到的对象，并且修改TAMS（Next Top at Mark Start）的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建新对象。（需要线程停顿，但耗时很短。）

**并发标记**：从GC Roots开始对堆中对象进行可达性分析，找出存活对象。（耗时较长，但可与用户程序并发执行）

**最终标记**：为了修正在并发标记期间因用户程序执行而导致标记产生变化的那一部分标记记录。且对象的变化记录在线程Remembered Set  Logs里面，把Remembered Set  Logs里面的数据合并到Remembered Set中。（需要线程停顿，但可并行执行。）

**筛选回收**：对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划。（可并发执行）

**G1收集器运行示意图：**

![1326194-20181017225802481-709835773.png](image/1326194-20181017225802481-709835773.png)

## 4.触发Full gc条件

### 4.1.调用System.gc

```
/**
* -XX:+UseSerialGC -Xms200M -Xmx200M -Xmn32m -XX:SurvivorRatio=8 -XX:+PrintGCDetails
* @author gxl
*
*/
public class SimulateFullGc
{
    public static void main(String[] args)
    {
        //模拟fullgc场景
        //场景1 使用System.gc
        List<Object> l = new ArrayList<Object>();
        for (int i =0; i< 100;i++)
        {
            l.add(new byte[1024*1024 ]);
            if (i % 10 ==0)
            {
                System.gc();
            }
        }

    }
}
```

### 4.2.老年代空间不足

```
/**
* -XX:+UseSerialGC -Xms200M -Xmx200M -Xmn32m -XX:SurvivorRatio=8 -XX:+PrintGCDetails
* @author gxl
*
*/
public class SimulateFullGc
{
    public static void main(String[] args)
    {
        //模拟fullgc场景
        //老年代空间不足
        //按照上面的参数推算:老年代大小: 200 -32m = 168M

        byte [] MAXOBJ = new byte [1024 * 1024 * 100]; // 100M

        byte [] MAXOBJ2 = new byte [1024 * 1024 * 70]; // 60M
        MAXOBJ = null;

        byte [] MAXOBJ3 = new byte [1024 * 1024 * 100]; // 60M
    }
}
```

### 4.3.永久代空间不足

```
/**
* -XX:+UseSerialGC -Xms200M -Xmx200M -Xmn32m -XX:SurvivorRatio=8 -XX:+PrintGCDetails -XX:MaxPermSize=10M
* @author gxl
*
*/
public class SimulateFullGc
{
    public static void main(String[] args)
    {
        //模拟fullgc场景
        //持久代空间不足
        //class 加载信息
        //需要cglib + asm (http://forge.ow2.org/projects/asm/)
        while (true)
        {
            Enhancer en = new Enhancer();
            en.setSuperclass(OOMObject.class);
            en.setUseCache(false);
            en.setCallback(new MethodInterceptor()
            {

                @Override
                public Object intercept(Object arg0, Method arg1, Object[] arg2,
                        MethodProxy arg3) throws Throwable
                {
                    // TODO Auto-generated method stub
                    return null;
                }
            });
            en.create();
        }
    }
    static class OOMObject
    {

    }
}
```

### 4.4.gc 担保失败


### 4.5.Cocurrent mode failure

```
发生在cms的清理sweep阶段,发现有新的垃圾产生,而且老年代没有足够空间导致的. 关于cms: 初始标记(STW) - >并发标记　－＞重新标记（STW）　－＞并发清除． STW = stop the world.
```

## 5.JVM调优化

jvm调优没有一个固定模板配置说必须如何操作，它需要根据系统的情况不同对待。

但是可以有如下建议：

1. 初始化内存和最大内存尽量保持一致，避免内存不够用继续扩充内存。最大内存不要超过物理内存，例如内存8g，你可以设置最大内存4g/6g但是不能超过8g否则加载类的时候没有空间会报错。

2. gc/full gc频率不要太高、每次gc时间不要太长、根据系统应用来定。

### **调优命令**

Sun JDK监控和故障处理命令有jps jstat jmap jhat jstack jinfo

* jps，JVM Process Status Tool,显示指定系统内所有的HotSpot虚拟机进程。
* jstat，JVM statistics Monitoring是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。
* jmap，JVM Memory Map命令用于生成heap dump文件
* jhat，JVM Heap Analysis Tool命令是与jmap搭配使用，用来分析jmap生成的dump，jhat内置了一个微型的HTTP/HTML服务器，生成dump的分析结果后，可以在浏览器中查看
* jstack，用于生成java虚拟机当前时刻的线程快照。
* jinfo，JVM Configuration info 这个命令作用是实时查看和调整虚拟机运行参数。

### **性能调优**

* 设定堆内存大小

-Xmx：堆内存最大限制。

* 设定新生代大小。 新生代不宜太小，否则会有大量对象涌入老年代

-XX:NewSize：新生代大小

-XX:NewRatio  新生代和老生代占比

-XX:SurvivorRatio：伊甸园空间和幸存者空间的占比

* 设定垃圾回收器 年轻代用  -XX:+UseParNewGC  年老代用-XX:+UseConcMarkSweepGC
