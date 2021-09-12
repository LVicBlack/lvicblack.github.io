---
title: redis存json数据时选择string还是hash
date: 2020-08-16 17:51:03
categories: 
- Redis
tags:
- Redis
- 面试
---

  我们在缓存json数据到redis时经常会面临是选择string类型还是选择hash类型去存储。接下来我从占用空间和IO两方面来分析这两种类型的优势。

### 1、占用空间

根据数据结构的共识我们知道hashtable类型是要比string类型更占用空间， 而ziplist类型与string类型占用的空间基本相差不大。

如下图就是ziplist的存储的格式

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/1490424-20210116110010652-1677330203.jpg)

那我们接下来分别分析redis的string和hash类型占用空间方面的知识

* string类型： string类型当然如其名，如果json数据以string类型去存储，那么它的空间占用方面肯定是相当的。
* hash类型： redis对hash类型是有两种编码方式，分别是ziplist和hashtable。
    > 当如下情况时redis的hash类型，底层是用ziplist编码的：
    > 
    > 1. 哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节；
    > 2. 哈希对象保存的键值对数量小于 512 个；
    > 
    > 不满足上述情况时，redis的hash类型，底层编码格式为hashtable。

### 2、IO

从IO的角度来分析string和hash类型，我们得有一个共识，我们知道redis是有服务端的，也就是部署redis的所在机器他们会运算能力的。

* string类型：
    * 取数据：根据redis键取对应的整个value值。
    * 存数据：根据redis键存这个value值
    * 更新数据： 根据redis键更新整个value值
* hash类型：
    * 取数据：根据redis键，然后遍历整个hash键值对（相对string的取数据更加耗时）。
    * 存数据：根据redis键，在value出存键值对
    * 更新数据：根据redis键和hash key更新对应的数据

### 3、总结

综上所述，那具体怎么选择是用string类型还是hash类型存储json数据呢？给出以下结论

* 如果你的业务类型中对于缓存的读取缓存的场景更多，并且更新缓存不频繁（或者每次更新都更新json数据中的大多数key），那么选择string类型作为存储方式会比较好。
* 如果你的业务类型中对于缓存的更新比较频繁（特别是每次只更新少数几个键）时， 或者我们每次只想取json数据中的少数几个键值时，我们选择hash类型作为我们的存储方式会比较好。#)

