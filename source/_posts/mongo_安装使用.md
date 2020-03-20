---
title: 安装使用
date: 2020-03-20
categories:
  - Mongo
tags:
  - Mongo
---


#### 概念
```
database 数据库
collection 数据表
document 数据记录行/文档
filed 数据域
index 索引
primary key 主键、默认 _id
```

#### 基本命令
```
1. show dbs 查看所有数据的列表
   db 显示当前数据库对象或者集合
   use local 切换到local这个db
2. use niuniu  db不存在时会自动创建
3. db.dropDatabase() 删除一个db

4. db.createCollection("niu") # 先创建集合、类似db中的表
   show tables/collections 查看所有的表
   db.niu.drop() # 删除一个table

5. db.niu.insert({"name":"niu"}) 插入文档
6. db.niu.save() 若不指定id, 类似于insert、若指定、会执行更新
7. db.help() 库的帮助信息
8. db.table.help() 表的帮助文档

```

#### 远程连接
```
mongo 192.168.1.100:27017/test

```
