---
title: redis运维命令
date: 2020-03-20
categories:
  - Redis
tags:
  - Redis
---
平时觉得没什么用的命令、关键时候救命、😁~~~
```

1. 查看db大小
   redis-cli -h host -p port -a password dbsize



2. cat a.txt | redis-cli 从文件读入redis



3. redis-cli -a password --csv lrange mylist offset count
    从redis中快速读取一些key



4. n command 重复执行n次command



5. help @
  @generic  @list  @hash  @set  @sorted_set @string  @hyperloglog、@server、@connection、@scripting、@pubsub、@transactions

help <command>



6. redis-cli --stat 实时监控redis运行状况
   redis-cli monitor



7. redis-cli --bigkeys 查看redis中比较大的key



8. redis-cli --scan --pattern '*my*' 用scan来查找key、并过滤
   redis-cli --scan --pattern '*my*' |wc -l  直接使用管道作为下个命令的输入




9.  redis-cli -h host --rdb /tmp/dump.rdb
    备份远程redis数据到本地文件



10. maxmemory size 设置redis使用的最大内存
      超过最大内存时、key过期策略

     # volatile-lru -> remove the key with an expire set using an LRU algorithm
   # allkeys-lru -> remove any key accordingly to the LRU algorithm
   # volatile-random -> remove a random key with an expire set
   # allkeys-random -> remove a random key, any key
   # volatile-ttl -> remove the key with the nearest expire time (minor TTL)
   # noeviction -> don't expire at all, just return an error on write operations



11. 使用客户端查看redis配置
     redis-cli -h host config get key
     redis-cli -h host config get slowlog-log-slower-than 单位 微秒
     redis-cli -h host config get slowlog-log-slower-than 慢操作日志队列大小
     redis-cli -h host slowlog get 10 查看最近10条慢日志
     hash-max-ziplist-entries 配置使用hashmap编码的最大字节数
     redis-cli -h host get client-output-buffer-limit 客户端buffer控制、在server与client的交互中、

每个连接都会有一个buffer关联、此buffer用来队列化等待被client接受的响应信息、若client不及时响应、
buffer中积压的数据达到阈值、将会导致连接失败、buffer被移除


12. redis以指定配置文件启动
      redis-server redis.conf &


13. redis-cli --lru-test 10000000 模拟1000w keys的执行情况
     在maxmemory无限制的情况下、测试时无意义的、因为内存无限制、命中率会是100%、计算机的ram会被耗尽
    记得配置key的过期策略

```
