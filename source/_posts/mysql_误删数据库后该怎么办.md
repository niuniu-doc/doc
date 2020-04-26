---
title: 误删数据库后该怎么办
date: 2020-03-25 08:49:55
tags: MySQL
categories: MySQL
---



>  误删数据库后除了跑路、还能怎么办 ?



#### 使用delete误删数据行

可以使用Flashback工具把数据恢复. 原理是: 修改binlog的内容、拿回原库重放. 前提是: 确保 binlog_format=row 和 binlog_row_image=FULL.



##### 单行事务处理:

 insert: binlog event的类型 Write_rows event -> Delete_rows event

 delete: 反之

 update: 对调binlog位置即可



##### 多行事务:

(A)delete ... (B)insert ... (C)update ...

写回的顺序应该是: 

(revert C)update ... (revert B)delete ... (revert A)insert ...

但: 以上方案只能作为`万一`的紧急预案、平时应该尽可能的做到提前预防、建议:

 把sql_safe_updates 参数设置为 on、这样一来、若忘记在delete或者update时写where条件、或者where条件中不包含索引字段的话、SQL执行会报错

 代码上线前、必须经过SQL审计



**注意:** 

\1. delete 全表是很慢的、需要生成undo log、写 redo log、binlog、从性能角度来看、应该优先考虑 truncate table或者drop table.

\2. truncate/drop table, drop database,即时配置binlog_format=row、执行这3个命令、记录的还是statement





#### 使用drop table或者truncate table误删数据表

这种情况就要使用全量备份 + 增量日志的方式了, 必须要有定期的全量备份, 并实时备份binlog才行. 两者都具备时、恢复流程:

1. 取最近一次全量备份、假设是一天一备、上次备份是当天凌晨

2. 用备份恢复出一个临时库

3. 从日志备份里、取出0点之后的日志

4. 把这些日志、除了误删数据的语句之外、全部应用到临时库.



**注意:**

1. 为了加快数据恢复、若临时库有多个数据库、可在使用mysqlbinlog时、加上-database指定库.

2. 应用日志时、需要跳过误操作语句的binlog、

  非gtid: --stop-position执行到误操作之前的日志、--start-position再从它之后开始操作

  gtid: 假设误操作的gtid是gtid1, set gtid_next=gtid1;begin;commit; 将gtid1先加入gtid集合、之后按顺序执行binlog时、会自动跳过误操作的语句.



不过、即使这样、mysqlbinlog方法恢复数据还是不够快、因为:

1. mysqlbinlog不能只解析其中一个表的日志

2. mysqlbinlog解析出日志、应用日志的过程只能是单线程.

一种加速方法是: 在用备份恢复出临时实例之后、将这个临时库设为线上备库的从库、这样:

1. 在start slave之前、先通过执行 change replication filter replication_do_table = tb、就可以让临时表只同步误操作的表

2. 也可以用上并行复制、加速数据恢复





#### 使用drop database误删数据库

##### 搭建延迟备库:

虽然可以通过并行复制来加速恢复数据、但恢复时间依然不可控, 若一个库的备份特别大、或者距离上一个全量备份的时间较长、恢复时间就特别长、若核心业务是无法等待这么久的、可以考虑搭建延迟复制的备库(MySQL 5.6引入)



change master to master_delay=n, 让备库与主库有Ns的延迟.

eg. 设为3600s、代表若主库数据被删、且在1h内发现、其它备库已同时被删除时、可到延迟复制备库执行 stop slave、跳过误删除目录、恢复出需要的数据.



##### 预防误删库的方法:

1. 账号分离、避免写错命令

2. 指定操作规范. 避免写错要删除的表名. 先重命名待删除表、业务无影响再操作删除



#### 使用rm命令、误删数据库实例

对一个有HA机制的Mysql集群、只要不恶意删除整个集群、只删除单节点的话、HA系统会开始工作、选出一个新的主库、保证集群正常工作. 这时、需要把节点数据恢复、再接入整个集群.



建议: 尽量把备份跨机房、最好跨城市保存.