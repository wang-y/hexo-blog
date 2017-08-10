---
title: MongoDB分片
date: 2017-08-10 16:39:48
categories: 技术
tags:
    - MongoDB
    - 复制
    - 分片
---
# 复制(副本集)

条件限制，所有实例都是单机，故需要配置hosts
```
127.0.0.1 replication0
127.0.0.1 replication1
127.0.0.1 replication2
```
<!-- more -->
**启动一个MongoDB服务**
```
mongod --port "PORT" --dbpath "YOUR_DB_DATA_PATH" --replSet "REPLICA_SET_INSTANCE_NAME"

--port 指定端口
--dbpath 指定数据库路径
--replSet 指定复制实例名称
```

例：
```
> mongod --port 1111 --dbpath /usr/rs0/data --replSet rstest
> mongod --port 1112 --dbpath /usr/rs1/data --replSet rstest
> mongod --port 1113 --dbpath /usr/rs2/data --replSet rstest
```

**启动后链接至某一个实例**
```
> mongo --port 1111

#启动一个新的副本集
> rs.initiate() 

#添加副本集成员
> #rs.add(HOST_NAME:PORT)
> rs.add("replication1:1112")
> rs.add("replication2:1113")

#查看副本集配置
> rs.conf()

#查看副本集状态
> rs.status()

#查看当前节点是否为主节点
> db.isMaster()
```

在副本集中 主节点宕机后，副节点会接管主节点成为新的主节电，所以不会出现宕机的情况。

# 分片

同样，现修改hosts
```
127.0.0.1 conf0
127.0.0.1 conf1
```

**启动Shard Server**

用于存储实际的数据块，实际生产环境中一个shard server角色可由几台机器组个一个replica set承担，防止主机单点故障
```
> mongod --port 1111 --dbpath /usr/rs0/data --shardsvr
> mongod --port 1112 --dbpath /usr/rs1/data --shardsvr
> mongod --port 1113 --dbpath /usr/rs2/data --shardsvr
```

**启动Config Server**

mongod实例，存储了整个 ClusterMetadata，其中包括 chunk信息。

MongoDB 3.4 后需要搭建副本集
```
> mongod --port 2222 --dbpath /usr/config0/data --configsvr --replSet conf
> mongod --port 2223 --dbpath /usr/config1/data --configsvr --replSet conf

> mongo --port 2222
> rs.initiate() 
> rs.add("conf1:2223")
```

**启动Query Routers**

前端路由，客户端由此接入，且让整个集群看上去像单一数据库，前端应用可以透明使用。

```
> mongos --port 3333 --configdb conf/conf0:1111,conf1:2223 --logpath=/usr/router/logs/router.log

> mongo admin --port 3333  #连接到Router
> db.runCommand({addshard:"127.0.0.1:1111"})  #添加Shard节点
> db.runCommand({addshard:"127.0.0.1:1112"})  #添加Shard节点
> db.runCommand({addshard:"127.0.0.1:1113"})  #添加Shard节点
> use test  #创建test数据库
> db.col.insert({"name":"data1"})  #创建col集合，并插入一条数据
> use admin  #使用admin库
> db.runCommand({ enablesharding:"test"})   #设置分片存储的数据库
> db.runCommand({shardcollection: "test.col", key: {id:1}})  #设分片集合，并设置分片字段
```

测试
```
> mongo admin --port 3333  #连接到Router
> use test
> for(var i=0;i<100000;i++){
> db.col.insert({"name":"data"+i}) 
> }
> 
> db.printShardingStatus()  #查看分片状态
```