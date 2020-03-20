---
title: linux性能监控---vmstat
date: 2020-03-20
categories:
  - Linux使用
tags:
  - Linux使用
---
![image.png](https://upload-images.jianshu.io/upload_images/14027542-63558050c585c684.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

各列含义如下：
`procs`: 
`r` 等待运行的进程数
 `b` 可中断休眠的进程数

`memory`: kb
`swpd` 虚拟内存的使用项目
`free` 空闲的内存
`buff` 被用来作为缓存的内存数

`swap`
`si` 从磁盘交换到内存的交换页数量、
`so` 从内存交换到磁盘的交换页数量、

`io`
`bi` 发送到块设备的块数
`bo` 从块设备接收到的块数

`system`
`in` 每秒的中断数包括时钟中断
`cs` 每秒的上下文切换数

`cpu`
`us` 用户cpu使用时间
`sy` 系统cpu使用时间
`id` 空闲时间
