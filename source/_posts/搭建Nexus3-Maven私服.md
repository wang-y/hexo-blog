---
title: 搭建Nexus3 Maven私服
date: 2017-12-15 11:26:43
categories: 技术
tags: 
    - maven
    - nexus
---

# Nexus 介绍

Nexus 是一个强大的 Maven 仓库管理器，它极大地简化了本地内部仓库的维护和外部仓库的访问。

如果使用了公共的 Maven 仓库服务器，可以从 Maven 中央仓库下载所需要的构件（Artifact），但这通常不是一个好的做法。

<!-- more -->

正常做法是在本地架设一个 Maven 仓库服务器，即利用 Nexus 私服可以只在一个地方就能够完全控制访问和部署在你所维护仓库中的每个 Artifact。

Nexus 在代理远程仓库的同时维护本地仓库，以降低中央仓库的负荷, 节省外网带宽和时间，Nexus 私服就可以满足这样的需要。

Nexus 是一套 “开箱即用” 的系统不需要数据库，它使用文件系统加 Lucene 来组织数据。

Nexus 使用 ExtJS 来开发界面，利用 Restlet 来提供完整的 REST APIs，通过 m2eclipse 与 Eclipse 集成使用。

Nexus 支持 WebDAV 与 LDAP 安全身份认证。

Nexus 还提供了强大的仓库管理功能，构件搜索功能，它基于 REST，友好的 UI 是一个 extjs 的 REST 客户端，它占用较少的内存，基于简单文件系统而非数据库。

为什么要构建 Nexus 私服？

如果没有 Nexus 私服，我们所需的所有构件都需要通过 maven 的中央仓库和第三方的 Maven 仓库下载到本地，而一个团队中的所有人都重复的从 maven 仓库下载构件无疑加大了仓库的负载和浪费了外网带宽，如果网速慢的话，还会影响项目的进程。很多情况下项目的开发都是在内网进行的，连接不到 maven 仓库怎么办呢？开发的公共构件怎么让其它项目使用？这个时候我们不得不为自己的团队搭建属于自己的 maven 私服，这样既节省了网络带宽也会加速项目搭建的进程，当然前提条件就是你的私服中拥有项目所需的所有构件。

总之，在本地构建 nexus 私服的好处有：

- 加速构建；
- 节省带宽；
- 节省中央 maven 仓库的带宽；
- 稳定（应付一旦中央服务器出问题的情况）；
- 控制和审计；
- 能够部署第三方构件；
- 可以建立本地内部仓库；
- 可以建立公共仓库.

# 安装

## 所需环境

-  jdk1.8
-  maven

## 安装Nexus

