---
title: 主库异常、从库怎么办
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
![一主多从基本架构.png](https://upload-images.jianshu.io/upload_images/14027542-d8b11878314944da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)
说明: 虚箭头表示主备关系、从库B、C、D指向主库A

>那么、当主库A发生故障时、A'会成为新的主库、具体怎么切换的呢 ?

一、基于位点的主备复制
当需要把节点B设置为A'的从库时、需要执行`change master`命令
```
change master to
master_host=$host_name
master_port=$port
master_user=$user_name
master_password=$password
master_log_file=$master_log_name
master_log_pos=$master_log_pos
```
其中, `master_log_file`和`master_log_pos`表示要从从库的`master_log_name`文件的`master_log_pos`位置开始同步, 即: 主库对应文件名和日志偏移量

1. 如何获取同步位点(偏移量)呢 ?
方法一：
```
1. 等待新主库`A'` 把中转日志 relay log全部同步完成
2. 在`A'`上执行 show master status命令、得到`A'`上最新的File和Position
3. 取原主库A故障的时刻T
4. 用mysqlbinlog工具解析`A'`的File、得到T时刻的位点

mysqlbinlog File --stop-datatime=T --start-datetime=T， 如下图所示 end_log_pos的值
```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-11317714a2bf6581.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
但这样得到的值并不精准、假设 T时刻、主库A完成insert操作、插入行R、将binlog传给了A' 和 B、传完的瞬间A故障, 此时系统状态为:
```
1. 从库B已经同步了binlog、R已存在
2. 在新的主库`A'`, R也存在、且是写在123这个位置后的、
3. 在从库B上执行 change master时、指向`A'`的123位置、R的binlog再次同步到从库B、造成主键冲突.
```
通常情况下会主动跳过错误, 有两种方式可以主动跳过:
1. 主动跳过一个事务
```sql
set global sql_slave_skip_counter=1;
start slave;
```
2. 跳过指定类型的错误
```sql
1062 insert唯一键冲突
1032 删除数据时、找不到行
slave_skip_errors='1032,1062'; 会主动跳过这两类的错误
```

方法二、GTID
通过sql_slave_skip_counter跳过事务和slave_skip_errors忽略错误的方法、虽然最终可以建立新的主备关系、但操作复杂、容易出错、Mysql5.6引入了`GTID(Global Transaction Identifier)`. 格式:`GTID=server_uuid:gno`
其中:
```
server_uuid 是实例第一次启动时生成的, 全局唯一
gno是一个整数、初始值是1, 每次提交事务时分配给这个事务、并+1
```
官方定义: `GTID=source_id:transaction_id`

GTID模式的启动很简单、只需要启动是增加参数: 
gtid_mode=on 和 enforce_gtid_consistency=on 即可.

GTID模式下、每个事务都会跟一个GTID对应、有两种生成方式, 具体使用哪种取决于 session 变量和 gtid_next 的值
```
1. 若 gtid_next=automatic, 代表使用默认值、Mysql会把server_uuid:gno 分配给它
   a. 记录binlog时、先记录一行set
       @@session.gtid_mext = 'server_uuid.gno'
   b. 把这个gtid加入到本实例的gtid集合 
2. 若gtid是固定的值、eg. 通过set gtid_next='current_gtid'指定、会有两种可能性:
   a. 若current_gtid已存在实例的gtid集合、接下来的事务会被系统忽略
   b. 若不再、就将该值分配给要执行的事务、系统不需要生成、gno也不用+1
```
这样、MySQL维护了一个GTID集合、用来对应`实例执行过的所有事务`

当出现主键冲突时、可以通过GTID加入的方式、跳过事务
```
set gtid_next='conflict gtid'; 
begin;
commit; // 通过提交一个空事务、将gtid加入到实例X的GTID集合
set gtid_next=automatic; // 恢复GTID自动分配模式
start slave; // 启动从库
```


二、基于GTID的主备切换
```sql
change master to
master_host=$host_name
master_port=$port
master_user=$user_name
master_password=$password
master_auto_position=1 // 表示主备关系使用的是gtid协议
```
主备切换逻辑:
```
1. 实例B指向主库`A'`, 基于主备协议建立连接
2. 实例B把 set_b 发送给主库 `A'`
3. 实例`A'`算出 set_a 与 set_b的差集、即: 所有存在于 set_a、但不存在于 set_b 的gtid集合、判断`A'`本地是否包含了这个差集需要的所有binlog事务
   a. 若不包含、表示 `A'` 已经把实例B需要的binlog删除掉、直接返回错误
   b. 若包含、`A'`从自己的binlog找到第一个不在 set_b 的事务、发给B
4. 从该事务开始、往后读文件、按顺序读取binlog发送给B执行.
```
其实、这里边包含了一个设计思想: 在基于GTID的主备关系里、系统认为只要建立主备关系、就必须要保证主库发给备库的日志是完整的、若实例B需要的日志已经不存在、`A'`就拒绝发送给B

这跟基于位点的主备协议不同. 基于位点的协议、是由备库决定的、备库指定哪个位点、主库就发送哪个位点、不做日志的完整性判断. GTID是备库自动判断的.


思考:
> 若在GTID模式下设置主从关系的时候、从库执行start slave命令后、主库发现需要的binlog已经被删除掉了、导致主备创建不成功. 怎么处理呢 ?

```
1. 若业务允许主从不一致、可以在主库上执行 `show global variables like gtid_purged`、得到主库已经删除的gtid集合、假设是 gtid_purged1, 然后先在从库执行 reset master, 再执行 set global gtid_purged=gtid_purged1; 最后执行 start slave, 就会从现存binlog开始同步.
2. 若需要主从数据一致的话、最好还是通过重新搭建从库来做
3. 若其它从库有全量binlog的话、可以把新的从库先接到全量binlog的从库、追上日志以后、若有需要、再切回主库
4. 若binlog有备份、可以先在从库应用缺失的binlog、然后再执行start slave.
```
