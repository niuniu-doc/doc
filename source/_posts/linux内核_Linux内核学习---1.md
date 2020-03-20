---
title: Linux内核学习---1
date: 2020-03-20
categories:
  - Linux内核
tags:
  - Linux内核
---

#### 开机加电 -> main 函数执行

> 分3步完成、实现从启动盘加载操作系统、完成 main 函数加载所需要的准备工作



##### 启动BIOS、完成实模式下的中断向量表和中断服务程序
Q: RAM ？
A: Random Access Memory. 随机存取存储器、eg. 内存条 
   特点是 在加电状态下、可随意读写、断电消失

Q: 加电瞬间、RAM中无任何程序、谁来完成 操作系统从软盘的加载 ？
A: BIOS

Q: 为什么必须把操作系统从软盘加载到RAM ？
A: CPU的逻辑电路被设计为只能从内存中运行、无法直接从软盘运行

Q: 为什么CPU的逻辑电路被设计为只能从内存运行 ？
A: ... 暂未清楚

Q: BIOS本身是如何启动的 ？
A: 固定地址: 0xFFFF0、 
   CPU逻辑上被设计为、在加电的瞬间、强行将 CS设为 0xF000, IP 设为 0xFFF0 
   这样、CS:IP 就指向 0xFFFF0(BIOS程序的入口地址、BIOS程序就开始运行)
   
![BIOS.jpg](https://upload-images.jianshu.io/upload_images/14027542-bc3706ad36a398b8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 从启动盘加载操作系统到内存、加载操作系统的工作就是利用 中断服务程序来完成的
开始加载操作系统....
> 对Linux 0.11而言、是分3步、
  1: 有BIOS中断 0x19 将第一扇区bootsect的内容加载到内存
  2: 在bootsect的引导下、将后边的4个扇区和240个扇区的内容依次加载到内存中
  
![BIOS程序加载.jpg](https://upload-images.jianshu.io/upload_images/14027542-8467ab9c24c4efbc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Q: 操作系统如何加载 ？
A: BIOS代码执行完毕、硬件完成开机自检、会和BIOS联手、让CPU收到 int 0x19中断、CPU在中断向量表中找到中断程序的位置(0x0E6F2)
   即: 启动记载服务程序的入口地址、
   so. in 0x19中断程序的作用就是把软盘第一扇区的代码512b加载到内存指定位置(BIOS设计、与操作系统无关) - 内存(0x07C00)处
   第一扇区的代码作用就是将软盘中的操作系统程序陆续加载到内存、所以称为引导程序(bootsect)
   第一扇区的载入、标记着操作系统代码要开始发挥作用
![响应int0x19中断.jpg](https://upload-images.jianshu.io/upload_images/14027542-482d2aaafa61ffa8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   
   
##### 操作系统的内存规划

实模式下最大的寻址为1MB

```
//代码路径：boot/bootsect.s  
    …  
.globl begtext, begdata, begbss, endtext, enddata, endbss  
.text  
begtext:  
.data  
begdata:  
.bss  
begbss:  
.text  
 
SETUPLEN= 4    ! nr of setup-sectors  
BOOTSEG = 0x07c0    ! original address of boot-sector  
INITSEG = 0x9000    ! we move boot here-out of the way  
SETUPSEG= 0x9020    ! setup starts here  
SYSSEG  = 0x1000    ! system loaded at 0x10000 (65536).  
ENDSEG  = SYSSEG + SYSSIZE  ! where to stop loading  
 
! ROOT_DEV:0x000 - same type of floppy as boot.  
!  0x301 - first partition on first drive etc  
ROOT_DEV= 0x306 
… 
![实模式下的内存使用规划.jpg](https://upload-images.jianshu.io/upload_images/14027542-2fd3e1e6ef913630.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```   
`SETUPLEN` -> `SETUPSEG 0x9020` (被加载到的位置)
`BOOTSEG(0x07c0)`(BIOS加载的位置) -> `INITSEG 0x9000`(被移动到的新的位置)
`SYSSEG 0x1000` 内核被加载的位置
`ENDSEG` 内核末尾位置
`ROOT_DEV 0x306` 根文件设备号
**注意:**
CPU的段寄存器（CS）指向 `0x07C0` -> 原来 BOOTSEG 所在的位置
  

##### 为执行32位的main函数做过度工作



#### tips

> 1. 实模式: 兼容80286之后80x86的兼容CPU的模式、特性是一个20位的存储器地址空间(2^20 1MB的存储器空间可被寻址)
可以直接软件访问BIOS及周边硬件、无硬件支持的分页机制和实时多任务概念
80286 开始、所有的CPU开机都是 实模式 、之前的CPU只有一种模式、类似 实模式

> 2. CPU: 计算机所有的计算操作都由它完成、可以理解为 一个有输入和输出功能的集成电路

> 3. CPU寻址: 使用CS、IP两个寄存器
     CS: Code Segment Register 即代码段寄存器、指向CPU当前执行代码在内存中的位置(代码段的起始位置)、
     实模式为绝对地址、16位、保护模式为线性地址 需要结合GDT才能找到段基址
     IP/EIP: Intruction Pointer. 指令寄存器、存在于CPU中、记录将要执行的代码在代码段中的偏移地址
     实模式为绝对地址、指令指针为16位、即IP、保护模式为32位、即EIP、
     CS:IP 寻址规则: 段基址*16 + 偏移地址
     
> 4. 中断向量表建立
     BIOS在最开始的位置(0x00000)用1k内存(0x00000 - 0x003FF)构建中断向量表、接下来256k(0x00400～0x004FF)构建BIOS数据区
     大概57k后的位置(0x0E05B)加载了大概8k左右的与中断向量表相关的中断服务程序 

> 5. 中断向量表 
     记录所有中断向量对应中断程序的位置、实模式中断机制实现的重要部分

> 6. 中断服务程序
     通过中断向量表的索引对中断程序进行响应、是一些具有特殊功能的代码
        
> 7. 磁盘磁头
     利用特殊材料的电阻值会随磁场变化的原来来读写盘片数据、将硬盘盘片上的磁信号转化为电信号向外传输
        
