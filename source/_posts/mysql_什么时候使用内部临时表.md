---
title: 什么时候使用内部临时表
date: 2020-03-25 09:30:23
tags: MySQL
categories: MySQL
---



#### union执行流程

```sql
create table t1(id int primary key, a int, b int, index(a));

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000) do
    insert into t1 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
```
然后执行
```sql
(select 1000 as f) union (select id from t1 order by id desc limit 2);
```
获取两个子查询结果的并集(非重复).

使用explain分析发现:
在做union的时候、使用了临时表(Using temporary)

执行流程如下:
1. 创建一个内存临时表、只有一个整型字段f、且f是主键字段
2. 执行第一个子查询、得到1000这个值、写入临时表
3. 执行第二个子查询:
   获取id=1000, 插入临时表、由于主键冲突、插入失败、取id=999.
4. 从临时表按行取出数据、返回结果、并清除临时表、结果中包含的两行数据 1000 和 999.

若union改成union all、无需去重、就不需要临时表了.

#### group by执行流程

```sql
select id%10 as m, count(*) as c from t1 group by m;
```

explain的extra字段、可以看到三个信息:
using index, 表示语句使用了覆盖索引、选择了索引a、不需要回表;
uusing temporary, 表示使用了临时表
using filesort, 表示需要排序.

SQL执行流程:
1. 创建内存临时表、表里有两个字段 m 和 c, 主键是 m;
2. 扫描表t1的索引a、依次取出叶子节点上id的值、计算 id%10的结果、记为x;
   若临时表没有主键为x的行、就插入一个记录(x,1);
   若表中有主键x的行、就将c的值+1;
3. 遍历完成后、根据字段m做排序、得到结果集返回给客户端.

若实际需求不需要排序、可以加上 order by null. 这样就会跳过排序阶段.

#### group by优化方法 -- 索引
不论是内存临时表还是磁盘临时表、group by逻辑都需要构造一个带唯一索引的表、执行代价较高.
group by为什么需要临时表呢 ? group by是统计不同值出现的次数, 但每一行的 id%100 的结果是无序的、所以需要一个临时表、记录统计结果. 如果保证 出现的数据是有序的呢 ?
就不需要额外排序了. 索引, 可以通过添加冗余字段、记录 id%10 的值、并在该列 添加索引来优化.

四、group by优化方法 -- 直接排序
若碰到无法使用加索引完成group by的逻辑、就只能老老实实排序, 此时 group by如何优化呢 ?
若明知group by 语句的结果集很大、依然要 先存到内存临时表、插入部分数据后、发现临时内存不够再转成磁盘临时表、这样倒不如直接使用磁盘临时表.

在group by中加入 sql_big_result 可以告诉优化器、结果集很大、直接使用磁盘临时表就好. 磁盘临时表使用的是 B+ 树存储、存储效率不如数组. 既然数据量大、那不如使用数组存储.
```sql
select sql_big_result id%100 as m, count(*) as c from t1 group by m;
```

执行流程如下:
1. 初始化 sort_buffer、确定放入一个整型字段m;
2. 扫描t1的索引a、依次取出id的值、将id%100 的值、存入sort_buffer;
3. 扫描完成后、对sort_buffer的字段m做排序(若sort_buffer不够用、就利用磁盘临时文件辅助排序)
4. 排序完成、得到有序数组.

那么、MySQL什么时候会使用内部临时表呢 ?
1. 若语句执行过程是一边读数据、一边直接得到结果、是不需要额外内存的、否则就需要额外内存存储中间结果.
2. join_buffer是无序数组、sort_buffer是有序数据、临时表是二维表结构.
3. 若执行逻辑需要用到二维表特性、就要优先考虑临时表.

#### 思考:
为什么同样是使用 order by null. 内存临时表得到的结果 0在最后一行、而使用磁盘临时表得到的结果、0在第一行 ?