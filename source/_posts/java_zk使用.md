---
title: zk使用
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---

# 参考配置项
`tickTime` 用于计算的时间单元、eg. session超时、N*tickTime
`initLimit` 用于集群、允许从节点连接、并同步到master节点的初始化连接时间、以tickTime的倍数来表示
`syncLimit` 用于集群、master主节点与从节点之间发送消息、请求和应答的时间长度(心跳机制)
`dataDir` 数据文件目录、必须
`dataLogDir` 日志目录、非必须、默认`dataDir`
`clientPort` 连接服务器的端口、默认 `2181`

# client 连接
`zkCli.sh` 默认连接2181

# client 命令
`ls` 与linux下`ls`同义
`ls` 等同于  `ls` + `stat`
`stat` 状态显示 zZxid zookeeper为数据分配的id pZxid 子节点的id
`get` 查看节点数据
`create` 创建节点 -e 临时节点
`set` 修改节点数据
`delete` 删除节点数据

# ACL
构成: scheme: 采用的权限机制
(word:anyone:[permissions] | 
auth:user:password:[permissions] | 
digest:user:BASE64(SHA1(pass)):[permissions]) 
(ip:192.168.1.1:[permission])
(super: 代表超管、拥有所有的权限)

id: 代表允许访问的机制 

permissons: 权限

权限字符缩写 crdwa
create: 创建子节点
read: 获取节点、子节点
write: 设置节点数据
delete: 删除子节点
admin: 设置权限

`getAcl` 获取节点的acl权限信息
`setAcl` 设置某个节点的acl权限信息
`addAuth` 输入认证权限信息、注册时、输入明文密码(登录)、但在zk的系统里、是以密文存在的

# four letter cmd
`stat` 当前节点的状态信息
`ruok` 当前节点是否ok
`conf` 查看服务器相关的配置
`cons` 展示连接到server的client信息
`envi` 打印环境变量信息
`mntr` 监控zk的健康信息
`wchs` watcher的信息
`wchc` session与watch对应关系信息
`wchp` path与watch对应关系
