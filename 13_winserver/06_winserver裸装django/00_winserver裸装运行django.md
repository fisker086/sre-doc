# winserver裸装解决wsgi模块

**windows不支持python中uwsgi模块，通过apache和mod_wsgi替代uwsgi。**

## 一、安装运行依赖库

https://visualstudio.microsoft.com/zh-hans/downloads/

## 二、mysql安装

### 1、下载安装包

![1](1.png)

![2](2.png)

### 2、解压到对应目录

![3](3.png)



### 3、配置环境变量

此电脑-->属性-->高级系统设置

![4](4.png)

![5](5.png)

### 4、创建数据存放目录

![6](6.png)

### 5、书写my.ini配置文件

```bash
[mysqld]
# 设置3306端口
port=3306
# 设置mysql的安装目录
basedir=C:\\www\\mysql-8.0.25-winx64
# 设置mysql数据库的数据的存放目录
datadir=C:\\www\\bbs_db_data
# 允许最大连接数
max_connections=200
#链接等待时间
wait_timeout=600
#允许最大packet
max_allowed_packet=64M
# 允许连接失败的次数。这是为了防止有人从该主机试图攻击数据库系统
max_connect_errors=10

#慢日志
slow_query_log=ON
slow_query_log_file=C:\\www\\mysql-8.0.25-winx64\\slow.log
#慢日志超时时间
long_query_time=1
#记录不走索引的sql语句到慢日志
log_queries_not_using_indexes=ON
#每分钟记录的慢日志数量
log_throttle_queries_not_using_indexes=10

#锁等待
lock_wait_timeout=60

#表名存储在磁盘是小写的，但是比较的时候是不区分大小写
lower_case_table_names=1

##binlog日志
log_bin=C:\\www\\mysql-8.0.25-winx64\\mysql-bin

#binlog日志格式
log-bin-index=C:\\www\\mysql-8.0.25-winx64\\mysql-bin.index

#bin_log日志大小
max_binlog_size=500M

#binlog日志记录格式
binlog_format=ROW

#binlog日志存放时间，30天
binlog_expire_logs_seconds=2592000

#binlog刷盘策略，每产生一次事务刷盘一次
sync_binlog=1

# 服务端使用的字符集默认为UTF8
character-set-server=utf8mb4
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB

##复制
##relaylog
relay_log=/var/lib/mysql/relay
relay_log_index=C:\\www\\mysql-8.0.25-winx64\\relay.index
max_relay_log_size=500M
# 在数据库启动后立即启动自动relay log恢复
relay_log_recovery=ON

##gtid复制
#开启GTID复制
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
server-id=1
#1062：容错：主键冲突，数据不一致
slave_skip_errors=1032,1062

[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8mb4
# 设置mysql数据库格式
prompt=siveco [\\d]>
[client]
# 设置mysql客户端连接服务端时默认使用的端口
port=3306
default-character-set=utf8mb4
```

### 6、数据库初始化

```bash
mysqld --user=mysql --initialize-insecure
```



### 7、将mysql加入系统服务管理

**只能通过cmd才能成功**

```bash
sc create MySQL binPath= "C:\www\mysql-8.0.25-winx64\bin\mysqld.exe"
```

![7](7.png)



### 8、mysql的启停

```bash
net start MySQL
net stop MySQL

##删除mysql服务（前提需要先停止 MySQL 服务）
mysqld -remove
```



### 9、修改密码

```bash
PS C:\Users\Administrator> mysql -uroot -p
Enter password:
mysql> alter user root@'localhost' identified by 'bbX123456';
mysql> flush privileges;
```



## 三、redis安装

### 1、下载安装包

**由于redis官方并不支持windows版本，但是巨硬在github维护过redis的windows版本**

https://github.com/microsoftarchive/redis/releases

**使用msi安装队windows来说是比较方便的方式**

![8](8.png)

### 2、双击运行安装包

![9](9.png)

![10](10.png)

![11](11.png)

### 3、修改redis配置

