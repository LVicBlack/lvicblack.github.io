---
title: Semisynchronous Replication 半同步复制
date: 2021-10-19 16:06:00
categories: 
- MySQL
- 复制机制
tags:
- MySQL复制机制
---

[MySQL8 半同步复制 Semisynchronous Replication](https://dev.mysql.com/doc/refman/5.5/en/replication-semisync.html)

### 前言

  MySQL 5.5版本之前默认的复制是异步(Asynchronous )模式的, MySQL 5.5 以plugins的方式提供了[Semisynchronous Replication ](http://dev.mysql.com/doc/refman/5.5/en/replication-semisync.html)模式。在介绍 semi sync 之前,我们先了解：半同步 Asynchronous 和 同步 Synchronous 。


异步复制模式
  主库将已经提交的事务event 写入binlog后，即返回成功给app，该模式下并不保证任何已经提交的事务会传递到任何slave并被成功应用。

全同步复制模式。
  **当主库提交一个事务 event，主库会等待该事务被传递到所有的slave上，且所有slave applay 该事务/event 通知主库之后，才会返回回话，事务已经成功。**

  **从定义中可以看出 异步模式不能保证数据的安全性，因为它不等待主库提交的事务在slave 上落盘，而全同步模式 由于要等待所有的slave 确认已提交事务成功被应用，如此则会带来事务处理上的延时。semi sync 则取了一个比较折中的方式，确保已提交的事务必须存在于至少两个机器(主库和任一备库)，立即返回给客户端 事务成功。**

### 一 Semisynchronous Replication 定义


 Semisynchronous Replication模式下,在主库上提交一个事务/event，它会等待至少一个slave通知主库，slave 已经接收到传递过来的events并写入relay log，才返回给回话层 写入成功，或者直到传送日志发生超时。
  ![img](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/Semisynchronous%20Replication.jpg) 

 

### 二 优缺点
- 优点：当事务返回成功给客户端时，则事务至少在两台机器上存在，增强数据安全性。相比异步模式和全同步模式，是一种折中。

- 缺点：半同步的确会对数据库性能有一定影响，因为事务的提交必须等待slave 反馈。性能损耗取决于tcp/IP 网络传输时间，也即传输已提交事务和等待slave 反馈已经接收事务的时间。

### 三 MySQL 半同步的特性
  1. 当slave 连接主库时，它会告知主库它是不是semi sync 模式。
  2. 如果主库启用了semi sync模式，且至少一个slave 也启用了semi sync模式，一个在主库操作事务的进程在事务提交之后，且至少一个slave 通知主库成功接收所有事务之前，该进程会处于blocks 等待状态或者直到超时发生。
  3. 当且仅当传递过来的events 传递到slave，被写入relay log，刷新到磁盘才会通知主库完成。
  4. Semisynchronous replication 必须在主备两端都同时启用，否则任何一个未设置，主备之间的复制模式将转变为异步复制模式。
  5. 当所有slave 在（rpl_semi_sync_master_timeout的默认值)时间内未返回给主库成功接收event，主备之间就会变回原来的异步状态。

 其中关于第二点 MySQL 5.7 已经做了优化，由[ack Collector (Col) thread](http://dev.mysql.com/worklog/task/?id=6630) 等待备库的成功接收事务的通知,这点后续会做详细介绍--《[5.7 Semisync replication 增强](http://blog.itpub.net/22664653/viewspace-1183057/)》。

### 四 异常处理
  当备库Crash时，主库会在某次等待超时后，关闭Semi-sync的特性，降级为普通的异步复制，这种情况比较简单。

MySQL的 error.log 会提示：
```
140523 22:26:00 [Warning] Timeout waiting for reply of binlog (file: mysql-bin.000002, pos: 465893519), semi-sync up to file , position 0.
140523 22:26:00 [Note] Semi-sync replication **switched OFF**.  
```
比较难以处理的情况是:当主机/主库Crash时，可能存在一些事务已经在主库提交，但是还没有来的及传给任何备库，也即这些事务都是没有返回给客户端的，所以发起事务的客户端并不知道这个事务是否已经完成--**"墙头事务"**。这时，如果客户端不做切换，只是等Crash的主库恢复后，继续在主库进行操作，客户端会发现前面的"墙头事务"都已经完成，可以继续进行后续的业务处理；另一种情况，如果客户端Failover到备库上，客户端会发现前面的“墙头事务”都没有成功，则需要重新做这些事务，然后继续进行后续的业务处理，**其实此时主备是不一致的，需要通过主备数据校验来检查哪一个库是正确的，然后进行修复。**

### 五 小结
  总之相比于MySQL 5.5 版本之前的异步复制模式 semi sync 已经有了很大的进步，增强了数据的安全性，以安全换一定的性能损耗还是可以接受的。
