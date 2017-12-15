---
title: S-S-R 服务端搭建(一)
date: 2017-11-24 14:27:17
categories: 科学上网
tags: 
    - SSR
---

# 部署ShadowSocksR

* 一键安装脚本
> [https://shadowsocks.be/9.html](https://shadowsocks.be/9.html)
> [https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocksR.sh](https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocksR.sh)

<!-- more -->

**使用此一键脚本安装的，不能使用其他命令更新，只能通过 备份配置文件，卸载再重新安装！也都不是事，备份一下配置文件，重新安装，覆盖配置文件，重启即可。**

## 安装SSR

- 复制以下命令执行
```
wget --no-check-certificate  https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocksR.sh
chmod +x shadowsocksR.sh
./shadowsocksR.sh 2>&1 | tee shadowsocksR.log
```
- 如图，回车。

![Alt text](../../../../images/ssr-1.jpg)

- 输入设定初始密码，也可以直接回车。

![Alt text](../../../../images/ssr-2.jpg)

- 输入初始端口，也直接回车

![Alt text](../../../../images/ssr-3.jpg)

- 回车

![Alt text](../../../../images/ssr-4.jpg)

- 等待一段时间的滚屏，最后会提示——成功

![Alt text](../../../../images/ssr-5.jpg)

## shadowsocks.json配置文件

**默认在/root文件夹下，要进入/etc文件夹下找到shadowsocks.json**

### 编辑shadowsocks.json文件

- 注释版

```
{
  "server":"0.0.0.0",
  "server_ipv6":"::",
  "local_address":"127.0.0.1",
  "local_port":1080,
  "port_password":{
   #纯 SS 不带混淆 端口25 密码为123456.
   "25":"123456",
   #端口443，密码123456 ，protocol选择auth_chain_a。obfs选择tls1.2_ticket_auth，具体插件的介绍如下参考资料中
   "443":{"protocol":"auth_chain_a", "password":"123456", "obfs":"tls1.2_ticket_auth", "obfs_param":""},
  #注意无论怎么变化，最后一个端口设置，不带逗号！
   "3389":{"protocol":"auth_aes128_md5", "password":"123456", "obfs":"tls1.2_ticket_auth", "obfs_param":""}#此处没有逗号！
  },
  "timeout":400,
  #默认全局的加密方式，即上边各个端口的默认加密方式。一般为aes-256-cfb，     此处，选择为chacha20，移动设备性能较好。
  "method":"chacha20",
  #protocol.协议定义插件的默认值，origin即使用原版SS协议，不混淆。即上面端口配置中，你没有设置 protocol 和 obfs 情况下，使用的默认值。
  "protocol": "origin",
  "protocol_param": "",
   #protocol.协议定义插件的默认值，plain即使用原协议，不混淆。
  "obfs": "plain",
  "obfs_param": "",
  "redirect": "",
  "dns_ipv6": true,
 #TCP FAST OPEN ，打开
  "fast_open": true,
 "workers": 1
}
```

- 无注释版

```
{
  "server":"0.0.0.0",
  "server_ipv6":"::",
  "local_address":"127.0.0.1",
  "local_port":1080,
  "port_password":{
    "25":"123456",
      "443":{"protocol":"auth_chain_a", "password":"123456", "obfs":"tls1.2_ticket_auth", "obfs_param":""},
      "3389":{"protocol":"auth_aes128_md5", "password":"123456", "obfs":"tls1.2_ticket_auth", "obfs_param":""}
  },
  "timeout":400,
  "method":"chacha20",
  "protocol": "origin",
  "protocol_param": "",
  "obfs": "plain",
  "obfs_param": "",
  "redirect": "",
  "dns_ipv6": true,
  "fast_open": true,
  "workers": 1
}
```

### 重启SSR

- 配置完成后，重启SSR,以root账户登陆securecrt，复制以下代码，重启ssr。

```
/etc/init.d/shadowsocks restart
```

- 会提示shadowsocksr重启成功。

![Alt text](../../../../images/ssr-6.jpg)

### 更新SSR

**vps端，需要先备份 shadowsocks.json文件**

```
./shadowsocksR.sh uninstall
```

之后重新执行安装脚本即可

# 混淆选择

## 混淆插件简介

- ShadowsocksR目前支持的混淆插件（此类型的插件用于定义加密后的通信协议）：plain ,http_simple ,http_post,random_head ,tls1.2_ticketauth 协议定义插件(用于定义加密前的协议): origin, auth_sha1, auth_sha1_v2, auth_sha1_v4, auth_aes128_md5/auth_aes128_sha1
- auth_aes128_md5/auth_aes128_sha1 支持 单端口多用户，即一个端口 可以配置 几个不同的密码，稍后更新。
- ~~[https://github.com/breakwa11/shadowsocks-rss/blob/master/ssr.md](ShadowsocksR 协议插件文档)~~

## 混淆插件选择(5.24)

- 通用
- 推荐auth_chain_a+tls1.2_ticket_auth
这种组合目前混淆效果最好，有利于个人VPS的长时间使用。
- (5.24)auth_chain_a可不使用用加密，即加密方式None（SSR作者语），但是吧，性能差不多的情况下，加个密没毛病。
- auth_aes128_md5或auth_aes128_sha1+随意，即使使用rc4加密亦可（SSR作者语）
- 玩游戏，或对延迟有要求，不要使用tls1.2_ticket_auth

---

- 网络封锁/监控环境下
- 例如学校教育网/公司内网/广电宽带等等，封杀了BT/禁止访问网盘等等等等。
- 使用http_simple、http_post或tls1.2_ticket_auth 混淆访问的目标网址。再配合443/80端口通常可以解决问题。

---

- Android
- (5.24)最好使用auth_chain_a。
- 如果之前使用的是auth_aes128_md5，推荐以auth_chain_a替换
- 手机运算能力较差的推荐使用auth_sha1_v4替换auth_aes128_md5

---

**单端口多用户**

- 适用场景:
    多人合租,减少vps开放端口.

~~[https://breakwa11.blogspot.ru/2017/01/shadowsocksr-mu.html?m=1](官方 ShadowsocksR单端口多用户配置方法)~~

**伪装正常网站(推荐)**

- 适用场景
    最大程度减少GFW主动扫描,被发现的可能性. (伪装网站后,无法使用单端口多用户)
- ~~[https://doub.io/ss-jc48/](逗比根据地的介绍，很详细，搭配最新的auth_chain_a 混淆，很好用。)~~
- http网站对应混淆选择http 或 post
- https 网站选择tls1.2
- 域名申请，免费！[https://my.freenom.com/clientarea.php](https://my.freenom.com/clientarea.php)
- DNS解析(目前国内网站 域名 CDN DNS解析 均需备案实名,故放弃)[https://www.cloudflare.com/](https://www.cloudflare.com/)

# VPS优化

## TCP优化

- 增加TCP连接数量

```
nano /etc/security/limits.conf

#添加两行：
* soft nofile 51200
* hard nofile 51200
#保存(Ctrl + X —— y ——回车)
```

- 设置ulimit：

```
ulimit -n 51200
```

## TCP-BBR(推荐)

- BBR (Bottleneck Bandwidth and RTT)是由google工程师编写的新的 TCP 拥塞控制算法，目的是要尽量跑满带宽, 并且尽量不要有排队的情况, 加速效果不比锐速差，
    完全开源，对隐匿性要求高而无法使用锐速的人士，也可以放心使用
    [https://github.com/google/bbr](开源地址)
- 启用TCP-BBR涉及VPS更换内核，所以如果步骤错误，或者VPS不兼容最新的内核，会导致无法开机等错误
- 目前在 Vultr / DO 上测试通过。其他主机提供商，请自行测试

---

- 连接SSH，运行下面的命令

```
wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh
chmod +x bbr.sh
./bbr.sh
```

- 提示需要重启 VPS，输入 y 并回车后重启，重连SSH
- 脚本会自动更新匹配的4.xx版本内核(6.28 目前是4.11），并启用TCP BBR
- 验证:
    + 输入  uname -r                  有4.9.0 以上就 表示 更新成功
    + 输入  lsmod | grep bbr          返回值有 tcpbbr 即bbr已启动。
- 添加一些优化内容:
    + 修改sysctl.conf
    
```
nano /etc/sysctl.conf

复制代码：

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

保存(Ctrl + X —— y ——回车)

应用(sysctl -p)

```

- 重启SSR






