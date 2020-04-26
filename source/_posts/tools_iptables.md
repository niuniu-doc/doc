---
title: iptables学习
date: 2020-03-30 18:04:17
tags: Tools
categories: Tools
---



#### 工作机制

##### 基础说明

```
iptables并不是真正的防火墙、它只是位于用户空间的一个命令行工具, netfilter 才是真正的安全框架、iptables只是将我们定义的规则转交给netfilter. 
netfilter 是Linux操作系统核心层的一个数据包处理模块
```



##### 规则链名称

规则链名称包括以下5个、也被称为5个钩子函数:

`INPUT链`: 处理输入数据包

`OUTPUT链`: 处理输出数据包

`FORWARD链`: 处理转发数据

`PREROUTING链`: 用于目的地址转换(DNAT)

`POSTROUTING链`: 用于源地址转换(SNAT)



##### 报文流向

```
本机到某进程: prerouting -> input
本机转发: prerouting -> forward -> postrouting (直接在内核空间转发)
本机进程发出(通常为响应报文): output -> postrouting
即: 当启用你过来防火墙功能时、报文需要经过不同的关卡(链)、但根据情况的不同、会经过不同的链
```



##### 为什么称为链

```
防火墙的作用就在于对经过的报文进行规则匹配、然后执行对应的动作、但是每个关卡上可能有很多规则、这些规则按照配置顺序从上到下执行、看起来就像是一个链
```



##### 表

```
这么多的链都放了一系列的规则、但是有些很类似、比如A类规则是对ip或者短裤过滤; B类规则是修改报文... , 那么这些类似功能的规则能不能放在一起呢 ?
答案是可以的、iptables提供了如下规则的分类(表):
filter表: 负责过滤功能 , 防火墙; 内核模块 iptable_filter
nat表: 网络地址转换; 内核模块: iptable_nat
mangle表: 解析报文、修改报文、并重新封装 内核模块: iptable_mangle
raw表: 关闭nat表上启用的连接追踪机制; 内核模块: iptable_raw

当这些表处于同一条链时优先级如下:
raw -> mangle -> nat -> filter
```



##### 防火墙的策略

`通`: 默认不允许、需要定义谁可以进入

`堵`: 默认允许、但需要证明自己的身份

filter功能: 定义允许或者不允许的策略, 工作在 input / output / forward 链

nat选项: 定义地址转换功能, 工作在 prerouting / output / postrouting 链

为了让这些功能都工作、制定了`表`的定义, 来区分不同的工作功能和处理方式



##### 常用功能

`filter` 定义允许或者不允许、工作在 `input`, `output`, `forward` 链上

`nat` 定义地址转换, 工作在 `prerouting`, ` output `, `postrouting` 链上

`mangle` 修改报文原数据, 5个链都可以工作

注意: 设置多个规则链时、要把规则严格的放在前边、因为它是从上到下检查的~



##### 规则动作

`accept`: 接收数据包

`drop`: 丢弃数据包

`redirect`: 重定向、映射、透明代理

`snat`: 源地址转换

`dnat`: 目标地址转换

`masquerade`: ip伪装(nat), 用于ADSL

`log`: 日志记录



##### 思考:

其实iptables就是帮我们实现了网络包选择、地址转换、报文修改这些功能、为了实现这些功能、它划分了很多链、包括 prerouting、input、forward、output、postrouting, 定义了规则动作 accept、drop、redirect、snat、dnat、log、masquerade来实现这些功能.



#### 实例

##### 清空当前所有规则和计数

```
iptables -F # 清空所有防火墙规则
iptables -X # 删除用户自定义的空链
iptables -Z # 清空计数
```

##### 配置允许ssh端口连接

```
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 22 -j ACCEPT
# 22: ssh端口 -s 192.168.1.0/24 被允许的网段、 -j accept 接受这种类型的请求
```



##### 允许本地回环地址可以正常使用

```
iptables -A INPUT -i lo -j accept
iptables -A INPUT -o lo -j accept
```



##### 设置默认规则

```
iptables -P INPUT DROP # 配置默认的不让进
iptables -P FORWARD DROP # 默认的不允许转发
iptables -P OUTPUT ACCEPT # 默认的可以出去
```



##### 配置白名单

```
iptables -A INPUT -p all -s 192.168.1.0/24 -j ACCEPT  # 允许机房内网机器可以访问
iptables -A INPUT -p all -s 192.168.140.0/24 -j ACCEPT  # 允许机房内网机器可以访问
iptables -A INPUT -p tcp -s 183.121.3.7 --dport 3380 -j ACCEPT 
# 允许183.121.3.7访问本机的3380端口
```



##### 开启相应的服务端口

```
iptables -A INPUT -p tcp --dport 80 -j ACCEPT # 开启80端口，因为web对外都是这个端口
iptables -A INPUT -p icmp --icmp-type 8 -j ACCEPT # 允许被ping
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT # 已经建立的连接得让它进来
```



##### 保存规则到配置文件

```
cp /etc/sysconfig/iptables /etc/sysconfig/iptables.bak # 任何改动之前先备份，请保持这一优秀的习惯
iptables-save > /etc/sysconfig/iptables
cat /etc/sysconfig/iptables
```



##### 列出已设置规则

```
iptables -L [-t 表名] [链名]

iptables -L -t nat                  # 列出 nat 上面的所有规则
#            ^ -t 参数指定，必须是 raw， nat，filter，mangle 中的一个
iptables -L -t nat  --line-numbers  # 规则带编号
iptables -L INPUT

iptables -L -nv  # 查看，这个列表看起来更详细
```



##### 端口映射

````
iptables -t nat -A PREROUTING -d 210.14.67.127 -p tcp --dport 2222  -j DNAT --to-dest 192.168.188.115:22
# 将本机的 2222端口 映射到虚拟机的 22端口
````



##### 防止SYN洪水攻击

```
iptables -A INPUT -p tcp --syn -m limit --limit 5/second -j ACCEPT
```



#### 写在最后

> iptables --help 可以看到更多的选项、可以看到每一个选项的解释、本文列出了一些简单使用、欢迎指正、交流~