---
title: 常用命令
date: 2020-03-20
categories:
  - Mongo
tags:
  - Mongo
---
#### 查看版本
```
db.version()
mongo --version
mongod --version
```

#### 查看各节点的健康状态
```
复制集状态查看  rs.status()
查看oplog状态 rs.printReplicationInfo();
查看复制延迟 rs.printSlaveReplicationInfo();
查看服务状态详情 db.serverStatus();
```
#### 查看数据库空间大小
```
db.stats() 默认返回字节
"dataSize" : 20,  //所有数据的总大小
"storageSize" : 21, //所有数据占的磁盘大小 
"fileSize" : 23,  //预分配给数据库的文件大小

db.stats(1073741824) // 得到的单位是 G
这里的objects以及avgObjSize还是bytes为单位的，不受参数影响
```

#### 查看总的记录数
```
db.dbname.find().count()
db.dbname.find().skip(n).limit(m).count(); 会返回所有的记录数
db.dbname.find().skip(n).limit(m).count(true); 会返回limit的记录数
```

#### 查看数据库信息
```
db.getName 返回当前所在的db名称
```

#### 允许副本集上的查询
```
rs.slaveOk()
```

#### 参考:
```
https://www.cnblogs.com/kevingrace/p/8178549.html
https://stackoverflow.com/questions/30088833/strange-mongodb-and-mongoose-error-not-master-and-slaveok-false-error/30089063
```
