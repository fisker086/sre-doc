# lnmp架构简单搭建

## 一、nginx的location配置

```bash
使用Nginx Location可以控制访问网站的路径,但一个server可以有多个location配置, 多个location的优先级该如何区分
```

### 1.语法

```bash
Syntax:	location [ = | ~ | ~* | ^~ ] uri { ... }
		location @name { ... }
Default:	—
Context:	server, location
```

### 2.location匹配符

| **匹配符** | **匹配规则**                 | **优先级** |
| ---------- | ---------------------------- | ---------- |
| =          | 精确匹配                     | 1          |
| ^~         | 以某个字符串开头             | 2          |
| ~          | 区分大小写的正则匹配         | 3          |
| ~*         | 不区分大小写的正则匹配       | 3          |
| /          | 通用匹配，任何请求都会匹配到 | 4          |

### 3.优先级验证

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/youxianji.conf
server {
    listen 80;
    server_name linux.test.com;
    location / {
        default_type text/html;
        return 200 "location /";
    }
 
    location =/ {
        default_type text/html;
        return 200 "location =/";
    }
 
    location ~ / {
        default_type text/html;
        return 200 "location ~/";
    }
 
    # location ^~ / {
    #   default_type text/html;
    #   return 200 "location ^~";
    # }
}
```

### 4.Locaiton应用场景

```bash
# 通用匹配，任何请求都会匹配到
location / {
    ...
}
 
# 严格区分大小写，匹配以.php结尾的都走这个location    
location ~ \.php$ {
    ...
}
 
# 严格区分大小写，匹配以.jsp结尾的都走这个location 
location ~ \.jsp$ {
    ...
}
 
# 不区分大小写匹配，只要用户访问.jpg,gif,png,js,css 都走这条location
location ~* .*\.(jpg|gif|png|js|css)$ {
    ...
}

http://linux.test.com/1.PHP
http://linux.test.com/1.JPG
http://linux.test.com/1.jsp
http://linux.test.com/1.Gif
http://linux.test.com/1.PnG
http://linux.test.com/1.JsP
```



## 二、LNMP架构

### 1.简介

```bash
LNMP是一套技术的组合，L=Linux、N=Nginx、M~=MySQL、P~=PHP
不仅仅只有这些服务，还有很多
redis\elasticsearch\kibana\logstash\zabbix\git\jenkins\kafka\hbase\hadoop\spark\flink
```

### 2.LNMP架构工作方式

```bash
首先Nginx服务是不能处理动态请求，那么当用户发起动态请求时, Nginx又是如何进行处理的。
	1.静态请求：请求的内容是静态文件就是静态请求
		1）静态文件：文件上传到服务器，永远不会改变的文件就是静态文件
		2）html就是一个标准的静态文件
	2.动态请求：请求的内容是动态的就是动态请求
		1）不是真实存在服务器上的内容，是通过数据库或者其他服务拼凑成的数据

当用户发起http请求，请求会被Nginx处理，如果是静态资源请求Nginx则直接返回，如果是动态请求Nginx则通过fastcgi协议转交给后端的PHP程序处理，具体如下图所示
```

### 3.访问流程

```bash
1.浏览器输入域名，浏览器会拿着域名取DNS服务器解析
2.DNS服务器会将域名解析成IP
3.浏览器会去与IP对应服务器建立TCP\IP连接
4.连接建立完成，会向服务器发起请求，请求nginx
5.nginx会判断请求是动态的还是静态的
	#静态请求
	location \.jpg$ {
        root /code;
	}
	#动态请求
	location \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        ... ...
	}
6.如果是静态请求，nginx去code目录获取，直接返回
7.如果是动态请求，nginx会通过fastcgi协议连接PHP服务的php-fpm管理进程
```

```bash
8.php-fpm管理进程会下发工作给 wrapper工作进程
9.wrapper工作进程判断是不是简单的php内容
10.如果只是php内容则使用php解析器解析后直接返回
11.如果还需要读取数据库，wrapper工作进程会去数据库读取数据，再返回数据
12.数据流转过程：
	1）请求：浏览器 > 负载均衡 > nginx > php-fpm > wrapper > mysql
	2）响应：mysql > wrapper > php-fpm > nginx > 负载均衡 > 浏览器
```



## 三、LNMP架构搭建

### 1.搭建nginx

#### 1）配置官方源

```bash
[root@web01 ~]# vim /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```

#### 2）安装nginx

```bash
[root@web01 ~]# yum install -y nginx
```

