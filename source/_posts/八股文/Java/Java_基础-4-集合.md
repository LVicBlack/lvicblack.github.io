---
title: Java 基础 - 4.集合
date: 2020-08-20 15:24:03
categories: 
- 八股文
- Java
tags:
- Java
- 面试
---


### 集合 7

#### **Q1：说一说 ArrayList**

**ArrayList** 是容量可变的非线程安全列表，使用数组实现，集合扩容时会创建更大的数组，把原有数组复制到新数组。支持对元素的快速随机访问，但插入与删除速度很慢。ArrayList 实现了 RandomAcess 标记接口，如果一个类实现了该接口，那么表示使用索引遍历比迭代器更快。

**elementData**是 ArrayList 的数据域，被 transient 修饰，序列化时会调用 writeObject 写入流，反序列化时调用 readObject 重新赋值到新对象的 elementData。原因是 elementData 容量通常大于实际存储元素的数量，所以只需发送真正有实际值的数组元素。

**size** 是当前实际大小，elementData 大小大于等于 size。

\*_modCount \*_记录了 ArrayList 结构性变化的次数，继承自 AbstractList。所有涉及结构变化的方法都会增加该值。expectedModCount 是迭代器初始化时记录的 modCount 值，每次访问新元素时都会检查 modCount 和 expectedModCount 是否相等，不相等就会抛出异常。这种机制叫做 fail\-fast，所有集合类都有这种机制。

---

#### **Q2：说一说 LinkedList**

**LinkedList** 本质是双向链表，与 ArrayList 相比插入和删除速度更快，但随机访问元素很慢。除继承 AbstractList 外还实现了 Deque 接口，这个接口具有队列和栈的性质。成员变量被 transient 修饰，原理和 ArrayList 类似。

LinkedList 包含三个重要的成员：size、first 和 last。size 是双向链表中节点的个数，first 和 last 分别指向首尾节点的引用。

LinkedList 的优点在于可以将零散的内存单元通过附加引用的方式关联起来，形成按链路顺序查找的线性结构，内存利用率较高。

---

#### **Q3：Set 有什么特点，有哪些实现？**

**Set** 不允许元素重复且无序，常用实现有 HashSet、LinkedHashSet 和 TreeSet。

**HashSet** 通过 HashMap 实现，HashMap 的 Key 即 HashSet 存储的元素，所有 Key 都使用相同的 Value ，一个名为 PRESENT 的 Object 类型常量。使用 Key 保证元素唯一性，但不保证有序性。由于 HashSet 是 HashMap 实现的，因此线程不安全。

HashSet 判断元素是否相同时，对于包装类型直接按值比较。对于引用类型先比较 hashCode 是否相同，不同则代表不是同一个对象，相同则继续比较 equals，都相同才是同一个对象。

**LinkedHashSet** 继承自 HashSet，通过 LinkedHashMap 实现，使用双向链表维护元素插入顺序。

**TreeSet** 通过 TreeMap 实现的，添加元素到集合时按照比较规则将其插入合适的位置，保证插入后的集合仍然有序。

---

#### **Q4：TreeMap 有什么特点？**

- TreeMap 基于红黑树实现，增删改查的平均和最差时间复杂度均为 O\(logn\) ，最大特点是 Key 有序。Key 必须实现 Comparable 接口或提供的 Comparator 比较器，所以 Key 不允许为 null。

- HashMap 依靠 hashCode 和 equals 去重，而 TreeMap 依靠 Comparable 或 Comparator。 TreeMap 排序时，如果比较器不为空就会优先使用比较器的 compare 方法，否则使用 Key 实现的 Comparable 的 compareTo 方法，两者都不满足会抛出异常。

- TreeMap 通过 put 和 deleteEntry 实现增加和删除树节点。插入新节点的规则有三个：① 需要调整的新节点总是红色的。② 如果插入新节点的父节点是黑色的，不需要调整。③ 如果插入新节点的父节点是红色的，由于红黑树不能出现相邻红色，进入循环判断，通过重新着色或左右旋转来调整。TreeMap 的插入操作就是按照 Key 的对比往下遍历，大于节点值向右查找，小于向左查找，先按照二叉查找树的特性操作，后续会重新着色和旋转，保持红黑树的特性。

---

#### **Q5：HashMap 有什么特点？**

- JDK8 之前底层实现是数组 \+ 链表，JDK8 改为数组 \+ 链表/红黑树，节点类型从Entry 变更为 Node。主要成员变量包括存储数据的 table 数组、元素数量 size、加载因子 loadFactor。

- table 数组记录 HashMap 的数据，每个下标对应一条链表，所有哈希冲突的数据都会被存放到同一条链表，Node/Entry 节点包含四个成员变量：key、value、next 指针和 hash 值。

- HashMap 中数据以键值对的形式存在，键对应的 hash 值用来计算数组下标，如果两个元素 key 的 hash 值一样，就会发生哈希冲突，被放到同一个链表上，为使查询效率尽可能高，键的 hash 值要尽可能分散。

- HashMap 默认初始化容量为 16，扩容容量必须是 2 的幂次方、最大容量为 1\<\< 30 、默认加载因子为 0.75。

---

#### **Q6：HashMap 相关方法的源码？**

**JDK8 之前**

**hash：计算元素 key 的散列值**

1. 处理 String 类型时，调用 stringHash32 方法获取 hash 值。

2. 处理其他类型数据时，提供一个相对于 HashMap 实例唯一不变的随机值 hashSeed 作为计算初始量。

3. 执行异或和无符号右移使 hash 值更加离散，减小哈希冲突概率。

**indexFor：计算元素下标**

将 hash 值和数组长度\-1 进行与操作，保证结果不会超过 table 数组范围。

