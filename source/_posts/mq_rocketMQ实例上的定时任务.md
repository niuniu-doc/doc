---
title: rocketMQ实例上的定时任务
date: 2020-03-20
categories:
  - MQ
tags:
  - MQ
---
```
1. 120s更新一次nameSrv地址表
2. 30s或指定配置时间pollNameServerInterval 更新一次 topic路由信息
3. 30s或指定时间 heartbeatBrokerInterval 更新一次broker信息
4. 5s或persistConsumerOffsetInterval时间、持久化一次消费偏移量信息
5. 每分钟动态调整一个threadpool大小

```
