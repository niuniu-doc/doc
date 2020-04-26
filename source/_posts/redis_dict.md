---
title: Redis数据结构-字典
date: 2020-04-24 19:25:45
tags: Redis
categories: Redis
---
> 字典又称`符号表`, `关联数组`, `映射`, 是一种用于保存键值对的抽象数据结构
>
> Redis构建了自己的字典实现, eg. set msg 'test'会构建`key`为`msg`, `value`为`test`的键值对.
>
> 除了表示数据库之外, 字典还是hash键的底层实现之一(另外一种是ziplist)



#### 字典实现

##### hash表

Redis字典所使用的哈希表由 dict.h/dictht 结构定义

```c
typedef struct dictht {
  dictEntry **table; // hash表数组
  unsigned long size; // hash表大小
  unsigned long sizemask; // hash表大小掩码, 用于计算hash索引值, 总是=size-1
  unsigned long used; // 已有节点数量
}dictht;
```

![空hash表结构示意](/images/image-hash.png)

##### hash表节点

```c
typedef struct dictEntry {
  void *key; // 键
  union {
    void *val;
    unit64_tu64;
    int64_ts64;
  }v; // 值
  struct dictEntry *next;// 指向下个hash节点、形成链表
}
```

`key`属性保存着键值对中的键、而`v`属性则保存着键值对中的值, 它可以是一个`指针`, 或者是`uint64_t`类型的整数, 或者是 `int64_t`的整数

`next`属性是指向另一个节点的指针, 这个指针可以将多个hash值相同的键值对连接在一起, 解决键冲突问题



##### 字典

Redis中的字典由 `dict.h/dict` 结构定义

```c
typedef struct dict {
  dictType *type; // 类型特定函数
  void *privdata; // 私有数据
  dictht ht[2]; // hash表
  in trehashidx; // rehash索引, 当rehash不在进行时、值为-1
} dict;
```

`type`属性是一个指向 `dictType`结构的指针, 每个 dictType结构保存了一簇用于操作特定类型键值对的函数, Redis会为不同类型的字典设置不同类型的特定函数

`privdata`属性则保存了需要传给那些特定类型函数的可选参数

`type`和`privdata`是针对不同类型的键值对, 为创建多态字典设置的

```c
typedef struct dictType {
  unsigned int (*hashFunction)(const void *key); // 计算hash值的函数
  void *(*keyDup)(void *privdata, const void *key); // 复制键的函数
  void *(valDup)(void *privdata, const void *obj); // 复制值的函数
  int (*keyCompare)(void *privdata, const void *key1, const void *key2); // 对比键的函数
  void (*keyDestructor)(void *privdata, void *key); // 销毁键的函数
  void (*valDestructor)(void *privdata, void *obj); // 销毁值的函数
}dictType;
```

`ht`属性是一个包含两个项的数组, 数组中的每个项都是一个dictht哈希表, 一般只使用ht[0], ht[1]只会在对ht[0] 进行`rehash`时使用.

`rehashidx`记录当前`rehash`的进度, 若没有在进行`rehash`, 则值为 `-1`



##### 完整的dict结构

![dict结构示意](/images/image-dict.png)



#### hash算法

Redis计算hash值的方式如下:

```c
hash = dict->type->hashFunction(key); // 使用hash字典设置的hash函数、计算key的hash值
index = hash & dict->ht[x].sizemask; // 根据sizemask和hash值、计算出hash索引
```

#### 解决键冲突

当有两个或以上数量的键被分配到hash表数组同一个索引上时, 称为`键冲突 collision`, Redis的hash表使用 `链地址法`(`separate chaining`)来解决冲突, 每个hash表节点上有一个next指针, 多个hash表节点使用next指针构成单链表

`dictEntry`总是将新增节点添加到链表头部(时间复杂度为 O(1)), 因为没有tail指针.

#### rehash

随着操作的不断进行、hash表维护的键值对会增加或减少, 为了让hash表的负载因子(load factor)维持在一个合理的额范围, 当hash表保存键值对数量太大或太小时, 程序需要对hash表扩容或缩容, 扩容或缩容通过`rehash`进行, `Rehash`步骤如下:

