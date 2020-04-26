---
title: mysql索引
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---

#### 常见的索引模型
```
1. hash 一种key-value结构、k-v映射、写入、等值查询很快、但范围查询比较慢、但适用于 等值查询的场景
2. 有序数组在等值查询和范围查询的性能表现上都很优秀
   但在插入时必须要挪动后续所有记录、成本太高、
   所以只适用于静态存储. eg. 城市信息表这种不经常变动的数据
3. 树 在查询和写入的效率上都还不错
```
![hash表示意图.png](https://upload-images.jianshu.io/upload_images/14027542-378ce5505cc95e86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### innodb的索引模型
> 在innodb中、表都是根据主键顺序、以索引的形式存放的, 这种存储方式的表称为索引组织表
innodb使用了B+树索引模型，数据都是存储在B+树的、每个索引对应一颗B+树


```sql
create table T-inx(
id int primary key ,
k int not null ,
name varchar(16) ,
index (k)
)engine=innodb
```
![i索引树示意图.png](https://upload-images.jianshu.io/upload_images/14027542-4047d1069d4fce5c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
从图中可以看出、索引树分为主键索引和非主键索引两种类型、
主键索引存储的是整行数据、在innodb里, 主键索引也被称为聚簇索引(clustered index).
非主键索引的叶子节点内容是主键值, 也称为二级索引(secondary index).

```

Q: 那么基于主键索引 和 基于普通索引的查询有什么区别 ?
```
若 sql 是 select * from T where ID=500; 即主键查询的方式、则只需要搜索ID这颗 B+ 树
若 sql 是 select * from T where k=5, 则需要先搜索k这颗索引树、得到id的值为500, 再到ID索引树搜索一次、这个过程称为 回表

也就是: 基于非主键索引的查询需要多扫描一颗索引树, 所以需要尽量的使用主键查询
```

Q: 索引维护
```
B+ 树为了维护索引有序性、在插入新值的时候需要做必要的维护.
若插入的新值较大、只需要再最后插入新的记录、若插入的值为中间值、则相对麻烦,需要先挪动后边的数据、腾出位置
更糟糕的是: 若最后一个数据所在的数据页已经满了、根据B+树的算法、需要申请一个新的数据页、然后挪动部分数据过去
这个过程称为 页分裂, 除了性能外、页分裂还影响数据页的利用率

当相邻页有数据删除之后、由于数据页的利用率很低、会进行页合并

场景: 自增主键的作用？
插入新的记录时、系统会获取当前最大值+1 作为下一条记录的id
即: 自增主键的插入数据模式、正好符合了递增插入、不涉及挪动其它记录、也不会触发叶子节点的分裂
而有业务逻辑的字段做主键、则往往不能保证有序写入、这样写数据的成本相对高

从存储上看、主键长度越小、普通索引的叶子节点就越小、索引占用的空间也就越小

so. 从性能和存储上来看、自增主键往往是最合理的选择
```


表初始化语句
```sql
create table T-index(
ID int primary key, 
k int not null default '',
index k(k)
)engine=Innodb;

insert into T-index values (100, 1, 'aa'), (200, 2, 'bb'), (300, 3, 'cc'), , (500, 5, 'ee'), (600, 6,'ff'),(700,7,'gg');
                                                                                                     
```

Q: 执行sql`select * from T-index where k between 3 and 5`需要执行几次树的搜搜操作、会扫描多少行 ？
A: 执行流程:
```
1. 在 k 索引树上找到 k=3 的记录、取得id=300;
2. 再到 id 索引树上查找 id=300 对应的记录R3;
3. 在 k 索引树上找到 下一个值 k=5, 取得 id=500;
4. 再回到 id 索引树上找到 id=500 对应的记录 R4;
5. 在k索引树上取下一个值 k=6, 不满足条件、循环两次.

在步骤 2、4中、回到主键索引查找数据的过程、称为回表
```
所以、一共执行了 `3次`搜索、`2次`回表、扫描`3行`

#### 覆盖索引
> 若执行的语句是
select ID from T-index where k between 3 and 5;
此时只需要查询 ID 的值、而ID的值已经在 K 索引树上了、可以直接提供查询结果、
无需回表、即: 在本次查询中、索引 k 已经覆盖了查询需求、
称为覆盖索引

* 在引擎内部使用覆盖索引在索引 k 上其实读了3个记录、R3~R5、
但是对于 MySQL的server来说、只从引擎拿到了2条记录、认为扫描行是2

so. 覆盖索引在一定程度上可以减少回表扫描的次数


Q: 在一个市民表上、是否有必要将身份证和名字建立联合索引 ?
```sql
create  table `user` (
`id` int(11) not null ,
`id_card` varchar(32) default null ,
`name` varchar(32) default null ,
`age` int(11) default null , 
`ismale` tinyint(1) default null ,
PRIMARY key (`id`) ,
key `id_card` (`id_card`),
key `name_age` (`name`, `age`)
) engine=Innodb
```
A: 
```
这种情况需要看实际的业务场景:
1. 若经常的查询需求是: 
   根据身份证号查询市民信息、则 只需要在身份证字段上建立索引, 再建(id,name)的联合索引就会浪费空间
   
2. 若经常的查询需求是:
   根据身份证号查询name, 则建立联合索引就有很大的意义、它可以在这个高频请求上用到覆盖索引, 
   不在需要回表查询， 提高查询速度

```

Q: 如果现在有一个非高频请求、根据身份证号查询家庭地址, 需要再设计一个联合索引么 ?
A: 
```
索引项是按照索引出现的字段排序的:
eg. 利用(name, age)的联合索引查找所有名字是 zhangsan 的人时, 可快速定位到 ID4, 然后向后遍历得到所有结果
    不只是索引的全部定义. 只要满足最左前缀原则、就可以利用索引来加速检索
所以: 基于已经建立了 (id_card, name) 的联合索引、无需再建立 (id_card, addr)的联合索引、
利用最左前缀的原则、它可以使用 (id_card, name) 的联合索引
```

#### 索引下推

思考: 
以(name, age)联合索引为例、需要检索所有 **名字第一个字为张,且年龄为10的男孩**
SQL如下:
```sql
select * from user where name like `张%` and age=10 and ismale=1;
```
拿到记录行之后、还要进一步判断其它条件是否满足、这个怎么处理的呢?

```
1. 在MySQL 5.6 之前、只能从 ID3 开始、一个个回表. 在主键索引上找到数据行、再对比字段值
2. 在5.6 引入了索引下推(index condition pushdown), 可以在索引遍历的过程中对索引包含的字段优先判断
   过滤掉不满足条件的记录、减少回表次数
   
   在无索引下推时、innodb不会看age的值、只是顺序把 `name`第一个字是`张`的记录取出、回表、需要回表4次
   有索引下推时、innodb在(name, age)内部就判断了age是否=10，不等于10的记录、直接判断并跳过
   只需要取回ID4、ID5两条记录、所以只需要回表2次
```
Q: 为什么需要重建索引？
```
索引因为删除、或者页分裂等原因、导致数据页有空洞、重建索引的过程会创建一个新的索引、把数据按顺序插入、这样页面的利用率最高、
也就是索引更紧凑、更省空间
```
Q: 
```
场景: DBA入职时、发现自己接手维护的库、有一个表定义：

create table `geek`(
`a` int(11) not null,
`b` int(11) not null,
`c` int(11) not null,
`d` int(11) not null,
primary key(`a`, `b`),
key `c`(`c`),
key `ca`(`c`, `a`),
key `cb`(`c`, `b`)
)engine=innodb;

```

> 为什么需要ca、cb这两个联合索引呢 ？同事的解释是: 因为业务常用下边的sql:

```sql
select * from geek where c=N order by a limit 1;
select * from geek where c=N order by b limit 1;
```
> 那么这种解释对么 ？
```
主键a、b的聚簇索引组织顺序相当于 order by a, b 即: 先按a排序、再按b排序、c无序
索引 ca的组织是先按c排序、再按a排序、同时记录主键b
索引 cb的组织是先按c排序、再按b排序、同时记录主键a
so. ca可以去掉、cb需要保留

```

Q: using where的时候、需要回表查询数据、然后把数据传输给server层、server来过滤数据、那么这些数据是存在哪儿的呢 ？
```
无需存储、就是一个临时表、读出来立马判断、然后扫描下一行是否可以复用
```

Q: limit起到限制扫描行数的作用、并且有using where的时候、limit这个操作在存储引擎层做的、还是在server层做的 ？
```
server层、接上面一个Q、读完以后、顺便判断一下limit够不够就可以了、够了就结束循环
```

Q: extra列线上 using index condition 代表使用了索引下推 ?
```
ICP代表可以下推、但 '可以、不一定是'
```

Q: 备库使用 --single-transaction 做逻辑备份的时候、若从主库的binlog传来一个DDL语句会如何?
A:备份关键语句:
```sql
Q1:SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
Q2:START TRANSACTION  WITH CONSISTENT SNAPSHOT;
/* other tables */
Q3:SAVEPOINT sp;
/* 时刻 1 */
Q4:show create table `t1`;
/* 时刻 2 */
Q5:SELECT * FROM `t1`;
/* 时刻 3 */
Q6:ROLLBACK TO SAVEPOINT sp;
/* 时刻 4 */
/* other tables */
```
```
在备份开始的时候、为了确保RR(可重复读)、再设置一次隔离级别Q1
启动事务、用 with consistent snapshot 来确保这个语句执行完可以得到一个一致性视图 Q2
设置保存点 Q3
show create 是为了那点表结构 Q4、正式导数据 Q5 回滚到savepoint ap是为了释放t1的MDL锁 Q6

DDL从主库传过来的时间不同、则影响不同
1. 若在Q4之前到达: 无影响、那点的是DDL后的表结构
2. 若在时刻2到达、表结构被修改过、Q5执行的时候、报 Table definition has changed. please retry transaction. mysqldump 终止
3. 在时刻2 和 3之间到达、mysqldump占着t1的MDL锁、binlog被阻塞、现象: 主从延迟、直到 Q6 完成.
4. 从时刻2开始、mysqldump释放了 MDL锁、现象: 无影响、备份拿到的是 DDL前 的表结构

```

Q: 
>删除100000行表记录时,有3种做法
1. 直接delete from T limit 100000;
2. 在一个连接中 delete from T limit 5000; loop 20
3. 在20个连接中 delete from T limit 5000; 20 connections;
如何选择?

A:
```
尽量的选择第二种方式
1. 单个语句占用的时间较长、锁的时间也会较长、而且打的事务也会造成主从延迟
3. 在20个连接中同时执行 delete from T limit 5000, 可能会造成认为锁冲突
```
