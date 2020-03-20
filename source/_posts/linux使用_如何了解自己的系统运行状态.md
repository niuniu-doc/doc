---
title: 如何了解自己的系统运行状态
date: 2020-03-20
categories:
  - Linux使用
tags:
  - Linux使用
---
#### 平均负载 loadavg
```
当系统变慢时、我们想到的很可能是 uptime 或者 top
它每一列的含义是什么 ？
```
![uptime](https://upload-images.jianshu.io/upload_images/14027542-7a4f7de3de8dc6be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

so. mark
`10:13:59` 是当前时间
`up 303 days, 22:27` 是系统允许时间
 `1 user` 是正在登录用户数
load average: 4.61, 4.37, 3.73 这三列呢 ？
依次是 过去`1min`、`5min`、`15min`的平均负载


#### 什么是平均负载 ？？？
```
知道了uptime是如何看的、那么平均负载多少算是合适 ？它又是代表什么含义呢 ？

简单来说平均负载就是单位时间内、系统处于可运行状态和不可中断状态的平均进程数、也就是平均活跃进程数，
之前、我以为是cpu的使用率来着的~~~xxxxx


可运行状态的进程：指正在使用CPU或者等待CPU的进程、ps看到的、处于R状态(Runnable/Running)的进程

不可中断进程：正处于内核关键流程中的进程、并且这些进程是不可打断的、eg. 等待硬件设备的io响应、也就是ps命令中的D状态(uninterrutible sleep)的进程
不可中断状态其实是系统对进程包含和硬件设备的一种保护机制

既然平均负载就是平均的活跃进程数、那么最理想的状态就是每个CPU上都刚好运行着一个进程(每个cpu充分利用、又不会过载)

eg. loadavg为2、意味着什么呢 ？
1) 假如系统有2个CPU、则cpu刚好都被占用
2) 在4个CPU的系统上、意味着50%的CPU空闲
3) 在只有1个CPU的系统上、则会有一半的进程竞争不到CPU

最理想的状态是、刚好等于CPU的核心数
那么如何查询系统有几个CPU ？？
1) 使用top
2) 从/proc/cpuinfo 获取 
```
![cpuinfo.png](https://upload-images.jianshu.io/upload_images/14027542-4cf1dcfb4a79d551.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 知道了cpu的核数和衡量指标、如何分析负载趋势？
```
1. 若1min、5min、15min的值基本相同或者相差不大、说明系统负载很平稳
2. 若1min内的值远小于15min的值、说明最近1min的负载在减小、而过去15min则负载很高
3. 反过来、若最近1imn的值远高于过去15min的负载、说明负载在明显升高、需要持续观察一下

eg. 单核cpu的 loadavg为：1.73  0.60  7.98 
说明过去1min有73%的超载、过去15min有698%的超载
总体来看、负载在降低

那么实际生产环境、负载达到多少的时候、我们应该开始关注呢 ？ 一般 超过70%就要持续观察下、看看系统负载的平均趋势
```

#### 案例
```
1. 命令
stress 压力测试工具
mpstat 多核CPU性能分析工具、可实时查看每个CPU的平均指标
pidstat 实时查看进程的cpu、内存、io及上下文切换等指标
watch 监控进程

2. 模拟cpu密集型(需要准备3个终端、每个终端执行一个命令)
stress --cpu 1 --timeout 600

watch -d uptime (-d代表高亮显示变化的区域)

mpstat -P ALL 5 (-p ALL代表监控所有CPU 5代表间隔5s输出1次)

3. 模拟IO密集型
stress -i 1 --timeout 600

4. 大量进程的场景
stress -c 8 --timeout 600

附：安装stress
sudo yum -y install epel-release
sudo yum -y install stress
```
![stress.png](https://upload-images.jianshu.io/upload_images/14027542-e8ae343bc07ff1ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![uptime.png](https://upload-images.jianshu.io/upload_images/14027542-e6c92ece83479acc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![mpstat.png](https://upload-images.jianshu.io/upload_images/14027542-5b6eb08f6768da70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