```bash
redis.windows-service.conf
...
#监听地址
bind 0.0.0.0
...
#登录密码
requirepass 123456
```

### 4、重启redis

> win+R -->cmd

```cmd
C:\Users\Administrator>net stop redis

Redis 服务已成功停止。


C:\Users\Administrator>net start redis
Redis 服务正在启动 .
Redis 服务已经启动成功。
```



### 5、安装redis桌面客户端

https://gitee.com/qishibo/AnotherRedisDesktopManager/releases



## 四、nginx安装

> 有的服务器默认安装了iis服务默认使用80端口，运行nginx需要该自己端口或者将iis服务关掉 

```cmd
iisreset　/RESTART 停止后启动
iisreset /START 启动IIS (如果停止)
iisreset /STOP 停止IIS (如果启动)
iisreset /REBOOT 重启电脑 
iisreset /REBOOTonERROR 如果停止IIS失败重启电脑
iisreset /NOFORCE 不用强迫IIS停止
iisreset /TIMEOUT:X 在X秒后,IIS被强制停止,除非 /NOFORCE 参数给出.  
 最方便的使用，当然你也可在CMD下运行：iisreset /start
```



### 1、下载安装包

http://nginx.org/en/download.html

![12](12.png)

### 2、解压到对应目录

![13](13.png)



### 3、下载winsw

**WinSW是一个可执行二进制文件，可用于包装和管理作为Windows服务的自定义进程**

https://repo.jenkins-ci.org/artifactory/releases/com/sun/winsw/winsw/2.9.0/

![14](14.png)

![15](15.png)

### 4、将下载好的程序放到nginx目录下并重命名一下

![16](16.png)

### 5、在nginx根目录编写nginx-service.xml文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<service>
 <id>Nginx</id>
 <name>Nginx</name>
 <description>High Performance Nginx Service</description>
 <logpath>C:\www\nginx-1.20.1\logs</logpath>
 <executable>nginx.exe</executable>
 <stopexecutable>nginx.exe</stopexecutable>
 <stopargument>-s</stopargument> 
 <stopargument>stop</stopargument>
 <logmode>rotate</logmode>
</service>
```

### 6、安装net framework 3.5

**添加新功能可参照mysql安装部分**

![17](17.png)

### 7、安装nginx系统服务

**运行winsw安装nginx**

>按下Win+X+A键
>打开命令提示符（管理员）

```cmd
C:\www\nginx-1.20.1\nginx-server.exe install
```

**winsw支持的其他操作**

>将nginx-server.exe所在目录加入环境变量可以直接操作

```bash
install     将服务安装到Windows Service Controller.
uninstall   卸载服务和上面相反的操作。
start       启动服务，该服务必须已经安装。
stop        停止服务。
stopwait    停止服务并等待，直到它实际上停止为止。
restart     重新启动服务。如果该服务当前未运行，则此命令的作用类似于start。
status      检查服务的当前状态。
```



### 8、安装完nginx服务后可以通过系统服务进行管理

```bash
net start nginx
net stop nginx
```



> IIS服务导致80端口占用问题

```cmd
iisreset　/RESTART 停止后启动
iisreset /START 启动IIS (如果停止)
iisreset /STOP 停止IIS (如果启动)
iisreset /REBOOT 重启电脑 
iisreset /REBOOTonERROR 如果停止IIS失败重启电脑
iisreset /NOFORCE 不用强迫IIS停止
iisreset /TIMEOUT:X 在X秒后,IIS被强制停止,除非 /NOFORCE 参数给出.  
 最方便的使用，当然你也可在CMD下运行：iisreset /start
