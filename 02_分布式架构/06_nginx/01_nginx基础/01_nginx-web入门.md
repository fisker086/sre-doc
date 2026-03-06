# Nginx web基础

## 一、Nginx介绍

### 1、概述

```bash
Nginx是一个开源且高性能、可靠的http web服务、代理服务

开源：直接获取源代码
高性能：支持海量开发
可靠：服务稳定
```

### 2、Nginx特点

#### 1）高性能，高并发

```bash
nginx支持很高的并发，nginx在处理大量并发的情况下比其他web服务要快
```

#### 2）轻量且高扩展性

```bash
#轻量
功能模块少，只保留核心模块，其他代码模块化 (易读，便于二次开发，对于开发人员非常友好)

#高扩展性
需要什么模块再安装模块，不需要全部安装，并且还支持第三方模块
```

#### 3) 高可靠性

```bash
只要不过分几乎不会出现问题
其他的web服务需要每隔一段时间进行重启，nginx不需要
nginx的宕机时间，是99999级别
```

#### 4) 支持热部署

```bash
nginx可以再运行期间，更新迭代，代码部署
```

#### 5) 大多数公司都在用nginx

```bash
1.Nginx技术成熟，具备的功能是企业最常使用而且最需要的
2.适合当前主流架构趋势, 微服务、云架构、中间层
3.统一技术栈, 降低维护成本, 降低技术更新成本。
```

#### 6) Nginx使用的是Epool网络模型

```bash
Select: 当用户发起一次请求，select模型就会进行一次遍历扫描，从而导致性能低下。
Epoll: 当用户发起一次请求，epoll模型会直接进行处理，效率高效，并无连接限制。
```

#### 7）nginx应用场景

![nginx 应用场景](nginx%20%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF.png)



### 3、其他的web服务

```bash
1.apache：httpd，最早期使用的web服务，性能不高，操作难
2.nginx
	tengine：Tengine是由淘宝网发起的Web服务器项目。它在Nginx的基础上，针对大访问量网站的需求，添加了很多高级功能和特性
	openresty-nginx：OpenResty 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。
3.IIS：windows下的web服务
4.lighttpd：是一个德国人领导的开源 Web 服务器软件，其根本的目的是提供一个专门针对高性能网站，安全、快速、兼容性好并且灵活的 Web Server 环境。具有非常低的内存开销，CPU 占用率低，效能好，以及丰富的模块等特点。
5.GWS：google web server
6.BWS：baidu web server

Tomcat
Resin
weblogic
Jboss
```

## 二、Nginx安装

### 1、安装方式

```bash
1.epol源安装 		web01
2.官方源安装		  web03
3.源码包安装		  web03

#三种安装方式都需要安装nginx运行的依赖环境
yum install -y gcc gcc-c++ autoconf pcre pcre-devel make automake wget httpd-tools vim tree
```

### 2、epol源安装

```bash
[root@web01 ~]# yum install -y nginx
```

### 3、官方源安装

#### 1) 配置官方源

```bash
[root@web02 ~]# vim /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[root@web02 ~]# yum install -y nginx
```

### 4、源码包安装

#### 1）下载安装包

```bash
[root@web03 ~]# wget http://nginx.org/download/nginx-1.18.0.tar.gz
```

#### 2)  解压源码包

```bash
[root@web03 ~]# tar xf nginx-1.18.0.tar.gz 
```

#### 3）配置安装的环境（运行用户，安装目录等）

```bash
1.创建用户和组，且不创建用户的家目录
[root@web03 ~]# groupadd www -g 666
[root@web03 ~]# useradd www -u 666 -g 666 -s /sbin/nologin -M

2.创建一个安装目录
公司不指定安装目录时，默认安装到/usr/local/软件名/
公司指定的话，就要按照公司要求来
```

#### 4）生成Makefile

```bash
[root@web03 ~/nginx-1.18.0]# cd nginx-1.18.0/
[root@web03 ~/nginx-1.18.0]#./configure --prefix=/usr/local/nginx-1.18.0 --user=www --group=www --without-http_gzip_module
```

#### 5）编译安装,结束后去安装目录查看安装结果

```bash
1.编译安装
[root@web03 nginx-1.18.0]# make && make install

2.查看安装结果
[root@web03 /usr/local]# cd /usr/local/nginx-1.18.0/
[root@web03 /usr/local/nginx-1.18.0]# ll
total 0
drwxr-xr-x 2 root root 333 Nov 26 15:40 conf
drwxr-xr-x 2 root root  40 Nov 26 15:40 html
drwxr-xr-x 2 root root   6 Nov 26 15:40 logs
drwxr-xr-x 2 root root  19 Nov 26 15:40 sbin
```

#### 6）做软连接

