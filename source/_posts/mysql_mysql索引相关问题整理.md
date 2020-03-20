---
title: mysql索引相关问题整理
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
#### count(1) count(*) count(id)
> mysiam:
将表的总行数保存到了磁盘上、所以对于无条件的count(*) 是很快返回的

> innodb实现：
count(*) 是把数据记录一行行的拿出来判断、
(innodb是索引组织表、主键索引的叶子节点是数据记录、普通索引的叶子节点是主键值、会小很多、mysql会选择最小的那颗索引树来遍历、在保证逻辑正确的情况下、尽量减少扫描的数据量)
count(1) 遍历表、但不取值、扫描记录时、返回1给server
count(id) 会把id返回给server
count(field) 
若定义为非null、会先判断该字段记录是否为null、非null才累加
若定义为null、从记录读出字段、累加


#### 为什么mysql不直接记录count数？
> innodb要保证事务执行、不同会话、commit前后 总行数是会发生改变的

#### show table status 替代？
> 得到的结果是通过采样估算得到的、不精准、最高偏差有40%-50%

#### 采样缓存系统保存计数?
> 1.redis重启、数据丢失
2.redis和db本身记录增加先后的问题会导致短时间的不精准

#### 使用mysql表保存总记录？
> 可以、使用mysql的事务来保证、但是会影响性能


