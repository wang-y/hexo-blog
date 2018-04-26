---
title: Hadoop 3.1.0 安装教程
date: 2018-04-26 15:21:28
categories: hadoop
tags:
    - hadoop
---


# 安装依赖环境

- Java
- ssh

```shell
$ sudo apt-get install ssh
$ sudo apt-get install pdsh
```

# 搭建Hadoop Cluster

## 准备工作

解压Hadoop压缩包。然后进入目录。编辑etc/hadoop/hadoop-env.sh的下列参数：
设置JAVA_HOME

```
# set to the root of your Java installation
export JAVA_HOME=/usr/java/latest 
```

保存退出后，执行以下命令:

```
$ bin/hadoop  
```

它将显示hadoop脚本可用的文件。
现在您可以开始从下列三种支持的Hadoop cluster模式中选择一种来进行环境搭建了：

- 本地（单机）模式
- 伪分布式模式
- 分布式模式

## 单机模式

默认情况下，Hadoop的配置是非分布式模式，就像一个Java进程一样，它有利于进行debug调试。
您可以复制下个例子中的代码到解压路径下进行执行根据指定的表达式，匹配查找每次输入的结果，并且对其进行输出到output路径下。

```shell
$ mkdir input  
$ cp etc/hadoop/*.xml input  
$ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.0.jar grep input output 'dfs[a-z.]+'  
$ cat output/*  
```

## 伪分布式设置

Hadoop也是支持在单节点上配置伪分布式模式的。在这样的情况下，Hadoop的守护进程都是单独的java进程。

### 配置

etc/hadoop/core-site.xml:
```xml
<configuration>  
    <property>  
        <name>fs.defaultFS</name>  
        <value>hdfs://localhost:9000</value>  
    </property>
    <property>  
        <name>hadoop.tmp.dir</name>  
        <value>/home/hadoop/hadoopdata</value>  
        <description>Abase for other temporary directories.</description>  
    </property>  
</configuration> 
```

etc/hadoop/hdfs-site.xml:
```xml
<configuration>  
    <property>  
        <name>dfs.replication</name>  
        <value>1</value>  
    </property>  
</configuration>  
```

### 配置ssh密码

现在检查是否可以在没有密码的情况下ssh到本地主机：

```shell
$ ssh localhost
```

如果不能再没有密码的情况下ssh到本地主机，请执行一下命令
```shell
$ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa  
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys  
$ chmod 0600 ~/.ssh/authorized_keys  
```

修改etc/hadoop/workers，注意：workers这个文件就是hadoop2.x版本中的slaves。
```
hadoop000  #主机名  
```

格式化文件系统：
```shell
$ bin/hdfs namenode -format 
```

启动NameNode守护进程和DataNode守护进程：
```shell
$ sbin/start-dfs.sh
```

hadoop守护进程日志输出到$HADOOP_LOG_DIR 路径 (默认为 $HADOOP_HOME/logs)。
通过浏览器访问NameNode的web接口，默认地址为:http://localhost:9870/。
创建需执行MapReduce jobs的HDFS目录：
```shell
$ bin/hdfs dfs -mkdir input  
$ bin/hdfs dfs -put etc/hadoop/*.xml input  
```

复制input文件到分布式文件系统:
```shell
$ bin/hdfs dfs -mkdir input  
$ bin/hdfs dfs -put etc/hadoop/*.xml input  
```

运行提供的示例程序:
```shell
$ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.0.jar grep input output 'dfs[a-z.]+'  
```

检查输出文件:从分布式文件系统复制output文件到本地然后进行检查:
```shell
$ bin/hdfs dfs -get output output  
$ cat output/*  
```
或者直接在分布式文件系统中进行查看:
```shell
$ bin/hdfs dfs -cat output/*  
```

当您已经完成上面所有的工作的时候，您可以停止守护进程了:
```shell
$ sbin/stop-dfs.sh  
```

## YARN上运行单点

您可以在单点模式的YARN上运行MapReduce job。您只需要增加ResourceManager守护进程和NodeManager守护进程设置就可以了。
其实现方式通过下面步骤进行配置：

修改配置参数:
etc/hadoop/mapred-site.xml:
```xml
<configuration>  
    <property>  
        <name>mapreduce.framework.name</name>  
        <value>yarn</value>  
    </property>
    <property>  
        <name>mapreduce.application.classpath</name>  
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>  
    </property>  
</configuration>  
```
etc/hadoop/yarn-site.xml:
```xml
<configuration>  
    <property>  
        <name>yarn.nodemanager.aux-services</name>  
        <value>mapreduce_shuffle</value>  
    </property>  
    <property>  
        <name>yarn.nodemanager.env-whitelist</name>  
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>  
    </property>  
</configuration>  
```

启动 ResourceManager守护进程和NodeManager守护进程:
```shell
$ sbin/start-yarn.sh  
```

通过浏览器访问ResourceManager的web接口，默认地址为: http://localhost:8088/

运行一个 MapReduce job.

当您已经完成上面所有的工作的时候，您可以停止守护进程了:
```shell
$ sbin/stop-yarn.sh  
```

# 异常问题解决：

如果启动HDFS时出现以下错误：

```
Starting namenodes on [hadoop000]  
ERROR: Attempting to operate on hdfs namenode as root  
ERROR: but there is no HDFS_NAMENODE_USER defined. Aborting operation.  
Starting datanodes  
ERROR: Attempting to operate on hdfs datanode as root  
ERROR: but there is no HDFS_DATANODE_USER defined. Aborting operation.  
Starting secondary namenodes [hadoop000]  
ERROR: Attempting to operate on hdfs secondarynamenode as root  
ERROR: but there is no HDFS_SECONDARYNAMENODE_USER defined. Aborting operation.  
2018-04-13 07:37:05,439 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable 
```

解决办法：

```
$ vim start-dfs.sh  
$ vim stop-dfs.sh  
#分别在文档的最前面，添加如下参数：  
HDFS_DATANODE_USER=root  
HADOOP_SECURE_DN_USER=hdfs  
HDFS_NAMENODE_USER=root  
HDFS_SECONDARYNAMENODE_USER=root   
```


如果启动YARN时出现以下错误：

```
Starting resourcemanager  
ERROR: Attempting to operate on yarn resourcemanager as root  
ERROR: but there is no YARN_RESOURCEMANAGER_USER defined. Aborting operation.  
Starting nodemanagers  
ERROR: Attempting to operate on yarn nodemanager as root  
ERROR: but there is no YARN_NODEMANAGER_USER defined. Aborting operation.  
```

解决办法：

```
$ vim start-yarn.sh  
$ vim stop-yarn.sh  
#分别在两个文件最前端，添加如下参数：  
YARN_RESOURCEMANAGER_USER=root  
HADOOP_SECURE_DN_USER=yarn  
YARN_NODEMANAGER_USER=root     
```