```bash
[root@web03 /usr/local/nginx-1.18.0]# ln -s /usr/local/nginx-1.18.0 /usr/local/nginx
[root@web03 /usr/local]# ll
total 0
drwxr-xr-x. 2 root root  6 Apr 11  2018 bin
drwxr-xr-x. 2 root root  6 Apr 11  2018 etc
drwxr-xr-x. 2 root root  6 Apr 11  2018 games
drwxr-xr-x. 2 root root  6 Apr 11  2018 include
drwxr-xr-x. 2 root root  6 Apr 11  2018 lib
drwxr-xr-x. 2 root root  6 Apr 11  2018 lib64
drwxr-xr-x. 2 root root  6 Apr 11  2018 libexec
lrwxrwxrwx  1 root root 23 Nov 26 15:46 nginx -> /usr/local/nginx-1.18.0	##成功
drwxr-xr-x  6 root root 54 Nov 26 15:40 nginx-1.18.0
drwxr-xr-x. 2 root root  6 Apr 11  2018 sbin
drwxr-xr-x. 5 root root 49 Nov 17 20:30 share
drwxr-xr-x. 2 root root  6 Apr 11  2018 src
```

#### 7）配置环境变量

```bash
[root@web03 ~]# vim /etc/profile.d/nginx.sh
export PATH=$PATH:/usr/local/nginx/sbin
~                                      
[root@web03 ~]# source /etc/profile			#在当前bash环境下读取并执行/etc/profile中的命令
```

#### 8）system管理配置

```bash
#源码包安装后没有办法使用system管理，需要我们自己配置
[root@web03 /usr/lib/systemd/system]# vim /usr/lib/systemd/system/nginx.service
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop

[Install]
WantedBy=multi-user.target

[root@web03 ~]# systemctl daemon-reload					#加载新的unit （*.service）配置文件

```

**练习**

```bash
[root@web03 /usr/lib/systemd/system]# vim /usr/lib/systemd/system/nginx.service
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStart=/usr/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/usr/sbin/nginx -s stop

[Install]
WantedBy=multi-user.target
```



## 三、Nginx命令

```bash
nginx #启动nginx。	等价于systemctl start nginx

nginx -s reopen #重启Nginx。	等价于systemctl restart nginx

nginx -s reload #重新加载Nginx配置文件，然后以优雅的方式重启Nginx。 等价于systemctl reload nginx

nginx -s stop #强制停止Nginx服务。	等价于systemctl stop nginx

nginx -s quit #优雅地停止Nginx服务（即处理完所有请求后再停止服务）

nginx -t #检测配置文件是否有语法错误，然后退出

nginx -?,-h #打开帮助信息

nginx -v #显示版本信息并退出

nginx -V #显示版本和配置选项信息，然后退出

nginx -V 2>&1 | sed "s/\s\+--/\n --/g"	#模块分行输出，格式化输出

killall nginx #杀死所有nginx进程

systemctl enable nginx	#加入开机自启
	Centos6：
		启动：nginx
			service nginx start
			/etc/init.d/nginx start
		加入开机自启：
			chkconfig nginx on

nginx -T #检测配置文件是否有语法错误，转储并退出

nginx -q #在检测配置文件期间屏蔽非错误信息

nginx -p prefix #设置前缀路径(默认是:/usr/share/nginx/)

nginx -c filename #设置配置文件(默认是:/etc/nginx/nginx.conf)

nginx -g directives #设置配置文件外的全局指令


```



## 四、Nginx服务平滑无感知添加模块和升级

### 1、模块解析

```bash
[root@web02 ~]# nginx -V		#显示版本和配置选项信息，然后退出
nginx version: nginx/1.18.0		
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie

############################内容解析#######################################

nginx version: nginx/1.18.0					#nginx版本
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 	#
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments:
 --prefix=/etc/nginx			#nginx文件的安装路径，其它选项如果使用相对路径，那么以此路径为基础路径
 --sbin-path=/usr/sbin/nginx	#二进制程序的安装目录
 --modules-path=/usr/lib64/nginx/modules	#模块安装路径
 --conf-path=/etc/nginx/nginx.conf	#设置nginx的conf文件路径
 --error-log-path=/var/log/nginx/error.log	#设置nginx错误日志路径
 --http-log-path=/var/log/nginx/access.log	#设置nginx的访问日志路径
 --pid-path=/var/run/nginx.pid	#设置nginx的pid文件路径
 --lock-path=/var/run/nginx.lock	#设置lock文件临时存放的路径
 --http-client-body-temp-path=/var/cache/nginx/client_temp	#设置http客户端主体临时缓存路径
 --http-proxy-temp-path=/var/cache/nginx/proxy_temp		#设置http反向代理临时路径
 --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp		#设置http的fastcgi临时缓存路径
 --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp	#设置uwsgi临时目录
 --http-scgi-temp-path=/var/cache/nginx/scgi_temp	#scgi临时存放目录
 --user=nginx	#设置启动 worker 进程时所使用的非特权用户名
 --group=nginx	#设置启动 worker 进程时所使用的非特权用户组名
 --with-compat
 --with-file-aio
 --with-threads
 --with-http_addition_module
 --with-http_auth_request_module
 --with-http_dav_module
 --with-http_flv_module
 --with-http_gunzip_module
 --with-http_gzip_static_module
 --with-http_mp4_module
 --with-http_random_index_module
 --with-http_realip_module
 --with-http_secure_link_module
 --with-http_slice_module
 --with-http_ssl_module
 --with-http_stub_status_module
 --with-http_sub_module
 --with-http_v2_module
 --with-mail
 --with-mail_ssl_module
 --with-stream
 --with-stream_realip_module
 --with-stream_ssl_module
 --with-stream_ssl_preread_module
 --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong
 --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC'
 --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'

```



