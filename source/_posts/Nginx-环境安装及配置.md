---
title: Nginx 环境安装及配置
date: 2017-07-23 15:44:47
categories: 技术
tags: 
    - nginx
---

# 安装
## 所需环境
1. gcc
2. openssl-devel
3. pcre-devel(使Nginx支持http rewrite的模块)
4. zlib-devel
<!-- more -->
```
#yum -y install gcc openssl-devel pcre-devel zlib-devel
#yum -y install libtool
#yum -y install gcc-c++
#yum -y install gcc automake autoconf libtool make

#tar zxvf pcre-8.31.tar.gz
#cd pcre-8.31
#./congigure
#make
#make install

```

## 安装Nginx
```
#tar zxvf nginx-1.2.4.tar.gz
#cd nginx-1.2.4
#./configure --with-http_stub_status_module --with-http_gzip_static_module --prefix=/usr/local/ngnix/nginx-1.12.0
#make
#makeinstall

// --with-http_stub_status_module 可以用来启用Nginx的NginxStatus功能，以监控Nginx的当前状态。
// --with-http_gzip_static_module 支持在线实时压缩输出数据流。

```

# 配置
## 定义Nginx运行的用户和用户组
```
#user root root;
#nginx进程数，建议设置为等于CPU总核心数。
worker_processes 4;
#全局错误日志定义类型，[ debug | info | notice | warn | error | crit ]
error_log  logs/nginx.log info;
#进程文件
pid logs/nginx.pid;
#一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（系统的值ulimit -n）与nginx进程数相除，但是nginx分配请求并不均匀，所以建议与ulimit -n的值保持一致。
worker_rlimit_nofile 1024;
```

## 工作模式与连接数上限
```
events{ 
    #设置网路连接序列化，防止惊群现象发生，默认为on
    accept_mutex on;   
    #设置一个进程是否同时接受多个网络连接，默认为off 
    multi_accept on;  
    #参考事件模型，use [ kqueue | rtsig | epoll | /dev/poll | select | poll ]; epoll模型是Linux 2.6以上版本内核中的高性能网络I/O模型，如果跑在FreeBSD上面，就用kqueue模型。
    use epoll; 
    #单个进程最大连接数（最大连接数=连接数*进程数）
    worker_connections 1024;
}
```

## 设定http服务器，利用它的反向代理功能提供负载均衡支持
```
http{
      include mime.types; #文件扩展名与文件类型映射表
      log_format myFormat '$remote_addr–$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for'; #自定义格式
      access_log  logs/access.log myFormat; #设定日志格式
      default_type application/octet-stream; #默认文件类型
      sendfile  on; #必须设为 on,如果用来进行下载等应用磁盘IO重负载应用，可设置为 off，以平衡磁盘与网络I/O处理速度，降低系统的uptime.
      charset utf-8; #默认编码
      server_names_hash_bucket_size 128; #服务器名字的hash表大小
      client_header_buffer_size 5m; #上传文件大小限制
      autoindex off; #开启目录列表访问，合适下载服务器，默认关闭。
      tcp_nopush on; #防止网络阻塞
      tcp_nodelay on; #防止网络阻塞
      keepalive_timeout 120; #长连接超时时间，单位是秒

      #FastCGI相关参数是为了改善网站的性能：减少资源占用，提高访问速度。下面参数看字面意思都能理解。
      fastcgi_connect_timeout 300;
      fastcgi_send_timeout 300;
      fastcgi_read_timeout 300;
      fastcgi_buffer_size 64k;
      fastcgi_buffers 4 64k;
      fastcgi_busy_buffers_size 128k;
      fastcgi_temp_file_write_size 128k;

      #gzip模块设置
      gzip on; #开启gzip压缩输出
      gzip_disable "MSIE [1-6]\.(?!.*SV1)";
      gzip_min_length 1k; #最小压缩文件大小
      gzip_buffers 4 16k; #压缩缓冲区
      gzip_comp_level 2; #压缩等级
      gzip_types text/plain application/x-javascript text/css application/xml;
      #压缩类型，默认就已经包含text/html，所以下面就不用再写了，写上去也不会有问题，但是会有一个warn。
      gzip_vary on;
      #limit_zone crawler $binary_remote_addr 10m; #开启限制IP连接数的时候需要使用

      #设定负载均衡的服务器列表(weigth参数表示权值，权值越高被分配到的几率越大)
      upstream mysvr {   
      server 12.15.0.106:8080 weight=3;
      server 12.15.0.108:8080 weight=2;  #热备
      }

     #虚拟主机的配置
     server{
       keepalive_requests 120; #单连接请求上限次数。
       listen 80;  #监听端口
       server_name mysvr; #定义使用www.xx.com访问(域名可以有多个，用空格隔开)
       access_log  logs/access.log  myFormat; #设定本虚拟主机的访问日志
       #方向代理默认请求
       location / {
                     #root   /root;      #定义服务器的默认网站根目录位置
                     #index index.php index.html index.htm;   #定义首页索引文件的名称
                     proxy_pass  http://mysvr;#请求转向mysvr 定义的服务器列表
                     #以下是一些反向代理的配置可删除.
                     proxy_redirect off;
                     #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
                     proxy_set_header Host $host; 
                     proxy_set_header X-Real-IP $remote_addr;
                     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                     client_max_body_size 50m;    #允许客户端请求的最大单文件字节数
                     client_body_buffer_size 256k;  #缓冲区代理缓冲用户端请求的最大字节数，
                     proxy_connect_timeout 90;  #nginx跟后端服务器连接超时时间(代理连接超时)
                     proxy_send_timeout 90;        #后端服务器数据回传时间(代理发送超时)
                     proxy_read_timeout 90;         #连接成功后，后端服务器响应时间(代理接收超时)
                     proxy_buffer_size 256k;             #设置代理服务器（nginx）保存用户头信息的缓冲区大小
                     proxy_buffers 4 256k;               #proxy_buffers缓冲区，网页平均在32k以下的话，这样设置
                     proxy_busy_buffers_size 256k;    #高负荷下缓冲大小（proxy_buffers*2）
                     proxy_temp_file_write_size 256k;  #设定缓存文件夹大小，大于这个值，将从upstream服务器
       }

   }
}

```

## 动态的由tomcat处理，静态的由Nginx处理(在server{}里添加)。
```
location ~  \.(jsp|do)$ { 
        proxy_pass http://mysvr;
        proxy_set_header  X-Real-IP  $remote_addr;
}
location ~.*.(htm|html|gif|jpg|jpeg|png|bmp|swf|ioc|rar|zip|txt|flv|mid|doc|ppt|pdf|xls|mp3|wma|js|css)${  
        root dandan;  
}  
```

# 常用命令
```
nginx/sbin/nginx -t  # 检查配置
nginx/sbin/nginx -c /path/nginx.conf # 指定使用path目录下的nginx.conf配置文件启动 
nginx/sbin/nginx -s stop  # 停止
nginx/sbin/nginx -s reload  # 重启
```