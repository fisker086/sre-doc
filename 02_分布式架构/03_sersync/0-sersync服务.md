## 一、NFS总结

### 1.NFS优点

```bash
1.NFS文件系统简单易用、方便部署、数据可靠、服务稳定、满足中小企业需求。
2.NFS文件系统内存放的数据都在文件系统之上，所有数据都是能看得见。
```

### 2.NFS缺点

```bash
1.存在单点故障, 如果构建高可用维护麻烦web->nfs()->backup
2.NFS数据明文, 并不对数据做任何校验。
3.客户端挂载NFS服务没有密码验证, 安全性一般(内网使用)
```

### 3.NFS应用建议

```bash
1.生产场景应将静态数据尽可能往前端推, 减少后端存储压力
2.必须将存储里的静态资源通过CDN缓存jpg\png\mp4\avi\css\js
3.如果没有缓存或架构本身历史遗留问题太大, 在多存储也无用
```



## 二、Rsync+NFS 解决单点故障

### 1.准备环境

| 主机   | 角色                   | IP                   |
| ------ | ---------------------- | -------------------- |
| web01  | nfs客户端、rsync客户端 | 172.16.1.7、10.0.0.7 |
| nfs    | nfs服务端、rsync客户端 | 172.16.1.31          |
| backup | rsync服务端            | 172.16.1.41          |



### 2.web01搭建上传作业代码

#### 1）关闭防火墙和selinux

#### 2）安装httpd和php

```bash
[root@web01 ~]# yum install -y httpd php
```

#### 3）配置httpd

```bash
[root@web01 ~]# vim /etc/httpd/conf/httpd.conf
User www
Group www
```

#### 4）创建用户

```bash
[root@web01 ~]# groupadd www -g 666
[root@web01 ~]# useradd www -u 666 -g 666
```

#### 5）启动服务

```bash
[root@web01 ~]# systemctl start httpd

#验证服务启动
[root@web01 ~]# ps -ef | grep httpd
```

#### 6）配置网站代码

```bash
[root@web01 ~]# cd /var/www/html/
[root@web01 html]# rz
[root@web01 html]# ll
total 28
-rw-r--r--. 1 root root 26995 Nov 22 16:47 kaoshi.zip
[root@web01 html]# unzip kaoshi.zip         
[root@web01 html]# ll
total 80
-rw-r--r--. 1 root root 38772 Apr 27  2018 bg.jpg
-rw-r--r--. 1 root root  2633 May  4  2018 index.html
-rw-r--r--. 1 root root    52 May 10  2018 info.php
-rw-r--r--. 1 root root 26995 Nov 22 16:47 kaoshi.zip
-rw-r--r--. 1 root root  1192 Jan 10  2020 upload_file.php
```

#### 7）修改站点目录权限

```bash
[root@web01 html]# chown -R www.www /var/www/html/
```



### 3.NFS服务器搭建NFS服务端

#### 1）关闭防火墙和selinux

#### 2）安装NFS和rpcbind

```bash
[root@nfs ~]# yum install -y nfs-utils
```

#### 3）配置NFS

```bash
[root@nfs ~]# vim /etc/exports
/data 172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)
```

#### 4）创建用户

```bash
[root@nfs ~]# groupadd www -g 666
[root@nfs ~]# useradd www -u 666 -g 666
```

#### 5）创建目录并授权

```bash
[root@nfs ~]# mkdir /data
[root@nfs ~]# chown -R www.www /data/
```

#### 6）启动服务

```bash
[root@nfs ~]# systemctl start nfs
```

#### 7）验证配置

```bash
[root@nfs ~]# cat /var/lib/nfs/etab 
/data	172.16.1.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=666,anongid=666,sec=sys,rw,secure,root_squash,all_squash)
```



### 4.web端挂载NFS服务器

#### 1）创建挂载目录并授权

