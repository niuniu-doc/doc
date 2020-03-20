---
title: lsof---一切皆文件
date: 2020-03-20
categories:
  - Linux使用
tags:
  - Linux使用
---
> lsof(list open files) 查看当前系统文件、Linux下任何事物都以文件的形式存在、通过文件不仅可以访问常规数据库、还可以访问网络连接(tcp、udp等)和硬件

**lsof 打开文件可以是：**
* 普通文件 
* 目录
* 网络文件系统的文件
* 字符或设备文件
* 共享库
* 管道、命名管道
* 符号连接
* 网络文件 (eg. NFS file、网络socket、unix 域名socket等)
* 其它类型的文件

**命令参数**
> -a 列出打开文件存在的进程
-c<进程名>
