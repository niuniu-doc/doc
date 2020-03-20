---
title: 常用命令
date: 2020-03-20
categories:
  - Linux使用
tags:
  - Linux使用
---
`cat /proc/cpuinfo | grep 'physical id' |  sort| uniq| wc -l` 查看物理CPU的个数
`cat /proc/cpuinfo| grep "cpu cores"| uniq` 查看每个物理CPU中cores的个数
`cat /proc/cpuinfo| grep "processor"| wc -l` 查看逻辑CPU的个数
