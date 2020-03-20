---
title: 索引简单使用
date: 2020-03-20
categories:
  - Mongo
tags:
  - Mongo
---
###索引管理
#### 索引创建
```
db.COLLECTION_NAME.ensureIndex(keys[,options])

keys: 要建立索引的参数列表 key字段名、1升序 -1降序

options: 可选参数
background: Boolean 在后台建立索引
unique: boolean 创建唯一索引、默认false
name String指定索引名称
dropDups Boolean创建唯一索引时、若若出现重复、删除后续出现的相同索引、只保留第一个
sparse Boolean 对文档中不存在的字段数据不启用索引、默认false
v index version 索引的版本号
weights document索引权重值、表示改索引相对其它索引字段的得分权重值
```

#### 重建索引
```
db.collectionName.reIndex()
作用类似于mysql的optimize、修复多次修改产生的文件空洞
```

#### 查看索引
```
db.collectionName.getIndexes()
查看索引大小
```

#### 删除索引
```
db.collectionName.dropIndex(IndexName)

删除所有索引
db.collectionName.dropIndexes;

```


### 基础索引与复合索引
#### 基础索引
```
1. 为一个集合中的某个字段创建索引
eg. db.useers.ensureIndex({age:1})

2. 当数据很多时、索引创建会比较慢、可以指定 background: true
eg. db.users.ensureIndex({age:1}, {background: true})
```

#### 组合索引
```
为users表的age和city字段、创建联合索引、分表按照升序和降序排列
eg. db.useers.ensureIndex({age:1, city:-1})
```

#### 查看索引
```
db.dbname.getIndexes() 返回当前集合所有的索引
1:升序  -1:降序
```

### 文档索引
```
1. mongodb可以为多个字段创建索引、当字段是子文档时、同样可以创建
eg. mongo中有下列数据
{name:"aaa", address:{city:"bj",district:"海淀"}}

可以为address创建索引如下:
db.users.ensureIndex({address:1})

建立索引后、查询时子文档的字段顺序要和查询顺序一致、才可使用索引

// 会使用索引
db.users.find({address:{city:"bj",district:"海淀"}})

// 不会使用索引
db.users.find({address:{district:"海淀",city:"bj"}})

也可以只对子文档的某一个或者部分字段创建索引
db.users.ensureIndex({"address.city":1})
```

### 唯一索引与强制索引
#### 创建唯一索引
```
添加索引时、指定 unique:true 等效于唯一索引
eg. 为users表的email字段添加唯一索引
db.users.ensureIndex({email:1},{unique:true})

创建唯一索引后、插入重复值时、mongo会报错..
```

#### 强制使用索引
```
db.users.find({name:'aaa', age:3}).hint({age:1})
```