```



### 9、强杀进程

> 如果nginx长时间关不掉可以强制杀死

**查看nginx服务进程**

```cmd
tasklist | findstr nginx
```



**杀死进程**

```cmd
tskill 进程ID
```



## 五、安装python

### 1、下载安装包，版本必须和开发高度统一

https://www.python.org/downloads

![18](18.png)

### 2、安装

![19](19.png)

![20](20.png)



### 3、上传代码

**先安装代码里面需要的依赖**

```bash
上传自己项目的代码，根据依赖先安装依赖
```



### 4、安装python依赖

#### 1.pip配置国内源

![21](21.png)

**创建pip文件夹，并在此文件夹下创建pip.ini**

![22](22.png)

**pip.ini文件内容**

```bash
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = https://pypi.tuna.tsinghua.edu.cn
```



#### 2.打开cmd

**运行之前先把requirements.txt里面的uwsgi删除，windows安装会报错**

```bash
pip install -r requirements.txt
```

**这里uwsgi会安装失败，windows不支持uwsgi，所以只能通过apache的方式去使用**

## 六、apache安装

### 1、下载安装包

>http://httpd.apache.org/

> vc14版本
>
> https://www.apachelounge.com/download/VC14/

![23](23.png)

![24](24.png)

![25](25.png)

![26](26.png)

### 2、解压到指定目录

![27](27.png)

### 3、配置环境变量

此电脑--》属性

![28](28.png)

![29](29.png)

![30](30.png)

![31](31.png)

### 4、配置apache配置文件

![32](32.png)

![33](33.png)

**配置文件校验**

![34](34.png)

### 5、apache主服务

httpd -k install -n Apache

![35](35.png)



### 6、apache启停

```bash
httpd -k start 		#不会提示详细的错误信息。
httpd -k start -n apache		#会提示详细的错误信息，其中的"apache"修改为你的Apache服务名,可以到计算机服务里找。 
httpd -k restart -n apache     #重启。
net start apache      #利用Windows托管服务命令。

httpd -k stop
httpd -k uninstall

Windows卸载服务命令：sc delete 服务名
```

**若Apache服务器软件不想用了，想要卸载，一定要先卸载apache服务，然后删除安装文件（切记，若直接删除安装路径的文件夹，会有残余文件在电脑，可能会造成不必要的麻烦），在cmd命令窗口先停止服务再卸载，（建议先停止服务再删除）：**

### 7、启动和访问apache

![36](36.png)

> http://127.0.0.1:10050/

![37](37.png)





## 七、mod_wsgi安装

> LoadModule wsgi_module modules/mod_wsgi.so 使用于apache配置为python2系列的运行环境，以插件so方式load可直接使用。

>在python3系列时，apache的管网各个版本中都找不到mod_wsgi.so，只能在https://www.lfd.uci.edu/~gohlke/pythonlibs/#mod_wsgi这个网站中下载whl文件，自行通过pip命令编译，集成到python的site-package包中，然后再到apache的配置文件指定python执行环境。

### 1、下载mod_wsgi文件

https://www.lfd.uci.edu/~gohlke/pythonlibs/#mod_wsgi

#### 1.文件命名说明

```bash
mod_wsgi‑4.7.0+ap24vc14‑cp35‑cp35m‑win_amd64.whl

文件格式说明：
mod_wsgi‑4.7.0:表示当前mod_wsgi版本是4.7.0
ap24vc14：表示apache版本是2.4，基于vc2014编译出来
cp35-cp35m：表示对应的python版本是3.5系列
win_amd64：表示对应的平台是windows 64bit
```

#### 2.根据python版本下载文件

> 我们python是3.6.7，系统是64位,那我们就选择cp36+amd64的版本文件

![38](38.png)



### 2、安装

**复制下载的mod_wsgi-4.7.1-cp36-cp36m-win_amd64.whl到python的scripts目录，然后执行pip3 install "C:\www\python3.6.7\Scriptsmod_wsgi-4.7.1-cp36-cp36m-win_amd64.whl"**

```cmd
C:\Windows\system32>cd C:\www\python3.6.7\Scripts
C:\www\python3.6.7\Scripts>pip3 install mod_wsgi-4.7.1-cp36-cp36m-win_amd64.whl
```

![39](39.png)



### 3、获取mod_wsgi配置

```cmd
mod_wsgi-express.exe module-config

