---
title: redis基础命令
date: 2020-03-20
categories:
  - Redis
tags:
  - Redis
---
#### redis基础命令
```
1. 返回给定pattern的所有key
   keys *
   keys list*

2. 确认key是否存在
   exists key
   
3. 删除key
   del key

4. 设置key过期
   expire key expire

5. 将当期数据库的某个key转移到另外一个db
   move key dbindex

6. 移除给定key的过期时间
   persist key

7. randomkey 随机返回一个key

8. rename key newname 重新命名一个key

9. type key 查看key的数据类型

10. dump key 返回序列化后的key

11. restore key 反序列化为redis键

12. expireat 和expire类型、接受的参数为 unix时间戳

13. migrate host port key destination-db timout [copy] [repalce]
    将key原子性的从当前实例转移到目标实例的指定db上、一旦传送成功、key会出现在目标实例、当前实例的key会被删除
    eg. redis-cli -h host migrate des_host 6379 operateKey 0 1000
    
14. Object 从内部观察redis对象
    Object encoding key 
    (str: raw & int(为了节省内存) list: ziplist & linkedlist set: intset & hashtable
     hash: zipmap & hashtable zset: ziplist & skiplist)
    Object refcount key
    object idletime key

15. renamenx key 当且仅当newkey不存在时、改名为newkey

16. sort 对列表、集合、有序集合中的key进行排序
    alpha对字符串进行排序  sort list alpha
    limit offset count 跳过offset个元素、返回count个元素
    sort list limit offset count
    
17. scan、sscan、hscan、zscan 迭代元素
    scan cursor 迭代获取元素
```