### 2.nginx服务添加模块

```bash
1.安装依赖
[root@web02 nginx-1.16.1]# yum install -y openssl openssl-devel

2.再生成一次
[root@web02 nginx-1.16.1]# ./configure --prefix=/usr/local/nginx-1.16.1-new --user=www --group=www --without-http_gzip_module --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'

3.安装
[root@web02 nginx-1.16.1]# make && make install

4.重做软连接
[root@web02 ~]# rm -rf /usr/local/nginx && ln -s /usr/local/nginx-1.16.1-new /usr/local/nginx

5.重启服务
[root@web02 nginx-1.16.1]# systemctl restart nginx
```



### 3.nginx升级

```bash
1.下载新版本的包
[root@web02 ~]# wget http://nginx.org/download/nginx-1.18.0.tar.gz

2.解压
[root@web02 ~]# tar xf nginx-1.18.0.tar.gz

3.生成
[root@web02 nginx-1.18.0]# cd nginx-1.18.0
[root@web02 nginx-1.18.0]# ./configure --prefix=/usr/local/nginx-1.18.0 --user=www --group=www --without-http_gzip_module --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module

4.编译安装
[root@web02 nginx-1.18.0]# make && make install

5.重做软连接
[root@web02 ~]# rm -rf /usr/local/nginx && ln -s /usr/local/nginx-1.20.0 /usr/local/nginx

6.重启服务
[root@web02 ~]# systemctl restart nginx
```

```bash
./configure --user=www --group=www --prefix=/usr/local/nginx-1.20.1 --sbin-path=/usr/sbin/nginx-1.20.1 --modules-path=/usr/lib64/nginx-1.20.1/modules --conf-path=/usr/local/nginx-1.20.1/conf/nginx.conf --error-log-path=/usr/local/nginx-1.20.1/logs/error.log --http-log-path=/usr/local/nginx-1.20.1/logs/access.log --pid-path=/usr/local/nginx-1.20.1/logs/nginx.pid --lock-path=/usr/local/nginx-1.20.1/logs/nginx.lock --http-client-body-temp-path=/usr/local/nginx-1.20.1/client_body_temp --http-proxy-temp-path=/usr/local/nginx-1.20.1/proxy_temp --http-fastcgi-temp-path=/usr/local/nginx-1.20.1/fastcgi_temp --http-uwsgi-temp-path=/usr/local/nginx-1.20.1/uwsgi_temp --http-scgi-temp-path=/usr/local/nginx-1.20.1/scgi_temp --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_ssl_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'
```

```bash
ln -s /usr/sbin/nginx /usr/bin/nginx
```



## 五、Nginx相关文件

## 

```bash
为了让大家更清晰的了解Nginx软件的全貌，可使用rpm -ql nginx查看整体的目录结构及对应的功能，如下表格整理了Nginx比较重要的配置文件
```

### 1.Nginx主配置文件

| **路径**                       | **类型** | **作用**         |
| ------------------------------ | -------- | ---------------- |
| /etc/nginx/nginx.conf          | 配置文件 | nginx主配置文件  |
| /etc/nginx/conf.d/default.conf | 配置文件 | 默认网站配置文件 |

### 2.Nginx代理相关参数文件

| **路径**                  | **类型** | **作用**            |
| ------------------------- | -------- | ------------------- |
| /etc/nginx/fastcgi_params | 配置文件 | Fastcgi代理配置文件 |
| /etc/nginx/scgi_params    | 配置文件 | scgi代理配置文件    |
| /etc/nginx/uwsgi_params   | 配置文件 | uwsgi代理配置文件   |

### 3.Nginx编码相关配置文件