1. 为字典的ht[1]hash表分配空间, hash表的空间大小取决于要执行的操作, 及ht[0]当前包含的键值对的数量(即: ht[0].used 属性值), 若为扩容, ht[1] 为第一个大于等于 (ht[0].used`*`2)d的*2<sup>n</sup>; 若为缩容, ht[1] 为第一个大于等于 (ht[0].used) 的 2<sup>n</sup> 
2. 将ht[0]中的所有键值对rehash到ht[1]上, `rehash`是指: 重新计算键的hash值和索引值, 然后将键值对放到ht[1]hash表的对应位置
3. 当ht[0]包含的所有键值对迁移到ht[1]之后(ht[0]变为空表), 释放ht[0], 将ht[1]设置为 ht[0], 并在 ht[1]新创建一个空白hash表, 为下一次rehash做准备

```
eg. 扩容操作:
ht[0].used = 4, 4*2=8, 恰好是第一个大于等于4的2的n次方, 所以rehash后、ht[1]size为8

缩容操作:
ht[0].used = 4, 4=2², 恰好是第一个大于等于4的2的n次方, rehash后 ht[1].size = 4
```



##### hash表的扩展与收缩

满足下述任意条件时, 会自动扩展:

1. server 当前未执行`bgsave` 或者 `bgrewriteaof`, 且hash表的负载因子大于等于1

2. server 当前正在执行`bgsave` 或者 `bgrewriteaof`, 且 hash表的负载因子大于等于 5

   (持久化时、优先持久化操作)

```c
load_factor = ht[0].used / ht[0].size; // 负载因子 = hash表已保存节数量 / hash表大小
```

**若`bgsave`或者`bgrewriteaof`执行时, Redis需要创建当前服务器进程的子进程, 大多数操作系统使用写时复制(`copy-on-write`)优化子进程使用效率, 所以子进程存在期间, Server会提高rehash操作的负载因子, 尽可能避免此时rehash, 避免不必要的内存写入操作, 最大限度的节省内存.**

**另外:**

hash表的负载因子小于`0.1`时, 程序会自动缩容

#### 渐进式rehash

> 其实, hash表的rehash操作、不是一次集中完成的, 而是多次迭代进行的. 因为若 ht[0]里保存的key有百万计, 甚至更多, 一次rehash导ht[1], 庞大的计算量可能会导致Server一段时间的停止服务.

渐进式rehash步骤如下:

1. 为ht[1]分配空间, 让字典同时持有ht[0] 和 ht[1]两个hash表
2. 在字典中维护一个索引计数器变量 `rehashidx`, 设置为0, 表示`rehash`已经开始
3. 在`rehash`期间, 每次对字典`add`, `del`, `find` or `update`操作, 程序除了执行指定操作外, 还会顺带将 ht[0] hash表在 `rehashidx`索引上的所有键值对 rehash 到 ht[1], rehash完成、将`rehashidx++`.
4. 随字典操作的不断执行、某个时间点, ht[0]的所有键值对rehash到ht[1], rehash完成、将 `rehashidx`设为 `-1`.

渐进式rehash过程中会维护 ht[0] 和 ht[1] 两个hash表, 字典的删除、查找、更新等操作都会在两个hash表上进行, eg. 查找: 会先在 ht[0] 上查找、若未找到则 到 ht[1] 上查找

但: 插入操作只会在 ht[1] 上操作

```c
ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
```



##### 字典API

| 函数             | 作用                                   | 时间复杂度        |
| ---------------- | -------------------------------------- | ----------------- |
| dictCreate       | 创建一个新的字典                       | O(1)              |
| dictAdd          | 将给定键值对添加到字典                 | O(1)              |
| dictReplace      | 添加or替换                             | O(1)              |
| dictFetchValue   | 返回给定键的值                         | O(1)              |
| dictGetRandomKey | 随机返回一个键值对                     | O(1)              |
| dictRelease      | 释放给定字典, 及字典中包含的所有键值对 | O(N), N为值的数量 |



#### dict总结

* 广泛用于实现Redis的各种功能, 包括数据库和hash键
* 作为hash表的底层实现, 每个字典有两个hash表、一个平时使用, 一个 rehash 时使用
* 使用 `Murmurhash2` 计算key的hash值
* 使用链地址法解决键冲突、同一个索引上的多个键值对连接成一个单链表
* rehash是渐进式完成、非一次性工作

 







