#### 3）配置nginx

```bash
[root@web01 ~]# vim /etc/nginx/nginx.conf 
user  www;
```

#### 4）创建用户

```bash
[root@web01 ~]# groupadd www -g 666
[root@web01 ~]# useradd www -u 666 -g 666 -s /sbin/nologin -M
```

#### 5）启动nginx

```bash
[root@web01 ~]# systemctl start nginx
[root@web01 ~]# systemctl enable nginx
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.

#验证启动
[root@web01 ~]# ps -ef | grep nginx
root       9953      1  0 11:17 ?        00:00:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
www        9954   9953  0 11:17 ?        00:00:00 nginx: worker process
```



### 2.安装PHP

#### 1）安装方式一

```bash
# rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

[root@nginx ~]# yum remove php-mysql-5.4 php php-fpm php-common

#配置第三方源
[root@nginx ~]# vim /etc/yum.repos.d/php.repo
[php-webtatic]
name = PHP Repository
baseurl = http://us-east.repo.webtatic.com/yum/el7/x86_64/
gpgcheck = 0

[root@nginx ~]# yum -y install php71w php71w-cli php71w-common php71w-devel php71w-embedded php71w-gd php71w-mcrypt php71w-mbstring php71w-pdo php71w-xml php71w-fpm php71w-mysqlnd php71w-opcache php71w-pecl-memcached php71w-pecl-redis php71w-pecl-mongodb
```

#### 2）安装方式二

```bash
1.上传包
[root@web01 ~]# rz
[root@web01 ~]# ll
-rw-r--r--. 1 root root 19889622 Nov 22 15:52 p-hp.tar.gz

2.解压包
[root@web01 ~]# tar xf php.tar.gz

3.本地安装php的rpm包
[root@web01 ~]# yum localinstall -y *.rpm
```

#### 3）配置php

```bash
[root@web01 ~]# vim /etc/php-fpm.d/www.conf 
user = www
group = www
```

#### 4）启动服务

```bash
[root@web01 ~]# systemctl start php-fpm
[root@web01 ~]# systemctl enable php-fpm
Created symlink from /etc/systemd/system/multi-user.target.wants/php-fpm.service to /usr/lib/systemd/system/php-fpm.service.
```

#### 5）验证启动

```bash
[root@web01 ~]# ps -ef | grep php-fpm
root      10195      1  0 11:29 ?        00:00:00 php-fpm: master process (/etc/php-fpm.conf)
www       10196  10195  0 11:29 ?        00:00:00 php-fpm: pool www
www       10197  10195  0 11:29 ?        00:00:00 php-fpm: pool www
www       10198  10195  0 11:29 ?        00:00:00 php-fpm: pool www
www       10199  10195  0 11:29 ?        00:00:00 php-fpm: pool www
www       10200  10195  0 11:29 ?        00:00:00 php-fpm: pool www
```



### 3.搭建交作业页面

#### 1）配置nginx

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/default.conf 
server {
    listen 80;
    server_name linux.zuoye.com;

    location / {
        root /code/zuoye;
        index index.html;
    }
}
```

#### 2）创建站点目录

```bash
[root@web01 ~]# mkdir /code/zuoye -p
```

#### 3）上传代码

```bash
[root@web01 ~]# cd /code/zuoye/
[root@web01 zuoye]# rz
[root@web01 zuoye]# ll
-rw-r--r--. 1 root root 26995 Nov 22 16:47 kaoshi.zip
[root@web01 zuoye]# unzip kaoshi.zip      
[root@web01 zuoye]# ll
-rw-r--r--. 1 root root 38772 Apr 27  2018 bg.jpg
-rw-r--r--. 1 root root  2633 May  4  2018 index.html
-rw-r--r--. 1 root root    52 May 10  2018 info.php
-rw-r--r--. 1 root root  1192 Jan 10  2020 upload_file.php
```

#### 4）修改代码中上传作业位置

```bash
[root@web01 ~]# vim /code/zuoye/upload_file.php
$wen="/code/zuoye/upload";
```

#### 5）授权

```bash
[root@web01 zuoye]# chown -R www.www /code/
```

#### 6）重启服务

```bash
[root@web01 ~]# systemctl restart nginx
```

#### 7）配置hosts访问测试

```bash
#配置hosts
10.0.0.7 linux.zuoye.com

#访问
http://linux.zuoye.com/

