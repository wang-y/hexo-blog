---
title: keepalived 环境安装及配置
date: 2017-07-23 15:45:06
categories: 技术
tags: 
    - keepalived
---

# 安装
## 所需环境
1. popt-devel
2. psmisc
<!-- more -->

```
#yum install -y popt-devel
#yum install psmisc
```

##安装keepalived
```
#tar zxvf keepalived-1.2.2.tar.gz 
#cd keepalived-1.2.2  
#./configure --prefix=/  
#make  
#make install 
```

# 配置
## 修改配置文件
```
修改配置文件
    vi /etc/keepalived/keepalived.conf
  
    #ConfigurationFile for keepalived  
    global_defs {
        notification_email {
        	xxxxxxxx@xx.com               #设置报警邮件地址，可以设置多个，每行一个。 需开启本机的sendmail服务
        }
        notification_email_from  xxxxxxxx@xxx.com        #设置邮件的发送地址
        smtp_server smtp.exmail.qq.com                   #设置smtp server地址
        smtp_connect_timeout 30                          #设置连接smtp server的超时时间
        router_id LVS_DEVEL                              #表示运行keepalived服务器的一个标识。发邮件时显示在邮件主题的信息
    }
    vrrp_script check_nginx {              #定义监控nginx的脚本  
        script "/root/check_nginx.sh"  
        interval 2                         #监控时间间隔  
        weight 2                           #负载参数  
    }  
   vrrp_instance vrrptest {                #定义vrrptest实例  
        state BACKUP                       #服务器状态  
        interface eth0                     #网卡名
        virtual_router_id 51               #虚拟路由的标志，一组lvs的虚拟路由标识必须相同，这样才能切换  
        priority 150                       #服务启动优先级，值越大，优先级越高，BACKUP 不能大于MASTER  
        advert_int 1                       #设定MASTER与BACKUP负载均衡器之间同步检查的时间间隔，单位是秒  
        authentication {  
            auth_type PASS                 #设置验证类型，主要有PASS和AH两种
            auth_pass orieange             #认证密码，一组lvs 服务器的认证密码必须一致  
        }  
        track_script {                     #执行监控nginx进程的脚本  
            check_nginx  
        }  
        virtual_ipaddress {                #设置虚拟IP地址，可以设置多个虚拟IP地址，每行一个
            12.15.0.168  
        }
		virtual_server 12.0.0.12 80 {      #设置虚拟服务器，需要指定虚拟IP地址和服务端口，IP与端口之间用空格隔开
		    delay_loop 6                   #设置运行情况检查时间，单位是秒
    		lb_algo rr                     #设置负载调度算法，这里设置为rr，即轮询算法
    		lb_kind DR                     #设置LVS实现负载均衡的机制，有NAT、TUN、DR三个模式可选
    		persistence_timeout 50         #会话保持时间，单位是秒。这个选项对动态网页是非常有用的，为集群系统中的session共享提供了一个很好的解决方案。
           			                       #有了这个会话保持功能，用户的请求会被一直分发到某个服务节点，直到超过这个会话的保持时间。
                  			               #需要注意的是，这个会话保持时间是最大无响应超时时间，也就是说，用户在操作动态页面时，如果50秒内没有执行任何操作，
                 			               #那么接下来的操作会被分发到另外的节点，但是如果用户一直在操作动态页面，则不受50秒的时间限制
    		protocol TCP                   #指定转发协议类型，有TCP和UDP两种

			real_server 12.15.0.106 80 {   #配置服务节点1，需要指定real server的真实IP地址和端口，IP与端口之间用空格隔开
        		weight 3                   #配置服务节点的权值，权值大小用数字表示，数字越大，权值越高，设置权值大小可以为不同性能的服务器
              		                       #分配不同的负载，可以为性能高的服务器设置较高的权值，而为性能较低的服务器设置相对较低的权值，这样才能合理地利用和分配系统资源
       			TCP_CHECK {                #realserver的状态检测设置部分，单位是秒
            		connect_timeout 10     #表示3秒无响应超时
            		nb_get_retry 3         #表示重试次数
            		delay_before_retry 3   #表示重试间隔
            		connect_port 80
				}
			}
		    real_server 12.15.0.108 80 {
		        weight 3
		        TCP_CHECK {
		            connect_timeout 10
		            nb_get_retry 3
		            delay_before_retry 3
		            connect_port 80
		        }
		    }
		}
	}

	/* 新建检查nginx脚本 */
	vi /check_nginx.sh  
    
    #!/bin/bash
    if [ "$(ps -ef | grep "nginx: master process"| grep -v grep)" == "" ]
    then
       /usr/local/nginix/nginx-1.12.0/sbin/nginx
       sleep 5
    if [ "$(ps -ef | grep "nginx: master process"| grep -v grep)" == "" ]
       then
       killall keepalived
    fi
    fi
    
    /* 增加执行权限 */
    chmod +x /root/check_nginx.sh

```

# 命令
```
启动keepalived(cp keepalived /etc/init.d/)
service keepalived start

验证
ip add show eth0
```