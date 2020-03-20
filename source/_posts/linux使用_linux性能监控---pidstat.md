---
title: linux性能监控---pidstat
date: 2020-03-20
categories:
  - Linux使用
tags:
  - Linux使用
---
1. cpu使用率监控
pidstat -t {pid}  1 3 -u 每隔1s输出1次、共输出3次系统状况
pidstat -t {pid}  1 3 -u -t 同时输出线程使用情况

2. io监控
pidstat -t {pid}  1 3 -d -t 查看线程io信息

3. 内存使用监控
pidstat -t {pid}  1 3 -m -t 查看线程内存使用
