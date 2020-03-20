---
title: mysql常用命令
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---

#### lock

1. 查看事务隔离级别
   
> select @@transaction_isolation;
   
2. 设置全局事务(影响新的会话、不影响本会话)
   
> set global transaction isolation level read committed; 
   
3. 设置会话事务(影响本会话)
   
> set session transaction isolation level read committed; 
   
4. 查看mysql默认读取的 my.cnf 的命令

> mysql --help | grep 'my.cnf'

查看mysql是否使用了指定目录的 my.cnf 
> ps aux | grep 'my.cnf'

4. 查看mysqlbinlog
   
   > mysqlbinlog --no-defaults  /var/lib/mysql/mysqld-bin.000001 --start-position=2425
   
5. 查看binlog的位置
   > show variables like '%log_bin%';

 开启binlog
  ```
  server-id=1
  log_bin=ON
  log_bin_basename=/var/lib/mysql/mysql-bin
  log_bin_index=/var/lib/mysql/mysql-bin.index
  ```
   
   查看所有的binlog文件
   > show binary logs;
   
   查看指定binlog文件的内存
   > show binlog events in '{name}';
   
   当前日志的文件名和偏移位置
   > show master status;
   
   刷新日志文件
   > flush logs;
   
   查看死锁配置
   > show variables like '%deadlock%'
   
6. mysqlbinlog工具查看
   基于时间: 
   >mysqlbinlog --start-datetime='2019-05-19 13:00' --stop-datetime='2019-05-19 14:00'
   
   基于偏移量
   > mysqlbinlog --start-postion=107 --stop-position=1000 -d {db} {binlog}
   
   row格式文件的查看 添加`-vv`参数


7. 开启一致性视图
   
   > start transaction with consistent snapshot;
   
8. 查看innodb页大小
   > show global status like 'innodb_page_size';
   > show variables like 'innodb_page_size';
   
9. 查看表基本信息
   
  > select * from information_schema.tables where TABLE_NAME like 'car_order_finished_collect_2019';
  
10. 开启profile
   
>  show variables like 'profiling'; set profiling=1;
   
11. 显示所有的profile
   
   > show profiles;
   
12. 显示第n个profile的详情
   > show profile for query n;
   > show profile all for query n; 
   
13. 查看innodb redo log配置
   > show variables like '%innodb_log_file%';
   innodb_log_file_size 单个redo log大小、
   innodb_log_files_in_group redo log文件数量
   
14. 查看innodb io控制
   
   > show variables like 'innodb_io_capacity'
   
15. innodb脏页比例
   
   > show variables like 'innodb_max_dirty_pages_pct'
   
16. 计算innodb当前脏页比例
   >  show global status like 'Innodb_buffer_pool_pages_%';
   Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total
   
   
