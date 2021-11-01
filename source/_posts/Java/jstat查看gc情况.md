---
title: jstat查看gc情况(JAVA1.8)
date: 2021-11-01 13:17:03
categories: 
- Java
- JVM
tags:
- Java
- JVM
---
>注意！！！：使用的jdk版本是jdk8.

### 总述
jstat通常用来分析系统的垃圾回收情况。

#### 命令：

```
jstat -gccause pid 2000     #每格2秒输出结果

或

jstat -gcutil pid  2000
```

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135030.png)

#### 分析：

- S0、S1 代表两个Survivor区；
- E 代表 Eden 区；
- O（Old）代表老年代；
- P（Permanent）代表永久代；
- YGC（Young GC）代表Minor GC次数；
- YGCT代表Minor GC耗时；
- FGC（Full GC）代表Full GC次数；
- GCT代表Minor & Full GC共计耗时。

Java 堆分为新生代和老年代，新生代一般划分为三块区域，Eden + From Survivor + To Survivor，Eden 和 Survivor 的内存比为8:1，每次只使用一个Eden 和一个 Survivor 区域，另一个Survivor 用于复制收集算法回收内存。

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135110.png)

对象一般尽量分配到新生代中，而对于大对象（长字符串和大数组）直接分配在老年代中，同时“年龄”长的的对象会从新生代自动晋升到老年代中。

Java 方法区称为永久代，只有 HotSpot 虚拟机才存在永久代。
 

首先向eden区申请分配空间，如果空间够，就直接进行分配，否则进行一次Minor GC。
minor GC 首先会对Eden区的对象进行标记，标记出来存活的对象。然后把存活的对象copy到From空间。
如果From空间足够，则回收eden区可回收的对象。
如果from内存空间不够，则把From空间存活的对象复制到To区，如果TO区的内存空间也不够的话，则把To区存活的对象复制到老年代。
如果老年代空间也不够（或者达到触发老年年垃圾回收条件的话）则触发一次**full GC。**

>jstat命令可以查看堆内存各部分的使用量，以及加载类的数量。命令的格式如下：
>jstat [-命令选项] [vmid] [间隔时间/毫秒] [查询次数]
 

### 类加载统计：
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135555.png)

Loaded:加载class的数量
Bytes：所占用空间大小
Unloaded：未加载数量
Bytes:未加载占用空间
Time：时间


### 编译统计
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135601.png)　

Compiled：编译数量。
Failed：失败数量
Invalid：不可用数量
Time：时间
FailedType：失败类型
FailedMethod：失败的方法


### 垃圾回收统计
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135609.png)　

S0C：第一个幸存区的大小
S1C：第二个幸存区的大小
S0U：第一个幸存区的使用大小
S1U：第二个幸存区的使用大小
EC：伊甸园区的大小
EU：伊甸园区的使用大小
OC：老年代大小
OU：老年代使用大小
MC：方法区大小
MU：方法区使用大小
CCSC:压缩类空间大小
CCSU:压缩类空间使用大小
YGC：年轻代垃圾回收次数
YGCT：年轻代垃圾回收消耗时间
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间


### 堆内存统计
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135631.png)　 

NGCMN：新生代最小容量
NGCMX：新生代最大容量
NGC：当前新生代容量
S0C：第一个幸存区大小
S1C：第二个幸存区的大小
EC：伊甸园区的大小
OGCMN：老年代最小容量
OGCMX：老年代最大容量
OGC：当前老年代大小
OC:当前老年代大小
MCMN:最小元数据容量
MCMX：最大元数据容量
MC：当前元数据空间大小
CCSMN：最小压缩类空间大小
CCSMX：最大压缩类空间大小
CCSC：当前压缩类空间大小
YGC：年轻代gc次数
FGC：老年代GC次数


### 新生代垃圾回收统计
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135640.png)　

S0C：第一个幸存区大小
S1C：第二个幸存区的大小
S0U：第一个幸存区的使用大小
S1U：第二个幸存区的使用大小
TT:对象在新生代存活的次数
MTT:对象在新生代存活的最大次数
DSS:期望的幸存区大小
EC：伊甸园区的大小
EU：伊甸园区的使用大小
YGC：年轻代垃圾回收次数
YGCT：年轻代垃圾回收消耗时间


### 新生代内存统计
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135652.png)　

NGCMN：新生代最小容量
NGCMX：新生代最大容量
NGC：当前新生代容量
S0CMX：最大幸存1区大小
S0C：当前幸存1区大小
S1CMX：最大幸存2区大小
S1C：当前幸存2区大小
ECMX：最大伊甸园区大小
EC：当前伊甸园区大小
YGC：年轻代垃圾回收次数
FGC：老年代回收次数


### 老年代垃圾回收统计
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135705.png)　

MC：方法区大小
MU：方法区使用大小
CCSC:压缩类空间大小
CCSU:压缩类空间使用大小
OC：老年代大小
OU：老年代使用大小
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间


### 老年代内存统计
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135716.png)　

OGCMN：老年代最小容量
OGCMX：老年代最大容量
OGC：当前老年代大小
OC：老年代大小
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间


### 元数据空间统计
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135730.png)　

MCMN: 最小元数据容量
MCMX：最大元数据容量
MC：当前元数据空间大小
CCSMN：最小压缩类空间大小
CCSMX：最大压缩类空间大小
CCSC：当前压缩类空间大小
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间


### 总结垃圾回收统计
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135740.png)　

S0：幸存1区当前使用比例
S1：幸存2区当前使用比例
E：伊甸园区使用比例
O：老年代使用比例
M：元数据区使用比例
CCS：压缩使用比例
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间


### JVM编译方法统计
![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20211101135750.png)　

Compiled：最近编译方法的数量
Size：最近编译方法的字节码数量
Type：最近编译方法的编译类型。
Method：方法名标识。































