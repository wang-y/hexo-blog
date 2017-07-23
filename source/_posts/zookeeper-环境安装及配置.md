---
title: zookeeper 环境安装及配置
date: 2017-07-23 16:22:02
categories: 技术
tags: 
    - zookeeper
---

# 安装及配置
## 所需环境
1. Java
<!-- more -->
```
#tar zxvf zookeeper-3.4.9.tar.gz
#cd zookeeper-3.4.9/conf/
#cp zoo_sample.cfg zoo.cfg

#修改zoo.cfg配置
#zookeeper 定义的基准时间间隔，单位：毫秒
tickTime=2000
initLimit=10
syncLimit=5
#数据文件夹
dataDir=/usr/local/zookeeper/data
#日志文件夹
dataLogDir=/usr/local/zookeeper/logs
#客户端访问 zookeeper 的端口号
clientPort=2181

#cd ../bin/
#vim /etc/profile
export ZOOKEEPER_HOME=/usr/local/zookeeper
export PATH=$ZOOKEEPER_HOME/bin:$PATH
```

# 常用命令及防火墙配置
```
 zkServer.sh start  # 启动
 zkServer.sh stop  # 停止
 zkServer.sh status  # 状态
 zkServer.sh restart  # 重启
```

```
firewall-cmd --zone=public --add-port=2181/tcp --permanent
firewall-cmd --reload
```

# 集群

1. 修改主机名:/etc/hostname
2. 修改/etc/hosts: 
    12.15.0.106 master
    12.15.0.108 slave
3. ookeeper配置中添加
    server.1=master:2888:3888
    server.2=slave:2888:3888
4. 在dataDir路径下创建一个myid文件。
5. 在myid文件中输入一个服务ID数字。