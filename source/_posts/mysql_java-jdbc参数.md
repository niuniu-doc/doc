---
title: java-jdbc参数
date: 2020-03-20
categories:
  - MySQL
tags:
  - MySQL
---
|参数名称|参数说明|缺省值|版本|
|--|--|---|--|
|user|数据库用户名| | all|
|password|密码||all|
|useUnicode|是否使用unicode字符集、若characterEncoding设置为gb2312或者gbk、则必须设置为true|false|>1.1|
|characterEncoding|指定字符编码|false|>1.1|
|autoReconnect|db异常时、是否自动重连？|false|1.1|
|autoReconnectForPools|是否使用针对db 连接池的重连策略|false|1.1|
|failOverReadOnly|自动重连成功后、连接是否设为只读|true|3.0.12|
|maxReconnects|autoReconnect设为true时、重试次数|3|1.1|
|initalTimeout|autoReconnect为true时、两次重试之间的时间间隔|2s|1.1|
|connectTimeOut|和数据库建立socket连接的超时时间ms|0-永不超时|3.0.1|
|socketTimeOut|socket读写超时ms|0-永不超时|3.0.1|
| zeroDateTimeBehavior|将db 0值转化为null||3.1.4|
