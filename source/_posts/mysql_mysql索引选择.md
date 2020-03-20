---
title: mysql索引选择
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
#### 普通索引和唯一索引

##### 场景：
```
假设维护一个市民系统、每个人有一个唯一的身份证号、经常会根据身份证号查询、sql如下:
select name from user where id_card='xxx';
一定会考虑在id_card上建立索引、那么、这个索引应该是唯一索引还是普通索引呢 ？
```
##### 思考:
> id_card字段比较大、不建议作为主键、那么、1.普通索引 2.唯一索引 如何选择？依据又是什么呢 ？


![mysql索引结构组织树.png](https://upload-images.jianshu.io/upload_images/14027542-a869a323d23b3991.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

##### 分析：
**查询过程**
```
假设执行的语句为 select t from T where k=5;
搜索从根开始、按层搜索到叶子节点、可以认为数据页内部通过二分法来定位
1. 对于普通索引、查找到第一条记录(5, 500) 之后、需要查找下一个记录、直到碰到第一个不满足k=5的记录
2. 对于唯一索引】由于定义了唯一性、查找到(5, 500) 之后、停止检索、直接返回
那么带来的性能差异呢 ？ - 微乎其微

innodb的数据是按照数据页为单位来读写的、即: 当需要读一条记录的时候、不是讲记录本身从磁盘读出、而是以页为单位、
将整个数据页读取到内存、数据页大小默认为16k

因为是按页读取、当找到k=5时、它所在的数据页已经在内存了、对于普通索引来说、多做的一次'查找和判断下一条记录'只需要一次指针查找和一次计算
不幸的是、恰好l=5是数据页的最后一条记录呢 ？....
必须读取下一个数据页, 对于整型字段、16k可以放近千个key、出现这种情况的概率很低、所以计算平均性能差异时、可认为这个操作成本对于现在的CPU来说可以忽略不计

```

**更新过程**
##### change_buffer
```
先说下change_buffer的概念：
更新数据页时，若数据页在内存中 -> 直接更新，
不在内存 -> 在不影响一致性读的情况下、会讲更新操作缓存在change_buffer
这样就不需要从磁盘读入数据页, 下次查询需要访问这个数据页时、将数据页读到内存、与change_buffer合并(称为merge)

虽然名字叫change_buffer、实际可持久化、在内存中有copy、也会被写入磁盘

触发merge:
1. 访问数据页
2. 后台线程定期merge
3. db正常关闭、shutdown
4. 达到change_buffer的可用最大内存、触发merge、然后淘汰

显然、如果更新操作先记录在change_buffer、减少读磁盘、可以提高sql执行效率
而且、数据读入内存、是需要占用buffer_pool的、这种方式还可以避免占用内存、提高内存使用效率

那么什么时候可以使用buffer_pool?
1. change_buffer 占用 buffer_pool 的内存、通过 innodb_change_buffer_max_size 调整、设为50、
   表示change_buffer 最多占用buffer_pool的50%
2. 不能无限增大、不能 > buffer_pool
3. 对唯一索引来说、所有的更新都要判断操作是否违反唯一约束、
   eg. 要插入(4, 400)这个记录、必须先判断记录是否存在、必须先将数据页读入内存
   若已读入内存、直接更新内存更快、无需使用 change_buffer、
   实际上、唯一索引的更新也不能使用change_buffer、只有普通索引可用
```

##### 理解了 change_buffer、看下插入(4，400)
```
1. 更新的记录目标页在内存 - 无差别(只有一个判断的差别)
   1) 唯一索引: 找到3、5之间位置、判断无冲突、直接插入
   2) 普通索引: 找到3、5之间、直接插入

2. 更新记录目标页不在内存
   1) 唯一索引: 将数据页读入内存、判断无冲突、插入
   2) 普通索引: 将记录更新在change_buffer、执行结束
   将数据从磁盘读到内存、涉及io随机访问、change_buffer减少了随意访问磁盘、性能会明显提升
   
3. 案例：
   业务库的内存命中率突然下降、整个系统处于阻塞状态、更新全部阻塞、
   深入排查后发现: 业务有大量的插入操作、而、前一天晚上上、将普通索引改成了唯一索引、使写操作的效率下降
```

##### change_buffer使用场景
```
change_buffer对于查询无明显效果、也只适用于普通索引、那么 普通索引的所有场景、change_buffer都能起到加速作用么 ？

1. merge 的时候是真正数据更新的时刻、change_buffer主要是将记录变更的动作缓存、so.merge前变更越多、收益越大
   对于写多、读少的业务、change_buffer的效果最好、常见业务模型: 账单类、日志类系统
   
2. 若业务场景是、写完立马会有查询、由于立马访问数据页、会立即触发merge、随机访问的io次数不会减少、
  反而增加了change_buffer的维护代价、不适合使用
```

##### 索引选择和实践
```
1. 若所有更新后跟着记录的查询、应关闭change_buffer
2. 若有一个机械硬盘的历史库、应尽量使用普通索引、调大change_buffer、确保历史数据的写入效率
```

##### change_buffer 和 redo log
```
假设执行更新
insert into t(id, k) values(id1, k1), (id2, k2)
当前k索引树的状态、k1所在的数据页在内存(innodb buffer pool)中、
```

![带change_buffer的数据更新.png](https://upload-images.jianshu.io/upload_images/14027542-0d07c7e2f5c7640c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)

```
涉及: 内存、redolog(ib_log_fileX)、数据表空间(t.ibd)、系统表空间(ibdata1) 四个部分
更新做了如下操作:
1. page1在内存、直接更新内存 (图中1)
2. page2不在内存、在内存的change_buffer区域、记录下 '往page2插入1行' 这个信息 (图中2)
3. 将1、2两个动作写入redo log(图中3、4)

所以、这条更新写了两处内存、一次磁盘(两次操作合写一次磁盘)还是顺序写、

图中两个虚线、是后台操作、不影响更新的响应时间
```

**读请求**
![带change_buffer的读.png](https://upload-images.jianshu.io/upload_images/14027542-ae14a2d3a50f1d8c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)
```
select * from t where k in (k1, k2)

1. 假设、读发生在更新不久、内存中数据还在、此时与系统表空间和redolog无关、直接读即可
2. 读page1时、从内存返回、读page2时、需要把page2从磁盘读入内存、然后应用change buffer的操作日志、合并数据

so. 简单对比这两个机制在提升更新性能上的收益的话、
redo log节省的是随机写 IO 的消耗、转为顺序写
change buffer节省的是随机读io的消耗
```

Q: change_buffer一开始是写内存的、此时若掉电重启、会导致change_buffer丢失么 ？ 若丢失、从磁盘读入时、就丢了merge、相当于丢了数据...

A:
```
虽然只是更新内存、但: 事务提交时、把change buffer的操作记录到了 redo log. 所以恢复时、change buffer可以找回

1.change buffer有一部分在内存有一部分在ibdata.
做purge操作,应该就会把change buffer里相应的数据持久化到ibdata
2.redo log里记录了数据页的修改以及change buffer新写入的信息
如果掉电,持久化的change buffer数据已经purge,不用恢复。主要分析没有持久化的数据
情况又分为以下几种:
(1)change buffer写入,redo log虽然做了fsync但未commit,binlog未fsync到磁盘,这部分数据丢失
(2)change buffer写入,redo log写入但没有commit,binlog以及fsync到磁盘,先从binlog恢复redo log,再从redo log恢复change buffer
(3)change buffer写入,redo log和binlog都已经fsync.那么直接从redo log里恢复


merge流程：
1. 从磁盘读入数据页到内存
2. 从change buffer找出这个数据页的change buffer记录(可能是多个)、依次应用、得到新版数据页
3. 写redo log. 这个redo log包含了数据的变更和change buffer的变更
merge流程完成、哈哈、此时 数据页和内存中change buffer对应的磁盘位置都还没修改、属于脏页、之后各自刷回自己的物理数据 -> 另外一个流程
```

#####
```
内存命中率: ib_bp_hit=1000 – (t2.iReads – t1.iReads)/(t2.iReadRequest – t1.iReadRequest)*1000

change buffer相当于推迟更新、对于MVCC是否有影响?比如加锁？
锁是单独的数据结构、若数据页上有锁、change buffer在判断能否使用时、会认为否

change buffer中、有此行记录的条件下、再次修改、是增加还是原地修改?
增加


```