**get：获取元素的 value 值**

1. 如果 key 为 null，调用 getForNullKey 方法，如果 size 为 0 表示链表为空，返回 null。如果 size 不为 0 说明存在链表，遍历 table\[0\] 链表，如果找到了 key 为 null 的节点则返回其 value，否则返回 null。

2. 如果 key 为 不为 null，调用 getEntry 方法，如果 size 为 0 表示链表为空，返回 null 值。如果 size 不为 0，首先计算 key 的 hash 值，然后遍历该链表的所有节点，如果节点的 key 和 hash 值都和要查找的元素相同则返回其 Entry 节点。

3. 如果找到了对应的 Entry 节点，调用 getValue 方法获取其 value 并返回，否则返回 null。

**put：添加元素**

1. 如果 key 为 null，直接存入 table\[0\]。

2. 如果 key 不为 null，计算 key 的 hash 值。

3. 调用 indexFor 计算元素存放的下标 i。

4. 遍历 table\[i\] 对应的链表，如果 key 已存在，就更新 value 然后返回旧 value。

5. 如果 key 不存在，将 modCount 值加 1，使用 addEntry 方法增加一个节点并返回 null。

**resize：扩容数组**

1. 如果当前容量达到了最大容量，将阈值设置为 Integer 最大值，之后扩容不再触发。

2. 否则计算新的容量，将阈值设为 newCapacity x loadFactor 和 最大容量 \+ 1 的较小值。

3. 创建一个容量为 newCapacity 的 Entry 数组，调用 transfer 方法将旧数组的元素转移到新数组。

**transfer：转移元素**

1. 遍历旧数组的所有元素，调用 rehash 方法判断是否需要哈希重构，如果需要就重新计算元素 key 的 hash 值。

2. 调用 indexFor 方法计算元素存放的下标 i，利用头插法将旧数组的元素转移到新数组。

**JDK8**

**hash：计算元素 key 的散列值**

如果 key 为 null 返回 0，否则就将 key 的 hashCode 方法返回值高低16位异或，让尽可能多的位参与运算，让结果的 0 和 1 分布更加均匀，降低哈希冲突概率。

**put：添加元素**

1. 调用 putVal 方法添加元素。

2. 如果 table 为空或长度为 0 就进行扩容，否则计算元素下标位置，不存在就调用 newNode 创建一个节点。

3. 如果存在且是链表，如果首节点和待插入元素的 hash 和 key 都一样，更新节点的 value。

4. 如果首节点是 TreeNode 类型，调用 putTreeVal 方法增加一个树节点，每一次都比较插入节点和当前节点的大小，待插入节点小就往左子树查找，否则往右子树查找，找到空位后执行两个方法：balanceInsert 方法，插入节点并调整平衡、moveRootToFront 方法，由于调整平衡后根节点可能变化，需要重置根节点。

5. 如果都不满足，遍历链表，根据 hash 和 key 判断是否重复，决定更新 value 还是新增节点。如果遍历到了链表末尾则添加节点，如果达到建树阈值 7，还需要调用 treeifyBin 把链表重构为红黑树。

6. 存放元素后将 modCount 加 1，如果 \+\+size \> threshold ，调用 resize 扩容。

**get ：获取元素的 value 值**

1. 调用 getNode 方法获取 Node 节点，如果不是 null 就返回其 value 值，否则返回 null。

2. getNode 方法中如果数组不为空且存在元素，先比较第一个节点和要查找元素的 hash 和 key ，如果都相同则直接返回。

3. 如果第二个节点是 TreeNode 类型则调用 getTreeNode 方法进行查找，否则遍历链表根据 hash 和 key 查找，如果没有找到就返回 null。

**resize：扩容数组**

重新规划长度和阈值，如果长度发生了变化，部分数据节点也要重新排列。

**重新规划长度**

1. 如果当前容量 oldCap \> 0 且达到最大容量，将阈值设为 Integer 最大值，return 终止扩容。

2. 如果未达到最大容量，当 oldCap \<\< 1 不超过最大容量就扩大为 2 倍。

3. 如果都不满足且当前扩容阈值 oldThr \> 0，使用当前扩容阈值作为新容量。

4. 否则将新容量置为默认初始容量 16，新扩容阈值置为 12。

**重新排列数据节点**

1. 如果节点为 null 不进行处理。

2. 如果节点不为 null 且没有next节点，那么通过节点的 hash 值和 新容量\-1 进行与运算计算下标存入新的 table 数组。

3. 如果节点为 TreeNode 类型，调用 split 方法处理，如果节点数 hc 达到6 会调用 untreeify 方法转回链表。

4. 如果是链表节点，需要将链表拆分为 hash 值超出旧容量的链表和未超出容量的链表。对于hash & oldCap == 0 的部分不需要做处理，否则需要放到新的下标位置上，新下标 = 旧下标 \+ 旧容量。

---

#### **Q7：HashMap 为什么线程不安全？**

JDK7 存在死循环和数据丢失问题。

**数据丢失：**

* **并发赋值被覆盖：**
* **已遍历区间新增元素丢失：**
* **新表被覆盖：**

**死循环：** 扩容时 resize 调用 transfer 使用头插法迁移元素，虽然 newTable 是局部变量，但原先 table 中的 Entry 链表是共享的，问题根源是 Entry 的 next 指针并发修改，某线程还没有将 table 设为 newTable 时用完了 CPU 时间片，导致数据丢失或死循环。

JDK8 在 resize 方法中完成扩容，并改用尾插法，不会产生死循环，但并发下仍可能丢失数据。可用 ConcurrentHashMap 或 Collections.synchronizedMap 包装成同步集合。
