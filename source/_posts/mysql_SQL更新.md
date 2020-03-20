---
title: SQL更新
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---

#### redolog(重做日志 - innodb引擎层特有)
```
MySQL使用AWL(Write-Ahead Logging)技术, 当有记录需要更新的时候、innodb就先把记录写到redo log、并更新内存、这时、更新就算完成. 当innodb引擎比较空闲的时候、会将操作更新到磁盘

触发更新的点:
1. redo log 写满时(redo log文件大小是固定的 - 循环覆盖) 
   write pos: 当前位置 
   check point: 当前要擦除的位置
   write和check间有空间、就代表可更新、否则、就要先擦除记录、推进checkpoint、
2. 
```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-e22222b71aa4fb6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)


#### binlog(归档日志 - MySQL server层实现)
```
mysql引擎实现的日志、与redo log 的差异？
1. redo log 是innodb特有的; binlog是mysql server层实现、所有引擎可以使用
2. redo log 记录的是物理日志、记录的是"在某个数据页上做了什么修改"
   binlog 是逻辑日志、记录的是原始逻辑、eg. 给id=2的c字段+1
3. redo log 循环写、空间固定 
    binlog 是追加写、日志写满会切换到下一个

```

#### 为什么有redo log 和 binlog ？
```
sync_binlog: 1 - 每次都持久化到磁盘
binlog: statment格式、记录的是sql语句、
           row格式、记录的是行内容、更新前后都有

mysql整体包含: 
server层 - 主要做的是mysql功能层面的事情、
引擎层 - 负责存储
最初、mysql无innodb引擎、MySQL自带的引擎是myisam、但它没有crash-safe的能力、server层的binlog日志、只能用于归档、innodb是另外一个公司以插件形式引入、用redo log 来实现crash-safe能力
```

#### update语句执行
```
update table set c=c+1 where id=2
1. 执行器先找引擎取 id=2 这一行、id是主键、引擎可以直接使用树搜索找到. 若 id = 2 所在的数据页本来就在内存中、直接返回给执行器; 否则、先从磁盘读入内存、再返回
2. 执行器拿到引擎给出的数据行、把值 +1 、得到新的一行数据、再调用引擎接口写入新的数据行
3. 引擎将新的数据行更新到内存、同时将更新操作记录到 redo log. 此时redo log 处于 prepare 状态、然后告诉执行器、执行完成、随时可以提交事务.
4. 执行器生成操作的binlog、并把binlog写入磁盘
5. 执行器调用引擎的提交事务接口、引擎把刚刚写入的redo log改成commit状态、更新完成

```
![update.png](https://upload-images.jianshu.io/upload_images/14027542-7b1034b758d49c13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

#### 为什么需要两阶段提交
```
 redo log 和 binlog 是两个独立的逻辑、若不使用两阶段提交、则会是2种情况
1. 先 redo log、 后binlog
   若redolog完成、binlog未完成、进程异常重启、redo log生效、为修改后的值 
  若使用binlog恢复临时库、由于binlog未写完就崩溃、会少一条记录、源库和备库就会出现不一致
2. 先binlog、后 redo log
    binlog写完后crash、redo log未写、在恢复时、认为事务无效、
    binlog已写完、恢复备份库时、认为有效、出现不一致...
所以、为了保证redo log 和 binlog提交状态的逻辑一致性

```

#### 疑问
```
1. binlog 是每次update都会写磁盘？
2. 如果 redo log 第一阶段完成(prepare状态)、binlog完成、此时crash、在系统重启之后、这个log会被重放吗
   会、满足prepare 和 binlog的完整、在重启时、会自动执行commit
3. 极端情况下、redo log被写满、新的事务进入、需要擦除redo log(被修改的脏页被迫刷新到磁盘) -> 数据在 commit 之前被持久化、此时如何处理 ？
这些数据在内存中属于无效事务、其它事务读不到、即时被写入磁盘也没关系、再次读入内存时、依然是原逻辑
4. redo log 是顺序写 ？ binlog是随机写 ？
```