#测试
上传代码出错，报错405，因为nginx没办法处理php代码程序
```



### 4.关联nginx与php

#### 1）关联语法

```bash
#fastcgi_pass，nginx连接php的代理协议
Syntax:	fastcgi_pass address;
Default:	—
Context:	location, if in location

#指定请求的文件
Syntax:	fastcgi_param parameter value [if_not_empty];
Default:	—
Context:	http, server, location

#指定默认的php页面
Syntax:	fastcgi_index name;
Default:	—
Context:	http, server, location
```

#### 2）配置

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/default.conf 
server {
    listen 80;
    server_name linux.zuoye.com;

    location / {
        root /code/zuoye;
        index index.html;
    }

    location ~* \.php$ {
        fastcgi_pass localhost:9000;
        fastcgi_param SCRIPT_FILENAME /code/zuoye/$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

#### 3）访问页面测试

```bash
1.访问页面
http://linux.zuoye.com/

2.上传图片文件
成功

3.上传一个大文件
失败，413报错，说文件过大
#解决：修改配置文件上传文件大小配置
	[root@web01 ~]# vim /etc/nginx/nginx.conf
	http {
		... ...
		client_max_body_size 200m;
		... ...
	}
	[root@web01 ~]# systemctl restart nginx
	
	[root@web01 ~]# vim /etc/php.ini
	upload_max_filesize = 200M
	post_max_size = 200M
	[root@web01 ~]# systemctl restart php-fpm

4.重新上传文件测试
成功
```



### 5.搭建mariadb

#### 1）安装

```bash
[root@web01 ~]# yum install -y mariadb-server
```

#### 2）启动服务

```bash
[root@web01 ~]# systemctl start mariadb
[root@web01 ~]# systemctl enable mariadb
Created symlink from /etc/systemd/system/multi-user.target.wants/mariadb.service to /usr/lib/systemd/system/mariadb.service.
```

#### 3）验证启动

```bash
[root@web01 ~]# ps -ef | grep mariadb
mysql     11006  10841  1 12:06 ?        00:00:00 /usr/libexec/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib64/mysql/plugin --log-error=/var/log/mariadb/mariadb.log --pid-file=/var/run/mariadb/mariadb.pid --socket=/var/lib/mysql/mysql.sock
```

#### 4）连接

```bash
[root@web01 ~]# mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 2
Server version: 5.5.68-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;			#查看数据库
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
+--------------------+
4 rows in set (0.00 sec)
```

#### 5）设置数据库密码

```bash
[root@web01 ~]# mysqladmin -uroot password '123'

#使用密码连接数据库
[root@web01 ~]# mysql -uroot -p123
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 5
Server version: 5.5.68-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```



### 6.测试PHP和mariadb关联

#### 1）编写php测试连接数据库的代码

```bash
[root@web01 ~]# vim /code/zuoye/test.php
<?php
    $servername = "localhost";
    $username = "root";
    $password = "123";

    // 创建连接
    $conn = mysqli_connect($servername, $username, $password);

    // 检测连接
    if (!$conn) {
        die("Connection failed: " . mysqli_connect_error());
    }
    echo "小哥哥,php可以连接MySQL...";
?>

<img style='width:100%;height:100%;' src=https://blog.drig>
```

#### 2）访问测试

```bash
http://linux.zuoye.com/test.php
```



## 四、搭建wordpress博客

### 1.上传代码

```bash
[root@web01 code]# rz
[root@web01 code]# ll
-rw-r--r--. 1 root root 11098483 Sep 12 17:52 wordpress-5.0.3-zh_CN.tar.gz
```

### 2.解压代码

```bash
[root@web01 code]# tar xf wordpress-5.0.3-zh_CN.tar.gz
```

### 3.授权

```bash
[root@web01 code]# chown -R www.www wordpress
```

### 4.配置nginx

```bash
[root@web01 code]# vim /etc/nginx/conf.d/linux.wp.com.conf
server {
    listen 80;
    server_name linux.wp.com;

    location / {
        root /code/wordpress;
        index index.php; 
    }

    location ~* \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME /code/wordpress/$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

### 5.重启访问

```bash
#检查配置
[root@web01 code]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

#重启
[root@web01 code]# systemctl restart nginx
```

### 6.访问测试

```bash
#配置hosts
10.0.0.7 linux.wp.com

#访问
http://linux.wp.com/
```

### 7.创建数据库

```bash
[root@web01 code]# mysql -uroot -p123

MariaDB [(none)]> create database wordpress;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
| wordpress          |
+--------------------+
5 rows in set (0.00 sec)
```

### 8.根据页面提示操作
