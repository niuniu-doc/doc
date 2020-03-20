---
title: DHCP与PXE--ip是怎么来的、又是怎么没的--
date: 2020-03-20
categories:
  - 网络协议
tags:
  - 网络协议
---
#### 如何配置ip地址
1. 使用net-tools
```
$ sudo ifconfig eth1 10.0.0.1/24
$ sudo ifconfig eth1 up
```
2. 使用iprote2
```
$ sudo ip addr add 10.0.0.1/24 dev eth1
$ sudo ip link set up eth1
```
但: 是可以随意配置的吗 ?
> 显然不是. 只要在网络上的包、都是完整的、可以有下层无上层、不能反之.
若配置为不同网段, 发送请求包时、linux的默认逻辑是: 跨网段调用、不会直接将包发送到网络上、而是企图发送到网关.
若配置同网段呢? -- 会配置失败

#### 动态主机配置协议(DHCP)
`DHCP`: 动态主机配置协议(Dynamic Host Configuration Protocol)
有了DHCP、网络管理员只需要配置一段共享的ip地址. 每一台新接入的机器会通过DHCP协议去共享的ip地址里申请、然后自动配置 

> 若数据中心里的服务器、ip一旦配置好、基本不会改变、相当于自己买房、自己装修. DHCP的方式类似于租房、临时借用、用完归还即可.

#### 解析DHCP的工作方式
刚加入网络的机器、暂无ip地址、处于`DHCP Discover`的状态.

它会使用ip地址 0.0.0.0 发送广播包、目的地址为: 255.255.255.255. 广播包封装了UDP、UDP封装了BOOTP. 若网络里配置了DHCP Server, Server会分配ip地址给该MAC地址、同时记录(不会分配给其它机器). 这个过程称为 `DHCP Offer`.

若收到多个DHCP Server回复、一般会选择接受第一个到达的、且向网络发送一个DHCP Request广播数据包, 包含Clineet的MAC地址、接受的ip地址、提供该ip的HDCP Server地址等, 告诉其它server它接受了哪一台server提供的ip、请求他们撤销提供的ip、以提供给下一个ip租用者.

DHCP Server 接收到Client的request后、会广播返回一个ACK消息.

client会在租期过去50%时、向为其提供ip地址的server发送request消息包, client 再根据接收到server回应的ack消息包中提供的新的租期及其他已更新的tcp/ip参数更新自己的配置.

#### 预启动执行环境(PXE)
预启动执行环境: 