| **路径**              | **类型** | **作用**              |
| --------------------- | -------- | --------------------- |
| /etc/nginx/win-utf    | 配置文件 | Nginx编码转换映射文件 |
| /etc/nginx/koi-utf    | 配置文件 | Nginx编码转换映射文件 |
| /etc/nginx/koi-win    | 配置文件 | Nginx编码转换映射文件 |
| /etc/nginx/mime.types | 配置文件 | Content-Type与扩展名  |

### 4.Nginx管理相关命令

| **路径**              | **类型** | **作用**                  |
| --------------------- | -------- | ------------------------- |
| /usr/sbin/nginx       | 命令     | Nginx命令行管理终端工具   |
| /usr/sbin/nginx-debug | 命令     | Nginx命令行与终端调试工具 |

### 5.Nginx日志相关目录与文件

| **路径**               | **类型** | **作用**              |
| ---------------------- | -------- | --------------------- |
| /var/log/nginx         | 目录     | Nginx默认存放日志目录 |
| /etc/logrotate.d/nginx | 配置文件 | Nginx默认的日志切割   |



## 五、nginx配置文件

## 

```bash
Nginx主配置文件/etc/nginx/nginx.conf是一个纯文本类型的文件，整个配置文件是以区块的形式组织的。一般，每个区块以一对大括号{}来表示开始与结束。

Nginx主配置文件整体分为三块进行学习，分别是CoreModule(核心模块)，EventModule(事件驱动模块)，HttpCoreModule(http内核模块)
```

### 1.配置文件内容

```bash
[root@web01 ~]# cat /etc/nginx/nginx.conf 
#########################核心模块####################
#指定启动的用户
user  www;
#nginx的worker进程的数量
worker_processes  1;
#指定错误日志存放的路径以及记录的级别 debug/info/notice/warn/error/emerg
error_log  /var/log/nginx/error.log warn;
#指定pid文件
pid        /var/run/nginx.pid;

########################事件驱动模块#################
events {
	#每个worker工作进程的最大连接数
    worker_connections  1024;
}

######################http内核模块###################
http {
	#包含，nginx可识别的文件类型
    include       /etc/nginx/mime.types;
    #当nginx不识别文件类型的时候，默认下载
    default_type  application/octet-stream;
	#指定日志格式，日志格式起个名字
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
	#指定访问日志存储路径与格式
    access_log  /var/log/nginx/access.log  main;
	#高效传输
    sendfile        on;
    #高效传输
    #tcp_nopush     on;
	#开启长连接
    keepalive_timeout  65;
	#开启压缩
    #gzip  on;
	#包含网站的配置文件
    include /etc/nginx/conf.d/*.conf;
    
    #一个server表示一个网站
    server {
    	#监听端口
        listen       80;
        #网站提供的域名
        server_name  localhost;
        #字符集
        charset utf8;
        #匹配、控制访问的网站站点
        location / {
        	#指定站点目录
            root   /usr/share/nginx/html;
            #指定默认访问的页面
            index  index.html index.htm;
        }
    }
}
```



## 六、搭建小游戏

### 1.编写史上最简单配置

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/game.conf
server {
    listen 80;
    server_name localhost;
    #server_name www.game.com;

    location / {
        root /code/tuixiangzi;
        index index.html;
    }
}
```

### 2.检查配置文件

```bash
[root@web01 code]# nginx -t
nginx: [warn] conflicting server name "localhost" on 0.0.0.0:80, ignored
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

#因为网站冲突
[root@web01 code]# mv /etc/nginx/conf.d/default.conf /tmp/
```

### 3.创建站点目录

```bash
[root@web01 ~]# mkdir /code/
```

### 4.上传代码包

```bash
[root@web01 code]# rz
[root@web01 code]# unzip tuixiangzi.zip
[root@web01 code]# mv HTML5 canvas小人推箱子小游戏 tuixinagzi
```

### 4）重载nginx

```bash
[root@web01 code]# systemctl restart nginx
```

### 6.访问页面玩游戏



## 七、再搭建一个游戏

### 1.编辑配置文件

```bash
[root@web01 code]# vim /etc/nginx/conf.d/gametwo.conf 
server {
    listen 80;
    server_name www.tank.com;

    location / {
        root /code/tank;
        index index.html;
    }
}

[root@web01 code]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

[root@web01 code]# systemctl restart nginx
```

### 2.上传代码

```bash
[root@web01 code]# cd /code/
[root@web01 code]# rz
[root@web01 code]# unzip tankedazhan.zip
[root@web01 code]# mv jQuery坦克大战网页小游戏 tank
```



### 3.配置windows下的hosts

```bash
位置：C:\Windows\System32\drivers\etc
10.0.0.7 www.game.com www.tank.com
```



## 作业：

```bash
1.准备三台机器，使用三种方式搭建nginx
2.在三台nginx上分别搭建三个小游戏
```