```bash
[root@web01 ~]# mkdir /var/www/html/upload
[root@web01 ~]# chown -R www.www /var/www/html/upload
```

#### 2）安装rpcbind和nfs

```bash
[root@web01 ~]# yum install -y nfs-utils rpcbind
```

#### 3）查看挂载点

```bash
[root@web01 ~]# showmount -e 172.16.1.31
Export list for 172.16.1.31:
/data 172.16.1.0/24
```

#### 4）挂载

```bash
[root@web01 ~]# mount -t nfs 172.16.1.31:/data /var/www/html/upload

#验证挂载
[root@web01 ~]# df -h
Filesystem         Size  Used Avail Use% Mounted on
/dev/sda3           18G  1.7G   17G   9% /
devtmpfs           476M     0  476M   0% /dev
tmpfs              487M     0  487M   0% /dev/shm
tmpfs              487M  7.7M  479M   2% /run
tmpfs              487M     0  487M   0% /sys/fs/cgroup
/dev/sda1         1014M  127M  888M  13% /boot
tmpfs               98M     0   98M   0% /run/user/0
172.16.1.31:/data   18G  1.6G   17G   9% /var/www/html/upload
```



### 5.backup服务器搭建rsync服务端

#### 1）安装

```bash
[root@backup ~]# yum install -y rsync
```

#### 2）配置rsync

```bash
[root@backup ~]# vim /etc/rsyncd.conf
uid = www
gid = www
port = 873
fake super = yes
use chroot = no
max connections = 200
timeout = 600
ignore errors
read only = false
list = true
auth users = rsync_backup
secrets file = /etc/rsync.passwd
log file = /var/log/rsyncd.log
#####################################
[web_data]
comment = "该备份文件是web端挂载到nfs服务器的文件"
path = /data
```

#### 3）创建用户

```bash
[root@backup ~]# groupadd www -g 666
[root@backup ~]# useradd www -u 666 -g 666
```

#### 4）创建密码文件并授权

```bash
[root@backup ~]# echo "rsync_backup:123456" > /etc/rsync.passwd

#授权
[root@backup ~]# chmod 600 /etc/rsync.passwd
```

#### 5）创建真实目录并授权

```bash
[root@backup ~]# mkdir /data
[root@backup ~]# chown -R www.www /data/
```

#### 6）启动服务

```bash
[root@backup ~]# systemctl start rsyncd

#验证启动
[root@backup ~]# ps -ef | grep rsync
root      25733      1  0 15:24 ?        00:00:00 /usr/bin/rsync --daemon --no-detach
```



### 6.NFS服务器实时备份data目录到rsync

#### 1）安装inotify-tools

```bash
[root@nfs ~]# yum install -y inotify-tools
```

#### 2）编写脚本

```bash
[root@nfs ~]# vim rsyn-inotify.sh
#!/bin/bash
export RSYNC_PASSWORD=123456
dir=/data
/usr/bin/inotifywait -mrq --format '%w %f' -e create,delete,attrib,close_write $dir | while read line;do
        cd $dir && rsync -az -R --delete . rsync_backup@172.16.1.41::web_data >/dev/null 2>&1
done &
```

#### 3）启动脚本

```bash
[root@nfs ~]# sh rsyn-inotify.sh 
[root@nfs ~]# ps -ef | grep rsyn
root       9224      1  0 15:30 pts/0    00:00:00 sh rsyn-inotify.sh
```



### 7.测试

```bash
1.访问交作业页面（可以配置windows的hosts   C:\Windows\System32\drivers\etc）
2.上传作业测试，页面成功
3.查看web服务器上是否有文件
    [root@web01 ~]# ll /var/www/html/upload
    total 32
    -rw-r--r--. 1 www www 30419 Nov 23 15:41 2_nfs.jpg
4.查看NFS挂载目录下是否有文件
    [root@nfs ~]# ll /data/
    total 32
    -rw-r--r-- 1 www www 30419 Nov 23 15:41 2_nfs.jpg
5.查看backup服务器上是否有文件
	[root@backup ~]# ll /data/
    total 100
    -rw-r--r--. 1 www www 30419 Nov 23 15:41 2_nfs.jpg
    -rw-r--r--. 1 www www 69097 Nov 23 15:43 3_nfs.jpg
```



