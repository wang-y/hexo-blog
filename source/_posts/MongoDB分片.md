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

MongoDB 副本集（Replica Set）是有自动故障恢复功能的主从集群，有一个Primary节点和一个或多个Secondary节点组成。

副本集中数据同步过程：Primary节点写入数据，Secondary通过读取Primary的oplog得到复制信息，开始复制数据并且将复制信息写入到自己的oplog。如果某个操作失败，则备份节点停止从当前数据源复制数据。如果某个备份节点由于某些原因挂掉了，当重新启动后，就会自动从oplog的最后一个操作开始同步，同步完成后，将信息写入自己的oplog，由于复制操作是先复制数据，复制完成后再写入oplog，有可能相同的操作会同步两份，不过MongoDB在设计之初就考虑到这个问题，将oplog的同一个操作执行多次，与执行一次的效果是一样的。

当Primary节点完成数据操作后，Secondary会做出一系列的动作保证数据的同步：

1. 检查自己local库的oplog.rs集合找出最近的时间戳。
2. 检查Primary节点local库oplog.rs集合，找出大于此时间戳的记录。
3. 将找到的记录插入到自己的oplog.rs集合中，并执行这些操作。

副本集的同步和主从同步一样，都是异步同步的过程，不同的是副本集有个自动故障转移的功能。其原理是：slave端从primary端获取日志，然后在自己身上完全顺序的执行日志所记录的各种操作（该日志是不记录查询操作的），这个日志就是local数据 库中的oplog.rs表，默认在64位机器上这个表是比较大的，占磁盘大小的5%，oplog.rs的大小可以在启动参数中设 定：--oplogSize 1000,单位是M。

注意：在副本集的环境中，要是所有的Secondary都宕机了，只剩下Primary。最后Primary会变成Secondary，不能提供服务。


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
> mongos --port 3333 --configdb conf/conf0:2222,conf1:2223 --logpath=/usr/router/logs/router.log

> mongo admin --port 3333  #连接到Router
> db.runCommand({addshard:"127.0.0.1:1111"})  #添加Shard节点
> db.runCommand({addshard:"127.0.0.1:1112"})  #添加Shard节点
> db.runCommand({addshard:"127.0.0.1:1113"})  #添加Shard节点
> use test  #创建test数据库
> db.col.insert({"name":"data1"})  #创建col集合，并插入一条数据
> use admin  #使用admin库
> db.runCommand({ enablesharding:"test"})   #设置分片存储的数据库(一旦enable了个数据库，mongos将会把数据库里的不同数据集放在不同的分片上。只有数据集也被分片，否则一个数据集的所有数据将放在一个分片上。)
> db.runCommand({shardcollection: "test.col", key: {id:1}})  #设分片集合，并设置片键(shard key)
```

> shard key:片键；MongoDB不允许插入没有片键的文档。2.4版本以后MongoDB开始支持基于哈希的分片，但它仍然是基于范围的，只是将提供的片键散列成一个非常大的长整型作为最终的片键。
> 在部署之前先明白片键的意义，一个好的片键对分片至关重要。片键必须是一个索引，数据根据这个片键进行拆分分散。通过sh.shardCollection加会自动创建索引。一个自增的片键对写入和数据均匀分布就不是很好，因为自增的片键总会在一个分片上写入，后续达到某个阀值可能会写到别的分片。但是按照片键查询会非常高效。随机片键对数据的均匀分布效果很好。注意尽量避免在多个分片上进行查询。在所有分片上查询，mongos会对结果进行归并排序。

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

## 数据平衡器

如果存在多个可用的分片，只要块的数量足够多，MongoDB就会把数据迁移到其他分片上，这个迁移过程叫做平衡(balancing)，由叫做平衡器(balancer)的进程负责执行。

### 概念

均衡器会周期的检查分片是否存在不平衡，每隔几秒，mongos就会尝试成为平衡器，s会对整个集群加锁，以防止配置服务器对集群进行修改。均衡并不会影响Mongos的正常路由，使用Mongos的客户端不会受到影响.

当Mongos成为均衡器之后，判断各个分片中使用块的数量，不关心数据量的大小。只有在分片数量不对称的时候，才会启动。移动过程发生在块数最大与最小的分片之间。

在数据的迁移过程中，所有的读写请求会被路由到旧的块中。一旦元数据更新之后，指向旧块的读操作将会失败，这个时候Mongos会将这些请求到新块中，即再次执行请求，这部分工作对于用户来说是透明的。

### 数据迁移过程

1. 平衡器进程发送movechunk命令到源分片中
2. 原分片开始启动moveChunk命令，在移动的过程中，所有的操作都还是会指向原来的分片。
3. 目标分片开始创建所需要的索引
4. 目标分片开始向原分片请求数据，并复制数据
5. 当数据全部写入到目标分片中，目标分片连接并更新config数据库中的对应块的元数据信息
6. 最后，原分片将这部分块删除。

在3.4版本中，mongodb可以并行的移动块，如果有N个块的分片，可以同时进行n/2的块迁移任务。

为了将平衡器产生的影响降低到最小，根据块数的不同，阈值也会不同。当分片中的集合数目小于2或者移动出现失败，平衡器会停止。平衡器不会等待当前块删除之后开始新的块移动。在一些情况下，删除过程会持续很久，如果很多的删除过程在排队，没有完成，那么副本集中的主节点可能会出现问题。

此在分片集群中经常会出现:

```
moveChunk failed to engage TO-shard in the data transfer: can'taccept new chunks because there are still XXX deletes from previous migration
```

可以采用_waitForDelete，等待上一个操作结束后，再进行下一个数据块的移动。但是这个参数的设置将会使得后面的操作被阻塞，如果等待删除的块比较多，最好的方式是那个可以解决很多问题的步骤：重启。

### 影响

均衡过程会增加系统负载：目标分片必须查询源分片块中的所有文档，将文档插入目标块中，需要删除源分片。向集群中添加新分片时，均衡器会试图向该分片写入数据，会有一个数据迁移的过程。那么对应的措施:

1. 为均衡器指定一个时间窗口，指定在某个特定的时间内执行数据移动操作。
2. 手动进行均衡，但是这种做法要非常小心

### 特大块

如果块中的文档数超过了250,000或者块的大小大于配置文件中的chunkSize大小的1.3倍。

### 启动与禁止平衡器

默认情况下，在任何需要移动块的时候，平衡器都会进行工作。可以使用sh.stopBalancer()来禁止平衡器，通过sh.getBalancerState()来观察当前平衡器的状态。启动平衡器：sh.setBalancerState(true)。

如果mongodb正在移动数据，不要尝试去改变分片集群中的元数据，不应该采取任何可能会修改元数据的操作。比如，创建或者删除数据库，创建或者删除集合，或者使用任何分片的命令，否则可能会得到不一致的快照结果。

当平衡器在运行的时候也不要进行备份，应该将其禁止。可以设置平衡窗口，这样子在备份期间得到的数据才会是正确的。但是在禁止平衡器时，它不一定马上就停止，会等到当前的平衡过程终止。所以在备份之前一定要确保平衡器没有在运行。对应的命令为：sh.getBalancerState()，sh.isBalancerRunning()。

禁止平衡器的操作可以应用于某个特定的集合。Sh.enableBalancing()。利用这个操作是否可以实现在运行期间，将数据库禁止平衡，但是允许某个集合进行平衡，可以对正在运行的数据库进行备份，不影响整体的性能。那么是否可以设置某个集合禁止平衡。

