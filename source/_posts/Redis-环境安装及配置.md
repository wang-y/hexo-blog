---
title: Redis 环境安装及配置
date: 2017-07-24 10:04:02
categories: 技术
tags: 
    - redis
---
# 安装
## 所需环境
1. gcc
```
#yum install gcc
```
<!-- more -->

## 安装Redis
```
#tar xzf redis-2.6.14.tar.gz 
#mv redis-2.6.14 redis
#cd redis
#make PREFIX=/home/xxx/redis install #安装到指定目录中
```
在安装redis成功后，你将可以在/home/xxx/redis看到一个bin的目录，里面包括了以下文件:
1. redis-benchmark  
2. redis-check-aof  
3. redis-check-rdb  
4. redis-cli  
5. redis-sentinel  
6. redis-server

# 服务
## 制作成服务
```
#cp utils/redis_init_script /etc/rc.d/init.d/redis
```
>如果这时添加注册服务：chkconfig --add redis 将报以下错误：redis服务不支持chkconfig(为此，我们需要更改redis脚本.)

## 更改redis脚本
```
打开使用vi打开脚本，查看脚本信息：vi /etc/init.d/redis
看到的内容如下(下内容是更改好的信息)： 
#!/bin/sh 
#chkconfig: 2345 80 90 
# Simple Redis init.d script conceived to work on Linux systems 
# as it does use of the /proc filesystem. 
   
REDISPORT=6379 
EXEC=/home/xxx/redis/bin/redis-server
CLIEXEC=/home/xxx/redis/bin/redis-cli

   
PIDFILE=/var/run/redis_${REDISPORT}.pid 
CONF="/etc/redis/${REDISPORT}.conf" 
   
case "$1" in 
    start) 
        if [ -f $PIDFILE ] 
        then 
                echo "$PIDFILE exists, process is already running or crashed" 
        else 
                echo "Starting Redis server..." 
                $EXEC $CONF & 
        fi 
        ;; 
    stop) 
        if [ ! -f $PIDFILE ] 
        then 
                echo "$PIDFILE does not exist, process is not running" 
        else 
                PID=$(cat $PIDFILE) 
                echo "Stopping ..." 
                $CLIEXEC -p $REDISPORT shutdown 
                while [ -x /proc/${PID} ] 
                do 
                    echo "Waiting for Redis to shutdown ..." 
                    sleep 1 
                done 
                echo "Redis stopped" 
        fi 
        ;; 
    *) 
        echo "Please use start or stop as first argument" 
        ;; 
esac 
```

> 修改EXEC、CLIEXEC参数： 
>> #chkconfig: 2345 80 90 （添加）
>> EXEC=/home/xxx/redis/bin/redis-server
>> CLIEXEC=/home/xxx/redis/bin/redis-cli
>> $EXEC $CONF & (注意后面的那个“&”，即是将服务转到后面运行的意思，否则启动服务时，Redis服务将占据在前台，占用了主用户界面，造成其它的命令执行不了)
> 将redis配置文件拷贝到/etc/redis/${REDISPORT}.conf 
>> mkdir /etc/redis 
>> cp redis.conf /etc/redis/6379.conf

这样，redis服务脚本指定的CONF就存在了。默认情况下，Redis未启用认证，可以通过开启6379.conf的requirepass 指定一个验证密码。 
以上操作完成后，即可注册yedis服务：chkconfig --add redis
启动redis服务 service redis start 

## 将Redis的命令所在目录添加到系统参数PATH中
```
 #vi /etc/profile  # 修改profile文件
 #export PATH="$PATH:/home/xxx/redis/bin"  # 在最后行追加
 #source /etc/profile  # 应用这个文件
```
这样就可以直接调用redis-cli的命令了，如下所示： 
> $ redis-cli   
至此，redis 就成功安装了。

# 修改默认配置
```
vi /etc/redis/6379.conf

daemonize yes #默认为on。yes为转为守护进程，否则启动时会每隔5秒输出一行监控信息  
save 900 1 #900秒内如果有一个key发生变化时，则将数据写入进镜像  
maxmemory 256000000 #分配256M内存 
requirepass ori18502800930(密码)
#bind 127.0.0.1(注释掉bind)
protected-mode no(禁用保护模式)
```
