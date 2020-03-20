---
title: 死锁日志分析
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
### innodb基于系统参数开启监控
```
1. set global  innodb_status_output=on; #开启innodb标准监控
2. set global innodb_status_output_locks=on; # 开启innodb锁监控
3. set global innodb_print_all_deadlocks=on; # 将死锁日志记录到错误日志文件
```

### 死锁分析

```
2019-05-21 21:46:44 7f9ac2fff700
*** (1) TRANSACTION:
TRANSACTION 14234356386, ACTIVE 0 sec fetching rows
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1184, 2 row lock(s)
MySQL thread id 101287388, OS thread handle 0x7f9ac472c700, query id 15913830552 10.20.00.217 zop Searching rows for update
update driver_statistics set statistics_status=0 , statistics_time=NOW() where statistics_status=1 and statistics_time <= DATE_SUB(NOW(),INTERVA
L 120 SECOND)
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 65 page no 203 n bits 208 index `PRIMARY` of table `platform`.`driver_statistics` trx id 14234356386 lock_mode X loc
ks rec but not gap waiting
*** (2) TRANSACTION:
TRANSACTION 14234356375, ACTIVE 0 sec updating or deleting, thread declared inside InnoDB 0
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 1
MySQL thread id 101570708, OS thread handle 0x7f9ac2fff700, query id 15913830539 10.20.00.217 zop updating
update driver_statistics
     SET status = 2,time = '2019-05-21 21:46:44.96',d_id = 44201,
        d_phone = '15601237231', d_name = 'k'
    where id = 54145
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 65 page no 203 n bits 208 index `PRIMARY` of table `platform`.`driver_statistics` trx id 14234356375 lock_mode X loc
ks rec but not gap
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 65 page no 7749 n bits 1616 index `idx_statistics_status` of table `platform`.`driver_statistics` trx id 14234356375
 lock_mode X locks rec but not gap waiting
*** WE ROLL BACK TRANSACTION (2)
------------
```

#### 先看事务一的信息
> ***(1) TRANSACTION:
TRANSACTION 14234356386, ACTIVE 0 sec fetching rows

`active 0 sec` 表示事务活跃时间
`fetching rows` 表示事务正在做的事儿
```
可能的事务有 inserting、updating、deleting、fetching rows等
```
> mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1184, 2 row lock(s)

`mysql tables in use 1` 表示mysql有一个表被使用
`locked 1` 表示有一个表锁
`LOCK WAIT` 表示事务正在等待锁
`4 lock struct(s)` 表示该事务的锁链表长度为 4, 表示该事务的锁链表的长度为 11，每个链表节点代表该事务持有的一个锁结构，包括表锁，记录锁以及 autoinc 锁等。
`heap size 1184` 为事务分配的锁堆内存大小
`2 row lock(s)` 表示当前事务持有的行锁个数, 通过遍历上边的4个锁结构、找到其中类型为LOCK_REC的记录数、

> MySQL thread id 101287388, OS thread handle 0x7f9ac472c700, query id 15913830552 10.20.00.217 zop Searching rows for update

`MySQL thread id .. OS...` 当前事务线程信息
`10.20.00.217 zop` 数据库的ip和dbname
`Searching rows for update` 当前事务的状态

> update d_statistics set statistics_status=0, statistics_time=now() where statistics_status=1 and statistics_time <=date_sub(now(),interval 120 seconds)

> 当前正在执行的sql语句

> *** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 65 page no 203 n bits 208 index `PRIMARY` of table `platform`.`driver_statistics` trx id 14234356386 lock_mode X loc
ks rec but not gap waiting

`lock_mode X` 表示改记录锁为排他锁
`RECORD LOCKS` 表示正在等待的是记录锁
`index PRIMARY` 表示要加锁的索引为主键索引
`n bits 208` 表示这个记录锁结构上保留有208个bit位(改page上的记录数+64)、
`lock_mode X`表示该记录锁为排他锁、
`locks rec but not gap waiting`表示要加的锁为记录锁、并且处于锁等待状态


