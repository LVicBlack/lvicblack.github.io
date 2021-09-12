---
title: GC算法之 标记-压缩算法
date: 2020-03-17 08:17:03
categories: 
- Java
tags:
- Java
- 面试
---


之前介绍了[标记\-清除](https://zhuanlan.zhihu.com/p/51095294)和[复制\-清除](https://zhuanlan.zhihu.com/p/51133338)两种最基本的GC算法，现在就介绍集这两种算法于一身的标记\-压缩算法。

## 什么是GC标记\-压缩算法？

GC标记\-压缩算法由标记阶段和压缩阶段构成。

标记阶段和之前的[标记\-清除](https://zhuanlan.zhihu.com/p/51095294)中提到的标记阶段完全一样，然后我们就通过遍历数次堆来进行压缩。这里的压缩指的就是[复制\-清除](https://zhuanlan.zhihu.com/p/51133338)里面的把存活对象重新装填，使对象都紧挨在一起，从而避免内存碎片的产生，同时保证内存的高速分配。不过与GC复制算法不同的是，标记\-压缩不需要牺牲额外的空间。

## Lisp2算法

提到标记\-压缩就不得不提著名的计算机学家[Donald E. Knuth](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/%25E5%2594%2590%25E7%25BA%25B3%25E5%25BE%25B7%25C2%25B7%25E5%2585%258B%25E5%258A%25AA%25E7%2589%25B9/1436781%3Ffr%3Daladdin)发明的Lisp2算法。

在详细介绍算法之前我们先了解一下这个算法结构中的对象结构。

![v2-5d2b2fb297c6419a2f5430e06af1407f_r.jpg](image/v2-5d2b2fb297c6419a2f5430e06af1407f_r.jpg)

Lisp2 算法中的对象

Lisp2在对象头里面会预留一个叫forwarding的指针，这个指针和[复制\-清除](https://zhuanlan.zhihu.com/p/51133338)里面forwarding的用法一样指向GC后对象新的内存空间。这里提前说明一点就是forwarding不为空就表示该对象并没有移动完毕。

## Lisp2算法的GC过程

初始状态

![v2-61d79bc9bd535686c462538de12490a5_r.jpg](image/v2-61d79bc9bd535686c462538de12490a5_r.jpg)

初始状态

从GC Roots出发，标记活动对象

![v2-934136557ed9871a57716d5bf1e36ddb_r.jpg](image/v2-934136557ed9871a57716d5bf1e36ddb_r.jpg)

标记阶段结束后

进行压缩

![v2-45219f19e96403388655d2bab1d8c6bb_r.jpg](image/v2-45219f19e96403388655d2bab1d8c6bb_r.jpg)

压缩阶段结束后

通过图我们知道活动对象B、C、D、F分别对应B'、C'、D'、F'。所以Lisp2算法并不会改变对象原本的排序顺序，只是把缩小对象间的间隙，然后聚集到一端。

整个压缩阶段用伪代码的话，如下表示

```
compaction_phase(){
         set_forwarding_ptr(); //设定forwarding 指针
         adjust_ptr(); //更新指针
         move_obj(); //移动对象
    }
```

如代码所示整个阶段分为三个步骤

1、设定forwarding指针

2、更新指针

3、移动对象

对于步骤1，程序会搜索整个堆，给活动对象设定forwarding指针

```
   void set_forwarding_ptr() {
        scan = new_address = $heap_start;
        while (scan < $heap_end)
            if (scan.mark == TRUE)
                scan.forwarding = new_address;
                new_address += scan.size;
        scan += scan.size;
    }
```

scan是用来遍历堆中对象的指针，new\_address是指向目标对象的指针，这两个指针在后续的操作中是很重要的。当scan指针找到活动对象时，就会令对象的forwarding指针指向newaddress,然后将new\_address按对象长度移动。遍历完整个堆后堆的状态就会变成

![v2-9f9e90daa1553ad0dcfaaa253a06f5d8_r.jpg](image/v2-9f9e90daa1553ad0dcfaaa253a06f5d8_r.jpg)

set\_forwarding\_ptr函数执行完毕之后

设置好forwarding之后，接下来我们做的是并不是马上移动对象而是先更新对象的指针adjust\_ptr\(\)。这是因为GC标记\-压缩算法中新空间和原空间是同一个空间，所以有可能出现先移动的对象把还没有来得及移动的对象覆盖掉的情况。另外我们还需要记录各对象之间引用关系，找到对象GC后的目标地址。所以在移动对象前，我们先把活动对象的指针全部更新到预计的位置，这样一来，之后只要移动对象，GC就结束了。

```
adjust_ptr(){
        for(r : $roots)
            *r = (*r).forwarding;

        scan = $heap_start;
        while(scan < $heap_end)
            if(scan.mark == TRUE)
                for(child : children(scan))
                    *child = (*child).forwarding;
        scan += scan.size
    }
```

adjust\_ptr\(\)第一步是重写roots结点的指针，后面才重写所有活动对象的指针。注意这已经第二次对整个堆执行搜索了。函数执行完毕后，堆的状态如下

![v2-0c8708eb9e46957728e644067dbc5268_r.jpg](image/v2-0c8708eb9e46957728e644067dbc5268_r.jpg)

adjust\_ptr\(\)执行后

最后一步搜索整个堆，将活动对象移动到forwarding指针的引用目标处。需要注意的是这已经是第三次搜索整个堆了。

```
 move_obj(){
        scan = $free = $heap_start;
        while(scan < $heap_end)
            if(scan.mark == TRUE)
                new_address = scan.forwarding;
                copy_data(new_address, scan, scan.size);
                new_address.forwarding = NULL;
                new_address.mark = FALSE;
                $free += new_address.size;
                scan += scan.size;
    }
```

这里的逻辑就比较简单，搜索堆找到活动对象时，就把对象复制到forwarding指针指向的地址，然后把forwarding置NULL,后清除mark标记。

![v2-c65d151986ca84d87c037cc81dd33b79_r.jpg](image/v2-c65d151986ca84d87c037cc81dd33b79_r.jpg)

move\_obj\(\)函数执行完毕后

介绍完Lisp2算法整个流程之后，我们再回来分析一下算法的优缺点。

优点： 可有效利用堆。相对于GC[复制\-清除](https://zhuanlan.zhihu.com/p/51133338)，GC标记\-压缩不会空出一个To空间，是利用了整个堆。相对于GC[标记\-清除](https://zhuanlan.zhihu.com/p/51095294)，GC标记\-压缩对活动对象进行了压缩，不存在碎片化的问题，所以有效率利用率高。

缺点：压缩花费计算成本大。为了对活动对象进行压缩，我们看到Lisp2的压缩过程必须进行三次堆搜索。堆搜索的花费时间是和堆的大小成正比的，所以GC标记\-压缩算法的吞吐量要劣于其它算法。在GC[标记\-清除](https://zhuanlan.zhihu.com/p/51095294)算法中，清除阶段也要搜索整个堆，不过搜索一次就够了。但GC标记\-压缩要搜索三次，这样就要花费大约三倍的时间，这是一个相当巨大的缺陷，特别是堆越大，所消耗的成本也越大。

所以后面有人就提出了另外一种算法Two\-Finger算法

## Two\-Finger算法

Two\-Finger 是由Robert A. Saunders 研究出来的一种高效算法，具体来说就只需要搜索二次堆就可以了。

## 算法前提

Two\-Finger算法有一个很大的限制条件就是，所有的对象大小必须整理成一致。之前介绍的所有算法都没有这种限制。另外与Lisp2不同的是Two\-Finger中的对象头不需要设置forwarding指针。

## 概要

Two\-Finger算法首先也要标记活动对象，标记过程与其它算法一样。不同之处在于Two\-Finger的压缩阶段只有以下两个步骤。

1、移动对象

2、更新指针

算法的一大特征移动对象

![v2-420889b15d6324cc9444c1f0b2626b00_r.jpg](image/v2-420889b15d6324cc9444c1f0b2626b00_r.jpg)

Two\-Finger算法中的对象移动

从图中我们看到与Lisp2算法不同的是Two\-Finger是通过移动对象来填补空白处来去进行压缩，而Lisp2是整体向左移动。也正是为了让对象能够恰好填补空间，所以必须让所有对象大小一致！

下面来看具体每个步骤的细节

步骤一 移动对象

首先定义两个指针$free、live，从两边向中间搜索。我们可以把这两个指针当作手指，所以我们把它叫做Two\-Finger。

$free是用于寻找非活动对象的指针，live指针是用于活动对象的指针。堆以及$free和live的初始化状态如图：

![v2-01f7c8a397180d93894845f356be6c57_r.jpg](image/v2-01f7c8a397180d93894845f356be6c57_r.jpg)

堆和两个指针的初始状态

当两个指针发现目标空间和原空间对象时会移动对象，移动过程如下

![v2-87fc45e32a06cff6ce9952ee8b2179b0_r.jpg](image/v2-87fc45e32a06cff6ce9952ee8b2179b0_r.jpg)

移动对象

 定义move\_obj\(\)函数，描述整个移动过程。

```
move_obj(){
        $free = $heap_start;
        live = $heap_end - OBJ_SIZE;
        while(TRUE)
            while($free.mark == TRUE)
                $free += OBJ_SIZE;
            while(live.mark == FALSE)  
                live -= OBJ_SIZE;
            if($free < live)
                copy_data($free, live, OBJ_SIZE)
                live.forwarding = $free; //记录新目标地址，更新时用       
                live.mark = FALSE;
            else 
                break;
    }
```

因为对象大小一致，所以设其大小为OBJSIZE，$free指针从左\($heapstart\)开始向右搜索堆，另一方面live指针从右\($heap\_end\-OBJ\_SIZE\)开始向左搜索。 

对于活动对象$free是直接跳过，对于非活动对象live指针直接跳过。当$free和live碰撞时就表示整个堆搜索完毕了，对象也就移动完成了。

步骤二 更新指针

步骤一执行后，活动对象都压缩在一起了。但对象指针还没有更新，所以下面我们要更新对象的指针。

![v2-31fcae770316c38e400fd90d813dcb84_r.jpg](image/v2-31fcae770316c38e400fd90d813dcb84_r.jpg)

对象移动结束后

当对象移动结束时，$free指针指向分块的开头，这时位于$free指针右边的是以下两者对象

* • 非活动对象
* • 移动前对象

因此，指向$free指针右边地址的指针引用的是移动前的对象，基于这点我们来看看adust\_ptr\(\)函数

```
adjust_ptr(){
         for(r : $roots)
             if(*r >= $free)
                 *r = (*r).forwarding

         scan = $heap_start;
         while(scan < $free)
             scan.mark = FALSE;
             for(child : children(scan))
                 if(*child >= $free)
                     *child = (*child).forwarding;
             scan += OBJ_SIZE;

    }
```

从代码中我们看出第一步更新GC roots指针

又因为$free的左边都是活动对象，所以这时只要遍历$free左边所有的对象并依次更新对象即可。

优点：

Lisp2 算法要事先确保每个对象都留有 1 个字用于 forwarding 指针，这就压迫了堆。然而 因为 Two\-Finger 算法能把 forwarding 指针设置在移动前的对象的域里，所以不需要额外的内存 空间以用于 forwarding 指针，因此在内存的使用效率上，该算法要比 Lisp2 算法的使用效率高。 

 此外，在 Two\-Finger 算法中，压缩所带来的搜索次数只有 2 次，比 Lisp2 算法少 1 次， 在吞吐量方面占优势 

缺点：

Two\-Finger不像GC[复制\-清除](https://zhuanlan.zhihu.com/p/51133338)算法，将具有引用关系的对象安排在堆中较近的 位置能够通过缓存来提高访问速度。Two\-Finger 算法则不考虑对象间的引用关系， 一律对其进行压缩，结果就导致对象的顺序在压缩前后产生了巨大的变化。因此，基本 上也无法期待这个算法能沾缓存的光。 

此外该算法还有一个限制条件，那就是所有对象的大小必须一致。因为能消除这个限制 的处理系统不太多，所以这点制约了 Two\-Finger 算法的应用范围 。

关于GC标记\-压缩算法的概念和实现就暂介绍这了，希望对大家GC这块有所帮助和收获。
