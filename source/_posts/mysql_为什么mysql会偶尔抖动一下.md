---
title: 为什么mysql会偶尔抖动一下
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---

#### 基本概念
> 脏页: 内存页跟磁盘数据页内容不一致的时候、称这个内存页为 脏页
> 干净页: 内存数据写入磁盘后、内存和磁盘上的数据页的内容一致、称为 干净页

**干净页和脏页都在内存**

```
不难想象、平时操作很快的sql、其实就是在写内存和日志、而mysql抖动的那一瞬间、可能就是在刷脏页
```

#### 什么场景下会引发数据库的flush过程呢 ？
```
1. innodb的redo log满了: 此时系统会停止所有更新、把checkpoint推进、redo log留出空间
   此时需要把原write pos和checkpoint之间的所有脏页都flush到磁盘上
2. 系统内存不足: 当需要新的内存页、而内存不够用的时候、就需要淘汰一些数据页、空出内存给别的数据页使用
   若淘汰的是脏页、则需要先将内容flush到磁盘
   为什么不直接淘汰内存页 ？ 
   若刷脏页、一定会写磁盘、就保证了每个数据页有两种状态
   1) 在内存里存在、内存肯定是正确的结果、直接返回
   2) 在内存里无数据、在磁盘数据文件上肯定是正确的结果、直接读入内存后返回、这样的效率最高 
3. mysql系统空闲的时候
4. mysql正常关闭的时候、mysql会把内存的脏页全部flush到磁盘、下次启动的时候、直接从磁盘读取、启动很快

其中：1、2会影响性能
   1. 会停止所有更新、性能会跌为0、应该尽可能的避免
   2. 是内存不够用、需要将脏页写到磁盘、其实是常态、innodb使用buffer pool来管理内存、缓存池中的内存页有3种状态
      1) 暂未使用的 2) 已用&干净页 3)已用&脏页
   innodb的策略是尽量使用内存、对一个长时间运行的db来说、未被使用的页面很少
   当需要读入的数据页不在内存的时候、就必须到缓存池申请一个数据页、只能把最久不使用的数据页从内存淘汰
   若淘汰的是干净页、直接释放出来复用、若淘汰的是脏页、需要先flush到磁盘
   
```

#### innodb刷脏页的控制策略
```
innodb_io_capacity 告诉innodb所在主机的io能力、innodb才知道全力刷脏页时、可以刷多块、
    建议设置为磁盘的iops、测试:  fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest 

innodb并不是一直全力脏页、毕竟还要服务用户请求、那么是如何控制的呢 ？
1. 脏页比例(innodb_max_dirty_pages_pct)  2.redo log写盘速度
要合理设置io能力、关注脏页比例、当前脏页比例是通过 .._dirty/.._total 计算的
innodb_buffer_pool_pages_dirty 当前脏页数量
innodb_buffer_pool_pages_total 总的页数量
 
innodb刷页策略: innodb_flush_neighbors 为1: 代表若刷脏页时、临近页也是脏页、则一并刷到磁盘
为0、则只刷本页、
1的策略在机械硬盘时期比较有用、可以减少很多随机io、现在iops都不是问题、设置0即可、减少sql响应时间
mysql8.0 默认为0

```

Q: 访问某条记录时、mysql是如何知道记录是否在内存中 ？
A: 
> 每个数据页都是有编号的、到内存中查看对应页的编号、若不在内存、去磁盘查找即可
