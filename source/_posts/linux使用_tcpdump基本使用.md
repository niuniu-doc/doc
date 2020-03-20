---
title: tcpdump基本使用
date: 2020-03-20
categories:
  - Linux使用
tags:
  - Linux使用
---
**选项**
* `-i any` 监听所有的网卡接口、用来查看是否有网络流量
* `-i eth0` 只监听eth0网卡流量
* `-D` 显示可用的接口列表
* `-n` 不解析主机名、直接使用ip
* `-nn` 显示端口
* `-q` 显示简化输出
* `-t` 显示可读的时间戳
* `-X` 以`hex`和`ASCII`两种形式显示包内容
* `-v` `-vv` `-vvv` 显示更加多的包信息
* `-XX` 与 `-X` 类似、增加以太网header的显示
* `-c` 只读取`x`个包, 然后停止
* `-s` 指定每个包捕获的长度、单位是 `byte`, 可以使用 `-s0` 捕获整个包
* `-S` 输出绝对的序列号
* `-e` 获取以太网 `header`
* `-w` 将捕获的数据包信息写入文件
* `-r` 加载之前保存的文件

**表达式**
在`tcpdump`中可以使用表达式过滤指定类型的流量：
* 类型type选项包含：`host`  `net` `port`
* 方向dir包含：`src`  `dst`
* 协议proto选项包含：`tcp` `udp` `ah`等

**示例**
> 捕获所有流量
tcpdump -i any

> 指定网卡接口、查看指定网卡发生了什么
tcpdump -i eth0

> 原生输出、不解析主机、端口、显示绝对序列号、可读的时间戳
tcpdump -ttttnnvvS

> 查看指定ip的流量
tcpdump host {ip}

> 使用`源`和`目的`过滤
tcpdump src {source ip}
tcpdump dst {dest ip}

> 过滤某个子网的数据包
tcpdump net 1.2.3.0/24

> 过滤指定端口相关的流量
tcpdump port {port}
tcpdump src port {port} -> 只显示发出

> 过滤指定协议的流量
tcpdump tcp

> 只显示ipv6流量
tcpdump ip6

> 基于包大小过滤流量
tcpdump less 32
tcpdump grater 64
tcpdump <=128

> 使用端口范围过滤
tcpdump portrange 21-23

> 保存到指定文件
tcpdump port 80 -w file

> 加载之前保存的文件
tcpdump -r file

**高级使用**
* `AND`: `and` or `&&`
* `OR`: `or` or `||`
* `except`: `not` or `!`
> 过滤指定源和目的端口
tcpdump -n src 1.1.1.2 and dst port 8080

> 过滤指定网络方向
tcpdump -n src net 1.1.1.2/16 and dst net 1.1.1.3/16

> 过滤到指定ip的非icmp报文
tcpdump dst 1.1.1.2 and src net and not icmp

> 构建规则过于复杂的时候、可以使用单引号将规则放到一起
tcpdump 'src 1.1.1.1 and (dst port 80 or 22)'

**隔离指定的TCP标识**
`tcp[13]`表示在tcp header中的偏移位置13开始、后边代表的是匹配的字节数
> 显示所有的urgent包(URG)
tcpdump 'tcp[13] & 32!=0'

> 显示所有的ACK包
tcpdump 'tcp[13] & 16!=0'

> 显示所有的push包
tcpdump 'tcp[13] & 8!=0'

> 显示所有的reset包
tcpdump '[tcp13] & 4!=0'

> 显示所有的SYN包
tcpdump '[tcp13] & 2!=0'

> 显示所有的FIN包
tcpdump '[tcp13] & 1!=0'

> 显示所有的SYN/ACK包
tcpdump 'tcp[13]=18'

**识别重要流量**
> 过滤同时设置SYN和RST标识的包（这在正常情况下不应该发生）
tcpdump 'tcp[13] = 6'

> 过滤明文的HTTP GET请求
tcpdump 'tcp[32:4] = 0x47455420'

> 通过横幅文本过滤任意端口的SSH连接
tcpdump 'tcp[(tcp[12]>>2):4] = 0x5353482D'

> 过滤TTL小于10的包（通常情况下是存在问题或者在使用traceroute）
tcpdump 'ip[8] < 10'

> 过滤恶意的包
tcpdump 'ip[6] & 128 != 0'


**理解报文内容**
```
08:41:13.729687 IP 192.168.64.28.22 > 192.168.64.1.41916: Flags [P.], seq 196:568, ack 1, win 309, options [nop,nop,TS val 117964079 ecr 816509256], length 372
```
`08:41:13.729687` 本地时间戳
`ip` 协议是ipv4、若是ipv6会显示为 ip6
`192.168.64.28.22` 源ip和端口
`192.168.64.1.41916` 目的ip和端口
`Flags [P.]` 报文标记段 
`seq 196:568` 代表该数据包包含该数据流的第 196 到 568 字节
`ack 1` 该数据包是数据发送方，ack 值为 1。在数据接收方，该字段代表数据流上的下一个预期字节数据，例如，该数据流中下一个数据包的 ack 值应该是 568
`win 309` 表示接收缓冲区中可用的字节数，后跟 TCP 选项如 MSS（最大段大小）或者窗口比例值
`length 372` 代表数据包有效载荷字节长度
> flag描述
S SYN Connection Start
F FIN Connection Finish
P PUSH Data Push
. ACK Acknowledment
