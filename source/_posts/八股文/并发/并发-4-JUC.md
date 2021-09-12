---
title: 并发 - 4.JUC
date: 2020-08-20 17:31:03
categories: 
- 八股文
- 并发
tags:
- 并发
- 面试
---

### **JUC 11**

#### **Q1：什么是 CAS？**

CAS 表示 Compare And Swap，比较并交换，CAS 需要三个操作数，分别是内存位置 V、旧的预期值 A 和准备设置的新值 B。CAS 指令执行时，当且仅当 V 符合 A 时，处理器才会用 B 更新 V 的值，否则它就不执行更新。但不管是否更新都会返回 V 的旧值，这些处理过程是原子操作，执行期间不会被其他线程打断。

在 JDK 5 后，Java 类库中才开始使用 CAS 操作，该操作由 Unsafe 类里的 compareAndSwapInt 等几个方法包装提供。HotSpot 在内部对这些方法做了特殊处理，即时编译的结果是一条平台相关的处理器 CAS 指令。Unsafe 类不是给用户程序调用的类，因此 JDK9 前只有 Java 类库可以使用 CAS，譬如 juc 包里的 AtomicInteger类中 compareAndSet 等方法都使用了Unsafe 类的 CAS 操作实现。

---

#### **Q2：CAS 有什么问题？**

CAS 从语义上来说存在一个逻辑漏洞：如果 V 初次读取时是 A，并且在准备赋值时仍为 A，这依旧不能说明它没有被其他线程更改过，因为这段时间内假设它的值先改为 B 又改回 A，那么 CAS 操作就会误认为它从来没有被改变过。

这个漏洞称为 ABA 问题，juc 包提供了一个 AtomicStampedReference，原子更新带有版本号的引用类型，通过控制变量值的版本来解决 ABA 问题。大部分情况下 ABA 不会影响程序并发的正确性，如果需要解决，传统的互斥同步可能会比原子类更高效。

---

#### **Q3：有哪些原子类？**

JDK 5 提供了 java.util.concurrent.atomic 包，这个包中的原子操作类提供了一种用法简单、性能高效、线程安全地更新一个变量的方式。到 JDK 8 该包共有17个类，依据作用分为四种：原子更新基本类型类、原子更新数组类、原子更新引用类以及原子更新字段类，atomic 包里的类基本都是使用 Unsafe 实现的包装类。

AtomicInteger 原子更新整形、 AtomicLong 原子更新长整型、AtomicBoolean 原子更新布尔类型。

AtomicIntegerArray，原子更新整形数组里的元素、 AtomicLongArray 原子更新长整型数组里的元素、 AtomicReferenceArray 原子更新引用类型数组里的元素。

AtomicReference 原子更新引用类型、AtomicMarkableReference 原子更新带有标记位的引用类型，可以绑定一个 boolean 标记、 AtomicStampedReference 原子更新带有版本号的引用类型，关联一个整数值作为版本号，解决 ABA 问题。

AtomicIntegerFieldUpdater 原子更新整形字段的更新器、 AtomicLongFieldUpdater 原子更新长整形字段的更新器AtomicReferenceFieldUpdater 原子更新引用类型字段的更新器。

---

#### **Q4：AtomicIntger 实现原子更新的原理是什么？**

AtomicInteger 原子更新整形、 AtomicLong 原子更新长整型、AtomicBoolean 原子更新布尔类型。

getAndIncrement 以原子方式将当前的值加 1，首先在 for 死循环中取得 AtomicInteger 里存储的数值，第二步对 AtomicInteger 当前的值加 1 ，第三步调用 compareAndSet 方法进行原子更新，先检查当前数值是否等于 expect，如果等于则说明当前值没有被其他线程修改，则将值更新为 next，否则会更新失败返回 false，程序会进入 for 循环重新进行 compareAndSet 操作。

atomic 包中只提供了三种基本类型的原子更新，atomic 包里的类基本都是使用 Unsafe 实现的，Unsafe 只提供三种 CAS 方法：compareAndSwapInt、compareAndSwapLong 和 compareAndSwapObject，例如原子更新 Boolean 是先转成整形再使用 compareAndSwapInt 。

---

#### **Q5：CountDownLatch 是什么？**

CountDownLatch 是基于执行时间的同步类，允许一个或多个线程等待其他线程完成操作，构造方法接收一个 int 参数作为计数器，如果要等待 n 个点就传入 n。每次调用 countDown 方法时计数器减 1，await 方\*\*\*阻塞当前线程直到计数器变为0，由于 countDown 方法可用在任何地方，所以 n 个点既可以是 n 个线程也可以是一个线程里的 n 个执行步骤。

---

#### **Q6： CyclicBarrier 是什么？**

循环屏障是基于同步到达某个点的信号量触发机制，作用是让一组线程到达一个屏障时被阻塞，直到最后一个线程到达屏障才会解除。构造方法中的参数表示拦截线程数量，每个线程调用 await 方法告诉 CyclicBarrier 自己已到达屏障，然后被阻塞。还支持在构造方法中传入一个 Runnable 任务，当线程到达屏障时会优先执行该任务。适用于多线程计算数据，最后合并计算结果的应用场景。

