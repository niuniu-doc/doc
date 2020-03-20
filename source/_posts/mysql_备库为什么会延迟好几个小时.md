---
title: 备库为什么会延迟好几个小时
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---

> 若备库执行日志的速度低于主库生成日志的速度、备库有可能很久都追不上主库
MySQL5.6版本之前、MySQL只支持单线程复制、主库并发高、TPS较高时、可能会出现主备延迟
5.6版本之后、sql_thread不会直接更新数据、而是交给 coordinator 来中转日志和分发事务、由worker进程来更新数据. worker的线程数由 slave_parallel_works 决定.
一般32核机器设置为 8~16、备库还要提供查询、不能把CPU都吃光了.

思考: 
1. 事务能不能轮询分发给各个worker ?
不可以. 事务分发给worker以后、不同worker就独立执行了、但由于CPU的调度策略、第二个事务可能会比第一个先执行、此时、若两个事务更新的是同一行数据、就会导致主备不一致.

2. 同一个事务的多个更新语句、能不能分发给多个worker执行 ?
不可以. 若Trx1 更新了t1和t2 两个表的各一行数据、若分发到不同的worker执行、虽然备库最终结果是一致的、但 t1更新完成的瞬间、会出现不一致, 破坏了事务的隔离性

所以: 分发器 coordinator 需要满足:
1. 不能造成覆盖更新、更新同一行数据的两个事务必须分发到同一个worker
2. 同一个事务必须分发到同一个worker.

![主备流程图.png](https://upload-images.jianshu.io/upload_images/14027542-6992c49571630b74.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


不同版本的并行复制策略:
1. 5.5版本
官方5.5版本不支持并行复制. 
   1) 按表分发. 每个worker线程维护一个hash表、用来保存当前正在这个worker上执行的事务所涉及的表. 有事务分配给该worker时、涉及的表会加入到对应hash表、worker执行完成、从hash表移除. 若与多个worker冲突、coordinator进入等待.
   事务分发时、与worker冲突情况:
   a. 跟所有worker都不冲突、coordinator直接分发给最空闲的worker线程
   b. 跟多于一个worker冲突、coordinator进入等待、直到和该事务冲突的worker进程只有1个
   c. 若只跟一个worker冲突、分配给存在冲突关系的worker.
   按表分发、在多表负载均匀的场景下效果不错、但若碰到热点表、比如: 所有的更新事务都涉及同一张表时、所有事务都会分发到同一个worker、就变成单线程了.
   
   2) 按行分发. 若两个事务没有更新相同行、可以在备库并行执行、要求binlog格式必须是row.
   此时判断冲突规则就是 库名 + 表名 + 唯一键值(还要考虑唯一索引)
   相比于按表分发、按行分发在决定线程分发的时候、需要消耗更多的计算资源. 
 
 按表分发和按行分发都有约束:
 1. binlog格式必须是Row
 2. 表必须要有主键
 3. 不能有外键、有外键时、级联更新的行不会记录binlog、这样冲突检测就不准确.
 按行分发有个问题: 大事务耗费CPU和内存.
 解决: 会设置一个阈值、若单事务超过设置行阈值(eg. 单事务更新行数超过10w)、就退化为单线程模式: 1. coordinator暂时hold住事务、2. 等所有worker变成空队列、coordinator直接执行事务 3. 恢复并行模式
 
2. 5.6按库并行
原理同上、并行效果取决于压力模型. 若主库上有多个DB、并且多个DB的压力均衡、使用库级别分发效果较好. 相对表级别和行级别优势:
1) 构造hash值很快、只需要库名、且一个实例上DB数量不会过多
2) 不要求binlog格式为row、statement也可以拿到库名.
但: 若主库上的所有表都放在同一个DB里、这个策略就没用了、若不同DB热点不同、也起不到并行效果.

3. MariaDB的并行复制策略
利用的是组提交的特性. 
1) 能够在同一组里提交的事务、一定不会修改同一行
2) 主库上可以并行执行的事务、备库上也一定可以并行执行

1. 在一组提交的事务、有一个相同的commit_id、下一组就是 commit_id + 1
2. commit_id直接写入binlog
3. 传到备库应用时、相同commit_id的事务分发到多个worker执行
4. 一组全部执行完、coordinator再取下一组数据.
可以模拟主库的并行模式. 但: 并未完全实现`模拟主库并发度`这个目标, 在主库上、一组事务commit时、下一组事务是同时处于`执行中`的. 而备库需要第一组事务执行完、第二组才能开始执行、这样系统的吞吐量就不够、另外、容易被大事务拖后腿. 
假设 trx1 / trx2 / trx3为一组事务、trx2为大事务、这样、trx1/trx3执行完后也必须等待trx2执行完成、下一组才能开始执行、这段时间只有一个worker线程在工作、是对资源的浪费.

4. MySQL5.7的并行复制策略
MySQL5.7提供了 slave-parallel-type 控制并行复制策略
slave-parallel-type=database; 表示使用mysql5.6的按库并行复制
slave-parallel-type=logical_clock; 表示使用类似mariadb的策略、但做了改进

思考: 同时处于`执行状态`的所有事务、是不是可以并行 ?
不能. 可能有由于锁冲突而处于锁等待状态的事务、若这些事务在备库上分配到不同的worker、会出现主备不一致. 所以、MariaDB的核心是、处于commit状态的事务并行处理.

而实际上: 处于redo log prepare阶段的事务、都已经通过了锁冲突检验、
所以, Mysql5.7的并行复制策略思想是:
1) 同时处于prepare状态的事务、在备库可以并行执行
2) 处于prepare状态的事务、与处于commit状态的事务之间、在备库执行时、也是可以并行的.
binlog_group_commit_sync_delay表示延迟多少微秒后调用fsync
binlog_group_commit_sync_no_Delay_count表示累积多少次后才调用fsync
这两个参数可以用于故意拉长binlog从write到fsync的时间、减少binlog的写盘次数、在5.7的复制策略里、可以用来制造更多处于`prepare阶段的事务`、增加备库并行度


5. Mysql5.7.22并行复制策略
18.4月发布的5.7.22版本、增加一个机遇writeset的并行复制
binlog-transaction-dependency-tracking控制是否使用新的策略
1) commit_order、根据同时进入prepare和commit来判断是否可以并行的策略
2) writeset、表示对于事务涉及更新的每一行计算hash值、组成writeset、若两个事务没有操作相同行、就可以并行复制
3) writeset_session、在主库上同一个线程先后执行的两个事务、在备库执行时、要保证相同的先后顺序、hash值是通过 db+table+index+value计算的、类似5.5按行复制
但: 1. writeset是在主库生成后直接写入binlog的、在备库执行时、不需要解析binlog、节省计算量
2. 不需要把整个事务的binlog都扫一遍、才决定分发到哪个worker、更省内存
3. 备库的分发策略不依赖binlog内容、binlog格式可以不局限于row
通用性更有保证.

思考: 假设5.7.22版本的主库、单线程插入很多数据、过了3个小时、想给这个主库搭一个相同版本的备库. 为了更快的追上主库、需要怎么选择并行复制策略呢 ?
1. commit_order 由于是单线程执行、每个事务的commit_id不同、从库也只能是单线程执行
2、 writeset_order 要求同一个线程的日志必须要与主库上的先后顺序相同、也会导致退化为单线程
所以应该选择 writeset 并行复制策略.
