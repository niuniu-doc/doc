---
title: Redis数据结构-链表
date: 2020-04-24 19:25:32
tags: Redis
categories: Redis
---

> 链表提供了高效的节点重排能力, 及顺序性节点访问方式, Redis构建了自己的链表实现



#### 链表和链表节点的实现

```c
typedef struct listNode{
  struct listNode *prev; // 前置节点
  struct listNode *next; // 后置节点
  void *value; // 节点值
}listNode;
```

多个`listNode`节点通过 `prev` 和 `next`指针组成双端链表, 虽然仅使用多个listNode就可以组成链表, 使用 `adlist.h/list` 来持有链表会更方便操作.

```c
typedef struct list{
  listNode *head; // 表头节点
  listNode *tail; // 表尾结点
  unsigned long len; // 链表包含的节点数量
  void *(*dup)(void *ptr); // 节点值复制函数
  void (*free)(void *ptr); // 节点值释放函数
  int (*match)(void *ptr, void *key); 节点值对比函数
} list;
```

list结构为链表提供了表头指针head、表尾指针tail, 及链表长度计数器len, 而udp, free, match成员则用于实现多态链表所需特定函数.

##### Redis 链表实现特性总结

* 双端: 有`prev`和`next`指针, 获取前置和后置节点的复杂度都是O(1)
* 无环: 表头的`prev`和表尾的`next`都指向`NULL`, 对链表的访问以`NULL`为终点
* 带表头和表尾指针: 通过list结构的head和tail指针获取链表的头结点和尾结点复杂度为 O(1)
* 带长度计数器: 程序使用list结构的len属性对list节点计数, 获取节点数量的复杂度O(1)
* 多态: 链表节点使用 `void *`指针保存节点值, 可通过list结构的`dup`, `free`, `match`三个属性为节点值设置类型特定的函数, 所以链表可用来保存不同类型的节点值







































