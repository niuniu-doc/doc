---
title: mq本地搭建
date: 2020-03-20
categories:
  - MQ
tags:
  - MQ
---
启动:
进入到编译target目录
eg. 我的下载路径是 /Users/nj/build/rocketmq-all-4.4.0
cd /Users/nj/build/rocketmq-all-4.4.0/distribution/target/apache-rocketmq/

1.启动namesrv
sh bin/mqnamesrv &

2. 启动broker
sh bin/mqbroker &

3. 测试:
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
报错:
```
17:28:23.123 [main] DEBUG i.n.u.i.l.InternalLoggerFactory - Using SLF4J as the default logging framework
org.apache.rocketmq.client.exception.MQClientException: No route info of this topic, TopicTest1
See http://rocketmq.apache.org/docs/faq/ for further details.
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(DefaultMQProducerImpl.java:610)
17:28:29.595 [NettyClientSelector_1] INFO  RocketmqRemoting - closeChannel: close the connection to remote address[] result: true
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1223)
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1173)
	at org.apache.rocketmq.client.producer.DefaultMQProducer.send(DefaultMQProducer.java:214)
	at com.lct.quickstart.Producer.main(Producer.java:56)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:140)
```

原因:
mac本地搭建使用VPN时、会有两个内网ip、用telnet找到可以访问的那个、
修改broker.conf
指定brokerIP的地址:
brokerIP1 = 192.168.10.54
brokerIP2 = 192.168.10.54

查看broker启动参数:
sh bin/mqbroker -m

mqAdmin使用: 
####查看集群情况
./mqadmin clusterList -n 127.0.0.1:9876

####查看broker状态
./mqadmin brokerStatus -n 127.0.0.1:9876 -b 192.168.10.54:10911

####查看topic列表
./mqadmin topicList -n 192.168.10.54:9876

####查看topic状态
./mqadmin topicStatus -n 127.0.0.1:9876 -t PushTopic

####查看topic路由
./mqadmin topicRoute -n 127.0.0.1:9876 -t PushTopic
