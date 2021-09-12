---
title: Redis - 5.set
date: 2020-08-16 17:55:03
categories: 
- 八股文
- Redis
tags:
- Redis
- 面试
---

### **set 4**

#### **Q1：简单说一说 set 类型**

集合类型也是用来保存多个字符串元素，和列表不同的是集合不允许有重复元素，并且集合中的元素是无序的，不能通过索引下标获取元素。一个集合最多可以存储 2^32^\-1 个元素。Redis 除了支持集合内的增删改查，还支持多个集合取交集、并集、差集。

---

#### **Q2：你知道哪些 set 的命令？**

**添加元素**

sadd key element \[element...\]，返回结果为添加成功的元素个数。

**删除元素**

srem key element \[element...\]，返回结果为成功删除的元素个数。

**计算元素个数**

scard key，时间复杂度为 O\(1\)，会直接使用 Redis 内部的遍历。

**判断元素是否在集合中**

sismember key element，如果存在返回 1，否则返回 0。

**随机从集合返回指定个数个元素**

srandmember key \[count\]，如果不指定 count 默认为 1。

**从集合随机弹出元素**

spop key，可以从集合中随机弹出一个元素。

**获取所有元素**

smembers key

**求多个集合的交集/并集/差集**

sinter key \[key...\]

sunion key \[key...\]

sdiff key \[key...\]

**保存交集、并集、差集的结果**

sinterstore/sunionstore/sdiffstore destination key \[key...\]

集合间运算在元素较多情况下比较耗时，Redis 提供这三个指令将集合间交集、并集、差集的结果保存在 destination key 中。

---

#### **Q3：set 的内部编码是什么？**

intset 整数集合：当集合中的元素个数小于配置值（默认 512 个时），使用 intset。

hashtable 哈希表：当集合类型无法满足 intset 条件时使用 hashtable。当某个元素不为整数时，也会使用 hashtable。

---

#### **Q4：set 的应用场景有什么？**

set 比较典型的使用场景是标签，例如一个用户可能与娱乐、体育比较感兴趣，另一个用户可能对例时、新闻比较感兴趣，这些兴趣点就是标签。这些数据对于用户体验以及增强用户黏度比较重要。

sadd = 标签、spop/srandmember = 生成随机数，比如抽奖、sadd \+ sinter = 社交需求。
