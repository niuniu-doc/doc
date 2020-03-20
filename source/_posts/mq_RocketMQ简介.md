---
title: RocketMQ简介
date: 2020-03-20
categories:
  - MQ
tags:
  - MQ
---
组件:

nameSrv作用: 

```
1.加载kv配置、创建nettyServer网络处理对象、
2.开启定时任务进行心跳监测
task1: nameSrv 每10s扫描一次broker、移除处于不激活状态的broker
task2: nameSrv 每10min打印一次kv配置
3.注册jvm钩子函数、监听broker、消息生产者的请求

broker如果挂掉、producer向broker的消息发送是否失败 ？ 如何处理 ？
10s之后才会更新nameSrv信息、producer才可以检测到

```

NameSrv功能实现
```
1. 路由注册、故障剔除
nameSrv 测主要作用是为Producer和Consumer提供topic信息、所以需要存储路由的基础信息、能够管理Broker节点、包括路由注册、故障剔除等

元信息:
topicQueueTable: topic 消息队列路由信息、消息发送时进行负载均衡
brokerAddrTable: broker基础信息、包含BrokerName、所属集群地址、主备broker地址
clusterAddrTable: broker集群信息、存储集群中brokerName
brokerLiveTable: broker状态信息、nameServer每次收到心跳包时更新
filterServerTable: broker上的FilterServer列表、用于消息过滤

一个topic有多个Queue、一个broker默认为每个topic创建4个读Q 和 4个写Q、多个broker组成一个集群、多台相同brokerName的broker机器组成M-S架构、brokerId=0代表M、

BrokerLiveInfo中的lastUpdateTimestamp存储上次收到broker心跳包的时间

路由注册通过broker和nameSrv的心跳实现、broker每隔30s向所有nameSrv发送心跳、nameSrv收到心跳包更新brokerLiveTable中的BrokerLiveInfo的心跳时间、10s扫描一次brokerLiveTable、若连续120s未收到心跳包、nameSrv会移除该broker

路由删除:
1. nameSrv定时扫描 brokerLiveTable、超过120s未收到心跳包、会移除broker信息
2. broker正常关闭时、主动调用unregisterBroker
1) 从brokerLiveTable、filterServerTable移除该broker
2) 维护brokerAddrList信息
3) 维护clusterAddrTable信息
4) 遍历topic、从路由信息移除该broker

路由发现:
1. 非实时、topic路由信息变化、nameSrv不会主动推送、而是client主动定时拉取、
从topicQueueTable、brokerAddrTable、filterServerTable中填充TopicQueueData的QueueData、BrokerData、filterServer地址表
2. 若从topic找到对应的路由信息为顺序消息、则从nameServer kvConfig中获取关于顺序消息相关的配置填充路由信息

```

Borker:

```
提供消息存储、
对于mq的消息存储、一般考虑 消息堆积能力 和 消息存储能力
rocketmq 引入内存映射机制、所有主题的消息顺序存储在同一文件
引入 消息文件过期机制 和 文件存储空间报警机制 - 避免消息无限堆积

```

Producer启动流程
```
1. 检查producerGroup是否符合要求、改变生产者的instanceName为进程id
2. 创建MQClientInstance实例、整个jvm中只存在一个MQClientManager实例、维护一个MQClientInstance的hashMap表、即: 同一个clientID只会创建一个MQClientInstance
   instance是封装了rocketMQ的网络处理API、是Producer、Consumer与NameSrv、Broker打交道的网络通道
3. 将producer注册到MQClientInstance管理中、方便后续进行网络请求、心跳检测等

```

消息发送过程

