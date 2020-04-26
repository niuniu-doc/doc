---
title: Mysql是怎么保证数据不丢的
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
![binlog写盘状态.png](https://upload-images.jianshu.io/upload_images/14027542-56a40a40793fd005.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

一、binlog的写入机制
1. binlog写入逻辑: 
   1) 事务执行过程中、先写日志导binlog cache、事务提交时、再把binlog cache写入到binlog文件中.
   2) 一个事务的binlog不能被拆开、因此不论事务有多大、也要确保一次性写入. 系统给binlog cache每个线程分配一片内存(binlog_cache_size大小), 超过会先暂存到磁盘.
   3) 事务提交时、执行器会把binlog cache里的完整事务写入binlog、清空binlog cache.
2. 每个线程有自己的binlog cache、但共用binlog文件.
   事务提交、先写入到文件系统的page cache(write), 然后调用fsync写入磁盘(占用IOPS)
   write和fsync的时机:
   sync_binlog=0, 每次事务提交只write、不fsync
   sync_binlog=1, 每次事务提交都执行fsync
   sync_binlog=N, 每次事务提交都write、累积N个事务才fsync
   所以、在出现IO瓶颈的场景里、可以将sync_binlog设置为一个比较大的值、可以提升性能.
   风险: 若主机异常重启、会丢失最近N个事务的binlog.

二、redo log的写入机制
事务执行过程中会先写redo log buffer, 然后才写redo log
![redo log存储状态.png](https://upload-images.jianshu.io/upload_images/14027542-c93a7e2e48dc724e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从redo log的三种状态说起:
1. 存在redo log buffer中、物理上是在 mysql 进程内存中.
2. 写到磁盘write、但未持久化fsync、物理上是在文件系统的Page Cache中
3. 持久化到磁盘

1、2的过程都很快、但3的速度就慢很多了. InnoDB提供了三种策略, 通过 innodb_flush_log_at_trx_commit参数控制:
1. 0, 表示每次事务提交只把redo log留在redo log buffer中
2. 1, 表示每次事务提交都将redo log直接持久化到磁盘
3. 2, 表示每次事务提交都把redo log写到Page Cache.
InnoDB 有一个后台线程、每隔1s、会把redo log buffer中的日志调用write写到FS Pae Cache、然后调用 fsync 持久化到磁盘.

注意: 事务执行过程中的redo log也是在buffer中、可能会被后台线程一起持久化到磁盘
还有两种场景会将一个未提交的事务redo log写入磁盘:
1. redo log buffer占用的空间即将达到innodb_log_buffer_size 一半的时候、后台线程会主动写盘. (此时事务未提交、只是write、并未fsync, 即: 只留在FS Page Cache)
2. 并行事务提交时, 顺带将该事务的 redo log buffer持久化到磁盘、eg. Trx A执行到一半、Trx B要把buffer数据写入磁盘、会顺带把Trx A的日志一起持久化到磁盘
注意: 若将 innodb_flush_log_at_trx_commit 设为1, redo log在prepare阶段就要持久化一次、因为有一个崩溃恢复依赖于prepare的redo log + binlog.
通常说的`双1`配置、是redo log 和binlog的刷盘机制都设为1, 即: 一个事务完整提交前、需要等待两次刷盘: redo log(prepare阶段) 和 binlog

思考: TPS 2w/s 的话、写盘就是 4w/s, 但磁盘能力只有2w/s, 是怎么实现的呢 ?
组提交: 三个并发事务trx1, trx2, trx3, 对应LSN(日志逻辑序列化、单调递增)分别为:50, 120, 160, trx1写盘时、这组(trx1->3)已经有3个事务、LSN也变成了160, 去写盘时、带的LSN=160, 等trx1返回时、所有LSN<160的redo log都已持久化到磁盘, trx2,trx3可直接返回.
![两阶段提交细化.png](https://upload-images.jianshu.io/upload_images/14027542-1044eab2470e5e22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)

在并发更新场景下、第一个事务写完 redo log buffer、调用fsync越晚、组员越多、节约IOPS效果越好.
binlog的write 和 fsync的时间间隔短、组提交优化不如redo log. 
可以通过设置以下参数来提升效果:
1. binlog_group_commit_sync_delay, 延迟x 微秒后才调用fsync
2. binlog_group_commit_sync_no_delay_count, 累积x次以后调用fsync
二者满足其一就调用 fsync

另: 不建议设置 innodb_flush_log_at_trx_commit=0, 因为这样redo log只保存在内存中、MySQL异常重启会丢失数据、风险太大. 而redo log写到 FS Page Cache的速度也是很快的、不会损失很多性能, 可以保证异常重启不丢数据、风险小很多.
