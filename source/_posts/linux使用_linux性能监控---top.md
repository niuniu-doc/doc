---
title: linux性能监控---top
date: 2020-03-20
categories:
  - Linux使用
tags:
  - Linux使用
---
![image.png](https://upload-images.jianshu.io/upload_images/14027542-15663f2f66d4722a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到`top`的前半部分是系统统计信息、后半部分是进程信息
第`1`行：任务队列信息，等同于`uptime`、从左到右依次为：
`系统当前时间`，`系统运行时间`、`登录用户数`
`load avg` 是系统平均负载、代表最近`1min`、`5min`、`15min` 的平均值
第`2`行：进程统计信息`总进程数`、 `正在运行进程数`、`休眠进程数`、`停止的进程数`、`僵死进程数`
第`3`行：cpu统计信息 `us:用户空间cpu占用率`、`sy:内核空间cpu占用率`、`ni用户进程空间改变过优先级的进程cpu占用率`、`id空闲cpu占用率`、`wa等待输入输出的cpu时间百分比`、`hi硬中断请求`、`si软中断请求`
第`4`行：内存统计信息 `total:物理内存总量` `free:空闲内存总量` `used:已使用内存总量` `内核缓冲使用量`
第`5`行：交换区使用情况

进程区字段含义：
`pid`: 进程id
`user`: 进程所有者的用户名
`pr`: 优先级
`ni`: nice值、负值表示高优先级
`virt`: 进程使用的虚拟内存总量
`res`: 进程使用的、未被换出的物理内存大小
`shr`: 共享内存大小
`s`: 进程状态
`%cpu`: 上次更新到现在的cpu时间占用百分比
`%mem`: 进程使用的物理内存百分比
`time+`: 进程使用的cpu时间总计，单位：`1/100s`
`command`: 命令名
