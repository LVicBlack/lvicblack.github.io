---
title: Redis - 4.list
date: 2020-08-16 17:54:03
categories: 
- 八股文
- Redis
tags:
- Redis
- 面试
---

### **list 4**

#### **Q1：简单说一说 list 类型**

list 是用来存储多个有序的字符串，列表中的每个字符串称为元素，一个列表最多可以存储 2^32^\-1 个元素。可以对列表两端插入（push）和弹出（pop），还可以获取指定范围的元素列表、获取指定索引下标的元素等。列表是一种比较灵活的数据结构，它可以充当栈和队列的角色，在实际开发中有很多应用场景。

list 有两个特点：① 列表中的元素是有序的，可以通过索引下标获取某个元素或者某个范围内的元素列表。② 列表中的元素可以重复。

---

#### **Q2：你知道哪些 list 的命令？**

**添加**

从右边插入元素：rpush key value \[value...\]

从左到右获取列表的所有元素：lrange 0 \-1

从左边插入元素：lpush key value \[value...\]

向某个元素前或者后插入元素：linsert key before|after pivot value，会在列表中找到等于 pivot 的元素，在其前或后插入一个新的元素 value。

**查找**

获取指定范围内的元素列表：lrange key start end，索引从左到右的范围是 0N\-1，从右到左是 \-1\-N，lrange 中的 end 包含了自身。

获取列表指定索引下标的元素：lindex key index，获取最后一个元素可以使用 lindex key \-1。

获取列表长度：llen key

**删除**

从列表左侧弹出元素：lpop key

从列表右侧弹出元素：rpop key

删除指定元素：lrem key count value，如果 count 大于 0，从左到右删除最多 count 个元素，如果 count 小于 0，从右到左删除最多个 count 绝对值个元素，如果 count 等于 0，删除所有。

按照索引范围修剪列表：ltrim key start end，只会保留 start ~ end 范围的元素。

**修改**

修改指定索引下标的元素：lset key index newValue。

**阻塞操作**

阻塞式弹出：blpop/brpop key \[key...\] timeout，timeout 表示阻塞时间。

当列表为空时，如果 timeout = 0，客户端会一直阻塞，如果在此期间添加了元素，客户端会立即返回。

如果是多个键，那么brpop会从左至右遍历键，一旦有一个键能弹出元素，客户端立即返回。

如果多个客户端对同一个键执行 brpop，那么最先执行该命令的客户端可以获取弹出的值。

---

#### **Q3：list 的内部编码是什么？**

ziplist 压缩列表：跟哈希的 zipilist 相同，元素个数和大小小于配置值（默认 512 个和 64 字节）时使用。

linkedlist 链表：当列表类型无法满足 ziplist 的条件时会使用linkedlist。

Redis 3.2 提供了 quicklist 内部编码，它是以一个 ziplist 为节点的 linkedlist，它结合了两者的优势，为列表类提供了一种更为优秀的内部编码实现。

---

#### **Q4：list 的应用场景有什么？**

**消息队列**

Redis 的 lpush \+ brpop 即可实现阻塞队列，生产者客户端使用 lpush 从列表左侧插入元素，多个消费者客户端使用 brpop 命令阻塞式地抢列表尾部的元素，多个客户端保证了消费的负载均衡和高可用性。

**文章列表**

每个用户有属于自己的文章列表，现在需要分页展示文章列表，就可以考虑使用列表。因为列表不但有序，同时支持按照索引范围获取元素。每篇文章使用哈希结构存储。

lpush \+ lpop = 栈、lpush \+ rpop = 队列、lpush \+ ltrim = 优先集合、lpush \+ brpop = 消息队列。
