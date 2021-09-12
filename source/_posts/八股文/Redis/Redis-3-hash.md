---
title: Redis - 3.hash
date: 2020-08-16 17:53:03
categories: 
- 八股文
- Redis
tags:
- Redis
- 面试
---

### **hash 4**

#### **Q1：简单说一说 hash 类型**

哈希类型指键值本身又是一个键值对结构，哈希类型中的映射关系叫 field\-value，这里的 value 是指 field 对于的值而不是键对于的值。

---

#### **Q2：你知道哪些 hash 的命令？**

**设置值**

hset key field value，如果设置成功会返回 1，反之会返回 0，此外还提供了 hsetnx 命令，作用和 setnx 类似，只是作用于由键变为 field。

**获取值**

hget key field，如果不存在会返回 nil。

**删除 field**

hdel key field \[field...\]，会删除一个或多个 field，返回结果为删除成功 field 的个数。

**计算 field 个数**

hlen key

**批量设置或获取 field\-value**

```
hmget key field [field...]
hmset key field value [field value...]
```

**判断 field 是否存在**

hexists key field，存在返回 1，否则返回 0。

**获取所有的 field**

hkeys key，返回指定哈希键的所有 field。

**获取所有 value**

hvals key，获取指定键的所有 value。

**获取所有的 field\-value**

hgetall key，获取指定键的所有 field\-value。

---

#### **Q3：hash 的内部编码是什么？**

ziplist 压缩列表：当哈希类型元素个数和值小于配置值（默认 512 个和 64 字节）时会使用 ziplist 作为内部实现，使用更紧凑的结构实现多个元素的连续存储，在节省内存方面比 hashtable 更优秀。

hashtable 哈希表：当哈希类型无法满足 ziplist 的条件时会使用 hashtable 作为哈希的内部实现，因为此时 ziplist 的读写效率会下降，而 hashtable 的读写时间复杂度都为 O\(1\)。

---

#### **Q4：hash 的应用场景有什么？**

缓存用户信息，每个用户属性使用一对 field\-value，但只用一个键保存。

优点：简单直观，如果合理使用可以减少内存空间使用。

缺点：要控制哈希在 ziplist 和 hashtable 两种内部编码的转换，hashtable 会消耗更多内存。
