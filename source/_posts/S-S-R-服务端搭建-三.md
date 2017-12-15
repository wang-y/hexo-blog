---
title: S-S-R 服务端搭建(三)
date: 2017-11-24 14:27:40
categories: 科学上网
tags: 
    - SSR
---

# SSH

- 有两种方案
    + 彻底关闭SSH登陆，改为密钥登陆，安全性最高，但配置较为复杂
    + 修改SSH登陆端口，同时开放默认端口，将爆破 ssh 密码的 IP 封停

<!-- more -->

- [https://ttt.tt/104/](密钥登陆)

**修改端口**

- CentOS 命令 如下，以下是将 SSH 端口改为 999 端口
```
sed -i 's/#Port 22/Port 999/g' /etc/ssh/sshd_config
```

+ debian 需要打开/etc/ssh/sshdconfig 文件，修改其中的port后面的数字

```
nano /etc/ssh/sshd_config
```

修改其中的port后面的数字，为你要ssh登陆的端口，cltrl + x保存，回车退出。

# fail2ban

- fail2ban 是 Linux 上的一个著名的入侵保护的开源框架，它会监控多个系统的日志文件，并根据检测到的任何可疑的行为自动触发不同的防御动作。将尝试爆破 ssh 密码的 IP 封停，默认10分钟。

- 详细配置安装方法在如下网址，这里使用默认配置即可。

    > [https://linux.cn/article-5067-1.html](https://linux.cn/article-5067-1.html) 

- 安装 fail2ban

CentOS 需要提前 设置 [https://linux.cn/article-2324-1.html](EPEL仓库)

以 root 用户登陆，非 root 用户需要 命令前 增加 sudo

```
//CentOS
yum install fail2ban
//Debian / ubuntu
apt-get install fail2ban
```

- 其他命令

```
service fail2ban restart #重启

fail2ban-client ping     #验证状态

Server replied: pong     #返回
```

- 设置开机自启动（debian不用前面验证完成，已加入开机启动）

```
// CentOS/RHEL 6
chkconfig fail2ban on
// CentOS/RHEL 7
systemctl enable fail2ban
```

# 禁用Linux多余端口

- 关闭多余端口是永远正确的选择！只留下常用端口 和 SSR端口
- 配置 iptables 警告 iptables 配置不是一般的复杂，谨慎操作

```
#清空默认规则
iptables -F

#允许22端口，给暴力破解留点空间 //doge
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT

#允许53端口 udp ，一般用做DNS服务器，如果你不需要则忽略此条
iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
iptables -A INPUT -p udp --sport 53 -j ACCEPT

#允许本机访问本机
iptables -A INPUT -s 127.0.0.1 -d 127.0.0.1 -j ACCEPT
iptables -A OUTPUT -s 127.0.0.1 -d 127.0.0.1 -j ACCEPT

#允许真正 SSH 端口
iptables -A INPUT -p tcp -s 0/0 --dport 999 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 999 -m state --state ESTABLISHED -j ACCEPT

#允许 80 443 端口，http 和 https
iptables -A INPUT -p tcp -s 0/0 --dport 80 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp -s 0/0 --dport 443 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT

#允许 SSR端口 以888 xxx 为例 你有几个端口，就添加几个
iptables -A INPUT -p tcp -s 0/0 --dport 888 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 888 -m state --state ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp -s 0/0 --dport xxx -j ACCEPT
iptables -A OUTPUT -p tcp --sport xxx -m state --state ESTABLISHED -j ACCEPT

#CentOS 保存配置
iptables-save > /etc/sysconfig/iptables

#重载 iptables
iptables -L

# debian 7 保存配置
iptables-save > /etc/iptables-rules
ip6tables-save > /etc/ip6tables-rules

随后修改/etc/network/interfaces文件，最后加入

pre-up iptables-restore < /etc/iptables-rules
pre-up ip6tables-restore < /etc/ip6tables-rules

重启执行`iptables -L`，看到配置已生效。
```

# 安装 CSF 防火墙
- 命令

```
rm -fv csf.tgz
wget http://download.configserver.com/csf.tgz
tar -xzf csf.tgz
cd csf
sh install.sh
//使用下边的命令来验证csf正确安装并已经运行
perl /usr/local/csf/bin/csftest.pl
```

## autoban 补丁（8.22）推荐
* 来自ss原作者@clowwindy 的补丁，ban掉那些尝试破译ss密码的ip，在此对原作者表示敬意！
* 原理是同一个ip出现3次以上尝试破译ss密码会被永久ban掉，ssr也可用
* 测试只在debian 7上进行，需要python环境，但是通过秋水逸冰脚本安装ssr 已经安装好了python环境。
* 以下内容参考
  >https://plus.google.com/104980438521301094845/posts/9zypCffhdZu

---

* 下载脚本

```
wget https://raw.githubusercontent.com/Jasper-1024/shadowsocksr/manyuser/utils/autoban.py
```

* ssr运行

```
python autoban.py < /var/log/shadowsocksr.log
```

查看哪些ip试图连接破解ssr密码。原版ss将log文件路径替换即可。

* 自动ban ip
* 
编辑/etc/init.d/rc.local，加入

```


python /root/autoban.py < /var/log/shadowsocksr.log

nohup tail -F /var/log/shadowsocksr.log | python /root/autoban.py >log 2>log &
```

保存退出即可。
