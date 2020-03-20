---
title: mysql锁
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
### 锁的种类和概念
*Shared and Exclusive Locks*
```
Shard Lock: 共享锁
官方: permits the transaction that holds the lock to read a row.
eg. select * from table where id=1 lock in share mode;

Exclusive Locks: 排他锁
官方: permits the transaction that holds the lock to update or delete a row
eg. select * from table where id=1 for update;

```

*Intention Locks*
```
锁是加在table上的、表示要对下一个层级(记录)加锁
Intention shared(IS): Transaction T intends to set S locks on individual rows in table t
Intention exclusive(IX): Transaction T intends to set X lock on those rows
在db层看到的是：
Table Lock table `db`.`table` trx_id 12121212 lock mode IX
```

*Record Locks*
```
在db层看到的是:
Record Locks space id in 281 page no 3 n bits 72 index PRIMARY of table `db`.`table` trx id 12121212 lock_mode X rec but not gap
锁是加在索引上的(从 index primary of table `db`.`table` ) 可以看出
记录锁有两种类型: 
1. lock_mode X locks rec but not gap
2. lock_mode S locks rect bot not gap
```

*Gap Locks*
```
在db层看到的是:
Record Locks space id 281 pages no 5 n bits 72 index idx_c of table `lc_3`.`a` trx id 133588125 lock_mode X locks gap before rec
gap锁是用来防止insert的
锁的不是记录、而是记录之间的间隙、eg. (10,20)
```

*Insert intention Locks*
```
在db层看到的是:
Record Locks space id 279 page no 3 n bits 72 index primary of table `lc_3`.`t1` trx id 133587907 lock_mode X insert intention waiting
```

*next-Key locks*
```
在db看到的是:
Record Locks space id 281 page no 5 n bits index idx_c of table `lc_3`.`t1` trx id 133587907 lock_mode X
Next-Key Locks = Gap Locks + Record Locks会同时锁住记录和间隙
```

*Anto-inc Locks*
```
在db层看到的结果是:
Table Lock table xx trx id 7498948 lock mode auto-inc waiting 
属于表级别的锁 http://keithlan.github.io/2017/03/03/auto_increment_lock/
```

*显式锁 vs 隐式锁*
```
显式锁(explicit lock)
显式加的锁, 在 show engine innoDB status 中能够看到、会在内存中产生对象、占用内存
eg. select ... for update, select ... lock in share mode ....

隐式锁(implicit lock)
是在索引中对记录逻辑的加锁、但实际上不产生锁对象、不占用内存空间
eg. insert into xx values(xx)
      update xx set t=t+1 where id=1;

implicit lock -> explicit lock
eg. 只有当implicit lock 产生冲突的时候、会自动转换成 explicit lock、降低锁开销
eg. A会话插入记录10、本身会加上 implicit lock、但是如果B 会话更新10这条记录、就会转换为 explicit lock
```

*metadata lock*
```
是server层实现的锁、与引擎无关
执行select时、若有ddl语句、会被阻塞、因为 select 会加上 metadata lock、防止元数据在访问过程中被修改
```

*锁迁移*
```
锁迁移、又叫锁继承
A锁住的记录是一条已经被标记为删除的记录、但是还没有被puge、然后这条被标记为删除的记录、被purge掉了、上边的锁就会给了下边一条记录、称为锁迁移
```

*锁升级*
```
一条全表更新的语句、db可能会对所有记录加锁、可能造成锁的开销很大、升级为页锁、或者表锁(mysql无锁升级)
```

*锁分裂*
```
1. innodb 实现的加锁、其实是在页上边做的、没办法直接对记录加锁
2. 一个页被读取到内存、会产生锁对象、锁对象里会有位图信息记录哪些heap no被锁住、heapno 表示的就是堆的序列号、可以认为就是定位到某一条记录
3. insert的时候、可能会产生页分裂
4. 若页分裂、原来对页上边的加锁位图信息也就变了、为了保持这种变化和锁信息、锁对象也会分裂、继续维护分裂后页的锁信息
```

*锁合并*
```
参考锁分裂
```

*latch vs lock*
```
latch: mutex、rw-lock、临界资源用完就释放、不支持死锁检测、应用程序维护 非db锁
lock: 事务结束后释放、支持死锁检测、db锁
```