mysql两阶段提交的过程：
![image.png](https://upload-images.jianshu.io/upload_images/14027542-0f3b96cc0f6f0519.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)


#### 在不同阶段crash对于系统的影响
```
1. 图中A时刻(redo log写完、binlog还没写的时候)crash、binlog还没写、redo log也没提交、重启事务会回滚~
2.若是在B、binlog写完、redo log还未commit时、crash会发生什么？
   a. 若redo log也是完整的-有了commit标识、直接提交
   b. 若redo log只有完整的prepare、则判断对应的binlog是否完整、
      完整-提交事务; 否则: 回滚事务
```

#### mysql如何知道binlog是完整的？
```
一个事务的binlog有完整的格式:
statement格式的: 最后会有commit
row格式的: 最后会有Xid event
MySQL5.6.2 之后、还引入了checksum检查日志中间出错的情况
```

#### redo log和binlog是如何关联的
```
有一个共同的字段XID、crash恢复的时候会按顺序扫描redo log:
1. 遇到既有prepare 又有commit的redo log直接提交
2. 遇到只有prepare的、就拿XID去对应的binlog查找事务
```

#### 处于prepare阶段的redo log + 完整的binlog重启就能恢复、为什么这么设计?
```
在时刻B、binlog就已经被写入了、若是应用于从库、从库就有了这条记录、为了保证主从数据的一致性、就必须保证主库也有这条记录、所以把redo log提交
```

#### 为什么不是先写完redo log再写binlog ？
```
比较典型的分布式问题
若redo log提交了、又不能回滚(回滚可能会覆盖掉其它的事务)、所以redo log直接提交、binlog写入失败的时候、由于不能回滚、就会比从库多一个事务
```

#### 为什么不直接使用binlog、不用redo log ？
```
若是历史原因: innodb本身不是mysql的原生引擎、原生引擎是mysiam、不支持崩溃恢复
innodb在加入mysql之前、就可以支持崩溃恢复和事务
innodb发现binlog没有崩溃恢复的能力、那就直接使用redo log吧、
如果用binlog支持崩溃恢复呢 ？流程如下图
在这样的流程下、binlog还是不能支持崩溃恢复、不能支持恢复数据页
若binlog2写完、未commit的时候crash、引擎内部事务2会回滚、应用binlog2可以补回来、但对于binlog1事务已经提交、不会再应用binlog1

innodb使用的是WAL、在写完内存和日志的时候、事务就算完成了、若以后崩溃、依赖日志恢复数据页、图中1位置crash 事务1可能会丢失、且是数据页级的丢失、binlog未记录数据页的更新细节、不支持数据页恢复
```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-9b6ac9029bcb1485.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

#### 能只用redo log不用binlog吗？
```
1. 只从崩溃恢复的角度来讲、可以关掉binlog、系统依然是crash-safe的、但binlog有redo log不可替代的功能
a. 归档. redo log是循环写、日志无法保留
b. mysql系统依赖于binlog、binlog作为mysql本身就有的功能、
c. 一些异构系统、需要消费binlog来更新数据
```

#### redo log一般设置多大？
```
redo log过小、会导致很快写满、不得不强行刷盘、这样WAL的能力就发挥不出来了、若磁盘在T级别、就直接设置为G级别吧~
```

#### 正常运行的实例、数据写入后的最终落盘是从redo log更新的还是从buffer poll更新的？
```
redo log 并没有记录数据页的完整数据、所以本身没有能力更新磁盘数据页、
1. 数据页被修改后与磁盘数据页不一致、最终落盘就是把内存中的数据写入磁盘、与redo log无关
2. 崩溃恢复的场景中、innodb如果判断到一个数据页可能在崩溃恢复的时候丢失了更新就会将它读入内存、然后让redo log更新内存内容、更新完、内存变成脏页、回到1的情况
```

#### redo log buffer是什么？是先修改内存、还是写写redo log ？
```
在事务更新过程中 redo log 是要多次写的、
eg. begin;
      insert into t1...;
      insert into t2...;
     commit;
这个事务要在两个表中插入记录、在插入的过程中、生成的日志都得先保存起来、但又不能在还没commit的时候写redo log

所以 redo log buffer就是一块内存、用来保存 redo log日志的. 即 在执行第一个insert的时候、数据的内存被修改了、redo log buffer也写入了日志
但真正写入redo log(ib_logfile+日志)是在执行commit语句的时候做的、
```

#### update记录为原值的时候、mysql如何操作？
```
1. update是先读后写、发现值本来就是原值、不操作、直接返回
2. mysql调用innodb引擎接口修改、引擎发现与原值相同、不更新、直接返回
3. innodb认真的执行了更新、改加锁的加锁?

答案是3、可以通过事务来验证
```
过程请参考: [https://time.geekbang.org/column/article/73479](https://time.geekbang.org/column/article/73479)


**varchar(255)是边界、>255需要两个字节存储、小于需要1个字节**



4. mysql 源码编译启动报错
```
mysql启动报错：Starting MySQL... ERROR! The server quit without updating PID file

看errlog 发现:
2018-01-24 07:57:03 67547 [ERROR] Fatal error: Can't open and lock privilege tables: Table 'mysql.user' doesn't exist
or: 2018-01-24 07:57:03 [ERROR]   Can't locate the language directory.

重新初始化db:
mysql_install_db --user=mysql --basedir=/home/devil/mysql57/ --datadir=/home/devil/mysql57/data/

出现:
2019-04-18 21:23:16 [WARNING] mysql_install_db is deprecated. Please consider switching to mysqld --initialize

so. 
mysqld --initialize --user=mysql --basedir=/home/devil/mysql57/ --datadir=/home/devil/mysql57/data/

可以看到初始化成功. 并生成了临时密码

```

5. mysql binlog 查看
```
a. 登录mysql查看
   1) 只查看第一个binlog文件的内容
        show binlog events;
   2) 查看指定binlog文件的内容
        show binlog events in 'mysql-binlog.000001';
   3) 查看当前正在写入的binlog
        show master status;
    4) 获取binlog文件列表
        show binary logs;

 b. 使用mysqlbinlog工具查看
     1) 本地查看 
         基于开始/结束时间: 
         mysqlbinlog --start-datatime='2019-04-10 00:00:00' --stop-datatime='2019-04-10 01:00:00'  
         基于pos值
         mysqlbinlog --start-postion=107 --stop-position=1000 -d 库名 二进制文件

      2)  远程查看
           mysqlbinlog -u{uname} -p{pass} -htest.com -P3306 \
--read-from-remote-server --start-datetime='2013-09-10 23:00:00' --stop-datetime='2013-09-10 23:30:00' mysql-bin.000001 > t.binlog
```

```
set optimizer_trace='enabled=on' 打开trace记录
select  trace from   `information_schema`.`optimizer_trace`; 查看trace记录

tmp_table_size 内存临时表大小
sort_buffer_size 用于排序的内存大小、超过会使用文件排序
max_length_for_sort_data 单行数据量超过这个值会使用rowid排序
```

```
查看sql被哪个语句阻塞
select * from t sys.innodb_lock_waits where locked_table=`'test'.'t'

select blocking_pid from sys.schema_table_lock_waits 可以找到阻塞的pid
```

 
