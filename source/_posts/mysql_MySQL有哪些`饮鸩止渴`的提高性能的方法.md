---
title: MySQL有哪些`饮鸩止渴`的提高性能的方法
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---

一、短链接风暴
正常的短连接模式是连接到数据库后、执行很少的SQL语句就断开、下次需要再重连、若使用的是短连接、在业务高峰期、可能会出现连接数暴涨的情况. MySQL建联的成本是比较高的、除正常的网络三次握手外、还需要登录权限判断和获得这个连接的数据读写权限.

DB压力小时、这些成本并不明显、但DB处理较慢时、连接数就会暴涨、max_connections参数、用来控制一个MySQL实例同时存在的连接数的上限、超过该值、系统会拒绝接下来的连接请求. 此时一个自然的想法是: 调高max_connections的值、但更多请求进来、可能会导致系统的负载进一步变大、大量的资源消耗在鉴权等逻辑上、已经拿到连接的线程拿不到CPU资源去执行请求, 适得其反. 

可以怎么做呢 ?
1. 先处理掉那些占着连接、但不工作的线程. 
   通过kill_connection主动踢掉. 类似提前设置wait_timeout参数(一个线程空闲wait_timeout秒之后、MySQL会主动断开连接)
information_schema.innodb_trx 表可以查看事务具体状态.
从服务端断开: kill connection + id. 此时Client处于sleep状态、连接被服务端主动断开后、Client不会马上知道、直到Client发起下一个请求, 才会收到 Lost Connection的错误.
此时若Client不重新建联、而直接使用原连接重试、会觉得MySQL一直没恢复

2. 减少连接过程的消耗
有的业务代码会在短时间内先申请大量数据库连接备用、若现在数据库确认是被连接打挂了、可以先让DB跳过鉴权. 重启DB、使用 --skip-grant-tables 参数启动.
但这样MySQL会跳过所有的鉴权、包括连接过程和语句执行过程, 风险极高、尤其是允许外网访问时. 
Mysql8.0, 跳过鉴权时、会默认打开--skip-networking只允许本地Client可见.

二、慢查询性能问题.
可能引发慢查询一般有3种可能:
1. 索引没设计好
2. SQL没写好
3. MySQL选错了索引

1. 索引未设计好、MySQL5.6以后创建索引都支持Online了、一般高峰期DB被SQL打挂时、最高效的做法就是快速添加索引. 若有主备、最高在备库先执行、
1) 在备库执行set sql_bin_log=off, 关闭binlog写入、执行alter语句添加索引
2) 执行主备切换、
3) 此时主库是B、备库是A、在A上执行 set sql_log_bin=off, 执行alter语句添加索引
平时做变更时、应考虑 gh-ost 更稳妥、但紧急处理时、上边的方案最高效.

2. 语句没写好
Mysql5.7以上的版本、提供了 query_rewrite 功能、可以将输入的语句改写成另外一种模式.
eg. select * from t where id+1=10000 会让索引失效、可以通过一个改写语句解决

```SQL 
insert into query_rewrite.rewrite_rules(pattern, replacement, pattern_database) 
values ("select * from t where id + 1 = ?", "select * from t where id = ? - 1", "db1");

call query_rewrite.flush_rewrite_rules(); // 让新插入的规则生效
```
3. 索引没选对、可以通过上线前的SQL分析来提早发现.

三、QPS突增问题
有时业务突然出现高峰、或者应用程序bug导致某个语句的QPS暴涨、也可能导致MySQL压力过大、影响服务. 
1. 若是新业务bug导致、且新业务可以下掉、只是时间没那么快、可以直接从DB上把白名单禁掉.
2. 若这个新功能使用的是独立账号、可以将该账号禁用, 这样由它引发的QPS就会降到0
3. 若新功能是跟主体功能部署在一起的、只能通过处理语句来限制, 可以使用上边的重写功能、将压力最大的SQL重写为 select 1.
当然该操作的风险很高、它可能导致:
1. 若别的功能也用到了该SQL、会被误伤
2. 很多业务不是靠一个SQL就完成逻辑的、这样会导致后边的业务一起失败.
