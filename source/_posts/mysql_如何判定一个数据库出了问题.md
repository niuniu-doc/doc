---
title: 如何判定一个数据库出了问题
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
> 主备切换有两种场景: 主动切换 和 被动切换. 被动切换大多是主库出了问题、由HA系统引发的. 那么: 如何判定主库出了问题呢 ?

一、 select 1 判定
实际上, select 1 成功返回、只能说明这个库的进程还在、并不能说明主库没问题. eg. 以下场景:
```sql
set global innodb_thread_concurrency=3; // 控制innodb并发线程上限. 超过、进入等待. 来避免线程数过多、上下文切换的成本过高.

create table `t`(
 `id` int(11) not null,
`c` int(11) default null
) engine=innodb;

insert into t values(1,1);
```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-a908616fd039d681.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在session D里、select 1是可执行成功的, 但查询表t会被堵住.
注意:
> 并发连接和并发查询不是一个概念. show processlist看到的几千个连接、指的就是并发连接.
当前正在执行的语句、才是并发查询.
连接多顶多占一些内存、而并发查询多、才是造成CPU压力的根本. 在线程进入锁等待之后、并发线程的计数会-1, 行锁(包括间隙锁)不占用CPU资源.


思考:
> Mysql会什么设计锁等待不计入并发线程数呢 ?
```
假设以下场景:
1. 线程1 执行begin; update t set c=c+1 where id=1; 启动事务trx1, 然后保持、此时线程处于空闲
状态、不计入并发线程
2.  线程2~129执行 update t set c=c+1 where id=1; 由于等行锁、进入等待、这样就有128个
线程处于等待状态.
3. 若处于等待状态线程计数不-1, innodb会认为线程用满、阻止其他查询语句进入引擎执行、线程1不能
提交、而另外的128个线程又处于锁等待状态、整个系统就阻塞了. 如下图:
此时innodb不能响应任何请求、且所有线程都处于等待状态、此时占用CPU为0, 很不合理、所以 
设计进入锁等待、将并发线程的计数器-1, 是合理的.

但: 若真的执行查询、还是计入并发线程的. eg. select sleep(100) from t;
```
![等待并发线程不减1, 系统锁死.png](https://upload-images.jianshu.io/upload_images/14027542-facf6013361bf644.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

二、查表判断
在系统库(mysql)新建一个表, eg. 命名为 health_check, 里边只放一行数据、定期检测, 可以发现由于并发线程数过多导致的db不可用.
```sql
select * from mysql.health_check;
```
但: 若空间满了、这种方法又不好使. 
更新事务要写binlog、而一旦binlog所在磁盘空间占用率达到100%, 那所有的更新和事务提交的commit语句都会被堵住, 但可以正常读取. 

三、更新判断
```sql
update mysql.headlth_check set t_modified=now();
```
节点可用性的检测都应该包含主库和备库、若用来检测主库的话、备库也要进行更新检测. 但: 备库的检测也是要写binlog的、一般会把A和B的主备关系、设计为双M架构、所以在备库B上执行的检测命令、也会发回给主库A.
但若A、B更新同一条数据、就可能发生行冲突、导致主备同步停止. 可以插入多条数据、使用A、B的server_id做主键.
```sql
create table `headlth_check`(
 `id` int(11) not null,
`t_modified` timestamp not null default current_timestamp,
primary key(`id`)
) engine=innodb;
# 检测命令
insert into mysql.health_check(id, t_modified) values(@@server_id, now()) on duplicate
```
这是一个比较常见的方案、但还存在一些问题, `判定慢`.
eg. 所有的检测逻辑都需要一个超时时间N、执行update、超过Ns不返回、认为系统不可用.
但: 假设日志盘IO利用率已经100%, 这个时候系统响应很慢、需要主备切换. 但检测使用的update需要的资源很少、拿到io就可以提交成功、超时之前就返回了、于是得到了`系统正常`的结论, 显然这是不合理的判定.

四、内部统计
针对磁盘利用率、若Mysql 可以告知每次请求的io时间、就靠谱多了.
5.6版本以后的performance_schema库、file_summay_by_event_name表里就统计了每次IO请求时间.
event_name='wait/io/file/innodb/innodb_log_file' 统计的是redo log的写入时间. 第一列 event_name表示统计的类型.
接下来3组、统计的是redo log操作时间

第一组5列、是所有IO类型的统计:
count_star 是所有IO总次数
sum、min、avg、max是统计的总和、最小、平均和最大值.(单位ps)

第二组6列、是读操作的统计
最后一列 sum_number_of_bytes_read统计的是、总共从redo log读多少个字节

第三组6列, 是写操作统计

最后四组时间、是对其它类型数据的统计、在redo log里、可以认为是对 fsync的统计.

binlog对应的event_name是: wait/io/file/sql/binlog 这行、统计逻辑同 redo log. 额外的统计这些信息、是有性能耗损的、大概在10%左右. 建议只开启需要的项. 

打开统计项之后、可以通过判断MAX_TIMER的值来判断数据库是否有问题. 
eg. 设定阈值、单次IO超过200ms属于异常、出现异常把之前的统计值清空、再次出现这个异常就可以加入监控累计值了.
