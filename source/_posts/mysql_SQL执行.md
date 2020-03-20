---
title: SQL执行
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
[toc]

### SQL执行流程

#### MySQL架构图
![image.png](https://upload-images.jianshu.io/upload_images/14027542-4270d2d72dee925f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)


#### server层、
> 包含连接器、查询缓存、分析器、优化器、执行器等、涵盖mysql大部分核心功能、
及mysql所有的内置函数(日期、时间、数学和加密函数等)、所有跨存储引擎的功能
(视图、存储过程、触发器等)都在这一层实现

##### 连接器
```
mysql -h$ip -P$port -u$user -p
若 用户名和密码 认证失败、会返回 Access denied for user 的错误
若认证通过、这个连接里的权限都会依赖此时权限表查到的权限
此时用admin账号修改该用户的权限、不会立即生效、会在重新连接时生效

连接完成后、无后续动作、则连接处于sleep状态、

client 长时间(wait_time参数控制)不操作、连接器会断开连接

连接断开之后、再发送请求、会收到 Lost connection to mysql server during query的响应

```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-60a2493de769f5b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/14027542-18905125b398ebe6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/14027542-ee3accb459dc1246.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 查询缓存
```
key 是 sql、value是查询结果 - 任意更新会导致缓存失效
不建议使用
```

##### 分析器
```
词法分析: 根据关键词识别语句类型 select/update/insert
语法分析: 若有语法错误、会返回 You have an error in your SQL 

elect 1 from t where id=1;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'elect 1 from t where id=1' at line 1
                         
``` 

##### 优化器
```
经过分析器、mysql知道了想要做什么、优化器干什么呢 ？
1. 表中有多个索引的时候、决定使用哪个索引
2. 多表join的时候、决定连表顺序

```

![image.png](https://upload-images.jianshu.io/upload_images/14027542-b2765ac683595a26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 执行器
```
在分析器、优化器之后、mysql知道了要做什么、怎么做、就开始进入到执行器阶段、开始执行

1. 命中查询缓存、会在查询缓存返回结果时进行权限验证
2. 非命中 先验证权限、无 -> 直接返回错误
                    有 -> 执行SQL
3. eg. mysql> select * from T where ID=10;
       
    若 id 无索引:
    1) 调用innodb引擎接口查询第一行、若id = 10、存入结果集、否则 跳过、继续
    2) 调用引擎接口取下一行、直至最后一行
    3) 执行器将满足结果的行作为结果集返回给client
    
    若id 有索引:
    1) 调用innodb查询 满足条件的 第一行、放入结果集
    2) 查询满足条件的下一行..
    3) 将结果集返回给client
  
  在db慢SQL中、有一个 rows_examined 字段、表示在SQL执行过程中扫描了多少行、
  是在 执行器 调用 引擎 获取数据行的时候累计的、
  注：有些情况下、执行器调用一次会扫描很多行、所以 实际扫描行数 != rows_examined
    
```
