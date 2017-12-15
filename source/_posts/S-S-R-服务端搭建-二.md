---
title: S-S-R 服务端搭建(二)
date: 2017-11-24 14:27:31
categories: 科学上网
tags: 
    - SSR
---

# TCP优化（非BBR适用）

**使用建议**

- 建议科学上网服务器都进行一遍TCP优化
- 与其他加速手段兼容
- 使用 BBR 加速可以略过此部分(已包含)

<!-- more -->

**具体步骤**

- 适用场景：高延迟搞丢包线路
- 增加TCP连接数量

```
nano /etc/security/limits.conf

#添加
* soft nofile 51200
* hard nofile 51200

#保存(Ctrl + X —— y ——回车)

```

- 设置ulimit

```
ulimit -n 51200
```

**修改内核参数适合的还是hybla（高延迟高丢包率环境）:** 

- 首先看一下VPS现有算法：

```
sysctl net.ipv4.tcp_available_congestion_control
```

- 没有hybla时，加载hybla算法.

```
/sbin/modprobe tcp_hybla
```

- 开始修改

```
nano /etc/sysctl.conf
```

- 复制代码

```
#TCP配置优化(不然你自己根本不知道你在干什么)
fs.file-max = 51200
#提高整个系统的文件限制
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.netdev_max_backlog = 250000
net.core.somaxconn = 4096
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.ip_local_port_range = 10000 65000
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_mem = 25600 51200 102400
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_congestion_control = hybla
#END OF LINE

#保存(Ctrl + X —— y ——回车)
```
    
- 应用

```
sysctl -p
```

- 重启SSR

```
/etc/init.d/shadowsocks restart
```

# TCP-BBR

- BBR (Bottleneck Bandwidth and RTT)是由google工程师编写的新的 TCP 拥塞控制算法，目的是要尽量跑满带宽, 并且尽量不要有排队的情况, 加速效果不比锐速差

- 开源，高效，强烈推荐首选！
- 同一线路，BBR与锐速可分别试用取舍，一般满足需求！
- 修改版 BBR 更为高效
- 不会造成流量大量浪费

## 安装

- [https://github.com/google/bbr](开源地址)
    + 测试环境 Debian 7 x64 Vultr
    + 启用TCP-BBR涉及VPS更换内核，所以如果步骤错误，或者VPS不兼容最新的内核，会导致无法开机等错误
    + 锐速不支持，更换后的 >4.9 内核

### 安装原版BBR

- 系统支持：CentOS 6+，Debian 7+，Ubuntu 12+
- 虚拟技术：OpenVZ 以外的，比如 KVM、Xen、VMware 等
- 内存要求：≥128M

- 连接SSH，输入下面命令，更新内核

```
wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh
chmod +x bbr.sh
./bbr.sh
```

- 安装完成后，脚本会提示需要重启 VPS，输入 y 并回车后重启
- 重连SSH
- 验证 内核版本  uname -r   最新内核版本大于4.9即可，最新4.12(7.03)
- 修改sysctl.conf

```
nano /etc/sysctl.conf
```

    复制代码

```
#TCP配置优化(不然你自己根本不知道你在干什么)
fs.file-max = 51200
#提高整个系统的文件限制
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.netdev_max_backlog = 250000
net.core.somaxconn = 4096
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.ip_local_port_range = 10000 65000
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_mem = 25600 51200 102400
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_congestion_control = bbr
#END OF LINE

#保存(Ctrl + X —— y ——回车)

sysctl -p
```

- 重启SSR

```
/etc/init.d/shadowsocks restart
```


### 修改版BBR

- 相对原版更为暴力，加速效果更好
- 提供 Vicer版 和 魔改版 两个试用后加速效果还不错的版本。

---

- 注意：修改版BBR的一键脚本，支持系统较少，安装前需要确认自己的系统支持，如不在支持列表，自行Google即可，有大量第三方脚本。

- 支持系统： Debian 8 / Ubuntu16 + /Debian 7(需要手动安装gcc4.9)

---

Debian 7手动安装gcc4.9(以root用户登陆，否则 命令前添加sudo)
- 修改系统更新源

```
nano /etc/apt/sources.list
```

- 添加如下两个更新源

```
deb http://ftp.cn.debian.org/debian/ jessie main non-free contrib
deb http://ftp.uk.debian.org/debian/ jessie main non-free contrib
```

- 保存(Ctrl + X —— y ——回车)

- 执行更新

```
apt-get update
```

- 检查可安装 gcc 版本列表

```
apt-cache search gcc
```

- 输出有gcc4.9 字样即可，数量无所谓

```
libx32gcc-4.8-dev - GCC support library (x32 development files)
cpp-4.9 - GNU C preprocessor
gcc-4.9 - GNU C compiler
gcc-4.9-base - GCC, the GNU Compiler Collection (base package)
gcc-4.9-locales - GCC, the GNU compiler collection (native language support files)
gcc-4.9-multilib - GNU C compiler (multilib files)
gcc-4.9-plugin-dev - Files for GNU GCC plugin development.
gcc-4.9-source - Source of the GNU Compiler Collection
gccgo-4.9 - GNU Go compiler
gccgo-4.9-multilib - GNU Go compiler (multilib files)
gcj-4.9 - GCJ byte code and native compiler for Java(TM)
gcj-4.9-jdk - GCJ and Classpath development tools for Java(TM)
gcj-4.9-jre-lib - Java runtime library for use with gcj (jar files)
gdc-4.9 - GNU D compiler (version 2), based on the GCC backend
gfortran-4.9 - GNU Fortran compiler
gfortran-4.9-multilib - GNU Fortran compiler (multilib files)
gobjc++-4.9 - GNU Objective-C++ compiler
gobjc++-4.9-multilib - GNU Objective-C++ compiler (multilib files)
gobjc-4.9 - GNU Objective-C compiler
gobjc-4.9-multilib - GNU Objective-C compiler (multilib files)
```


- 命令 __apt-get install g++-4.9__ 安装g++-4.9

#### Vicer版BBR

- 脚本来自[https://moeclub.org/2017/06/24/278/](Debian/Ubuntu TCP BBR 改进版/增强版)

- 一键安装

```
wget --no-check-certificate -qO 'BBR_POWERED.sh' 'https://moeclub.org/attachment/LinuxShell/BBR_POWERED.sh' && chmod a+x BBR_POWERED.sh && bash BBR_POWERED.sh
```

- 遇到 Error! Header not be matched by Linux Kernel. 参看[https://moeclub.org/2017/06/06/249/](Debian/Ubuntu 开启 TCP BBR 拥塞算法),依照 作者开启bbr 的脚本执行一遍即可。

- 遇到 Error! Install gcc-4.9. debian 7 重新安装一遍 gcc-4.9 ，其他系统 执行 apt-get update

- 安装完成后 执行 __lsmod |grep 'bbr_powered'__ 结果不为空,则加载模块成功
- 重启SSR __/etc/init.d/shadowsocks restart__

#### 魔改版BBR

- 一键脚本

```
wget -N --no-check-certificate https://raw.githubusercontent.com/FunctionClub/YankeeBBR/master/bbr.sh && bash bbr.sh install
```

之后选择重启系统，重连SSH

- 输入命令 __bash bbr.sh start__ 即可完成安装
- 验证 __sysctl net.ipv4.tcp_available_congestion_control__ 返回命令有 tsunami 即可。
- 重启SSR __/etc/init.d/shadowsocks restart__







