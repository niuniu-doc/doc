---
title: 垃圾回收---JDK监控工具
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
#### JDK监控工具

`jps`: JVM Process Status Tool. 显示指定系统内所有HotSpot vm 进程
`jstat`: JVM statistic Monitor Tool. 用于收集vm各方面的运行数据
`jinfo`: Configuration Info For Java. 显示虚拟机配置信息
`jmap`: Memory Map For Java. 生成内存转储快照文件(headdump文件)
`jhat`: JVM Head Dump Browser. 用于分析dump文件、
`jstack`: 显示虚拟机的线程快照

#### `jps`
`usage`: `jps [opt] [hostid]`
> `-q` 只输出LVMID 省略主类的名称
> `-m` 输出虚拟机的进程启动时传递给主类函数的参数
> `-l` 输出主类的全名、若主类注销的是jar包、输出jar包路径
> `-v` 输出虚拟机启动时的jvm参数

#### `jstat` 
`usage`: `jstat [opt vmid [interval][s|ms] [count]]`
> `-class` 监视类装载、卸载数量、总空间及类装载所需时间
> `-gc` 监视Java堆情况、包括Eden区、两个Survivor区、老年代、永久代的容量、已用空间、GC时间合计等信息
> `-gccapacity` 监视内容与`-gc`基本相同、主要关注java堆各个区域用到的最大、最小空间
> `-gcutil`  监视内容与`-gc`基本相同、主要关注已使用空间与总空间的比率
> `-gccause` 与`-gcutil`相同、会额外输出导致上次gc的原因
> `-gcnew` 监视新生代gc的状况
> `-gcnewcapacity` 与`gcnew`基本相同、输出主要关注最大、最小空间
> `-gcold` 监视老年代gc的状况
> `-gcoldcapacity` 与`gcold`基本相同、输出主要关注最大、最小空间
> `-gcpermcapacity` 输出永久代最大、最小空间
> `-compiler`  输出jit编译过的方法、耗时等信息
> `-printcompalition` 输出已被编译过的方法
```
eg. jstat -gcutil  41610 250 3
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
  0.00   0.00   4.00   0.43  62.81  65.59      1    0.002     1    0.006    0.008
  0.00   0.00   4.00   0.43  62.81  65.59      1    0.002     1    0.006    0.008
  0.00   0.00   4.00   0.43  62.81  65.59      1    0.002     1    0.006    0.008
S0: survivor0区、都是空的
S1: survivor0区、都是空的
E: Eden区、4%的使用率
O: Old区、0.43%的使用率
M:Perm区、62.81%的使用率
CCS: class Space、 65.59%的使用率
YGC: young gc、1次
YGCT： young gc 耗时 0.002s
FGC: full gc 1次
FGCT: full  gc 耗时 0.006s
GCT: 总的gc耗时 0.008s
```

#### `jinfo` 
`usage` : `jinfo [opt] pid`
> 实时查看和调整虚拟机启动参数
eg.  jinfo -flag CMSInitiatingOccupancyFraction 41610
> 使用` [+|-] name` 修改一部分运行时虚拟机参数值
> 也可以使用 `jps -v` 来查看启动时默认参数
> `-XX:+PrintFlagsFinal` 可以将参数打印出来

#### `jmap` 
> `-dump` 生成快照文件
eg. jmap -dump:format=b, file=path pid
> `-finalizerinfo` 线上在F-Queue中等等Finalizer线程执行finalize方法的对象
> `-heap` 线上Java堆详细信息
> `-histo` 显示堆中对象的统计信息
> `-permstat` 以classLoader为统计路径、显示永久代内存使用情况
> `-F` 虚拟机不响应-dump时、使用-F强制生成dump文件
```
jmap -histo[:live] pid 显示堆中活跃对象
jmap -dump:[live,], file=filename 生成java堆
```

#### `jhat`
`Usage:` `jhat -port 9000 dump-file`
> `-port` 指定端口
分析结果以`包`为单位、分析内存泄露时、可能会用到 Heap Histogram(与 jmap -histo 功能一致)

#### jstack 
`stack trace for java`生成虚拟机当前时刻的快照
`Usage`: `jstack opt vmid`
> `-F` 正常的输出请求不被响应时、强制输出线程堆栈
> `-l` 除堆栈外、显示线程的锁信息
> `-m` 若调用本地方法、显示C/C++的线程堆栈
