---
title: rocketMQ消息存储
date: 2020-03-20
categories:
  - MQ
tags:
  - MQ
---


```
rocketmq消息格式
totalSize: 消息长度、4字节
magicCode: 魔术 4字节
bodycrc: 消息体crc校验码 4字节
queueId: 消息队列id 4字节
flag: 消息flag、mq不处理、
ququeoffset: 消息在队列中的偏移量
physicaloffset: 消息在commitlog中的偏移量
sysflag: 消息系统flag、eg 是否压缩、是否是事务消息等 4字节
bronTimeStamp: 生产者调用消息发送API时的时间戳 8字节
bornHost: 消息发送者ip、端口 8字节
storeTimeStamp: 消息存储时间戳  8字节
storeHostAddress: broker服务器ip+port 8字节
reconsumetimes: 消息重试次数 4字节
bodylength: 消息长度
body: 消息内容
topicLength: 主题存储长度
topic: 主题
propertiesLength: 消息属性长度、2字节
Properties: 消息属性
```
```

1. 存储设计
   commitLog: 消息存储文件、所有topic的消息都存储在commitLog
   大小默认1G、以文件中第一个偏移量为文件名、偏移量小于20位、用0补全
   
   consumeQueue: 消息消费队列、消息到达commitLog后悔异步转发到Queue、
   indexFile: 消息索引文件、主要存储key与offset的对应关系
   
   事务状态服务: 存储每条消息的事务状态
   定时消息服务: 每一个延迟级别对应一个消息消费队列、存储延迟队列的消息拉取进度
   
2. msg发送存储流程
   若当前broker挂掉或者broker为slave或者不支持写入时、拒绝写入
   若topic的长度>256个字符、或者msg的属性长度>65535时、拒绝写入
   
   若消息的延迟级别>0, 将消息的原主题名称和原消息队列id存入消息属性、使用延迟消息队列schedule_topic
   
   transientStorePoolEnable 是否开启缓存池. rocketmq单独创建一个mappedByteBuffer内存缓存池、临时存储数据、数据会先写入该内存映射、然后由commit线程复制到与物理文件对应的内存映射中、主要是提供一种内存锁定、将当前堆外内存一直锁定在内存中、避免被进程将该内存交互到磁盘、提高存储性能
   
   
3. 存储文件组织与内存映射机制
   doAppend只是将消息存储到byteBuffer中、然后创建AppendMessageResult、只是将消息存储在MappedFile对应的内存映射buffer中、并未刷到磁盘
   handleDiskFlush方法会处理数据持久化
   handleHA会处理m-s
   
4. 存储文件
   commitlog: 消息存储目录
   config运行期间配置信息
   consumequque 消息消费队列存储目录
   index 消息索引文件存储目录
   abort 启动时创建、正常退出之前删除
   checkpoint 文件检测点、存储commitlog文件最后一次刷盘时间戳、consumeQueue最后一次刷盘时间、index文件最后一次刷盘时间戳
   
5. 消费队列、索引文件构成和机制
6. 文件恢复机制
   rocketmq将消息全量存储在commitlog中、然后异步转发任务更新consumeQueue、index文件、若消息成功存储在commitLog转发任务执行失败、eg. broker宕机、则存储三者不一致的情况、commitlog中的消息可能永远不会被消费、rocketmq是如何保障最终一致性的 ？
   1) 判断上次退出是否正常、在broker启动时check是否存储abort文件、若存在、说明是非正常退出、需要修复
   2) 加载延迟队列
   3) 加载commitlog文件
   4) 加载消息消费队列
   5) 加载checkpoint文件、
   6) 加载索引文件、索引文件上次刷盘时间<该索引文件最大的消息时间戳时 说明索引文件不完备、立即销毁
   7) 根据broker是否正常停止、执行不同的恢复策略
   
7. 刷盘机制
   rocketMQ的刷盘是基于NIO的内存映射机制MappedByteBuffer、消息存储时先将消息追加到内存、再根据配置的刷盘策略在不同时间进行刷写磁盘。
   同步刷盘时: 消息追加到内存后悔同步调用force()方法
   消息生产者将在消息服务器端将消息内容追加到内存映射文件后、需同步将内存内容立刻刷到磁盘、调用内存映射文件MapppedByteBuffer的force方法可将内存中的数据写入磁盘
   
   异步刷盘时: 消息写到内存后、会立即返回给producer、mq单独起一个线程按照固定频率刷盘
   若开启 transientStorePoolEnable机制、RocketMQ会单独申请一个与目标物理文件commitlog相同大小的堆外内存、它将会使用内存锁定、不会被置换到虚拟内存中、消息先追加到堆外内存、然后提交到与物理文件的内存映射内存中、再flush到磁盘
   若未开启transientStorePoolEnable机制、消息直接追加到与物理文件直接映射内存中、然后刷写到磁盘
   1) 先将消息直接追加到ByteBuffer(堆外内存DirectlyByteBuffer), WrotePosition随消息增加不断向后移动
   2) CommiteRealTimeService线程默认每200ms将ByteBuffer中新追加的内容(WritePosition-commitedPosition)提交到MappedByteBuffer中
   3) MappedByteBuffer在内存中追加提交的内存、wrotePosition向前后移动、然后返回
   4) commit操作成功返回、将commitedPosition想前后移动本次提交的内容长度、此时wrotePosition指针依然可以向前推进
   5) flushRealTimeService线程默认每500ms将MappedByteBuffer中新追加的内容wrotePosition-上次刷写位置 flushedPositiont通过调研MappedByteBuffer#force方法讲数据刷写到磁盘
   
8. 文件删除机制
   rocketMQ操作commitLog、conssumeQueue是基于内存映射机制并在启动的时候加载commitLog、consumeQueue下所有的文件、为了避免内存与磁盘浪费、引入文件过期机制:
   超过一定时间间隔内没有更新的文件被认为过期文件、默认72h、可通过fileReservedTime来修改
   满足以下条件会删除:
   1) 指定删除文件的时间点、固定时间执行删除 默认凌晨4点
   2) 磁盘不足时、主动触发过期文件删除操作 磁盘分区使用率超过90%不可写入
   3) 预留、手工触发
   
```
