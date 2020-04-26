---
title: 为什么还有kill不掉的语句
date: 2020-03-25 08:54:45
tags: MySQL
categories: MySQL
---



> 为什么还要kill不掉的语句 ?



```
Mysql 中有两个kill命令: 
kill query + 线程id; // 终止线程中正在执行的语句; 
kill connection + 线程id; // 断开线程的连接、里边若有语句郑州执行、也会停止执行
那么什么情况下、会出现: 使用了kill命令、未断开连接、show processlist显示的是killed状态的现象呢 ?
```



#### 收到kill以后、线程做什么 ?

kill 若直接终止、退出、会导致原来占有的锁无法释放、所以: kill 不会马上停止、而是告诉线程`可以执行停止的逻辑了`. 实际上执行`kull query thread_id_B`时、处理了两件事:

1. 将session B的运行状态改成 THD::KILL_QUERY(将killed赋值为THD::KILL_QUERY);

2. 给session B的执行线程发一个信号.



#### kill不掉的场景

##### 线程并发数不够

 innodb_thread_concurrency 不够用、线程未执行到判断线程逻辑的状态.

  此时command列会显示killed, 但实际上服务端该语句还在执行中. 为什么不会直接退出呢 ?

  1) 执行kill query时、

  在实现上、等行锁、使用的是pthread_cond_timedwait函数、这个等待状态可以被唤醒. 虽然线程状态已经被置为kill_query, 但在这个等待进入innodb的循环过程中、并未判断线程状态、所以不会进入终止逻辑

  2) kill connection

  将线程状态置为 kill_connection、关掉线程的网络连接、所以被kill的session收到断开连接的提示. 为什么command列会显示为killed呢 ?

  show processlist 有一个特别逻辑: 若线程状态为 kill_connection、就显示为 killed.

  所以、即使Client退出了、线程状态仍然是等待中、直到满足进入innodb的条件后 session C才继续执行、然后才有可能判断到线程状态已经变成了kill_query 或者 kil_connection状态、再进入终止逻辑.



##### 终止逻辑耗时长. 

  此时从processlist结果列上来看也是Killed、需要等终止逻辑完成、才算真正完成. 场景:

  1) 超大事务执行期间被Kill、此时回滚事务需要对事务执行期间生成的所有数据版本做回收操作、耗时很长

  2) 大查询回滚、若查询过程中生成了较大的临时文件、加上此时文件系统压力大、删除临时文件可能需要等待IO资源、导致耗时较长

  3) DDL命令执行到最后阶段、若被Kill、需要删除中间过程的临时文件、可能也要受到IO资源耗时久的影响.



##### 直接在客户端Ctrl+C是否可以终止线程呢 ?

  不可以. 客户端只能操作客户端线程、Client和Server通过网络交互、不可能操作Server线程.

  而由于MySQL是停等协议、所以、线程执行的语句还未返回的时候、再往连接发命令也是无效的. 实际上、Ctrl+C时、Mysql Client是另外启动一个连接、发送 kill query 命令.

  



#### 两个关于Client的误解

##### 若库里包含的表特别多、连接就会很慢. 

  实际上、每个Client和Server建立连接时、需要做的事情就是 TCP握手、用户校验、获取权限、与库里表的数量无关. 但: 使用默认参数连接时、Mysql Client会提供本地库名和表名补全功能. 为了实现这个功能、Client在连接成功后、需要多一些操作:

  1) show databases;

  2) 切到db1、执行show tables;

  3) 将两个命令的结果构建一个本地hash表.

  最耗时的是操作(3). 所以、我们感知到的连接过程慢、其实不是连接慢、也不是Server慢、而是Client慢. 若不使用 或者很少使用 自动补全的话、建议默认加 -A

  其实-quick(-q) 也可以跳过这个阶段. 但这个参数可能会降低Server性能. Mysql 发送请求后、接收Sever返回结果的方式有两种:

  1) 本地缓存: 在本地开一片内存、将结果缓存、若使用API开发、对应:mysql_store_result.

  2) 不缓存: 读一个处理一个、若使用API、对应: mysql_use_result.

  MySQL默认采用本地缓存、若加上-quick会采用第二种. 此时、若本地处理的较慢、就会导致服务端发送结果被阻塞、让服务端变慢.

  ##### 为什么叫 -quick呢 ?

  1) 可以跳过表名自动补全

  2) mysql_store_result需要申请本地内存来缓存查询结果、若结果太大、会耗费较多本地内存、影响Client本机性能.

  3) 不会把执行命令结果记录到本地的命令历史文件.

  所以: -quick 是让Client变得更快.

  

#### 思考: 

若碰到一个被Killed的事务一直处于回滚状态、应该把Mysql进程强制重启、还是让它执行完呢 ?

因为重启后依然会继续回滚操作、所以 从恢复速度的角度来说、应该让它自己执行结束.

若这个语句可能会占用别的锁、或者由于占用IO资源过多、影响到别的语句执行、就需要做主备切换、切到新的主库提供服务. 切换之后别的线程都断开了连接、自动停止执行, 接下来还是等它自己执行完成(减少系统压力、加速终止逻辑).