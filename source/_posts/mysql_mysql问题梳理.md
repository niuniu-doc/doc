---
title: mysql问题梳理
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---

```
1. 备份一般都会在备库上执行，在用–single-transaction 方法做逻辑备份的过程中
   若主库的一个小表做了DDL、eg 添加一列
   在从库会看到什么样的现象 ？
   
   Q1:SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
   Q2:START TRANSACTION  WITH CONSISTENT SNAPSHOT；
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

   Q1、确保事务隔离级别为可重复读
   Q2、确保得到一个一致性视图
   Q3、设置一个保存点、Q6 回滚到这个保存点
   
   若DDL语句在Q4之前到达、无影响、备份表结构为修改后的表结构
   若在Q5之前到达、则insert和create的结构不一致、会提示 Table definition has changed, 
   retry transaction、mysqldump终止
   若在时刻2和3之间到达、会造成主从延时 
   若在Q6之前到达、不影响、备份结构是修改前的表结构
   

2. 删除表数据的比较好的方式？
   1) delete from T limit 100000；
   2）delete from T limit 5000；执行20次
   3）分20个线程去执行
   1会造成大事务、单语句占用时间长、锁的时间比较长、可能造成主从延迟
   3可能会人为造成锁冲突
   2相对比较好一些

3. 唯一索引和普通索引如何选择？

4. change buffer 一开始是写内存的、然后写磁盘
   假如刚写入内存就断电重启、会不会导致change buffer丢失 ？
   不会丢失、虽然只是更新内存、但在事务提交的时候、把change buffer的操作也记录在了redo log里、崩溃恢复的时候可以找回
   
   
5. 扫描行数如何判断 ？
   mysql在真正执行前并不能精确的知道满足条件的记录有多少条、只能根据统计信息来预估
   这个统计信息就是索引的区分度、称为 索引基数(cardinality)、也就是索引基数越大、索引的区分度越好
   show index from tt; 
   可以看到索引信息
   
6. mysql是如何得到索引基数的呢 ？
   将整张表取出来一行行统计 - 可以得到精确值、单代价太高
   所以采用 采样统计、当表数据持续变更的时候、变更的数据行超过 1/M 时、自动触发索引重新统计
   show variables like 'innodb_stats_persistent'; 查看存储索引统计方式
   on 代表统计信息会持久化存储、默认 N=20 M=10
   off 的时候、代表统计信息只存储在内存中， 默认 N=8 M=16
   由于是采样统计、不管N是多少、基数都很容易不准
   索引统计值虽然不够精准、大体上还是差不多的、选错索引还有别的原因
   
7. 如何修正索引？
   1 使用force index 强制选择一个索引
   2 使用语句引导索引 order by b, a
   3 新建更合适的索引或者删掉无用索引

8. 何时会出现mysql抖动、平时很快执行的sql在某些瞬间执行很慢？
   1. redo log log大小超过配置值时、需要刷新到磁盘、此时系统会停止所有的更新操作、应尽可能的避免
   2. 系统内存不足的时候、当需要新的内存页、内存不够用的时候、就淘汰一些数据页、给别的数据页使用、
      若淘汰的是脏页、就需要将脏页写到磁盘
      innodb使用缓冲池 buffer pool来管理内存、缓冲池中的内存页有3种状态
      1）暂未使用的  2）使用了、并且是干净页(内存数据页和磁盘数据页内容一致) 3）使用了、并且是脏页
      innodb的策略是尽量使用内存、对一个长时间运行的库来说、未被使用的页面很少
      当要读入的数据页没有在内存的时候、要到缓冲池申请一个数据页、只能把最久不使用的数据页从内存淘汰
      若淘汰掉的是一个干净页、就直接释放出来复用、若淘汰的是脏页、就先刷新到磁盘
      
      所以：
      1. 一个查询要淘汰的脏页个数过多的时候、会导致查询时间明显变长
      2. 日志写满会将更新全部堵塞、写性能跌为0、应尽可能的避免
      
      那么innodb是如何控制刷脏页的策略的呢 ？
      一个是系统io能力、可以使用  fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest 来测试
      一个是脏页比例上限：show variables like 'innodb_max_dirty_pages_pct'; 默认75%
      
      另外：mysql在刷脏页的时候、若 innodb_flush_neighbors = 1 恰好连着的页也是脏页
      会被一起刷掉、若 情况不妙、连着的连着页也是脏页。。。。会导致你的查询更慢
      这个策略在机械硬盘时代是有意义的、可以减少io、但SSD设备的话、可以设置为0 减少sql执行时间
      8.0 已默认为0
      
   3. mysql认为系统空闲的时候、
   4. mysql正常关闭的时候

9. 为什么数据删除、表空间确没有释放
   数据删除流程：假如删除 2、10之间的6记录、它只被标记删除、并不会之间删除
   之后需要insert 1~10之间的数据时、会复用这个位置
   若删掉了一个数据页上的所有记录、整个数据页可以被直接复用
   记录的复用 只限于符合范围条件的数据、
   页的复用、可以用于任何数据
   若相邻页利用率都很小、系统会把这两个页合并到其中一个数据页、另外一个标记为可复用
   这些可以被复用、而未被复用的空间、称为空洞
   
   不止delete会造成空洞、insert也会、随机插入会造成索引的数据页分裂
   所以、大量增删改的表是可能存在空洞的、若能去掉空洞、就可以达到收缩表空间的目的
   1. 重建表 可以新建一个与表A结构相同的表B、按照主键ID递增的顺序、把数据行一行一行的从表A读出、再插入到B
      若将B作为临时表、数据从A导入B的操作完成后、用B替换A、就起到了收缩表A空间的作用
      可以使用 alter table A engine=innodb 来重建、
      执行流程类似、只是不需要自己创建临时表(是在server完成的、需要数据导入)、mysql自己完成转存数据、交换表名、删除旧表
      
      显然整个过程最耗时的是、插入数据、在整个DDL过程、A表不能有更新、非online
      
      mysql 5.6 版本引入 online ddl
      流程：新建临时表、扫描A主键的所有数据页、
           使用数据页中的A的记录生成B+树、存储到临时文件(存储引擎创建、整个DDL过程在innodb内部完成、无需数据挪到临时表、是原地操作)
           生成临时文件的过程中、将所有对A的操作记录在一个日志文件中、row log、
           临时文件生成后、将日志文件中的操作应用到临时文件、得到一个逻辑数据上与表A相同的数据文件、
           用临时文件替换表A的数据文件
      
      iplace和online是同一概念 ？？
      no、只是ddl过程若是 online 的、一定是 inplace 的
      若、inplace的DDL、可能不是online的、
      eg 给一个innodb的表字段加fulltext索引、整个过程是inplace的、会阻塞 增删改查操作、不是online的
      
            
10. alter table t engine=innodb 其实是 recreate 重建表的过程、非online
    analyze table t 不是重建表、不修改数据、只是对表的索引信息进行重新统计、会加MDL 读锁
    optimize table t => recreate + analyze
   
   
11. 若一个高配的机器、redo log设置的太小、会发生什么 ？
    每次事务提交都有些redo log、若设置太小、会被很快写满、系统不得不停止所有更新、去刷新redo log
    同时、change buffer的优化作用也会失效、因为checkpoint一直往前推进、会触发merge操作、然后又进一步的触发脏页刷新
    看到的现象会是 磁盘压力小、但是数据库出现间歇性的性能下降
    
12. count(*) 是怎么工作的 ？
    mysiam将数据总行记录在了磁盘上、所以查询 count(*) 会直接返回
    innodb 因为考虑事务的原因、不能这么记录、是从记录一条条遍历出来的、但是会遍历最小的那颗树
    
    so：
    mysiam count(*) 快、但是不支持事务、
    show table status 返回快、但不准确
    count(*) 会遍历全表、导致性能问题
    
    那么如何自己实现计数？
    1）使用缓存系统 - 数据易丢失、统计不精准(先写redis、记录未插入、则多1、先写db redis未写入 则少1)
    2）使用统计表 使用事务保存
    
    count(*)、count(id)、count(1) 都标识返回满足条件的结果集的总行数
    count(field)表示 filed不为null的总个数
    
    count(id) innodb会遍历整张表、取出每行的id值、返回给server层、server拿到id后、判断不为空、累计行
    count(1) inndb会遍历整张表、但不取值、server对于返回的每一行、放数字1进去、判断不为空、按行累加
    所以、count(1)比count(id)快、因为从引擎返回id会涉及到数据行解析、及copy字段值得操作
    
    count(field) not null字段、独处记录、判断不能为null、按行累加
    允许为null的字段、执行的时候判断到有可能为null、会先取出值、判断、不为null才累加
    
    count(*) 不会取出全部字段、专门做了优化、不取值
    所以 效率： count(field) < count(主键) < count(1) ~ count(*)    
    
13. mysql 怎么知道binlog是完整的 ？
    一个事务的binlog是有完整格式的、statement格式的binlog最后会有commit
    raw格式的binlog、最后会有一个 xid
    5.6之后的版本、还引入了 binlog-checksum 来保证内容的正确性
    
14. redo log 和 binlog 如何关联：
    有一个共同的数据段、xid、崩溃修复的时候会按顺序扫描 redo log
    若既有prepare又有commit、则直接提交
    若只有prepare 则拿 xid去找binlog 对应的事务
    
    处于prepare阶段的redo log + 完整binlog 重启就能恢复、why ？
    binlog写完之后、mysql崩溃、此时binlog已经写入、就会被从库使用
    所以在主库上也要提交这个事务、保证主从数据的一致性
    
    为何不是先写 redo log再写binlog、然后直接两个log完整时才恢复数据 ？
    对于innodb来说、redo log提交完成、事务就不能回滚(会覆盖到别的事务的更新)、
    而如果redo log 直接提交、然后binlog失败的时候、inndo回滚不了、数据和binlog就又不一致了
    两阶段提交就是为了给所有人一个机会、啊哈~
    
15. 对于正常运行的实例、数据写入后的最终落盘、是从redo log更新的还是buffer pool ？
    redo log 并未记录数据页的完整数据、所以、没有能力自己去更新磁盘数据页、不存在redo log更新进去的情况
    a.若数据页被修改之后、跟磁盘的数据页不一致、称为脏页、最终数据落盘就是把内存中的数据页写盘
    b.若在崩溃恢复的场景中、innodb判断到一个数据页可能在崩溃中丢失了更新、就将其读到内存、
    然后让redo log更新内存内容、回到 a 的情况
    
   
   fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest 
 
16. order by 是如何工作的 ？
    explain 中的 using filesort 代表需要排序
    mysql 会给每个线程分配一块内存用于排序、称为 sort_buffer
    
    1) 初始化sort_buffer 确定放入 name city age 三个字段、
    2) 从索引 city 找到 第一个满足 city=xx 条件的主键id
    3) 从主键id索引取出整行、取name city age 三个字段的值、写入 sort_buffer 中
    4) 从索引city取下一条满足条件的记录id
    5) 重复3、4 直到无满足条件的记录
    6) 对sort buffer中的数据按照name进行快排
    7) 取结果集前1000行返回给client
    
    若单行数据超过 max_length_for_sort_data 的值、则会先排序、再回表取数据
    1) 初始化sort_buffer 确定放入 name city age 三个字段、
    2) 从索引 city 找到 第一个满足 city=xx 条件的主键id
    3) 从主键id索引取出整行、取name & id 两个个字段的值、写入 sort_buffer 中
    4) 从索引city取下一条满足条件的记录id
    5) 重复3、4 直到无满足条件的记录
    6) 对sort buffer中的数据按照name进行快排
    7) 取结果集前1000行、回原表扫描city、name和age三个字段值 返回给client  
    
    此时假如 name本身有索引、天然有序、则不需要额外排序、会变成
    1) 从索引 city name 找到满足 city=杭州的 条件的主键
    2) 回主键id索引查出正行、取name、city、age的值 作为结果集的一部分直接返回
    3) 从索引 name city取出下一个满足记录的主键id
    4) 重复2、3、直到足够1000条记录、或者无满足条件的记录
    
    
    
    使用 optimizer_trace 进行sql追踪、一般情况下、一个新的跟踪会覆盖掉以前的跟踪
    
    
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
    
    

   
   
   CREATE TABLE `tt` (
     `id` int(11) NOT NULL,
     `a` int(11) DEFAULT NULL,
     `b` int(11) DEFAULT NULL,
     PRIMARY KEY (`id`),
     KEY `a` (`a`),
     KEY `b` (`b`)
   ) ENGINE=InnoDB；
   
   delimiter ;;
   create procedure idata()
   begin
     declare i int;
     set i=1;
     while(i<=100000)do
       insert into tt values(i, i, i);
       set i=i+1;
     end while;
   end;;
   delimiter;
   call idata();

    select * from t where a between 10000 and 20000;
    
    
    set long_query_time=0;
    select * from t where a between 10000 and 20000; /*Q1*/
    select * from t force index(a) where a between 10000 and 20000;/*Q2*/


SELECT IFNULL(driver_id, 0) driverId, count(*) serviceCnt, MAX(booking_start_time) lastBookingDate FROM car_order_finished_collect_2018 WHERE driver_id IS NOT NULL  AND order_status > 43 AND order_status < 60 AND booking_user_id = 1148735 AND driver_id in (25311)  GROUP BY driver_id ORDER BY COUNT(*) DESC, MAX(booking_start_time) DESC\G;
SELECT IFNULL(driver_id, 0) driverId, count(*) serviceCnt, MAX(booking_start_time) lastBookingDate FROM car_order_finished_collect_2018 WHERE driver_id IS NOT NULL  AND order_status > 43 AND order_status < 60 AND booking_user_id = 80004692 AND driver_id in (10101406)  GROUP BY driver_id ORDER BY COUNT(*) DESC, MAX(booking_start_time) DESC\G;
                                     
```
