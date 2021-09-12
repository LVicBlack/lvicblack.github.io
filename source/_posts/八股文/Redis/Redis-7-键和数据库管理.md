---
title: Redis - 7.键和数据库管理
date: 2020-08-16 17:57:03
categories: 
- 八股文
- Redis
tags:
- Redis
- 面试
---

### **键和数据库管理 5**

#### **Q1：如何对键重命名？**

rename key newkey

如果 rename 前键已经存在，那么它的值也会被覆盖。为了防止强行覆盖，Redis 提供了 renamenx 命令，确保只有 newkey 不存在时才被覆盖。由于重命名键期间会执行 del 命令删除旧的键，如果键对应值比较大会存在阻塞的可能。

---

#### **Q2：如何设置键过期？**

expire key seconds：键在 seconds 秒后过期。

如果过期时间为负值，键会被立即删除，和 del 命令一样。persist 命令可以将键的过期时间清除。

对于字符串类型键，执行 set 命令会去掉过期时间，set 命令对应的函数 setKey 最后执行了 removeExpire 函数去掉了过期时间。setex 命令作为 set \+ expire 的组合，不单是原子执行并且减少了一次网络通信的时间。

---

#### **Q3：如何进行键迁移？**

* move 命令用于在 Redis 内部进行数据迁移，move key db把指定的键从源数据库移动到目标数据库中。
    move
* 可以实现在不同的 Redis 实例之间进行数据迁移，分为两步：
    ①dump key，在源 Redis 上，dump 命令会将键值序列化，格式采用 RDB 格式。
    ②restore key ttl value，在目标 Redis 上，restore 命令将序列化的值进行复原，ttl 代表过期时间， ttl = 0 则没有过期时间。
    整个迁移并非原子性的，而是通过客户端分步完成，并且需要两个客户端。
    dump \+ restore
* 实际上 migrate 命令就是将 dump、restore、del 三个命令进行组合，从而简化操作流程。migrate 具有原子性，支持多个键的迁移，有效提高了迁移效率。实现过程和 dump \+ restore 类似，有三点不同：
    ① 整个过程是原子执行，不需要在多个 Redis 实例开启客户端。
    ② 数据传输直接在源 Redis 和目标 Redis 完成。
    ③ 目标 Redis 完成 restore 后会发送 OK 给源 Redis，源 Redis 接收后根据 migrate 对应选项来决定是否在源 Redis 上删除对应键。
    migrate

---

#### **Q4：如何切换数据库？**

select dbIndex，Redis 中默认配置有 16 个数据库，例如 select 0 将切换到第一个数据库，数据库之间的数据是隔离的。

---

#### **Q5：如何清除数据库？**

用于清除数据库，flushdb 只清除当前数据库，flushall 会清除所有数据库。如果当前数据库键值数量比较多，flushdb/flushall 存在阻塞 Redis 的可能性。
