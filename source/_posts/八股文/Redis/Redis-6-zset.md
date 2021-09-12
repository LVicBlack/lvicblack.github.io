---
title: Redis - 6.zset
date: 2020-08-16 17:56:03
categories: 
- 八股文
- Redis
tags:
- Redis
- 面试
---

### **zset 4**

#### **Q1：简单说一说 zset 类型**

有序集合保留了集合不能有重复成员的特性，不同的是可以排序。但是它和列表使用索引下标作为排序依据不同的是，他给每个元素设置一个分数（score）作为排序的依据。有序集合提供了获取指定分数和元素查询范围、计算成员排名等功能。

---

#### **Q2：你知道哪些 zset 的命令？**

**添加成员**

zadd key score member \[score member...\]，返回结果是成功添加成员的个数

Redis 3.2 为 zadd 命令添加了 nx、xx、ch、incr 四个选项：

* nx：member 必须不存在才可以设置成功，用于添加。
* xx：member 必须存在才能设置成功，用于更新。
* ch：返回此次操作后，有序集合元素和分数变化的个数。
* incr：对 score 做增加，相当于 zincrby。

zadd 的时间复杂度为 O\(logn\)，sadd 的时间复杂度为 O\(1\)。

**计算成员个数**

zcard key，时间复杂度为 O\(1\)。

**计算某个成员的分数**

zscore key member ，如果不存在则返回 nil。

**计算成员排名**

zrank key member，从低到高返回排名。

zrevrank key member，从高到低返回排名。

**删除成员**

zrem key member \[member...\]，返回结果是成功删除的个数。

**增加成员的分数**

zincrby key increment member

**返回指定排名范围的成员**

zrange key start end \[withscores\]，从低到高返回

zrevrange key start end \[withscores\]， 从高到底返回

**返回指定分数范围的成员**

zrangebyscore key min max \[withscores\] \[limit offset count\]，从低到高返回

zrevrangebyscore key min max \[withscores\] \[limit offset count\]， 从高到底返回

**返回指定分数范围成员个数**

zcount key min max

**删除指定分数范围内的成员**

zremrangebyscore key min max

**交集和并集**

zinterstore/zunionstore destination numkeys key \[key...\] \[weights weight \[weight...\]\] \[aggregate sum|min|max\]

* destination
* numkeys
* key
* weight
* aggregate sum|min|max

---

#### **Q3：zset 的内部编码是什么？**

ziplist 压缩列表：当有序集合元素个数和值小于配置值（默认128 个和 64 字节）时会使用 ziplist 作为内部实现。

skiplist 跳跃表：当 ziplist 不满足条件时使用，因为此时 ziplist 的读写效率会下降。

---

#### **Q4：zset 的应用场景有什么？**

有序集合的典型使用场景就是排行榜系统，例如用户上传了一个视频并获得了赞，可以使用 zadd 和 zincrby。如果需要将用户从榜单删除，可以使用 zrem。如果要展示获取赞数最多的十个用户，可以使用 zrange。