### 8.backup服务器安装NFS服务端

#### 1）安装NFS

```bash
[root@backup ~]# yum install -y nfs-utils
```

#### 2）配置NFS

```bash
[root@backup ~]# cat /etc/exports
/data 172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)
```

#### 3）启动服务

```bash
[root@backup ~]# systemctl start nfs
```



### 9.故障切换

#### 1）NFS端搞事情

```bash
[root@nfs ~]# systemctl stop nfs
```

#### 2）切换挂载的机器

```bash
[root@web01 ~]# umount -lf /var/www/html/upload
[root@web01 ~]# mount -t nfs 172.16.1.41:/data /var/www/html/upload
```





## 三、sersync 实时同步

### 1.什么是实时同步

```bash
实时同步是一种只要当前目录发生变化则会触发一个事件，事件触发后会将变化的目录同步至远程服务器。
```

### 2.为什么使用

```bash
保证数据的连续性, 减少人力维护成本,解决nfs单点故障
```

### 3.实时同步原理

```bash
利用inotify通知接口，监控本地目录变化，只要监控目标发生变化，就触发事件，执行相应操作。
```

### 4.实时同步工具选择

```bash
sersync + RSYNC(√)、inotify + rsync

Inotify是一个通知接口，用来监控文件系统的各种变化，如果文件存取，删除，移动。可以非常方便地实现文件异动告警，增量备份，并针对目录或文件的变化及时作出响应。rsync + inotify 可以做到实时同步

sersync是国人基于rsync+inotify-tools开发的工具，不仅保留了优点同时还强化了实时监控，文件过滤，简化配置等功能，帮助用户提高运行效率，节省时间和网络资源。

sersync项目地址：https://github.com/wsgzao/sersync
```



### 5.安装sersync

| 角色   |             |              |
| ------ | ----------- | ------------ |
| NFS    | 172.16.1.31 | nfs/sersync  |
| backup | 172.16.1.41 | rsync-server |

#### 1）安装依赖环境

```bash
[root@nfs ~]# yum install -y inotify-tools rsync
```

#### 2）上传或下载sersync包

```bash
[root@nfs ~]# rz sersync2.5.4_64bit_binary_stable_final.tar.gz

[root@nfs ~]# wget                                                                                          https://raw.githubusercontent.com/wsgzao/sersync/master/sersync2.5.4_64bit_binary_stable_final.tar.gz
```

#### 3）解压安装

```bash
[root@nfs ~]# tar xf sersync2.5.4_64bit_binary_stable_final.tar.gz
```

#### 4）移动目录并改名

```bash
[root@nfs ~]# mv GNU-Linux-x86 /usr/local/sersync
```

#### 5）配置sersync