LoadFile "c:/www/python3.6.7/python36.dll"
LoadModule wsgi_module "c:/www/python3.6.7/lib/site-packages/mod_wsgi/server/mod_wsgi.cp36-win_amd64.pyd"
WSGIPythonHome "c:/www/python3.6.7"
```

![40](40.png)



### 4、将获取的配置复制到apache配置目录当中

![41](41.png)



## 八、部署django

### 1、数据库配置

#### 1.导入数据

**sqlmould里面可能没有建库语句，需要手写一下**

```mysql
create database bbspy charset utf8mb4;
USE bbspy;
```

```bash
mysql -uroot -p < C:\www\sqlmould\bbs_mould.sql
```



#### 2.创建授权tcp/root用户

```mysql
create user root@'%' identified by 'bbX123456';
grant all on *.* to root@'%' with grant option;
flush privileges;
```



### 2、上传前端代码

#### 1.前端主站代码

![42](42.png)



#### 2.前端微信代码

![43](43.png)



### 3、上传后端代码

![](44.png)

### 4、修改项目配置

#### 1.数据库

![](45.png)



#### 2.redis

![](46.png)





### 5、apache配置

```bash

...
##--------------- Django项目部署配置 ---------------##
# 声明项目根目录变量
Define DjangoRoot "C:/www/bbs2.0/backed/bbs2.0"

#wsgi配置
LoadFile "c:/www/python3.6.7/python36.dll"
LoadModule wsgi_module "c:/www/python3.6.7/lib/site-packages/mod_wsgi/server/mod_wsgi.cp36-win_amd64.pyd"
WSGIPythonHome "c:/www/python3.6.7"

#指定项目的wsgi.py配置文件路径
WSGIScriptAlias / "C:/www/backed/bluebeeSync/wsgi.py"

#指定项目目录,并配置访问权限。WSGIPythonPath取代DocumentRoot配置，或者保留DocumentRoot一致
WSGIPythonPath "C:/www/backed/bbs2.0"
<Directory "C:/www/backed/bbs2.0/bluebeeSync">
<Files wsgi.py>
    Require all granted
</Files>
</Directory>

...
```



### 6、nginx配置

#### 1.主配置文件目录创建conf.d和cert目录

![](47.png)

#### 2.修改nginx主配置文件

```bash
#user  nobody;
worker_processes  1;

error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  120s 120s;
	keepalive_requests 10000;

    #gzip  on;
    include conf.d/*.conf;

}
```



#### 3.配置nginx子配置文件

>proxy_params 

```bash
proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_connect_timeout 60s;
proxy_read_timeout 60s;
proxy_send_timeout 60s;
proxy_buffering on;
proxy_buffer_size 8k;
proxy_buffers 8 8k;
```

>www.conf

```bash
upstream api {
    server 127.0.0.1:10050;
}

server {
    listen      80;
    server_name localhost;
   
    charset     utf-8;
    access_log C:/www/nginx-1.20.1/logs/bs2.0_demo_access.log;
    error_log C:/www/nginx-1.20.1/logs/bbs2.0_demo_error.log;
    client_max_body_size 75M;
    proxy_connect_timeout 300s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;
    send_timeout 300s;

    gzip on;
    gzip_min_length 1k;
    gzip_comp_level 6;
    gzip_buffers 4 16k;  
    gzip_http_version 1.1;
    gzip_types *;


    location /{
        proxy_pass http://api;
        include proxy_params;
    }
    location /w {
        alias C:/www/bbs2.0/front/bbs_w/dist/;
		try_files $uri $uri/ /w/index.html;
		rewrite ^/w/e=(.*) /w/?e=$1 redirect;
        rewrite ^/w/(.*)/e=(.*) /w/$1/?e=$2 redirect;
   }

    location /static/ {   
        alias C:/www/bbs2.0/front/bbs/dist/static/;
        index index.html index.htm;
    }

    location /home{
	    alias C:/www/bbs2.0/front/bbs/dist/;
        index index.html;
	    try_files $uri $uri/ /home/index.html;
   }
   
}
```