1.在 [Nexus官网](https://www.sonatype.com/) 下载免费的OSS版本.

2.解压

```
tar -zxvf nexus-3.6.x-xx-unix.tar.gz
```

3.启动 nexus3

```
cd nexus-3.6.x-xx/bin/
./nexus start
```

4.开启远程访问端口

5.访问 http://ip:8081/ , 若网页能够正常访问就说明安装成功了.

```
nexus3默认端口是:8081
nexus3默认账号是:admin
nexus3默认密码是:admin123
```

6.设置开机自启动

```
#CentOS
ln -s /nexus-3.6.x-xx/bin/nexus /etc/init.d/nexus3
chkconfig --add nexus3
chkconfig nexus3 on
```

7.修改 nexus3 的运行用户为 root

```
vim nexus.rc

//设置
run_as_user="root"
```

8.修改 nexus3 启动时要使用的 jdk 版本

```
vim nexus

# 第 14 行:
INSTALL4J_JAVA_HOME_OVERRIDE=/path/jdk1.8.0_144
```

9.修改 nexus3 默认端口 (可选)

```
cd nexus-3.6.x-xx/etc/
vim nexus-default.properties

#默认端口: 8081
application-port=8081
```

10.修改 nexus3 数据以及相关日志的存储位置 (可选)：

```
[root@MiWiFi-R3-srv bin]# cd nexus-3.6.x-xx/bin/
[root@MiWiFi-R3-srv bin]# vim nexus.vmoptions

#添加或修改
-XX:LogFile=./sonatype-work/nexus3/log/jvm.log
-Dkaraf.data=./sonatype-work/nexus3
-Djava.io.tmpdir=./sonatype-work/nexus3/tmp
```

# 概念

**component name 的一些说明:**

1. maven-central: maven 中央库,默认从 repo1.maven.org/maven2/ 拉取jar;
2. maven-releases: 私库发行版 jar;
3. maven-snapshots: 私库快照(调试版本) jar;
4. maven-public: 仓库分组,把上面三个仓库组合在一起对外提供服务,在本地 maven 基础配置 settings.xml 中使用.

**Nexus 默认的仓库类型有以下四种:**

1. group(仓库组类型): 又叫组仓库，用于方便开发人员自己设定的仓库;
2. hosted(宿主类型): 内部项目的发布仓库(内部开发人员，发布上去存放的仓库);
3. proxy(代理类型): 从远程中央仓库中寻找数据的仓库(可以点击对应的仓库的 Configuration 页签下 Remote Storage Location 属性的值即被代理的远程仓库的路径);
4. virtual(虚拟类型): 虚拟仓库(这个基本用不到，重点关注上面三个仓库的使用).

**Policy(策略):**

1. 发布 (Release) 版本仓库;
2. 快照 (Snapshot) 版本仓库.

**Public Repositories 下的仓库:**

1. 3rd party: 无法从公共仓库获得的第三方发布版本的构件仓库,即第三方依赖的仓库,这个数据通常是由内部人员自行下载之后发布上去;
2. Apache Snapshots: 用了代理 ApacheMaven 仓库快照版本的构件仓库
3. Central: 用来代理 maven 中央仓库中发布版本构件的仓库
4. Central M1 shadow: 用于提供中央仓库中 M1 格式的发布版本的构件镜像仓库
5. Codehaus Snapshots: 用来代理 CodehausMaven 仓库的快照版本构件的仓库
6. Releases: 内部的模块中 release 模块的发布仓库，用来部署管理内部的发布版本构件的宿主类型仓库；release 是发布版本；
7. Snapshots: 发布内部的 SNAPSHOT 模块的仓库，用来部署管理内部的快照版本构件的宿主类型仓库；snapshots 是快照版本，也就是不稳定版本

所以自定义构建的仓库组代理仓库的顺序为：Releases，Snapshots，3rd party，Central。也可以使用 oschina 放到 Central 前面，下载包会更快。

**Nexus 仓库分类的概念**

1）Maven 可直接从宿主仓库下载构件, 也可以从代理仓库下载构件, 而代理仓库间接的从远程仓库下载并缓存构件
2）为了方便, Maven 可以从仓库组下载构件, 而仓库组并没有时间的内容 (下图中用虚线表示, 它会转向包含的宿主仓库或者代理仓库获得实际构件的内容).

![Alt text](../../../../images/nexus-1.png)

# Nexus 的 web 界面功能介绍

## Browse Server Content

![Alt text](../../../../images/nexus-2.png)

### Search

这个就是类似 Maven 仓库上的搜索功能，就是从私服上查找是否有哪些包。

### Browse

1）Assets
这是能看到所有的资源,包含 Jar,已经对 Jar 的一些描述信息.

2）Components
这里只能看到 Jar 包。

## Server Adminstration And configuration

看到这个选项的前提是要进行登录的，如上面已经介绍登陆方法，右上角点击 “Sign In” 的登录按钮，输入 admin/admin123, 登录成功之后，即可看到此功能，如图所示：

![Alt text](../../../../images/nexus-3.png)

### Blob Stores

文件存储的地方，创建一个目录的话，对应文件系统的一个目录，如图所示：

![Alt text](../../../../images/nexus-4.png)

### Repositories

![Alt text](../../../../images/nexus-5.png)

一 > **Proxy**

这里就是代理的意思，代理中央 Maven 仓库，当 PC 访问中央库的时候，先通过 Proxy 下载到 Nexus 仓库，然后再从 Nexus 仓库下载到 PC 本地。

这样的优势只要其中一个人从中央库下来了，以后大家都是从 Nexus 私服上进行下来，私服一般部署在内网，这样大大节约的宽带。

创建 Proxy 的具体步骤

1.点击 “Create Repositories” 按钮

![Alt text](../../../../images/nexus-6.png)

2.选择要创建的类型

![Alt text](../../../../images/nexus-7.png)

3.填写详细信息

* Name：就是为代理起个名字
* Remote Storage: 代理的地址，Maven 的地址为: repo1.maven.org/maven2/
* Blob Store: 选择代理下载包的存放路径

![Alt text](../../../../images/nexus-8.png)

二 > **Hosted**

Hosted 是宿主机的意思，就是怎么把第三方的 Jar 放到私服上。

Hosted 有三种方式，Releases、SNAPSHOT、Mixed

* Releases: 一般是已经发布的 Jar 包
* Snapshot: 未发布的版本
* Mixed：混合的

Hosted 的创建和 Proxy 是一致的，具体步骤和上面基本一致。

具体如下:

![Alt text](../../../../images/nexus-9.png)

![Alt text](../../../../images/nexus-10.png)

![Alt text](../../../../images/nexus-11.png)

**注意事项：**
Deployment Pollcy: 需要把策略改成 “Allow redeploy”。

![Alt text](../../../../images/nexus-12.png)

三 > **Group**

能把两个仓库合成一个仓库来使用，目前没使用过，所以没做详细的研究。


# 代理中央仓库

只要在 pom.xml 文件中配置私服的地址（比如 http://ip:8081）即可，配置如下：

```xml
<repositories>
    <repository>
        <id>maven-central</id>
        <name>maven-central</name>
        <url>http://ip:8081/repository/maven-central/</url>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
        <releases>
            <enabled>true</enabled>
        </releases>
    </repository>
</repositories>
```

# Snapshot 包的管理

1）修改 Maven 的 settings.xml 文件，加入认证机制

```xml
<servers>
    <server>
        <id>nexus</id>
        <username>admin</username>
        <password>admin123</password>
    </server>
</servers>
```

2）修改工程的 pom.xml 文件

```xml
<distributionManagement>
    <snapshotRepository>
        <id>nexus</id>
        <name>Nexus Snapshot</name>
        <url>http://ip:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
    <site>
        <id>nexus</id>
        <name>Nexus Sites</name>
        <url>dav:http://ip:8081/repository/maven-snapshots/</url>
    </site>
</distributionManagement>
```

**注意事项:**

pom.xml 中的 <id>nexus</id> 应与 setting.xml 中的 <id>nexus</id> 相互对应.

3）上传到 Nexus 上

    1– 项目编译成的 jar 是 Snapshot(POM 文件的头部)

```xml
<groupId>com.zhisheng</groupId>
<artifactId>test-nexus</artifactId>
<version>1.0.0-SHAPSHOT</version>
<packaging>jar</packaging>
```

    2– 使用 mvn deploy 命令运行即可（运行结果在此略过）
    3– 因为 Snapshot 是快照版本，默认他每次会把 Jar 加一个时间戳，做为历史备份版本。

# Releases 包的管理

1）与 Snapshot 大同小异，只是上传到私服上的 Jar 包不会自动带时间戳
2）与 Snapshot 配置不同的地方，就是工程的 pom.xml 文件，加入 repository 配置

```xml
<distributionManagement>
    <repository>
        <id>nexus</id>
        <name>Nexus Snapshot</name>
        <url>http://ip:8081/repository/maven-releases/</url>
    </repository>
</distributionManagement>
```

3）打包的时候需要把 Snapshot 去掉

```xml
<groupId>com.zhisheng</groupId>
<artifactId>test-nexus</artifactId>
<version>1.0.0</version>
<packaging>jar</packaging>
```