```bash
[root@nfs ~]# confxml.xml confxml.bak					#备份配置文件
[root@nfs ~]# vim /usr/local/sersync/confxml.xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<head version="2.5">
    <host hostip="localhost" port="8008"></host>		#本机ip地址和端口
    <debug start="false"/>							  #是否打开调试模式
    <fileSystem xfs="false"/>						  #是否支持xfs文件系统
    <filter start="false">							  #是否过滤，是否排除名称中含有制定字符串的文件的同步
	<exclude expression="(.*)\.svn"></exclude>
	<exclude expression="(.*)\.gz"></exclude>
	<exclude expression="^info/*"></exclude>
	<exclude expression="^static/*"></exclude>
    </filter>
    <inotify>
    	#inotify 监控的动作
        <delete start="true"/>						  #删除动作
        <createFolder start="true"/>				   #创建文件夹动作
        <createFile start="true"/>					  #创建文件动作
        <closeWrite start="true"/>					  #写入完成动作
        <moveFrom start="true"/>					  #移动来自动作
        <moveTo start="true"/>						  #移动到动作
        <attrib start="true"/>						  #属性被更改
        <modify start="true"/>						  #修改动作
    </inotify>
  
    <sersync>
        <localpath watch="/data">                       #监控的目录
            <remote ip="172.16.1.41" name="web_data"/>        	#远端rsync服务器的地址和模块
        </localpath>
        <rsync>
            <commonParams params="-az"/>        		#rsync的参数
            <auth start="true" users="rsync_backup" passwordfile="/etc/rsync.passwd"/>
            #开启认证				#虚拟用户					#指定虚拟用户的密码文件
            #如果远端rsync服务不是873端口，则开启并修改
            <userDefinedPort start="false" port="874"/><!-- port=874             #如果远端rsync服务不是873端口，则开启并修改
            <timeout start="false" time="100"/><!-- timeout=100 -->			#超时时间
            <ssh start="false"/>
        </rsync>
        #错误日志存储路径
        <failLog path="/tmp/rsync_fail_log.sh" timeToExecute="60"/><!--default every 60mins execute once-->        #定时任务，开启后，600分钟默认全备一次
        #定时任务，开启后，600分钟默认全备一次
        <crontab start="false" schedule="600"><!--600mins-->        #定时任务，开启后，600分钟默认全备一次
            <crontabfilter start="false">
                <exclude expression="*.php"></exclude>
                <exclude expression="info/*"></exclude>
            </crontabfilter>
        </crontab>
        <plugin start="false" name="command"/>
    </sersync>
  <plugin name="command">  #扩展插件功能的配置举例
	<param prefix="/bin/sh" suffix="" ignoreError="true"/>	<!--prefix /opt/tongbu/mmm.sh suffix-->
	<filter start="false">
	    <include expression="(.*)\.php"/>
	    <include expression="(.*)\.sh"/>
	</filter>
    </plugin>

    <plugin name="socket">		#扩展插件功能的配置举例
	<localpath watch="/opt/tongbu">
	    <deshost ip="192.168.138.20" port="8009"/>
	</localpath>
    </plugin>
    <plugin name="refreshCDN">				#扩展插件功能的配置举例
	<localpath watch="/data0/htdocs/cms.xoyo.com/site/">
	    <cdninfo domainname="ccms.chinacache.com" port="80" username="xxxx" passwd="xxxx"/>
	    <sendurl base="http://pic.xoyo.com/cms"/>
	    <regexurl regex="false" match="cms.xoyo.com/site([/a-zA-Z0-9]*).xoyo.com/images"/>
	</localpath>
    </plugin>
</head>

```

#### 6）创建虚拟用户密码文件

```bash
[root@nfs ~]# echo '123456' > /etc/rsync.passwd
[root@nfs ~]# chmod 600 /etc/rsync.passwd 
```

#### 7）启动服务

```bash
[root@nfs ~]# /usr/local/sersync/sersync2 -h
set the system param
execute：echo 50000000 > /proc/sys/fs/inotify/max_user_watches
execute：echo 327679 > /proc/sys/fs/inotify/max_queued_events
parse the command param
_______________________________________________________
参数-d:启用守护进程模式
参数-r:在监控前，将监控目录与远程主机用rsync命令推送一遍
参数-n: 指定开启守护线程的数量，默认为10个
参数-o:指定配置文件，默认使用confxml.xml文件
参数-m:单独启用其他模块，使用 -m refreshCDN 开启刷新CDN模块
参数-m:单独启用其他模块，使用 -m socket 开启socket模块
参数-m:单独启用其他模块，使用 -m http 开启http模块
不加-m参数，则默认执行同步程序

[root@nfs ~]# /usr/local/sersync/sersync2 -dro /usr/local/sersync/confxml.xml
```



