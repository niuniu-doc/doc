---
title: hbase
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---

### hbase

#### install
```
1. brew install hbase
2. http://hbase.apache.org/book.html#quickstart
   tar xzvf hbase-3.0.0-SNAPSHOT-bin.tar.gz
   cd hbase-3.0.0-SNAPSHOT/
```

#### modify config
```
1. conf/hbase-env.sh 添加Java jdk路径
vim xxxx/libexec/conf/hbase-env.sh

export JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home"

2. conf/hbase-site.xml 添加hbase文件存储路径

<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>file:///usr/local/var/hbase</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.clientPort</name>
    <value>2181</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/usr/local/var/zookeeper</value>
  </property>
</configuration>
```

#### start server
```
/usr/local/Cellar/hbase/{version}/libexec/bin/start-hbase.sh
```

#### start shell
```
hbase shell
```

#### command
#### base command
```
status 查询服务器的状态
version 查询hbase版本
whoami 查看连接用户

list 查询库中所有表
```

#### DDL
```
1. create
   create 'table_name', 'cf1', 'cf2'

2. drop
   disable 'table_name' 删除之前需要先让表失效、修改表结构是也是
   drop 'table_name' 删除
   
3. exists
   exists 'table_name' 查看表是否存在

4. describe 查看表结构
   describe 'table_name'

5. enable 使表有效
   enable 'table_name'

6. modify table structure
   1) alter 'table_name' 'cf3'   # 新增列族
   2) alter 'test', {NAME=>'cf3', METHOD=>'delete'} # 删除列族
```

#### DML
```
1. add
put 'table_name', 'rowkey', '列族名 1: 列名 1', 'value' #同一个rowkey、执行两次put、认为是更新操作
eg. put 'test','row_key1','cf1:a','a'

2. incr

3. count 查询表的行数(比较耗时)
   count 'table_name'

4. select
    1) get 'table_name', 'rowkey', '列族名：列名' #查询指定列族的指定列的值
       eg. get 'test', 'row_key1', 'cf1:a,cf4' 
       
    2) get 'table_name', 'rowkey' #获取指定row_key的所有数据
       eg. get 'test', 'row_key1'
       
    3) get 'table_name', 'rowkey', {COLUMN=>'列族名：列', TIMESTAMP=>1373737746997} # 获取指定时间戳的数据
        get 'test', 'row_key1', {COLUMN=>'cf1:a',TIMESTAMP=>1559367213477}
        
    4) get 'table_name', 'rowkey', {COLUMN => '列族名：列名', VERSIONS => 2} 获取多个版本值、默认返回第一个
       get 'test', 'row_key1', {COLUMN=>'cf1:a',VERSIONS=>2}

5. delete

1) delete 'table_name', 'rowkey', '列族名：列名' 删除指定rowkey的指定列族的列名数据
   eg. delete 'test','row_key1','cf1:a'

2) delete 'table_name', 'rowkey', '列族名'  删除指定rowkey指定列族的数据
   eg. delete 'test','row_key1','cf1'

3) deleteall 'table_name', ’rowkey' 删除rowkey所有column的calue、删除整行数据

6. scan 'table_name' 全表扫描
   scan 'test'

7. truncate 'table_name' 删除全表数据

8. hbase shell test.hbaseshell 执行shell脚本

```

#### 说明
```
1. 行以 rowkey 作为唯一标识、最大长度 64KB
   不支持order by只能按照rowkey或者range或者全表扫描

2. 列族是列的集合 列族:列

3. HBase 通过 row 和 column 确定一份数据
   tableName + RowKey + ColumnKey + Timestamp => value 唯一确定。Cell 中数据没有类型，字节码存储

```

#### reference
[http://einverne.github.io/post/2017/02/hbase-introduction-and-use.html](http://einverne.github.io/post/2017/02/hbase-introduction-and-use.html)

[http://einverne.github.io/post/2017/02/hbase-shell-command.html](http://einverne.github.io/post/2017/02/hbase-shell-command.html)
