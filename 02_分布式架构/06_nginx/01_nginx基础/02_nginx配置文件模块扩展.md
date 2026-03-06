## 一、nginx回顾

### 1.安装

```bash
1.epol源安装
2.官方源安装
3.源码包安装
	1）下载
	2）解压
	3）生成
	4）编译
	5）安装
```

### 2.nginx配置文件

```bash
[root@web01 ~]# cat /etc/nginx/nginx.conf
##################核心模块###############
user  www;
worker_processes  1;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
#################事件驱动模块###########
events {
    worker_connections  1024;
}
################http内核模块############
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  65;
    #tcp_nodelay     on;
    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
    
    server {
        listen       80;
        server_name  localhost;
        charset utf8;
        #access_log  /var/log/nginx/host.access.log  main;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        location /download {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
    }
}
```



## 二、Nginx虚拟主机

### 1.虚拟主机方式

```bash
1.基于多IP的方式
2.基于多端口的方式
3.基于多域名的方式
```

### 2.基于多IP的方式

#### 0）网卡添加子IP

```bash
[root@web01 ~]# ifconfig eth0:1 10.0.0.3/24
[root@web03 ~]# ip addr add 10.0.0.91/24 dev eth0
```

#### 1）第一个配置文件

```bash
[root@web03 /etc/nginx/conf.d]# vim addr1.conf 
server {
        listen 10.0.0.9:80;

        location / {
                root /code/addr1;
                index index.html;
        }
}
```

#### 2）第二个配置文件

```bash
[root@web03 /etc/nginx/conf.d]# vim addr2.conf 
server {
        listen 10.0.0.91:80;

        location / {
                root /code/addr2;
                index index.html;
        }
}
```

#### 3）根据配置文件配置环境

```bash
[root@web03 /etc/nginx/conf.d]# mkdir /code
[root@web03 /etc/nginx/conf.d]# mkdir /code/addr1
[root@web03 /etc/nginx/conf.d]# mkdir /code/addr2
[root@web03 /etc/nginx/conf.d]# echo "9" >/code/addr1/index.html
[root@web03 /etc/nginx/conf.d]# echo "91" >/code/addr2/index.html
```

#### 4）检查配置

```bash
[root@web01 ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

#### 5）重启访问

```bash
[root@web01 ~]# systemctl restart nginx