## 四、扩展分享

### 1.rsync配置多个模块及目录

#### 1）rsync服务端

```bash
[root@backup ~]# vim /etc/rsyncd.conf 
uid = www
gid = www
port = 873
fake super = yes
use chroot = no
max connections = 200
timeout = 600
ignore errors
read only = false
list = true
auth users = rsync_backup
secrets file = /etc/rsync.passwd
log file = /var/log/rsyncd.log
#####################################
[web_data]
comment = "该备份文件是web端挂载到nfs服务器的文件"
path = /data

[backup]
comment = "该目录备份服务器日常备份文件"
path = /backup
auth users = rsync_user
secrets file = /etc/rsync.password
```

#### 2）rsync客户端

```bash
[root@nfs ~]# rsync -avz . rsync_backup@172.16.1.41::web_data   
Password: 123456
sending incremental file list
./
sent 47 bytes  received 23 bytes  28.00 bytes/sec
total size is 0  speedup is 0.00

#使用两个模块推送
[root@nfs ~]# rsync -avz . rsync_user@172.16.1.41::backup
Password: 678910
sending incremental file list
./
sent 47 bytes  received 23 bytes  20.00 bytes/sec
total size is 0  speedup is 0.00

#使用密码文件推送
[root@nfs ~]# echo '123456' > /etc/rsync.passwd
[root@nfs ~]# echo '678910' > /etc/rsync.password

[root@nfs ~]# chmod 600 /etc/rsync.passwd
[root@nfs ~]# chmod 600 /etc/rsync.password

[root@nfs ~]# rsync -avz . rsync_backup@172.16.1.41::web_data --password-file=/etc/rsync.passwd
[root@nfs ~]# rsync -avz . rsync_user@172.16.1.41::backup --password-file=/etc/rsync.password
```



### 2.NFS配置多个挂载目录

#### 1）NFS配置多个可挂载目录

```bash
[root@nfs ~]# vim /etc/exports
/data 172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)
/pic 172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)
```

#### 2）NFS客户端查看

```bash
[root@web01 ~]# showmount -e 172.16.1.31
Export list for 172.16.1.31:
/pic  172.16.1.0/24
/data 172.16.1.0/24
```



### 3.sersync配置启动多个实时备份

#### 1）第一个配置文件

```bash
[root@nfs ~]# cat /usr/local/sersync/confxml.xml
... ...
    <host hostip="localhost" port="8008"></host>
    	<localpath watch="/data">
	    <remote ip="172.16.1.41" name="web_data"/>
	... ...
	<rsync>
	    <commonParams params="-az"/>
	    <auth start="true" users="rsync_backup" passwordfile="/etc/rsync.passwd"/>
	... ...
... ...
```

#### 2）第二个配置文件

```bash
[root@nfs ~]# cat /usr/local/sersync/backup.xml
... ...
    <host hostip="localhost" port="8009"></host>
    	<localpath watch="/backup">
	    <remote ip="172.16.1.41" name="backup"/>
	... ...
	<rsync>
	    <commonParams params="-az"/>
	    <auth start="true" users="rsync_user" passwordfile="/etc/rsync.password"/>
	... ...
... ...
```

#### 3）启动多个sersync

```bash
[root@nfs ~]# /usr/local/sersync/sersync2 -dro /usr/local/sersync/confxml.xml
[root@nfs ~]# /usr/local/sersync/sersync2 -dro /usr/local/sersync/backup.xml

#查看进程
[root@nfs ~]# !ps
ps -ef | grep sersync
root       9601      1  0 17:03 ?        00:00:00 /usr/local/sersync/sersync2 -dro /usr/local/sersync/confxml.xml
root       9938      1  0 17:52 ?        00:00:00 /usr/local/sersync/sersync2 -dro /usr/local/sersync/backup.xml
root       9976   9270  0 17:56 pts/1    00:00:00 grep --color=auto sersync
```