```
支持3种方式: 同步sync、异步async、单向oneway
sync: 同步等待broker将结果返回
async: 指定回调函数、不阻塞producer线程、消息结果通过回调通知
oneway: 不等待消息存储结果、亦不提供回调函数、不关心是否存储成功

1. 消息长度验证、消息长度最大4M
2. 查找路由信息
   首次发msg时、本地未缓存topic路由信息、查询nameSrv获取、未找到时会尝试创建、若路由信息变化则更新broker地址列表
3. queue选择
若2m-2s架构、queueNum=4、rocketmq会如何选择 ？
消息发送时会多次执行queue选择算法、lastBrokerName是上次send failed的broker、第一次执行Q选择时、last为null、直接在所有Q中选择、
再次执行时、会先判断、选择brokerName!=lastBrokerName的来进行规避

如果再次选择、得到的依然是不可用的Queue呢 ？会再次重试、造成资源浪费
broker故障不会立即摘除(nameSrv 10s才检测一次、Producer也是30s才更新路由信息、最快感知也需要30s)、

broker延迟故障机制
1) 获取一个小小队列
2) 验证是否可用
3) 若可用、移除latencyFaultTolerance关于该topic的条目、表明broker故障恢复

消息发送

1. 消息队列如何负载 ？
2. 如何实现高可用？
3. 批量消息如何实现一致性？
```

Consumer消息消费
```
集群模式: 同一topic下一条消息只能允许被其中一个消费者消费、消费进度保存在broker端
广播模式: 同一topic下一条消息可以被同一消费组的所有消费者消费、消息消费进度保存在消费端



消息队列负载与重新分布
消息消费模式
消息拉取方式
消息进度反馈
消息过滤
顺序消息
```
消费者启动流程
```
1. 构建订阅主题信息SubscriptionData并加入到RebalanceImpl的订阅消息中、订阅关系来自:
   1) 调用subscribe方法
   2) 订阅重试主题消息 以消费组为单位 命名: %RETRY%+消费组名
2. 初始化MQClientInstance、RebalanceImpl等
3. 初始化消费进度、broadcast保存在broker、cluster保存在consumer
4. 根据是否顺序消费、创建消费端消费线程服务、ConsumeMessageService主要负责消息消费、内部维护一个线程池
5. 向MQClientInstance注册消费者、并启动MQInstance、在一个JVM中的所有消费者、生产者持有同一个MQClientInstance、只启动一次
```

消息可用性保障

```
1. Broker 正常关机
2. Broker 异常Crash
3. OS Crash
4. 机器断电、可立即恢复供电
5. 机器无法开机、可能是CPU、主板、内存等关键设备损坏
6. 磁盘设备损坏

1-4 在同步刷盘机制下、可以确保不丢失消息、异步刷盘模式下丢失少量消息
5、6属于单点故障、一旦发生、节点上的消息全部丢失、若开启了异步复制可保证只丢失少量消息、双写机制下 不会丢失消息
```

消息延迟

```
在正常不发生消息堆积的情况下、以长轮询方式实现准实时消息推送
```

消息堆积

```
rocketmq消息存储使用磁盘文件(内存映射)并且在物理布局上为多个大小相等的文件组成逻辑文件组、可无限循环使用、提供消息过期机制、默认保留3天

```

消息过滤:

```
1. broker端过滤、
2. consumer过滤、
```

msg status 说明:
```
 SEND_OK, // 发送成功
 FLUSH_DISK_TIMEOUT, // 在规定时间内没有完成刷盘、在 flushDiskType 为 SYNC_FLUSH 时才出现 
 FLUSH_SLAVE_TIMEOUT, // 在主备方式下、broker被设置为 sync_master方式时、未在指定时间内完成主从同步
 SLAVE_NOT_AVAILABLE, // 在主备方式下、broker被设置为 sync_master方式时、未找到slave
```

疑问

```
1. broker收到消息会先写commitlog、为什么producer发消息的时候会选择一个Queue去发 ？而不是broker ？再由broker分发到Queue ？

2. 写commitlog的时候、同步刷盘指针和异步刷盘指针写的是同一个commitlog文件、如何保证消息的有序存储？

3. 文件清除何时进行 ？

4. 消息队列如何进行负载均衡？

5. 消息发送如何保证高可用

6. 批量消息如何保持一致性

7. broker失效后120s才能将该broker从路由表中移除、若生产者获取到的路由信息包含已宕机的broker如何处理？

8. 
```
