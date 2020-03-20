---
title: order-by是怎么工作的
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
#### 场景
```
假设有一张市民表、需要查询 城市是 杭州 的所有人的名字、以name排序、返回前1000个
```
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;

#查询sql
select city,name,age from t where city='杭州' order by name limit 1000  ;
```

![全字段排序.png](https://upload-images.jianshu.io/upload_images/14027542-c6445f922b6b14ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/14027542-ad05011df1ab8624.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 全字段排序
```
using filesort 表示需要排序、mysql会给每个线程分配一块内存用于排序、sort_buffer

sql执行流程:
1. 初始化sort_buffer、确定放入name、city、age字段
2. 从索引city找到第一个满足 city='杭州' 条件的主键id、也就是图中的ID_X
3. 到主键ID索引取出整行、取name、city、age的值 放入sort_buffer、
4. 从索引city找到下一个满足条件的记录、
5. 重复3、4直到无满足条件的记录、对应的主单id就是图中的ID_Y
6. 对sort_buffer中的数据、按name做快排
7. 按排序结果取前1000行给client

按name排序可以在内存完成、或者靠外部排序，取决于排序所需内存和sort_buffer_size
```

```
/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a 保存 Innodb_rows_read 的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b 保存 Innodb_rows_read 的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算 Innodb_rows_read 差值 */
select @b-@a;

其中number_of_files 表示的是使用的临时文件的个数、需要外部临时文件排序时、mysql会分散到多个文件中进行排序、然后进行归并排序、
examined_rows 表示参与排序的行数
```

#### rowid排序
```
单行数据较大时(超过 max_length_for_sort_data)、sort_buffer中存放的字段过多、内存能同时放下的行较少、需要很多临时文件、排序性能会特别差、innodb会采用另外一种排序、只在sort_buffer放入排序字段和id、然后流程变为:
1. 初始化sort_buffer、放入name和id
2. 从索引city找到第一个满足 city='杭州' 条件的主键id、也就是图中的ID_X
3. 回到主键索引取出整行、取id、name这两个字段、放入sort_buffer
4. 从索引city取下一条满足条件的记录、
5. 重复3、4直到无满足条件的记录
6. 对sort_buffer中的记录按照name排序、取前1000行
7. 回id索引取出所有满足条件的记录的name、age、city返回client

SET max_length_for_sort_data = 16;

```

#### 结论
```
对于innodb表、会优先选择内存排序、内存不足时、会优先选择全字段排序
```

那么对于内存表呢 ？
```
回表只是简单的根据行的位置得到数据、所以排序行会越小越好、优先选择rowid排序
```

`tmp_table_size` 内存临时表的大小、默认16M


Q: 假设表中已有city name的联合索引、sql如下: 
```sql
mysql> select * from t where city in ('杭州'," 苏州 ") order by name limit 100;
```
这个sql执行需要排序吗
A: 
```
对于不同city、name不是有序的、需要排序
可以将结果拿到业务层去排  
```

**总结**
```
tmp_table_size 会影响是否使用内存临时表(否则使用磁盘临时表)
sort_buffer_size 使用使用内存完成排序(否则需要文件排序)
max_length_for_sort_data 会影响排序策略(全字段/rowid)
内存排序时、>5.6版本会使用优先队列排序
```
