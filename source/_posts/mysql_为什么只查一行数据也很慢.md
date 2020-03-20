---
title: 为什么只查一行数据也很慢
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
1. 可能在等待锁
```
show process list;
如果看到 State列为: waiting for table metata lock; 则证明在等待MDL表锁

select blocking_pid from sys.schema_table_lock_waits 可以找到阻塞的pid
```

2. 可能在等待flush
```
State列为: waiting for table flush
```
Mysql对表的flush一般有两种方式
```
1. flush tables t with read lock. - 只关闭表t
2. flush tables with read lock. - 关闭所有表
```
正常情况下、这两个操作都很快、除非被阻塞...

3. 等行锁
```
mysql> select * from t where id=1 lock in share mode; 

>5.7 版本可以通过看到被哪个sql阻塞了
mysql> select * from t sys.innodb_lock_waits where locked_table=`'test'.'t'`\G
```

4. sql慢