CountDownLacth 的计数器只能用一次，而 CyclicBarrier 的计数器可使用 reset 方法重置，所以 CyclicBarrier 能处理更为复杂的业务场景，例如计算错误时可用重置计数器重新计算。

---

#### **Q7：Semaphore 是什么？**

信号量用来控制同时访问特定资源的线程数量，通过协调各个线程以保证合理使用公共资源。信号量可以用于流量控制，特别是公共资源有限的应用场景，比如数据库连接。

Semaphore 的构造方法参数接收一个 int 值，表示可用的许可数量即最大并发数。使用 acquire 方法获得一个许可证，使用 release 方法归还许可，还可以用 tryAcquire 尝试获得许可。

---

#### **Q8： Exchanger 是什么？**

交换者是用于线程间协作的工具类，用于进行线程间的数据交换。它提供一个同步点，在这个同步点两个线程可以交换彼此的数据。

两个线程通过 exchange 方法交换数据，第一个线程执行 exchange 方法后会阻塞等待第二个线程执行该方法，当两个线程都到达同步点时这两个线程就可以交换数据，将本线程生产出的数据传递给对方。应用场景包括遗传算法、校对工作等。

---

#### **P9：JDK7 的 ConcurrentHashMap 原理？**

ConcurrentHashMap 用于解决 HashMap 的线程不安全和 HashTable 的并发效率低，HashTable 之所以效率低是因为所有线程都必须竞争同一把锁，假如容器里有多把锁，每一把锁用于锁容器的部分数据，那么多线程访问容器不同数据段的数据时，线程间就不会存在锁竞争，从而有效提高并发效率，这就是 ConcurrentHashMap 的锁分段技术。首先将数据分成 Segment 数据段，然后给每一个数据段配一把锁，当一个线程占用锁访问其中一个段的数据时，其他段的数据也能被其他线程访问。

get 实现简单高效，先经过一次再散列，再用这个散列值通过散列运算定位到 Segment，最后通过散列算法定位到元素。get 的高效在于不需要加锁，除非读到空值才会加锁重读。get 方法中将共享变量定义为 volatile，在 get 操作里只需要读所以不用加锁。

put 必须加锁，首先定位到 Segment，然后进行插入操作，第一步判断是否需要对 Segment 里的 HashEntry 数组进行扩容，第二步定位添加元素的位置，然后将其放入数组。

size 操作用于统计元素的数量，必须统计每个 Segment 的大小然后求和，在统计结果累加的过程中，之前累加过的 count 变化几率很小，因此先尝试两次通过不加锁的方式统计结果，如果统计过程中容器大小发生了变化，再加锁统计所有 Segment 大小。判断容器是否发生变化根据 modCount 确定。

---

#### **P10：JDK8 的 ConcurrentHashMap 原理？**

主要对 JDK7 做了三点改造：① 取消分段锁机制，进一步降低冲突概率。② 引入红黑树结构，同一个哈希槽上的元素个数超过一定阈值后，单向链表改为红黑树结构。③ 使用了更加优化的方式统计集合内的元素数量。具体优化表现在：在 put、resize 和 size 方法中设计元素总数的更新和计算都避免了锁，使用 CAS 代替。

get 同样不需要同步，put 操作时如果没有出现哈希冲突，就使用 CAS 添加元素，否则使用 synchronized 加锁添加元素。

当某个槽内的元素个数达到 7 且 table 容量不小于 64 时，链表转为红黑树。当某个槽内的元素减少到 6 时，由红黑树重新转为链表。在转化过程中，使用同步块锁住当前槽的首元素，防止其他线程对当前槽进行增删改操作，转化完成后利用 CAS 替换原有链表。由于 TreeNode 节点也存储了 next 引用，因此红黑树转为链表很简单，只需从 first 元素开始遍历所有节点，并把节点从 TreeNode 转为 Node 类型即可，当构造好新链表后同样用 CAS 替换红黑树。

---

#### **P11：ArrayList 的线程安全集合是什么？**

可以使用 CopyOnWriteArrayList 代替 ArrayList，它实现了读写分离。写操作复制一个新的集合，在新集合内添加或删除元素，修改完成后再将原集合的引用指向新集合。这样做的好处是可以高并发地进行读写操作而不需要加锁，因为当前集合不会添加任何元素。使用时注意尽量设置容量初始值，并且可以使用批量添加或删除，避免多次扩容，比如只增加一个元素却复制整个集合。

适合读多写少，单个添加时效率极低。CopyOnWriteArrayList 是 fail\-safe 的，并发包的集合都是这种机制，fail\-safe 在安全的副本上遍历，集合修改与副本遍历没有任何关系，缺点是无法读取最新数据。这也是 CAP 理论中 C 和 A 的矛盾，即一致性与可用性的矛盾。
