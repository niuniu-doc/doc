---
title: 表数据删了一半、表大小不变
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
> innodb表包含两部分: 表定义和表数据、在mysql8.0之前、表定义单独存放、.frm 文件、8.0之后允许把表结构定义放在系统数据表中了(表结构定义占用的空间很小)

#### innodb_file_per_table
```
off: 表数据放在系统共享表空间、跟数据字典放在一起
on: 每个innodb表数据存储在一个.ibd结尾的文件中(drop table时直接删除文件)
```

#### 数据删除流程
![索引树示意图](https://upload-images.jianshu.io/upload_images/14027542-9ef7fd91e2dcd722.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
1. 假设删掉R4这个记录、innodb引擎只会把这个记录标记为已删除、之后再插入一个300-600之间的记录时、可能会复用这个位置、但是空间不会释放
2. innodb的数据是按页存放的、如果删掉了一个数据页上的所有内容呢 ？
    整个页可以复用
3. 数据页的复用: 可以复用到任意位置
    记录复用: 插入数据若不在400-600之间、无法复用
4. 若相邻两个数据页的利用率都很小、系统会把数据合并到一个页】另一个标记为可复用
5. delete删除整个表数据呢？所有数据页都被标记为可复用
所以delete操作只是标记页面可复用、这些可复用但未被复用的空间、就像是空洞、空间不会回收

6. 插入数据也会造成空洞
    数据若不是顺序插入、如果需要在某个页的末尾插入一条记录、就会申请新的页、把要插入位置之后的数据转移到新的页去: 如下图
```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-dede264180fcc1f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

#### 如何去除空洞
```
1. 重建表 alter table A engine=Innodb
2. 5.6之后的版本、引入Online DDL、重建流程变为: 
    1. 建立一个临时文件、扫描表A主键的所有数据页、
    2. 用数据页中表A的记录生成B+树、存储到临时文件中、
    3. 生成临时文件过程中、所有表A的操作记录在日志文件(row log) 中、
    4. 临时文件生成后、将日志文件应用于临时文件、得到与表A逻辑相同的文件
    5. 用临时文件替换表A的数据文件
    与以前不同之处在于: 在重建表的过程中允许对表A的操作

```
![重建表流程.png](https://upload-images.jianshu.io/upload_images/14027542-92ae8e30bfdb3194.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Online DDL.png](https://upload-images.jianshu.io/upload_images/14027542-79acd12bb61357d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 完全Online ？
```
不是, DDL之前是要拿MDL锁的、但这个写锁会在copy数据之前退化成读锁、就不会阻塞CURD操作了
可以直接解锁？不行、为了保证自己在copy的过程中表结构不会再次被修改、为了保护自己
对大表来说、最耗时的是copy文件、可以认为MDL读锁的时间可以忽略、但操作会比较消耗IO和CPU、安全操作可以使用 gh-ost来做
```

#### online和inplace
```
图三中、表A导出数据存放位置叫tmp_table、是一个临时表、在server层创建
图四种、表A重建的数据放在tmp_file、是innodb内部创建出来的、整个DDL过程在innodb内部完成、对server来说、没有挪到数据到临时表、是原地操作、inplace

假如磁盘空间是1.2T、现有一个1T的表、可否完成一个inpalce的DDL？
不能、inplace也是需要临时空间的、alter table t engine=innodb,ALGORITHM=inplace;
copy表其实是: alter table t engine=innodb,ALGORITHM=copy; 对应图三

inplace和online并无直接关系
eg. 添加索引、是inplace操作、但不是online的
```

#### alter table、analyze table 、 optimize table
```
1. 5.6之后、默认上图四操作、就是recreate
2. analyze table不是重建表、是重新统计索引信息、过程会加MDL读锁
3. optimize = recreate+analyze
```
