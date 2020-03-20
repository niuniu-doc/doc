---
title: java秒级定时任务的实现
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
`@Scheduled` 注解、简单轻量级task配置

`@Scheduled` 使用：
* `@Scheduled(cron = "0 5 * * * * ?")` 10分钟执行1次
* `@Scheduled(fixedRate = 2000)` 每2s执行1次、不用等待上次执行完成
* `@Scheduled(fixedDelay = 2000)` 等待上次请求结束后、delay 2s 执行下次任务
* `@Scheduled(fixedDelay = 2000, initDelay = 2000)` 项目启动成功后、延迟2s执行任务 
