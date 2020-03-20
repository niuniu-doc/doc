---
title: curd操作
date: 2020-03-20
categories:
  - Mongo
tags:
  - Mongo
---

#### 聚合操作

`$sum`: 计算总和
```
db.mycol.aggregate([{$group : {_id:"$f1", num_tutorial:{$sum:"$f2"}}}])
-> select count(1) from table group by f1; 

db.mycol.aggregate([{$group : {_id:"$f1", num_tutorial:{$sum:"$f2"}}}])
->select field,sum(f2) from table group by f1;

```
`$avg`: 计算平均值
```
db.test.aggregate([{$group:{_id:"$field", avg:{$avg:$f}}}])
-> select field,avg(f) from table group by field;
```
`$min`: 获取集合中所有文档对应的最小值
```
db.test.aggregate([{$group:{_id:"$field",min:{$min:"$f2"}}}])
-> select min(f2) from table group by field;
```
`$max`: 获取集合中所有文档对应的最大值
```
参考min
```
`$push`: 在结果文档中插入值到一个数组
```
db.t.aggregate([{$group:{_id:"$by_user",url:{$push:"$url"}}}])
```
`$first`: 根据资源文档的排序获取第一个文档数据
```
db.t.aggregate([{$group:{_id:"$by_user",first_url:{$first:"$url"}}}])
```
`$last`: 根据资源文档的排序获取最后一个文档数据
```
参考first
```

`$project` 修改输出文档的结构
`$match` 用于过滤数据、
`$limit` 限制返回条数
`$skip` 跳过指定条数的文档
`$unwind` 将某个数组类型字段拆分成多条
`$group` 将集合中的文档分组、可用于统计结果
`$sort` 将文档排序后输出

eg.
```
db.t.aggregate({$project:{_id:0,title:1,tags:1}})
db.t.aggregate({$project:{_id:0,title:1,tags:1}},[{$match:{score:{$gt:70,$lte:90}}},{$group:{_id:null,count:{$sum:1}}}])

```

#### mapReduce
```
map ：映射函数 (生成键值对序列,作为 reduce 函数参数)。
reduce 统计函数，reduce函数的任务就是将key-values变成key-value，也就是把values数组变成一个单一的值value。。
out 统计结果存放集合 (不指定则使用临时集合,在客户端断开后自动删除)。
query 一个筛选条件，只有满足条件的文档才会调用map函数。（query。limit，sort可以随意组合）
sort 和limit结合的sort排序参数（也是在发往map函数前给文档排序），可以优化分组机制
limit 发往map函数的文档数量的上限（要是没有limit，单独使用sort的用处不大）
```
