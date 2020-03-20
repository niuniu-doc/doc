---
title: mysql事务
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
### mysql 事务

1. ACID原则:
```
Automicity: 原子性
Consistency: 一致性
Isolation: 隔离性
Durability: 持久性
```


2. SQL隔离级别
```
1. 读未提交: read uncommited, 事务未提交时、别的事务就能读到它的变更
2. 读提交: read commited, 事务提交之后、才能被别的事务读到变更
3. 可重复读: repeatable read, 事务在执行过程中看到的数据、始终保持跟事务启动时看到的一致
4. 串行化: seriable, 对同一记录、写会加写锁、读会加读锁、读写锁冲突的时候、必须等到前一个锁释放才能执行
```

![image.png](https://upload-images.jianshu.io/upload_images/14027542-374aecc9f9ddf1bd.png?imageMogr2/auto-orient/strip%7CimageView2/1/w/240)

#### 不同事物隔离级别下、事物A的返回结果

* 若隔离级别是`读未提交`, 则 `V1` 的值是`2`、此时、B未提交A可以读到、
  `V2`, `V3` 的值也都是2
* 若.. 是`读提交`, 则`V1`是1、`V2` 是2、事物B的更新在提交后可以被A读到、则`V3`的值也是2
* 若隔离级别是`可重复读`、则`V1`, `V2`的值是1、`V3`的值2、因为`V2`在`事务A`提交之前、所以、`V1`, `V2` 的值为1
* 若隔离级别是`可串行化` 则`事务B`在执行`1->2`的过程会被锁、直到A提交、所以`V1`, `V2`是1 、

#### 查看事务隔离级别
```
mysql> show variables like 'transaction_isolation';
+-----------------------+-----------------+
| Variable_name         | Value           |
+-----------------------+-----------------+
| transaction_isolation | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.04 sec)
```

```
mysql> select @@transaction_isolation;
+-----------------+
| @@transaction_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
```

##### mysql 全局事务隔离级别修改后、在本会话不会生效、只影响后续会话 

```
mysql> set global transaction isolation level read committed;
Query OK, 0 rows affected (0.01 sec)

mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| REPEATABLE-READ         |
+-------------------------+
1 row in set (0.00 sec)

```
##### mysql session级别事务修改、只影响当前会话
```
mysql> set session  transaction isolation level read uncommitted;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| READ-UNCOMMITTED        |
+-------------------------+
1 row in set (0.00 sec)
```
*Q: 
```
1.合适需要RR级别呢 ？
   假设正在对数据做校对、是不是希望在校对过程中、用户产生的交易不会影响校对的结果 ？~~
2. 回滚日志什么时候删除？
系统会判断当没有事务需要用到这些回滚日志的时候，回滚日志会被删除
3. 什么时候不需要了？
  当系统里么有比这个回滚日志更早的read-view的时候
4. 为什么尽量不要使用长事务
  长事务意味着系统里面会存在很老的事务视图，在这个事务提交之前，回滚记录都要保留，这会导致大量占用存储空间。除此之外，长事务还占用锁资源，可能会拖垮库
5. 
```

#### 事务隔离的实现
```
事务隔离的实现：每条记录在更新的时候都会同时记录一条回滚操作。同一条记录在系统中可以存在多个版本，这就是数据库的多版本并发控制（MVCC）
```

Q: 由上所述、若事务隔离级别为RR、事务T启动的时候会创建一个一致性视图read-view、执行期间、若有其它事务修改了数据、事务T看到的数据不变
但: 这个事务要更新R1时、恰好R1被T2占有行锁、则T会进入等待状态、此时: 当它拿到行锁、可以执行更新的时候、读到的值是什么呢 ？
```sql
create table `t`(
`id` int(11) not null , 
`k` int(11) default null ,
primary key (`id`)
)engine=innodb;

insert into t(id, k) values (1, 1),(2, 2);
```
SQL执行顺序如下：
![image.png](https://upload-images.jianshu.io/upload_images/14027542-cd319103b6d27133.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)

那么两个查询语句得到的结果分表是什么呢？

A: 先看下几个概念:
```
1. begin/start transaction 并不是事务的七点、在执行到它们之后的第一个innodb表语句、事务才真正启动、马上执行事务、可以使用：
   start transaction with consistent snapshot;

2. mysql中两个视图的概念
   1) view: 是用查询语句创建的虚拟表、在调用的时候执行查询语句并生成结果、create view ... 
   2) innodb在实现MVCC时用到的一致性读视图、即 cinsistent read view, 用于执行 RC(Read Committed, 读提交)
       和RR(Repeatable Read, 可重复读)隔离级别的实现
   
```
> 两种启动事务的方式
  1.一致性视图是在执行第一个快照语句时创建
  2 在执行 start transaction with consistent snapshot 时创建
  
大前提:
```
1. 事务隔离级别为RR
2. autocommit=1
3. 注意事务启动的时机
```
结果:
```
A 查询得到1 B查询得到3 C更新成功
```

so. 一脸迷茫了、^.^...
来看下: 快照在MVCC里是如何工作的
```
在RR级别下、事务在启动的时候就拍了个快照(基于db)

快照如何实现？
1. innodb每个事务有一个唯一id、叫 transaction id. 在事务开始时向事务系统申请、严格递增
2. 每行数据有多个版本、每次更新都会生成一个新的数据版本、并把trx id赋值给这个数据版本的事务id、记为: row trx_id
   旧的数据版本要保留、且在新的数据版本中可以拿到、即: 表中一行记录、有多个版本row、每个版本有自己的row trx_id
   一个记录被连续更新后的状态如下:
```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-249460f611087ba6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)
```
虚线内是同一行数据的4个版本、当前最新版本是V4、k的值是22、是被 transaction id 为25的事务更新的

undo log呢? 
上图中的三个虚线就是undo log、而V1 V2 V3 并不是物理存在的、而是在需要的时候根据当前版本和undo log计算的
事务只认、事务启动之前提交的内容、如果提交之后的并不认、必须要找到它的上一个版本、若上一个版本也不可见、则继续查找

实现: 
innodb为每个事务构造了一个数组、用来保存事务启动的瞬间、当前正在活跃(启动但未提交)的所有事务id

数组里id的最小值即为 低水位、当前系统已创建过的事务id的最大值+1 即为 高水位
视图数组和高水位组成了当前事务的一致性视图 read-view
数据版本的可见性规则、就是基于row trx_id和一致性视图的对比结果得到的

```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-be120e1119794bcf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

```
对于当前事务启动的瞬间、一个数据版本的row trx_id有几种情况
1. 若落在绿色部分、表示已提交事务或当前事务自己生成的、可见
2. 红色部分、表示这个版本是将来启动的事务生成的、不可见
3. 黄色部分、
   a. 若 row trx_id 在数组中、表示由未提交的事务生成的、不可见
   b. 若不在数组中、表示已提交事务生成、可见
   
   有了row trx_id、事务的快照就是静态的了...
```

假设:
1. 事务开启前、系统只有一个活跃事务id 99
2. 事务A、B、C的版本号 100、101、102 且当前系统只有这4个事务
3. 事务开始前、(1,1)这行数据的 row trx_id是90

事务查询逻辑图:
![image.png](https://upload-images.jianshu.io/upload_images/14027542-7602ef99ebe62a71.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

所以、事务A的查询流程:
```
1. 找到(1,3) 判断trx_od=101 比高水位大、处于红色区域、不可见
2. 找到上个版本、row trx_id=102 比高水位大、处于红色区域、不可见
3. 继续、找到(1,1) row trx_id=90、比低水位小、处于绿色区域、可见

虽然这行数据被修改过、但事务A无论在何时查询、结果都是一致的、称为一致性读
```

#### 更新逻辑
那么是不是有个疑问:
按照一致性读、好像 事务B的update语句、是不对的 ？

![image.png](https://upload-images.jianshu.io/upload_images/14027542-9c0c0ec70b636639.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)
```
B进行update时、是不是看不到(1,2)？怎么计算出来的(1,3) ？

是的: 如果在事务B更新之前查询一次数据、会发现、返回的 k 的确是1
但是更新数据的时候、就不能再历史版本上更新了、否则、事务C的更新就丢失了
so. 事务B此时的更新是在(1,2) 的基础上进行的操作

规则:
更新都是先读后写的、这个读是当前读(current read)、只能读当前的值

其实、除了update、若select加锁、也是当前读

mysql> select k from t where id=1 lock in share mode; // 读锁(S锁、共享锁)
mysql> select k from t where id=1 for update; // 写锁(X锁、排他锁)

```

再往前一步：
若 C不是马上提交的、而是事务C' 会如何？

![image.png](https://upload-images.jianshu.io/upload_images/14027542-11fddfc9015b0955.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

```
事务 c' 不同的是、未马上提交、在它提交前、事务B的更新发起、而此时 C' 未提交、但是(1,2)这个版本也生成了、并且是当前最新版本
那么B如何处理？

此时就要考虑 两阶段锁协议、事务C'未提交、则写锁未释放、事务B是当前读、必须读最新版本、且必须加锁、
则B被阻塞、必须等到C'释放、才可继续

```

Q: 为何表结构不支持 可重复读?
```
表结构无对应的数据行、也没有 row trx_id 所以、只能遵循当前读
```

Q: 使用如下表结构和初始化语句作为实验环境、事务隔离级别是可重复读、想把 字段 c和id 等值的行 的c值清零、发现 并未改掉
解释这种情况出现的场景及原理, 及如何避免？
```sql
create table `t`(
`id` int(11) not null ,
`c` int(11) default null ,
primary key (`id`)
)engine=innodb
insert into t(id, c) values (1,1), (2,2), (3, 3), (4, 4);
```
A: 场景一

```
会话A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t;
+----+------+
| id | c    |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.01 sec)

mysql> update t set c=0 whete id=c;

会话B
mysql> update t set c=c+1;
Query OK, 4 rows affected (0.01 sec)
Rows matched: 4  Changed: 4  Warnings: 0

会话A
mysql> select * from t;
+----+------+
| id | c    |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)

会话A
mysql> commit;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t;
+----+------+
| id | c    |
+----+------+
|  1 |    2 |
|  2 |    3 |
|  3 |    4 |
|  4 |    5 |
+----+------+
4 rows in set (0.00 sec)
```
即：
![image.png](https://upload-images.jianshu.io/upload_images/14027542-eae83ec8f3ccedc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)


场景二：
![image.png](https://upload-images.jianshu.io/upload_images/14027542-7b63a20e1a0be038.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)

**记住:update时、为当前读 current read**



#### 幻读
```
1. 在 可重复读 RR 隔离级别下、普通的查询是快照读、是不会看到别的事务插入数据的、
   因此 幻读是在 当前读 下才会发生
2. 数据被修改、在当前读可以看到修改后的结果、这个不叫幻读、
   幻读 专指 新插入的行被当前读读到
```

#### 间隙锁-gaps lock
```
间隙锁: 锁的是两个值之间的空隙
跟间隙锁存在冲突关系的是: 往这个间隙插入记录、间隙锁之间不存在冲突关系
 
行锁冲突关系: 读-读: no 读-写:no 写写: yes
即: 跟间隙锁存在冲突关系的、是另外一个行锁

间隙锁的引入会导致同样的语句锁住更大的范围、影响并发
间隙锁只有在RR的隔离级别下才会生效、

把隔离级别设为读提交的话、就没有间隙锁了、但是、要解决可能出现的数据和日志不一致的问题、
需要把binlog的格式设置为row
```
