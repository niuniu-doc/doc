---
title: Linux基本认知-常用命令&系统调用
date: 2020-03-20
categories:
  - 操作系统
tags:
  - 操作系统
---
1. `rpm -qa` 查看安装的软件列表
   `-q` `--query` 查看  `-a` `--all` 所有
   后可用管道输出 eg. `rpm -qa | less` `rpm -qa | grep mysql`
   `-i` `--install` 从软件包安装
   `-e` `--erase` 删除软件包

2. `man rpm` 查看rpm帮助文档

3. yum配置文件 `/etc/yum.repos.d/CentOS-Base.repo`

   ```
   [base]
   name=CentOS-$releasever - Base - 163.com
   baseurl=http://mirrors.163.com/centos/$releasever/os/$basearch/
   gpgcheck=1
   gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
   
   ```

4. `nohup` `no hang up` 程序不挂起、退出命令行、程序依然执行

5. `&` 表示后台运行

   ```
   nohup command > out.file 2>&1 &
   1表示文件描述符1, 代表标准输出
   2表示文件描述符2, 代表标准错误输出
   2>&1 合并标准错误输出和标准输出 输出到out.file
   ```

   > ps -ef | grep command | awk '{print $2}' | xargs kill -9 
   >
   > 关闭进程

6. 创建新的进程 `fork`

   ```
   父进程返回0、子进程返回进程号、调用execve来执行另外的程序(先copy再修改)
   每个进程有独立的内存空间、相互不干扰
   代码段: Code Segment 存放程序代码
   数据段: Data Segment 存放运行中数据
          局部变量、当前函数执行时分配、运行结束释放
          动态分配、长时间保存、指明才销毁 堆heap
   一个进程的内存空32位是 4G、64位
   
   进程内存的分配是在使用时、进程需要使用内存的时候调用内存管理系统登记、但此时不会真正分配、真正写入数据的时候发现无对应物理内存、才会触发中断、分配物理内存
   
   需要内存小的时候、使用brk分配和原来的数据堆连在一起、
   需要内存大的时候、使用mmap、重新划分一整块区域
   
   
   ```

   等待子进程 `waitpid`

7. 文件管理

   * 已有文件, 使用 `open` 打开文件, `close` 关闭文件
   * 不存在的文件, 使用 `create` 创建文件
   * 打开后的文件, 使用 `lseek` 跳转到指定位置
   * 文件读取 `read` 文件写入 `write`

    

   > linux 的核心思想: 一切皆文件

   * 启动一个进程、需要一个程序文件、这是 二进制文件

   * 启动时的配置文件, 如 `*.yml`, `*.properties` 等、是文本文件、日志也是文本文件

   * 命令交互控制台输出、是标准输出 `stdout文件`

   * A进程的输出可以作为另外一个进程的输入、这个方式成为 `管道` 也是一种文件

   * 不同进程间的通信、通过`socket` 也是文件

   * 进程需要访问外部设备、`设备`也是文件

   * `文件夹`也是文件

   * 进程运行时、`/proc` 下对应的`进程号` 也是文件

     

8. 进程间通信

   * `消息队列`: `Message Queue` 在内核实现, `msgget` 创建、`msgsnd` 发送、`msgrcv` 接收, 消息内容不大时使用
   * `共享内存`: 交互消息比较大的时候, `shmget`创建 `shmat` 将共享内存映射到自己的内存空间, 通过信号量`  semaphore` 来处理`竞争`
     * 对于只允许一个人访问的资源、将信号量设为1、第二个访问请求进入会调用 `sem_wait` 进行等待、访问结束调用 `ssem_post` 释放资源、结束等待

9. `Glibc` 提供`字符串处理`, `数学运算`等用户态服务, 提供系统调用封装

   * 每个特定的系统调用至少对应一个`Glibc` 封装的库函数、

     eg. 打开文件的系统调用 `sys_open` 对应的是 `Glibc` 中的 `open` 函数

   * `Glibc` 中一个单独的API可能对应多个系统调用、

     eg. `Glibc` 对应的`printf` 就会调用 `sys_open`, `sys_mmap`, `sys_write`, `sys_close` 等系统调用

   * 也可能多个api对应同一个系统调用、

     eg. `Glibc` 下实现的 `malloc`, `calloc`, `free` 等函数用来分配和释放内存、都利用了内核的 `sys_brk` 的系统调用


     
![Linux系统调用.png](https://upload-images.jianshu.io/upload_images/14027542-beca9374617ce5cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


参考: 
[https://time.geekbang.org/column/article/89251](https://time.geekbang.org/column/article/89251)

【感谢极客时间、有很多优秀的专栏】
