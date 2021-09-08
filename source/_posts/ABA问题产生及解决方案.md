---
title: ABA问题产生及解决方案
date: 2019-05-20 06:17:03
categories: 
- Java
tags:
- Java
- 面试
---

## 1、基本的ABA问题

在CAS算法中，需要取出内存中某时刻的数据(由用户完成)，在下一时刻比较并交换(CPU保证原子操作)，这个时间差会导致数据的变化。 假设有以下顺序事件： 
1. 线程1从内存位置V中取出A 
2. 线程2从内存位置V中取出A 
3. 线程2进行了写操作，将B写入内存位置V 
4. 线程2将A再次写入内存位置V 
5. 线程1进行CAS操作，发现V中仍然是A，交换成功尽管线程1的CAS操作成功，但线程1并不知道内存位置V的数据发生过改变

## 2、ABA问题示例

```
public class ABADemo {
    private static AtomicReference<Integer> atomicReference = new AtomicReference<Integer>(100);

    public static void main(String[] args) {
        new Thread(() -> {
            atomicReference.compareAndSet(100, 101);
            atomicReference.compareAndSet(101, 100);
        },"t1").start();

        new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(atomicReference.compareAndSet(100, 2019) + "\t修改后的值:" + atomicReference.get());
        },"t2").start();
    }
}
```

- 初始值为100，线程t1将100改成101，然后又将101改回100
- 线程t2先睡眠1秒，等待t1操作完成，然后t2线程将值改成2019
- 可以看到，线程2修改成功

```
输出结果: 
true 
修改后的值:2019
```




<h1>3、ABA问题解决</h1>
要解决ABA问题，可以增加一个版本号，当内存位置V的值每次被修改后，版本号都加1

### AtomicStampedReference

> AtomicStampedReference内部维护了对象值和版本号，在创建AtomicStampedReference对象时，需要传入初始值和初始版本号，当AtomicStampedReference设置对象值时，对象值以及状态戳都必须满足期望值，写入才会成功


#### 示例
```
public class ABADemo {
    private static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<Integer>(100,1);

    public static void main(String[] args) {
        new Thread(() -> {
            System.out.println("t1拿到的初始版本号:" + atomicStampedReference.getStamp());

            //睡眠1秒，是为了让t2线程也拿到同样的初始版本号
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicStampedReference.compareAndSet(100, 101,atomicStampedReference.getStamp(),atomicStampedReference.getStamp()+1);
            atomicStampedReference.compareAndSet(101, 100,atomicStampedReference.getStamp(),atomicStampedReference.getStamp()+1);
        },"t1").start();

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println("t2拿到的初始版本号:" + stamp);

            //睡眠3秒，是为了让t1线程完成ABA操作
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("最新版本号:" + atomicStampedReference.getStamp());
            System.out.println(atomicStampedReference.compareAndSet(100, 2019,stamp,atomicStampedReference.getStamp() + 1) + "\t当前 值:" + atomicStampedReference.getReference());
        },"t2").start();
    }
}
```

- 初始值100，初始版本号1
- 线程t1和t2拿到一样的初始版本号
- 线程t1完成ABA操作，版本号递增到3
- 线程t2完成CAS操作，最新版本号已经变成3，跟线程t2之前拿到的版本号1不相等，操作失败

```
输出结果：
t1拿到的初始版本号:1 
t2拿到的初始版本号:1 
最新版本号:3 
false 
当前值:100
```

#### AtomicMarkableReference

>AtomicStampedReference可以给引用加上版本号，追踪引用的整个变化过程，如：A -> B -> C -> D - > A，通过AtomicStampedReference，我们可以知道，引用变量中途被更改了3次
但是，有时候，我们并不关心引用变量更改了几次，只是单纯的关心是否更改过，所以就有了AtomicMarkableReference
AtomicMarkableReference的唯一区别就是不再用int标识引用，而是使用boolean变量——表示引用变量是否被更改过


#### 示例

```
public class ABADemo {
    private static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<Integer>(100,1);

    public static void main(String[] args) {
        new Thread(() -> {
            System.out.println("t1拿到的初始版本号:" + atomicStampedReference.getStamp());

            //睡眠1秒，是为了让t2线程也拿到同样的初始版本号
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicStampedReference.compareAndSet(100, 101,atomicStampedReference.getStamp(),atomicStampedReference.getStamp()+1);
            atomicStampedReference.compareAndSet(101, 100,atomicStampedReference.getStamp(),atomicStampedReference.getStamp()+1);
        },"t1").start();

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println("t2拿到的初始版本号:" + stamp);

            //睡眠3秒，是为了让t1线程完成ABA操作
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("最新版本号:" + atomicStampedReference.getStamp());
            System.out.println(atomicStampedReference.compareAndSet(100, 2019,stamp,atomicStampedReference.getStamp() + 1) + "\t当前 值:" + atomicStampedReference.getReference());
        },"t2").start();
    }
}
```

- 初始值100，初始版本号未被修改 false
- 线程t1和t2拿到一样的初始版本号都未被修改 false
- 线程t1完成ABA操作，版本号被修改 true
- 线程t2完成CAS操作，版本号已经变成true，跟线程t2之前拿到的版本号false不相等，操作失败

```
输出结果：
t1版本号是否被更改:false 
t2版本号是否被更改:false 
是否更改过:true 
false 
当前值:100
```
