---
title: rocketMq入门
date: 2020-03-20
categories:
  - MQ
tags:
  - MQ
---
#### 专业术语

* `Producer` 

  消息生产者、负责生产消息、一般由业务系统负责生产消息

* `Consumer`

  消息消费者、负责消费消息、一般由后台系统负责异步消费

  * `Push Consumer`

    Consumer的一种、通常向consumer对象注册一个Listener接口、一旦收到消息、Consumer对象立即回调Listener接口方法

  * `Pull Consumer`

    Consumer的一种、应用通常主动调用Consumer的拉取消息放啊从Broker拉取消息、主动权由应用控制

* `Producer Group`

  一类Producer的集合名称、这类Producer通常发送一类消息、且发送逻辑一致

* `Consumer Group`

  一类Consumer的集合名称、消费同一类消息、消费逻辑一致

* `Broker`

  消息中转角色、负责存储消息、转发消息、一般也称为`Server`、在JMS规范中称为`Provider`

* `广播消费`

  一条消息被多个Consumer消费、即使这些Consumer属于同一个group、可以认为在消息划分方面无意义

* `集群消费`

  一个Consumer Group中的Consumer实例平均分摊消费消息、eg. 某个topic有9条消息、其中一个Consumer Group有3个实例、则 每个实例只消费其中的3条消息

* `顺序消息`

  消费消息的顺序要同发送消息的顺序一致、在RocketMQ中、主要指的是局部顺序、即: 同一类消息为满足顺序性、必须Producer单线程顺序发送、且发送到同一个队列、

* `普通顺序消费`



![RocketMQ物理部署结构.png](https://upload-images.jianshu.io/upload_images/14027542-720659236fc9c351.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




#### RocketMQ网络部署特点

* NameServer 几乎无状态节点、可集群部署、节点间无任何信息同步

* Broker部署相对复杂、分为master和slave、0表示master、非0表示slave、

  每个Broker与Name Server集群中的所有节点建立长连接、定时注册Topic信息到所有的Name Server

* Producer 与NameServer中的随机一个节点建立长连接、定期从NameServer取topic信息、并向提供topic服务的Master建立长连接、定时向master发送心跳、Producer完全无状态、可集群部署

* Consumer与NameServer中随机一个节点建立长连接、从NameServer取topic路由信息、并向提供topic服务的Master、Slave建立长连接、定时向master、slave发送心跳、即可以从master订阅消息、也可以从slave订阅消息

#### RocketMQ逻辑部署结构

(![逻辑部署结构.png](https://upload-images.jianshu.io/upload_images/14027542-0a5961f3b407c08b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