#### 再看事务二的分析
`undo log entries 1`表示当前事务有1个undo log记录、因为二级索引不记undo log、表示该事务更新了1个聚簇索引
```
 (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 65 page no 203 n bits 208 index `PRIMARY` of table `platform`.`driver_statistics` trx id 14234356375 lock_mode X loc
ks rec but not gap
```
表示事务2 持有主键索引的X记录锁、

```
(2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 65 page no 7749 n bits 1616 index `idx_statistics_status` of table `platform`.`driver_statistics` trx id 14234356375
 lock_mode X locks rec but not gap waiting
```
表示事务2正在等待二级索引上的X记录锁

#### mysql 加锁方式

[https://www.aneasystone.com/archives/2017/11/solving-dead-locks-two.html](https://www.aneasystone.com/archives/2017/11/solving-dead-locks-two.html)
```
批量update时、mysql server会根据where条件读取第一条满足条件的记录、innodb引擎返回第一条记录、并加锁(current read)、待mysql server收到这条加锁的记录后、再发起update请求、更新记录
server和innodb的交互是一条一条进行的、加锁也是一条一条进行的、
在给二级索引加X锁的同时、会给主键索引也加上X锁、
记录锁是加在索引上的、若表未建索引、db也会隐式的创建一个索引
```

**注意**
```
若sql无法使用索引时会走主索引实现全表扫描、mysql会给全表所有的记录行加记录锁、若一个where条件无法索引快速过滤、存储引擎就会将所有的记录加锁后返回、mysql server在进行过滤时、若发现不满足、会调用unlock_row把不满足的记录释放、但是每条记录的加锁操作还是不能省略的、so在没有索引时、会消耗大量的锁资源、增加db开销、降低db的并发性

RC级别不会加间隙锁
```

#### 加锁情况分析
```
1. 主键索引命中:
    RR和RC 一样、都只加记录锁

2. 主键索引非命中
    RC不加锁、RR加 gap 锁

3. 唯一二级索引、查询命中
   RC和RR都只加记录锁、加二级索引锁的同时会加主键索引锁

4. 唯一二级索引、查询非命中
   RC不加锁、RR会加GAP锁、但只加二级索引、不加主键索引

5.二级非唯一索引、查询命中
  RC会加记录锁、RR还会加GAP锁、

6.二级非唯一索引、查询未命中
  RC不加锁、RR会加GAP锁、

7. 无索引
   若where条件不能走索引、只能走全表扫描
   RC会给所有记录加行锁、RR会给所有记录加行锁 + 所有聚簇索引之间加GAP锁

8.聚簇索引范围查询
   包含等于时、会给下一条记录也加上next-key lock

9.二级索引范围查询

```


#### 参考
[https://www.aneasystone.com/archives/2018/04/solving-dead-locks-four.html](https://www.aneasystone.com/archives/2018/04/solving-dead-locks-four.html)
[https://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html](https://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html)
[https://blog.csdn.net/enweitech/article/details/52447006](https://blog.csdn.net/enweitech/article/details/52447006)
[https://blog.csdn.net/miyatang/article/details/78227344](https://blog.csdn.net/miyatang/article/details/78227344)
[https://time.geekbang.org/column/article/75659](https://time.geekbang.org/column/article/75659)
[https://my.oschina.net/u/2408025/blog/535941](https://my.oschina.net/u/2408025/blog/535941)
[https://dbarobin.com/2015/01/27/innodb-lock-wait-under-mysql-5.5/](https://dbarobin.com/2015/01/27/innodb-lock-wait-under-mysql-5.5/)
[https://www.aneasystone.com/archives/2018/04/solving-dead-locks-four.html](https://www.aneasystone.com/archives/2018/04/solving-dead-locks-four.html)
[https://www.cnblogs.com/LBSer/p/5183300.html](https://www.cnblogs.com/LBSer/p/5183300.html)
