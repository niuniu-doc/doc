---
title: mysql锁简析
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
### mysql锁

> 根据加锁的范围、MySQL里边的锁可以分为全局锁、表级锁和行锁三类

#### 全局锁
```
全局锁: 就是要对整个db实例加锁. mysql提供了一个全局锁支持: 
flush tables with read lock;(FTWRL)
此时: update、ddl、dml语句会被阻塞

典型场景: 全库逻辑备份

但是让整个db处于只读状态、比较危险:
若是操作主库、则在整个备份期间、都不能执行更新】整个业务基本上处于停滞状态
若操作从库、则备份期间从库不能执行主库同步过来的binlog、会导致主从延迟

那么、如果实现不影响业务的备份呢 ？

1. innodb引擎可以使用一致性读
   mysql自带备份工具mysqldump使用参数 --single-transaction 时、mysql就好启动一个事务、来确保拿到一致性视图

2. 为何不选择 set global readonly=true ?
   1) 在某些系统中、readonly可能被用来做其它逻辑、比如用来判断db是主还是备库 ？
   2) 在异常处理上的差异
      若执行FTWRL之后、mysql异常断开、mysql会自动释放这个全局锁、整个库可以回到正常更新的状态
      设置readonly、若client异常、则db会一直保持readonly状态、会导致整个库长时间不可写、风险较高
   3) MDL 在slave库上对super权限无效
      
```

#### 表级锁
```
mysql的表级锁有2种: mysql表锁 和 元数据锁(meta data lock, MDL)

表锁的语法是: lock tables .. read/write. 与FTWRL类似、使用unlock tables主动释放锁、也可以在client断开时自动释放
既限制别的线程、也限制本线程
eg. 某个线程A中执行 lock tables t1 read, t2 write; 
则其它线程写t1, 读写t2 都会被阻塞、同时、线程A在执行unlock tables之前、也只能执行读t1, 读写t2的操作
但innodb可以支持行锁、一般就不使用 lock tables了、毕竟影响还是很大的

元数据锁:
不需要显式使用、在访问一个表的时候会被自动加上、MDL的作用是 保证读写的正确性
eg. 正在遍历某表的数据、在执行期间另外一个线程对这个表的结构做了变更、减少了一列、查询结果就会有问题

在mysql5.5的版本中引入了MDL、当对表curd操作时、会加MDL读锁、对表结构变更时、会加MDL写锁

* 读锁之间不互斥、可以同时对一张表增删改查
* 写锁、读写锁之间互斥、用来保证表结构变更操作的安全性
so. 两个线程同时对一个表做结构变更时、其中一个要等另外一个执行完才开始执行 

```

##### 案例分析
case: 给一个小表加字段、却导致db挂掉
```
note: 给表加字段或者修改字段或者加索引、都会导致全表扫描数据
在对大表操作时、都会特别小心、避免对线上造成影响、小表一般认为很快结束、会比较大意、其实、小表操作不当也会造成db挂掉

实验环境: mysql5.6 t是小表
```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-94717b6aa177cad2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 可以看到: session A先启动、这时会对t加MDL读锁、session B操作不被影响
session C需要MDL写锁、blocked、此时db表现为 不可读写
若: 此时db的写十分频繁、且client有重试机制、超时后起一个新的session、这些session很快就会把连接打满

Q: 那么如何安全的给表加字段呢 ？
```

首先要解决长事务、事务不提交、就会一直占着MDL锁. 在mysql的information_schema库的innodb_trx表中、可以看到当前
正在执行的事务、若要做ddl变更的表刚好有长事务在执行、要考虑先暂停DDL或者kill掉事务

若要变更的是热点表、请求频繁、虽然数据量不大、但请求特别频繁、而不得不加字段、如何做 ?

此时kill未必管用、因为新的请求马上就进来了、此时比较理想的机制是: 
在alter table语句里设定等待时间、若在这个等待时间里能拿到MDL写锁最好、拿不到也不要阻塞后边的业务、先放弃
之后再重复

语法: 
alter table table nowait add column...
alter table table wait n add column...

```


#### 查看锁表情况

> show status like 'table%';

```
table_locks_immediate 指能够立即获取表级锁的次数
table_locks_waited 指的是不能立即获取表级锁而需要等待的次数、num越大、锁等待越多、有锁争用的情况
```

#### 查看正在被锁定的表
> show open tables where in_use>0;

#### 锁涉及表说明
```
information_shcema库
innodb_trx 当前innodb内核中的活跃事务
innodb_locks 当前状态下产生的innodb锁、仅在有锁等待时打印
innodb_lock_waits 当前状态产生的innodb锁等待 仅在有锁等待时打印
```

**innodb_trx表结构说明**
```
字段名                 说明

trx_id innodb          存储引擎内部唯一的事物ID 
trx_state              当前事务状态（running和lock wait两种状态） 
trx_started            事务的开始时间 
trx_requested_lock_id  等待事务的锁ID，如trx_state的状态为Lock wait，那么该值带表当前事物等待之前事物占用资源的ID，若trx_state不是Lock wait 则该值为NULL 
trx_wait_started       事务等待的开始时间 
trx_weight             事务的权重，在innodb存储引擎中，当发生死锁需要回滚的时，innodb存储引擎会选择该值最小的进行回滚 
trx_mysql_thread_id     mysql中的线程id, 即show processlist显示的结果 
trx_query               事务运行的SQL语句 
```

**innodb_locks表结构说明**
```
字段名       说明

lock_id      锁的ID 
lock_trx_id  事务的ID 
lock_mode    锁的模式（S锁与X锁两种模式） 
lock_type    锁的类型 表锁还是行锁（RECORD） 
lock_table   要加锁的表 
lock_index   锁住的索引 
lock_space   锁住对象的space id 
lock_page    事务锁定页的数量，若是表锁则该值为NULL 
lock_rec     事务锁定行的数量，若是表锁则该值为NULL 
lock_data    事务锁定记录主键值，若是表锁则该值为NULL（此选项不可信)
```

**innodb_lock_waits表结构说明**
```
字段名             说明 

requesting_trx_id  申请锁资源的事物ID 
requested_lock_id  申请的锁的ID 
blocking_trx_id    阻塞其他事物的事物ID 
blocking_lock_id   阻塞其他锁的锁ID
```

#### 加锁的原则
```
1. 基本单位 nex-key-lock
2. 查找过程中访问到的对象才加锁
3. 索引上的等值查询、给唯一错音加锁的时候、next-key lock退化为行锁
4. 索引上的等值查询、向右遍历时且最后一个值不满足等值条件时、退化为间隙锁
5. 唯一索引上的范围查询会访问到不满足条件的第一个值为止
```