http://10.0.0.7/
http://10.0.0.3/
```



### 3.基于多端口的方式

#### 1）第一个配置文件

```bash
[root@web01 ~]# cat /etc/nginx/conf.d/game.conf
server {
    listen 80;
    server_name localhost;

    location / {
	root /code/tuixiangzi;
	index index.html;
    }
}
```

#### 2）第二个配置文件

```bash
[root@web01 ~]# cat /etc/nginx/conf.d/gametwo.conf
server {
    listen 81;
    server_name localhost;

    location / {
	root /code/tank;
	index index.html;
    }
}
```

#### 3）检查配置

```bash
[root@web01 ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

#### 4）重启访问

```bash
[root@web01 ~]# systemctl restart nginx

http://10.0.0.7/
http://10.0.0.7:81/
```



### 4.基于多域名的方式

#### 1）第一个配置文件

```bash
[root@web01 ~]# cat /etc/nginx/conf.d/game.conf
server {
    listen 80;
    server_name www.tuixiangzi.com;

    location / {
        root /code/tuixiangzi;
        index index.html;
    }
}
```

#### 2）第二个配置文件

```bash
[root@web01 ~]# cat /etc/nginx/conf.d/gametwo.conf
server {
    listen 80;
    server_name www.tank.com;

    location / {
        root /code/tank;
        index index.html;
    }
}
```

#### 3）检查配置

```bash
[root@web01 ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

#### 4）重启

```bash
[root@web01 ~]# systemctl restart nginx
```

#### 5）配置本地hosts访问

```bash
#修改windows的hosts文件
C:\Windows\System32\drivers\etc\hosts
10.0.0.7 www.tuixiangzi.com www.tank.com

#访问
http://www.tuixiangzi.com/
http://www.tank.com/
```

### 总结：

​	在nginx的虚拟主机类型中，基于域名的虚拟主机应用最为广泛。

```bash
1，基于端口和IP的虚拟主机类型，用户体验不好。
2，基于IP类型的虚拟主机，如在公网环境下使用，会产生额外费用。
3，基于域名的虚拟主机，一次付费，用户体验较好。

不管使用哪种虚拟主机，最终使用的都是本地主机现有资源。
```









### 5.日志配置

#### 1）第一个配置

```bash
[root@web01 ~]# cat /etc/nginx/conf.d/game.conf 
server {
    listen 80;
    server_name www.tuixiangzi.com;

    access_log /var/log/nginx/www.tuixiangzi.com.log main;	#访问日志以main格式存放到目标地址

    location / {
	root /code/tuixiangzi;
	index index.html;
    }
}
```

#### 2）第二个配置

```bash
[root@web01 ~]# cat /etc/nginx/conf.d/gametwo.conf 
server {
    listen 80;
    server_name www.tank.com;

    access_log /var/log/nginx/www.tank.com.log main;

    location / {
	root /code/tank;
	index index.html;
    }
}
```

#### 3）重启访问测试

```bash
[root@web01 ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@web01 ~]# systemctl restart nginx

[root@web01 ~]# ll /var/log/nginx/
total 136
-rw-r-----. 1 nginx adm  103660 Nov 27 09:31 access.log
-rw-r-----. 1 nginx adm   21972 Nov 27 09:36 error.log
-rw-r--r--. 1 root  root    666 Nov 27 09:36 www.tank.com.log
-rw-r--r--. 1 root  root    190 Nov 27 09:36 www.tuixiangzi.com.log
```

总结：nginx运行优先遵循server内配置，再遵循http..。所以日志可以分类储存

## 三、nginx日志

```bash
Nginx有非常灵活的日志记录模式，每个级别的配置可以有各自独立的访问日志。日志格式通过log_format命令定义格式
```

### 1.log_format语法

```bash
Syntax:	log_format name [escape=default|json|none] string ...;
Default:	log_format combined "...";
Context:	http
```

### 2.默认日志格式

```bash
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
                      
10.0.0.1 - - [27/Nov/2020:09:36:08 +0800] "GET /images/tank.ico HTTP/1.1" 200 25214 "http://www.tank.com/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.89 Safari/537.36" "-"

10.0.0.1 - - [2020-11-27T10:13:58+08:00] "GET /images/modewin/help0.png HTTP/1.1" 200 27947 "http://www.tank.com/css/tank.css" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.89 Safari/537.36" "-"
```

### 3.日志常用变量

```bash
$remote_addr        # 记录客户端IP地址
$remote_user        # 记录客户端用户名
$time_local         # 记录通用的本地时间
$time_iso8601       # 记录ISO8601标准格式下的本地时间
$request            # 记录请求的方法以及请求的http协议
$status             # 记录请求状态码(用于定位错误信息)
$body_bytes_sent    # 发送给客户端的资源字节数，不包括响应头的大小
$bytes_sent         # 发送给客户端的总字节数
$msec               # 日志写入时间。单位为秒，精度是毫秒。
$http_referer       # 记录从哪个页面链接访问过来的
$http_user_agent    # 记录客户端浏览器相关信息
$http_x_forwarded_for #记录经过的所有服务器的IP地址
$X-Real-IP		   #记录起始的客户端IP地址和上一层客户端的IP地址
$request_length     # 请求的长度（包括请求行， 请求头和请求正文）。
$request_time       # 请求花费的时间，单位为秒，精度毫秒
# 注:如果Nginx位于负载均衡器，nginx反向代理之后， web服务器无法直接获取到客 户端真实的IP地址。
# $remote_addr获取的是反向代理的IP地址。 反向代理服务器在转发请求的http头信息中，
# 增加X-Forwarded-For信息，用来记录客户端IP地址和客户端请求的服务器地址。
```

### 4.nginx日志切割

```bash
[root@web01 ~]# vim /etc/logrotate.d/nginx 
#指定要切割的日志
/var/log/nginx/*.log {
	  
        daily	#每天切割日志
        missingok	#忽略日志丢失
        rotate 52	#日志保留时间 52天
        compress   #日志压缩
        delaycompress  #延时压缩
        not if empty	#不切割空日志
        create 640 nginx adm	#切割好的日志权限
        sharedscripts 	#开始执行脚本
        postrotate	#标注脚本内容
                if [ -f /var/run/nginx.pid ]; then#判断nginx启动
                        kill -USR1 `cat /var/run/nginx.pid`	#重新生成一个access.log

                fi
        endscript	#脚本执行完毕
}
```



## 四、nginx常用模块

### 1.目录索引模块

```bash
# ngx_http_autoindex_module

ngx_http_autoindex_module模块处理以斜杠字符（'/'）结尾的请求，并生成目录列表。

当ngx_http_index_module模块找不到索引文件时，通常会将请求传递给ngx_http_autoindex_module模块。
```

#### 1）语法

```bash
Syntax:	autoindex on | off;
Default:	autoindex off;
Context:	http, server, location
```

#### 2）配置

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/www.autoindex.com.conf 
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;

    location / {
        root /code/autoindex;
        autoindex on;
    }
}
```

#### 3）访问网站正常，加download跳转目录页面

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/www.autoindex.com.conf 
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;

    location / {
        root /code/autoindex;
        index index.html;
    }

    location /download {
        root /code/autoindex;
        autoindex on;
    }
}

#创建站点目录
[root@web01 ~]# mkdir /code/autoindex/download -p
[root@web01 ~]# echo "测试autoindex模块" > /code/autoindex/index.html

#访问
http://www.autoindex.com/		为主站
http://www.autoindex.com/download/		为下载文件的目录
```

#### 4）常用优化参数

```bash
#显示文件字节大小，默认是显示字节大小，配置为off之后，显示具体大小 M/G/K
Syntax:	autoindex_exact_size on | off;
Default:	autoindex_exact_size on;
Context:	http, server, location

#显示文件的修改的具体时间，默认显示的时间与真实时间相差8小时，所以配置 on
Syntax:	autoindex_localtime on | off;
Default:	autoindex_localtime off;
Context:	http, server, location
```

#### 5）完整配置

```bash
[root@web01 ~]# cat /etc/nginx/conf.d/www.autoindex.com.conf 
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;

    location / {
	root /code/autoindex;
	index index.html;
    }

    location /download {
	root /code/autoindex;
	autoindex on;
	autoindex_exact_size off;
	autoindex_localtime on;
    }
}
```



### 2.Nginx访问控制模块

```bash
#ngx_http_access_module
```

#### 1）语法

```bash
#允许访问的语法
Syntax:	allow address | all;
Default:	—
Context:	http, server, location, limit_except

#拒绝访问的语法
Syntax:	deny address | all;
Default:	—
Context:	http, server, location, limit_except

#如果配置允许，则也要配置拒绝；配置拒绝可以单独配置
```

#### 2）配置访问控制示例

##### 1>拒绝指定的IP,其他全部允许

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/www.autoindex.com.conf 
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;

    location / {
        root /code/autoindex;
        index index.html;
    }

    location /download {
        root /code/autoindex;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        deny 10.0.0.1;
        allow all;
    }
}
```

##### 2>只允许指定IP能访问, 其它全部拒绝

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/www.autoindex.com.conf 
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;

    location / {
        root /code/autoindex;
        index index.html;
    }

    location /download {
        root /code/autoindex;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        allow 10.0.0.1;
        #如果使用all，一定放在最后面
        deny all;
    }
}
```

##### 3>只允许10.0.0.1访问，拒绝该网段其他IP

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/www.autoindex.com.conf 
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;

    location / {
        root /code/autoindex;
        index index.html;
    }

    location /download {
        root /code/autoindex;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        allow 10.0.0.1;
        #如果使用all，一定放在最后面
        deny 10.0.0.0/24;
    }
}
```



### 3.Nginx访问认证模块

```bash
# ngx_http_auth_basic_module
```

#### 1）语法

```bash
#开启的登录认证，没有卵用
Syntax:	auth_basic string | off;
Default:	auth_basic off;
Context:	http, server, location, limit_except

#指定登录用的用户名密码文件
Syntax:	auth_basic_user_file file;
Default:	—
Context:	http, server, location, limit_except
```

#### 2）创建密码文件

```bash 
#创建密码文件需要用到 htpasswd
[root@web01 ~]# htpasswd -c /etc/nginx/auth_basic lhd
New password: 
Re-type new password: 
Adding password for user lhd

#添加一个登录用户
[root@web01 ~]# htpasswd /etc/nginx/auth_basic egon
New password: 
Re-type new password: 
Adding password for user egon

#密码文件内容
[root@web01 ~]# cat /etc/nginx/auth_basic
lhd:$apr1$A7d4BWYe$HzlIA7pjdMHBDJPuLBkvd/
egon:$apr1$psp0M3A5$601t7Am1BG3uINvuBVbFV0
```

#### 3）配置访问登录

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/www.autoindex.com.conf 
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;
    access_log /var/log/nginx/www.autoindex.com.log main;

    location / {
        root /code/autoindex;
        index index.html;
    }

    location /download {
        root /code/autoindex;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        auth_basic "性感荷官在线发牌！！！";
        auth_basic_user_file /etc/nginx/auth_basic;
    }
}
```



### 4.Nginx状态监控模块 

```bash
# ngx_http_stub_status_module

ngx_http_stub_status_module模块提供对nginx基本状态信息的访问。
默认情况下不构建此模块，应使用--with-http_stub_status_module配置参数启用它
```

#### 1）语法

```bash
Syntax:	stub_status;
Default:	—
Context:	server, location
```

#### 2）配置

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/www.autoindex.com.conf 
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;
    access_log /var/log/nginx/www.autoindex.com.log main;

    location / {
        root /code/autoindex;
        index index.html;
    }

    location /download {
        root /code/autoindex;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        auth_basic "性感荷官在线发牌！！！";
        auth_basic_user_file /etc/nginx/auth_basic;
    }

    location = /basic_status {
        stub_status;
    }
}
```

#### 3）访问

```bash
#访问 http://www.autoindex.com/basic_status

#nginx七种状态
Active connections: 2 
server accepts handled requests
 		2 		2 		2 
Reading: 0 Writing: 1 Waiting: 1

Active connections		#活跃的连接数
accepts					#TCP连接总数
handled					#成功的TCP连接数
requests				#成功的请求数
Reading					#读取的请求头
Writing					#响应
Waiting					#等待的请求数，开启了keepalive

# 注意, 一次TCP的连接，可以发起多次http的请求, 如下参数可配置进行验证
keepalive_timeout  0;   # 类似于关闭长连接
keepalive_timeout  65;  # 65s没有活动则断开连接
```

### 5.连接限制模块

```bash
# ngx_http_limit_conn_module
```

#### 1）语法

```bash
#设置限制的空间
Syntax:	limit_conn_zone key zone=name:size;
Default:	—
Context:	http

limit_conn_zone 	#设置空间的模块
key 				#指定空间存储的内容
zone				#指定空间
=name				#空间名字
:size;				#空间的大小

#调用限制的空间
Syntax:	limit_conn zone number;
Default:	—
Context:	http, server, location

limit_conn			#调用空间的模块
zone 				#空间的名字
number;				#指定可以同时连接的次数
```

#### 2）配置

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/www.autoindex.com.conf 
limit_conn_zone $remote_addr zone=conn_zone:10m;	#设置一个存储ip地址，空间名字为conn_zone,空间大小为10M的空间
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;;
    limit_conn conn_zone 1;		#调用conn_zone空间，限制每个ip同时只能连接一次
   
    location / {
        root /code/autoindex;
        index index.html;
    }
}
```



### 6.请求限制模块

#### 1）语法

```bash
#设置空间的语法
Syntax:	limit_req_zone key zone=name:size rate=rate [sync];
Default:	—
Context:	http

limit_req_zone			#设置空间的模块
key						#空间存储的内容
zone					#指定空间
=name					#空间的名字
:size 					#空间的大小
rate=rate [sync];		#读写速率

#调用的语法
Syntax:	limit_req zone=name [burst=number] [nodelay | delay=number];
Default:	—
Context:	http, server, location

limit_req 				#调用控件模块
zone=name 				#指定空间=空间的名字
[burst=number]			#允许多请求几次
[nodelay | delay=number]; #延时
```

#### 2）配置

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/www.autoindex.com.conf 

limit_conn_zone $remote_addr zone=conn_zone:10m;
limit_req_zone $remote_addr zone=req_zone:10m rate=1r/s;	#设置一个储存ip地址，储存大小为10m，空间名字req_zone，一秒只能请求一次的空间。
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;
    limit_conn conn_zone 1;
    limit_req zone=req_zone;	#调用空间名字是req_zone的空间
    #limit_req zone=req_zone burst=5 nodelay;			#调用空间名字是req_zone的空间，最大限度可以同时访问五次，没有延迟

    location / {
        root /code/autoindex;
        index index.html;
    }
}
```

#### 3）测试

```bash
[root@web01 ~]# ab -n 20000 -c 20 http://www.autoindex.com/index.html 
		-n		  #请求的次数
		-c		  #一次请求并发的次数
```
