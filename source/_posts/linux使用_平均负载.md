---
title: 平均负载
date: 2020-03-20
categories:
  - Linux使用
tags:
  - Linux使用
---
```
uptime:
 输出: 11:39:32 up 13 days,  1:51, 18 users,  load average: 6.21, 5.86, 4.91
 说明: 当前时间、系统运行时间、正在登陆用户数、最近1min、5min、15min的平均负载 load average
 
```



`平均负载`: 是单位时间内、系统处于可运行状态和不可中断状态的平均进程数, 即: 平均活跃进程数、和CPU的使用率无直接关系

`可运行状态的进程`: 正在使用CPU或者等待CPU的进程、ps的R(Runnning / Runable) 状态

`不可中断状态`: 正处于内核关键流程中的进程、且这些流程是不可打断的、eg. 等待硬件设备的IO响应、ps的D状态(Uninterruptible Sleep,即: Disk Sleep)

> eg. 当一个进程向磁盘写数据时, 为了保证数据一致性、在得到磁盘回复前、是不能被打断的、若被打断, 容易出现磁盘数据和进程数据不一致
>
> 其实: 不可中断状态其实是系统堆进程和硬件设备的一种保护机制



#### 平均负载合理性评估

平均负载为2怎么解读 ？

```
1. 在只有2个CPU的系统上、意味着CPU刚好可以全部占用
2. 在4个CPU的系统上、意味着有50%的空闲
3. 在1个CPU的系统上、意外着会有一半的进程竞争不到CPU
```



CPU核数查看

```
cat /proc/cpuinfo | grep 'model name' | wc -l
lscpu
```



> 经验值: cpu 的负载到达70%的时候、就需要关注了



平均负载与CPU使用率

```
1. 平均负载: 单位时间内处于可运行状态和不可中断状态的进程数
   包含 正在使用CPU的进程 + 等待CPU的进程 + 等待io的进程
2. CPU使用率: 单位时间内CPU繁忙情况的统计、不一定与平均负载对应
   eg. cpu密集型、使用大量CPU会导致load升高、 - 变化趋势一致
       io密集型、等待io也导致load升高、CPU使用率不一定很高
       
```



案例分析

一、CPU密集型进程

```
终端一:
stress --cpu 1 --timeout 600
--cpu CPU压力测试

终端二:
watch -d uptime 
-d: 高亮显示变化区域

终端三:
mpstat -P ALL 5
-P ALL: 表示监控所有CPU
5表示间隔5s输出一组数据

终端四:
pidstat -u 5 1 间隔5s输出一组数据、查找CPU高的线程

```

![stress.png](https://upload-images.jianshu.io/upload_images/14027542-e170f3f20c5f39de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




二、IO密集型进程

查看方式同上
