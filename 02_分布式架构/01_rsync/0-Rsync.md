# Rsync服务

## 一、备份

### 1.什么是备份？

```bash
备份就是把重要的数据或者文件复制一份保存到另一个地方，实现不同主机之间的数据同步
```

### 2.为什么做备份？

```bash
数据在公司中是很重要！！！！
备份就是为了恢复
```

### 3.能不能不做备份

```bash
对于重要的数据一定要备份
对于不重要的数据可以不备份或者备份一部分
```

### 4.备份的工具

```bash
本地备份：cp
远程备份：scp   rsync
```



### 5、scp命令及参数

#### 1、概念及参数

```bash
Linux scp 命令用于 Linux 之间复制文件和目录。
scp 是 secure copy 的缩写, scp 是 linux 系统下基于 ssh 登陆进行安全的远程文件拷贝命令。
scp 是加密的，rcp 是不加密的，scp 是 rcp 的加强版。

选项参数
-1： 强制scp命令使用协议ssh1
-2： 强制scp命令使用协议ssh2
-4： 强制scp命令只使用IPv4寻址
-6： 强制scp命令只使用IPv6寻址
-B： 使用批处理模式（传输过程中不询问传输口令或短语）
-C： 允许压缩。（将-C标志传递给ssh，从而打开压缩功能）
-p：保留原文件的修改时间，访问时间和访问权限。
-q： 不显示传输进度条。
-r： 递归复制整个目录。
-v：详细方式显示输出。scp和ssh(1)会显示出整个过程的调试信息。这些信息用于调试连接，验证和配置问题。
-c cipher： 以cipher将数据传输进行加密，这个选项将直接传递给ssh。
-F ssh_config： 指定一个替代的ssh配置文件，此参数直接传递给ssh。
-i identity_file： 从指定文件中读取传输时使用的密钥文件，此参数直接传递给ssh。
-l limit： 限定用户所能使用的带宽，以Kbit/s为单位。
-o ssh_option： 如果习惯于使用ssh_config(5)中的参数传递方式，
-P port：注意是大写的P, port是指定数据传输用到的端口号 
-S program： 指定加密传输时所使用的程序。此程序必须能够理解ssh(1)的选项。

```

#### 2、应用实例

```bash
一、从本地复制文件到远程
    1、复制本地文件到远程目录
    scp /home/space/music/1.mp3 root@www.runoob.com:/home/root/others/music 
    scp 					 	#命令
    /home/space/music/1.mp3 	  #本地文件
    root						#远端服务器的的系统用户	
    @							#分隔符,以哪个用户身份登录服务器
    www.runoob.com 				 #远程服务器的ip或域名
    :							#分隔符，指定服务器的里面所在的目录
    /home/root/others/music/	  #复制到远程服务器的目录地址

    2、复制本地文件到远程目录下，并重命名
    scp /home/space/music/1.mp3 root@www.runoob.com:/home/root/others/music/001.mp3
    scp 					 		#命令
    /home/space/music/1.mp3 	      #本地文件
    root							#远端服务器的的系统用户	
    @								#分隔符,以哪个用户身份登录服务器
    www.runoob.com 					 #远程服务器的ip或域名
    :								#分隔符，指定服务器的里面所在的目录
    /home/root/others/music/001.mp3    #复制到远程服务器的目录下并重命名001.mp3

二、从本地复制目录到远程目录下
    1、指定用户名，命令执行后需要再输入密码
    scp -r /home/space/music/ root@www.runoob.com:/home/root/others/ 
    scp 						#命令
    -r 						    #选项
    /home/space/music/ 			 #本地目录
    root					    #远端服务器的的系统用户
    @						    #分隔符,以哪个用户身份登录服务器
    www.runoob.com				 #远程服务器的ip或域名
    :						    #分隔符，指定服务器的里面所在的目录
    /home/root/others/			 #复制到远程服务器的目录地址

	2、不指定用户名，命令执行后需要输入用户名和密码
	scp -r /home/space/music/ www.runoob.com:/home/root/others/ 
    scp 						#命令
    -r 						    #选项
    /home/space/music/ 			 #本地目录
    www.runoob.com				 #远程服务器的ip或域名
    :						    #分隔符，指定服务器的里面所在的目录
    /home/root/others/			 #复制到远程服务器的目录地址

三、从远程复制到本地
	1、scp root@www.runoob.com:/home/root/others/music /home/space/music/1.mp3 
	   scp -r www.runoob.com:/home/root/others/ /home/space/music/
	   
```

### 6、rsync常用参数

```bash
-a           #归档模式传输, 等于-tropgDl    -t -r -o -p -g -D -l
-v           #详细模式输出, 打印速率, 文件数量等
-z           #传输时进行压缩以提高效率
-r           #递归传输目录及子目录，即目录下得所有目录都同样传输。
-t           #保持文件时间信息
-o           #保持文件属主信息
-p           #保持文件权限
-g           #保持文件属组信息
-l           #保留软连接
-P           #显示同步的过程及传输时的进度等信息
-D           #保持设备文件信息
-L           #保留软连接指向的目标文件
-e           #使用的信道协议,指定替代rsh的shell程序
--exclude=PATTERN   #指定排除不需要传输的文件
--exclude-from=file #排除不需要的文件
--bwlimit=100       #限速传输
--partial           #断点续传
--delete            #让目标目录和源目录数据保持一致
--password-file=xxx #使用密码文件
--port	#指定端口传输
```

#### 1、--exclude-from=file  排除不需要的文件

```bash
#创建多个文件
[root@web01 ~]# touch {1..10}.txt
[root@web01 ~]# ll
total 0
drwxr-xr-x. 2 root root 6 Nov 19 08:44 dir
-rw-r--r--. 1 root root 0 Nov 19 08:44 file
-rw-r--r--. 1 root root 0 Nov 19 08:59 txt1
-rw-r--r--. 1 root root 0 Nov 19 08:59 txt10
-rw-r--r--. 1 root root 0 Nov 19 08:59 txt2
-rw-r--r--. 1 root root 0 Nov 19 08:59 txt3
-rw-r--r--. 1 root root 0 Nov 19 08:59 txt4
-rw-r--r--. 1 root root 0 Nov 19 08:59 txt5
-rw-r--r--. 1 root root 0 Nov 19 08:59 txt6
-rw-r--r--. 1 root root 0 Nov 19 08:59 txt7
-rw-r--r--. 1 root root 0 Nov 19 08:59 txt8
-rw-r--r--. 1 root root 0 Nov 19 08:59 txt9

#编辑文件写入要排除的文件名字
[root@web01 ~]# vim 1.txt 
txt1
txt2
txt3
txt4

#指定排除文件推送内容
[root@web01 ~]# rsync -avz ./* rsync_backup@172.16.1.41::backup --exclude-from=1.txt
sending incremental file list
1.txt
txt10
txt5
txt6
txt7
txt8
txt9
dir/

sent 468 bytes  received 165 bytes  422.00 bytes/sec
total size is 20  speedup is 0.03
#发现没有传输1.txt中写入名字的文件
```

#### 2、--bwlimit=100       限速传输

```bash
#创建一个1G的文件
[root@web01 ~]# dd if=/dev/zero of=./1.txt bs=1M count=1000

#限速1M每秒推送
[root@web01 ~]# rsync -avzP 1.txt rsync_backup@172.16.1.41::backup --bwlimit=1
sending incremental file list
1.txt
    114,130,944  10%    1.01MB/s    0:15:06

#限速10M每秒推送
[root@web01 ~]# rsync -avzP 1.txt rsync_backup@172.16.1.41::backup --bwlimit=10
sending incremental file list
1.txt
    262,078,464  24%    9.89MB/s
```

#### 3、delete 数据一致 （无差异同步）

```bash
#查看客户端数据
[root@web01 ~]# ll
total 0
-rw-r--r--. 1 root root 0 Nov 19 09:17 txt2
-rw-r--r--. 1 root root 0 Nov 19 09:17 txt3
-rw-r--r--. 1 root root 0 Nov 19 09:17 txt4
-rw-r--r--. 1 root root 0 Nov 19 09:17 txt5
-rw-r--r--. 1 root root 0 Nov 19 09:17 txt6
-rw-r--r--. 1 root root 0 Nov 19 09:17 txt7
-rw-r--r--. 1 root root 0 Nov 19 09:17 txt8
-rw-r--r--. 1 root root 0 Nov 19 09:17 txt9

#删除数据
[root@web01 ~]# rm -rf txt2
[root@web01 ~]# rm -rf txt3
[root@web01 ~]# rm -rf txt4

#执行数据一致同步
[root@web01 ~]# rsync -avz ./ rsync_backup@172.16.1.41::backup --delete
sending incremental file list
deleting txt4
deleting txt3
deleting txt2
./

sent 332 bytes  received 52 bytes  768.00 bytes/sec
total size is 7,746  speedup is 20.17

#查看服务端
[root@backup backup]# ll
total 0
-rw-r--r--. 1 rsync rsync 0 Nov 19 09:17 txt5
-rw-r--r--. 1 rsync rsync 0 Nov 19 09:17 txt6
-rw-r--r--. 1 rsync rsync 0 Nov 19 09:17 txt7
-rw-r--r--. 1 rsync rsync 0 Nov 19 09:17 txt8
-rw-r--r--. 1 rsync rsync 0 Nov 19 09:17 txt9

#注意：
拉取时：客户端数据与服务端数据一致，以服务端数据为准
推送时：服务端数据一客户端数据一致，一客户端数据为准
```



## 二、rsync服务介绍

### 1.简介

```bash
    rsync英文称为remote synchronizetion，从软件的名称就可以看出来，rsync具有可使本地和远程两台主机之间的数据快速复制同步镜像、远程备份的功能，这个功能类似于ssh带的scp命令，但是又优于scp命令的功能，scp每次都是全量拷贝，而rsync可以增量拷贝。当然，rsync还可以在本地主机的不同分区或目录之间全量及增量的复制数据，这又类似cp命令。但是同样也优于cp命令，cp每次都是全量拷贝，而rsync可以增量拷贝。

rsync官方地址：https://rsync.samba.org/
rsync监听端口：873
rsync运行模式：C/S   client/server

rsync简称叫做远程同步，可以实现不同主机之间的数据同步，还支持全量和增量
```

### 2.rsync特性

```bash
支持拷贝特殊文件，如连接文件、设备等。
可以有排除指定文件或目录同步的功能，相当于打包命令tar的排除功能。
可以做到保持原文件或目录的权限、时间、软硬链接、属主、组等所有属性均不改变 –p。
可以实现增量同步，既只同步发生变化的数据，因此数据传输效率很高（tar-N）。
可以使用rcp、rsh、ssh等方式来配合传输文件（rsync本身不对数据加密）。
可以通过socket（进程方式）传输文件和数据（服务端和客户端）*****。
支持匿名的活认证（无需系统用户）的进程模式传输，可以实现方便安全的进行数据备份和镜像。
```

### 3.生产场景备份方案

```bash
1.借助cron+rsync把所有客户服务器数据同步到备份服务器。
2.针对公司重要数据备份混乱状况和领导提出备份全网数据的解决方案。
3.通过本地打包备份，然后rsync结合inotify应用把全网数统一备份到一个固定存储服务器，然后在存储服务器上通过脚本检查并报警管理员备份结果。
4.定期将IDC机房的数据备份公司的内部服务器，防止机房地震及火灾问题导致数据丢失。
5.实时同步，解决存储服务器等的单点问题。
```



## 三、Rsync应用场景

### 1.备份方式

#### 1）全量备份

```bash
将数据完整的复制一份保留下了
```

![全量备份](%E5%85%A8%E9%87%8F%E5%A4%87%E4%BB%BD.png)

#### 2）增量备份

```bash
备份上一此备份后新增的数据
```

![增量备份](%E5%A2%9E%E9%87%8F%E5%A4%87%E4%BB%BD.png)



### 2.rsync的传输方式

```bash
push 推：
客户端将数据从本地推送至服务端

pull 拉：
客户端将数据从服务端拉取到本地
```

![上传](%E4%B8%8A%E4%BC%A0.png)

![下载，拉](%E4%B8%8B%E8%BD%BD%EF%BC%8C%E6%8B%89.png)



### 3.传输存在的问题

```bash
1.推的问题：当客户端服务器数量过多，容易造成数据推送缓慢
2.拉的问题：当客户端服务器数量过多，容易造成服务端压力过大
```



### 4.大量服务器备份场景

```bash
现在有2000台服务器，怎么有效快速的缓解推和拉存在的问题
```

![多server备份](%E5%A4%9Aserver%E5%A4%87%E4%BB%BD.png)



### 5.异地备份实现思路

![异地备份存储](%E5%BC%82%E5%9C%B0%E5%A4%87%E4%BB%BD%E5%AD%98%E5%82%A8.png)





## 四、Rsync传输模式

### 1.传输模式

```bash
1.本地方式（类似于cp，不支持推送和拉取，只是单纯的复制）
2.远程方式（类似于scp，又不同于scp），scp只支持全量备份，rsync支持增量备份和差异备份
3.守护进程方式（客户端和服务端）
```

### 2.本地方式

```bash
#语法：
rsync [OPTION]... SRC [SRC]... DEST
命令   选项        源文件       目标地址

#语法实例
[root@web01 ~]# rsync -avz 1.txt /tmp/

[root@web01 ~]# rsync -avz 1.txt /tmp/
			  命令   选项 源文件 目标目录 

#命令拆分
rsync 				#命令
-avz 				#选项
1.txt		 		#源文件
/tmp/				#目标目录

#类似于cp，但是cp是全量复制并且会修改文件属性，rsync是增量复制，会保证文件属性不变
```

### 3.远程方式

#### 1）pull 拉取数据的命令

```bash
#语法：
rsync [OPTION]... [USER@]HOST:SRC [DEST]


#实例：
[root@web01 ~]# rsync -avz root@172.16.1.41:/tmp/1.txt ./

#语法拆分
rsync 			#命令
-avz 			#选项
root			#远端服务器的系统用户
@				#分隔符
172.16.1.41		#远程主机的地址
:				#分隔符，代表主机下的....
/tmp/1.txt 		#远程主机的目录及文件
./				#当前主机的当前目录
```

#### 2）push 推送数据命令

```bash
#语法
rsync [OPTION]... SRC [SRC]... [USER@]HOST:DEST

#实例
[root@web01 ~]# rsync -avz ./1.txt root@172.16.1.41:/tmp

#语法拆分
rsync 				#命令
-avz 				#选项
1.txt				#当前服务器的本地文件
root				#远端服务器的系统用户
@					#分隔符
172.16.1.41			#远端主机的IP地址
:					#分隔符，代表主机下的....
/tmp				#远程主机的目录
```

#### 3)注意事项

```bash
1、[root@web01 ~]# rsync -avz root@172.16.1.41:/tmp/1.txt ./2.txt	#将远程服务器1.txt文件全量备份到当前目录下并重命名为2.txt

2、[root@web01 ~]# rsync -avz root@172.16.1.41:/tmp/ ./a/	#将远程服务器tmp目录全量备份到当前目录下并重命名为a

3、指定目录"/a"时，意思是"/a"这个目录及这个目录下的文件。指定目录"/a/"时，意思是"/a/"目录下的文件不包括目录
```





### 4.守护进程传输模式

#### 1）为什么使用守护进程模式

```bash
1.rsync传输时，使用的是系统用户和系统用户的密码，非常的不安全
2.使用普通用又会出现权限问题
```

#### 2）守护进程传输模式语法

##### 1> push 推送语法

```bash
#语法：
rsync [OPTION]... SRC [SRC]... [USER@]HOST::DEST

#实例：
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::backup

#语法拆分
rsync 				#命令	
-avz 				#选项
1.txt		 		#当前服务器的文件
rsync_backup		#rsync服务端配置的虚拟用户
@					#分隔符
172.16.1.41			#远程主机IP地址
::backup			#模块名

#推送过程中服务端目录必须是服务端配置的启动用户权限
```

##### 2> pull 拉取语法

```bash
#语法：
rsync [OPTION]... [USER@]HOST::SRC [DEST]

#示例：
[root@web01 ~]# rsync -avz rsync_backup@172.16.1.41::backup /tmp/

#语法拆分：
rsync 				#命令
-avz 				#选项
rsync_backup		#服务端定义的虚拟用户
@					#分隔符
172.16.1.41			#远程主机IP地址
::backup 			#模块名字
/tmp/				#当前主机的目录

#拉取过程中服务端目录不用设置rsync用户权限
```



### 5.守护进程模式实践

#### 1）环境准备

| 主机   | IP        | 主机角色    |
| ------ | --------- | ----------- |
| web01  | 10.0.0.7  | rsync客户端 |
| backup | 10.0.0.41 | rsync服务端 |

#### 2）服务端rsync

##### 1、服务端安装修改配置文件

```bash
[root@backup ~]# yum install -y rsync

#查找配置文件
[root@backup ~]# rpm -qc rsync
/etc/rsyncd.conf

#编辑配置文件
[root@backup ~]# cat /etc/rsyncd.conf
uid = rsync
gid = rsync
port = 873
fake super = yes
use chroot = no
max connections = 200
timeout = 600
ignore errors
read only = false
list = false
auth users = rsync_backup
secrets file = /etc/rsync.passwd
log file = /var/log/rsyncd.log
#####################################
[backup]
comment = welcome to xiaowuedu backup!
path = /backup

#配置文件详解
uid = rsync							#启动服务的用户id
gid = rsync							#启动服务用户的组id
port = 873							#服务默认监听端口
fake super = yes					#无须使用root用户启动
use chroot = no						#安全机制
max connections = 200				#最大连接数
timeout = 600						#超时时间
ignore errors						#忽略错误
read only = false					#只读权限
list = false						#查看模块列表
auth users = rsync_backup			 #定义虚拟用户（rsync传输过程使用的用户）
secrets file = /etc/rsync.passwd	 #定义虚拟用户的密码
log file = /var/log/rsyncd.log		 #日志文件
#####################################
[backup]								#模块
comment = welcome to xiaowuedu backup!	   #模块的备注
path = /backup							#服务器真实的路径
```

##### 2、创建配置文件运行需要的环境

```bash
1.创建用户
[root@backup ~]# useradd -M -s /sbin/nologin rsync

2.创建密码文件（一定不能有空格）
[root@backup ~]# echo "rsync_backup:123456" > /etc/rsync.passwd
[root@backup ~]# chmod 600 /etc/rsync.passwd

3.创建备份目录
[root@backup ~]# mkdir /backup

4.授权备份目录
[root@backup ~]# chown -R rsync.rsync /backup/
```

##### 3、启动服务

```bash
[root@backup ~]# systemctl start rsyncd

#验证启动
[root@backup ~]# netstat -lntp | grep 873
tcp        0      0 0.0.0.0:873             0.0.0.0:*               LISTEN      26370/rsync         
tcp6       0      0 :::873                  :::*                    LISTEN      26370/rsync         
[root@backup ~]# ps -ef | grep rsync
root      26370      1  0 11:08 ?        00:00:00 /usr/bin/rsync --daemon --no-detach
root      26408  25098  0 11:09 pts/1    00:00:00 grep --color=auto rsync
```

#### 3）客户端rsync

```bash
#下载安装
yum install -y rsync

#创建认证文件并修改权限
echo "123456" >/etc/rsync.passwd
chmod 600 /etc/rsync.passwd
```



### 6.客户端推拉数据方法

#### 1）方法一：自己输入密码

```bash
#输入密码的方式
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::backup
Password: 123456
sending incremental file list

sent 55 bytes  received 20 bytes  21.43 bytes/sec
total size is 194  speedup is 2.59
[root@web01 ~]# 
```

#### 2）方法二：设置密码文件，运行时读取

```bash
#指定密码文件的方式
1.创建密码文件
[root@web01 ~]# echo "123456" > /etc/rsync.passwd
[root@web01 ~]# chmod 600 /etc/rsync.passwd 

2.使用参数传输
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::backup --password-file=/etc/rsync.passwd
sending incremental file list
rewriteip.sh

sent 207 bytes  received 43 bytes  166.67 bytes/sec
total size is 194  speedup is 0.78
```

#### 3）方法三：添加环境变量

```bash
[root@web01 ~]# export RSYNC_PASSWORD=123456			#临时添加
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::backup
sending incremental file list

sent 55 bytes  received 20 bytes  50.00 bytes/sec
total size is 194  speedup is 2.59
```



### 7.rsync常见报错

```bash
1.报错内容：
[root@web01 ~]# rsync -avz 1.txt rsync_backu@172.16.1.41::backup
Password: 
@ERROR: auth failed on module backup
rsync error: error starting client-server protocol (code 5) at main.c(1649) [sender=3.1.2]
#原因：1）虚拟用户的用户名或者密码错误，2）服务端密码文件权限不为600

2.报错内容：
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::backup --password-file=/etc/rsync.passwd
ERROR: password file must not be other-accessible
rsync error: syntax or usage error (code 1) at authenticate.c(196) [sender=3.1.2]
#原因：客户端密码文件权限不是600

3.报错内容：
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::backu
@ERROR: Unknown module 'backu'
rsync error: error starting client-server protocol (code 5) at main.c(1649) [sender=3.1.2]
#原因：模块名字错误

4.报错内容：
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::/backup
ERROR: The remote path must start with a module name not a /
rsync error: error starting client-server protocol (code 5) at main.c(1649) [sender=3.1.2]
#原因：双冒号后面跟的是模块名字，而不是目录名字不要加/

5.报错内容：
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::backup
rsync: failed to connect to 172.16.1.41 (172.16.1.41): No route to host (113)
rsync error: error in socket IO (code 10) at clientserver.c(125) [sender=3.1.2]
#原因：防火墙开启，没有配置防火墙规则
[root@backup ~]# firewall-cmd --add-port=873/tcp
success

6.报错内容：
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::backup
Password: 
sending incremental file list
rewriteip.sh
rsync: mkstemp ".rewriteip.sh.vx4Cry" (in backup) failed: Permission denied (13)

sent 207 bytes  received 128 bytes  44.67 bytes/sec
total size is 194  speedup is 0.58
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1179) [sender=3.1.2]
#原因：selinux没有关闭

7.报错内容：
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::backup
sending incremental file list
rsync: delete of stat xattr failed for "rewriteip.sh" (in backup): Permission denied (13)

sent 55 bytes  received 114 bytes  338.00 bytes/sec
total size is 194  speedup is 1.15
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1179) [sender=3.1.2]
#原因：服务端备份目录权限不是rsync

8.报错内容
[root@web01 ~]# rsync -avz 1.txt rsync_backup@172.16.1.41::backup
rsync: failed to connect to 172.16.1.41 (172.16.1.41): Connection refused (111)
rsync error: error in socket IO (code 10) at clientserver.c(125) [sender=3.1.2]
#原因：服务端服务没有启动

9.报错内容
[root@web01 ~]# rsync -avz 1.txt rsync_backup@10.0.0.41::backup 
sending incremental file list
rsync: read error: Connection reset by peer (104)
rsync error: error in socket IO (code 10) at io.c(785) [sender=3.1.2]
#原因：服务端配置错误，导致启动问题
```





## 五、Rsync备份案例

### 1、准备服务器

| 主机   | IP        | 身份   |
| ------ | --------- | ------ |
| web01  | 10.0.0.7  | 客户端 |
| backup | 10.0.0.41 | 服务端 |

### 2.了解需求

```bash
客户端需求:
1.客户端提前准备存放的备份的目录，目录规则如下:/backup/nfs_172.16.1.31_2018-09-02
2.客户端在本地打包备份(系统配置文件、应用配置等)拷贝至/backup/nfs_172.16.1.31_2018-09-02
3.客户端最后将备份的数据进行推送至备份服务器
4.客户端每天凌晨1点定时执行该脚本
5.客户端服务器本地保留最近7天的数据, 避免浪费磁盘空间

服务端需求:
1.服务端部署rsync，用于接收客户端推送过来的备份数据
2.服务端需要每天校验客户端推送过来的数据是否完整
3.服务端需要每天校验的结果通知给管理员
4.服务端仅保留6个月的备份数据,其余的全部删除
```



### 3、客户端需求

#### 1、创建备份目录

```bash
#尝试获取信息
[root@web01 ~]# mkdir /backup
[root@web01 ~]# hostname
web01
[root@web01 ~]# hostname -I
10.0.0.7 172.16.1.7 
[root@web01 ~]# hostname -I | awk '{print $2}'
172.16.1.7
[root@web01 ~]# date +%F
2020-11-19

#结合信息创建目录
[root@web01 ~]# mkdir /backup/$(hostname)_$(hostname -I | awk '{print $2}')_$(date +%F)
[root@web01 ~]# mkdir /backup/`hostname`_`hostname -I | awk '{print $2}'`_`date +%F`

[root@web01 ~]# ll /backup/
drwxr-xr-x. 2 root root 6 Nov 19 10:00 web01_172.16.1.7_2020-11-19
```

#### 2、打包数据

```bash
#打包
[root@web01 ~]# tar zcf conf.tar.gz /var/log/messages
#移动
[root@web01 ~]# mv conf.tar.gz /backup/$(hostname)_$(hostname -I | awk '{print $2}')_$(date +%F)/

#或者使用下面方式

#直接打包到目录
[root@web01 ~]# tar zcPf /backup/$(hostname)_$(hostname -I | awk '{print $2}')_$(date +%F)/conf.tar.gz /var/log/messages
```

#### 3、推送文件

```bash
[root@web01 ~]# rsync -avz /backup/ rsync_backup@172.16.1.41::backup
sending incremental file list
./
web01_172.16.1.7_2020-11-19/
web01_172.16.1.7_2020-11-19/conf.tar.gz

sent 124,299 bytes  received 54 bytes  248,706.00 bytes/sec
total size is 130,519  speedup is 1.05
```

#### 4、将以上步骤写成脚本

```bash
[root@web01 ~]# vim rsync_client.sh 
#!/bin/bash
#1、定义变量
DIR=/backup
HOSTNAME=$(hostname)
IP=$(hostname -I | awk '{print $2}')
DATE=$(date +%F)
SRC=${DIR}/${HOSTNAME}_${IP}_${DATE}

#2、创建备份目录
mkdir -p $SRC

#3、打包文件
tar zcPf $SRC/conf.tar.gz /var/log/messages

#4、推送文件
export RSYNC_PASSWORD=123456
rsync -az $DIR/ rsync_backup@172.16.1.41::backup

```

#### 5、将脚本加入定时任务

```bash
[root@web01 ~]# crontab -e
#每天凌晨1点执行备份脚本
0 1 * * * /bin/bash /root/rsync_client.sh

```



#### 6、只保留七天数据

```bash
#模拟到今天的数据
[root@web01 ~]# for i in {1..19};do date -s 2020/11/$i;sh rsync_client.sh;done

#删除七天前的数据
[root@web01 ~]# find /backup/ -type d -mtime +7 | xargs rm -rf
```

#### 7、加入脚本

```bash
[root@web01 ~]# vim rsync_client.sh 
#!/bin/bash
#1、定义变量
DIR=/backup
HOSTNAME=$(hostname)
IP=$(hostname -I | awk '{print $2}')
DATE=$(date +%F)
SRC=${DIR}/${HOSTNAME}_${IP}_${DATE}

#2、创建备份目录
mkdir -p $SRC

#3、打包文件
tar zcPf $SRC/conf.tar.gz /var/log/messages

#4、推送文件
export RSYNC_PASSWORD=123456
rsync -az $DIR/ rsync_backup@172.16.1.41::backup

#5、删除七天之前的数据
find $DIR/ -type d -mtime +7 | xargs rm -rf
```

#### 8、客户端先判断文件是否存在

```bash
[root@web01 ~]# vim rsync_client.sh 
 #!/bin/bash
 #1、定义变量
 DIR=/backup
 HOSTNAME=$(hostname)
 IP=$(hostname -I | awk '{print $2}')
 DATE=$(date +%F)
 SRC=${DIR}/${HOSTNAME}_${IP}_${DATE}
 
 #2、创建备份目录
 [ -d $SRC ] || mkdir -p $SRC
  
 #3、打包文件
 [ -d $SRC/conf.tar.gz ] || tar zcPf $SRC/conf.tar.gz /var/log/messages
  
 #4、推送文件
 export RSYNC_PASSWORD=123456
 rsync -az $DIR/ rsync_backup@172.16.1.41::backup
 
 #5、删除七天之前的数据
 find $DIR/ -type d -mtime +7 | xargs rm -rf

```



### 4、服务端需求

#### 1、部署rsync服务端



#### 2、客户端加入校验码操作

```bash
#!/bin/bash
#1、定义变量
DIR=/backup
HOSTNAME=$(hostname)
IP=$(hostname -I | awk '{print $2}')
DATE=$(date +%F)
SRC=${DIR}/${HOSTNAME}_${IP}_${DATE}

#2、创建备份目录
 [ -d $SRC ] || mkdir -p $SRC

#3、打包文件
[ -d $SRC/conf.tar.gz ] || tar zcPf $SRC/conf.tar.gz /var/log/messages

#4.生成校验码
md5sum $SRC/conf.tar.gz > $SRC/flag_$DATE

#5、推送文件
export RSYNC_PASSWORD=123456
rsync -az $DIR/ rsync_backup@172.16.1.41::backup

#6、删除七天之前的数据
find $DIR/ -type d -mtime +7 | xargs rm -rf

```

#### 3、校验文件

```bash
[root@backup backup]# md5sum -c /backup/*_$(date +%F)/flag_2020-11-19
```

#### 4、使用邮件发送消息

```bash
#1.服务端配置邮件功能
[root@backup ~]# yum install mailx -y
[root@backup ~]# vim /etc/mail.rc 
set from=1426115933@qq.com
set smtp=smtps://smtp.qq.com:465
set smtp-auth-user=1426115933@qq.com
set smtp-auth-password=pdaoywghqhxuggcg
set smtp-auth=login
set ssl-verify=ignore
set nss-config-dir=/etc/pki/nssdb/

#2.测试发送邮件
[root@backup backup]# mail -s "校验结果" 1426115933@qq.com < 1.txt
```

#### 5、服务端脚本

```bash
[root@backup ~]# vim rsync_server.sh
#!/bin/bash
#1.定义变量
DIR=/backup
HOSTNAME=$(hostname)
IP=$(hostname -I | awk '{print $2}')
DATE=$(date +%F)
SRC=${DIR}/${HOSTNAME}_${IP}_${DATE}

#2.校验文件
md5sum -c $DIR/*_$DATE/flag_$DATE > $DIR/result.txt

#3.将校验结果发送给管理员邮箱
mail -s "$DATE备份文件 校验结果" 1426115933@qq.com < $DIR/result.txt &> /dev/null

#4.删除6个月之前的数据
find $DIR/ -type d -mtime +180 | xargs rm -rf

```

#### 6、将服务脚本加入定时任务

```bash
[root@backup ~]# crontab -e
#服务端每天7点将校验备份结果发给管理员
0 7 * * * /bin/bash /root/rsync_server.sh
```



## 六、Rsync结合inotify

### 1、安装inotify

```bash
[root@web01 ~]# yum -y install inotify-tools
```

### 2、常用参数

```bash
-m 持续监控
-r 递归
-q 静默，仅打印时间信息
--timefmt 指定输出时间格式
--format 指定事件输出格式
	%Xe 事件
	%w 目录
	%f 文件
-e 指定监控的事件
	access 访问
	modify 内容修改
	attrib 属性修改
	close_write 修改真实文件内容
	open 打开
	create 创建
	delete 删除
	umount 卸载
```

### 3、测试命令

```bash
/usr/bin/inotifywait  -mrq  --format '%Xe  %w  %f' -e create,modify,delete,attrib,close_write  /backup
```

### 4.实时备份脚本编写

#### 1）粗略版

```bash
[root@backup ~]# vim rsyn-inotify.sh
#!/bin/bash
dir=/backup
/usr/bin/inotifywait  -mrq  --format '%w %f' -e create,delete,attrib,close_write  $dir | while read line;do
        cd  $dir  && rsync -az -R  --delete  .  rsync_backup@172.16.1.31::backup --password-file=/etc/rsync.passwd >/dev/null 2>&1
done  &
```

#### 2）精细版

```bash
#!/bin/bash
src=/data
des=backup
rsync_passwd_file=/etc/rsync.passwd
ip1=172.16.1.41
user=rsync_backup
cd ${src}
/usr/bin/inotifywait -mrq --format  '%Xe %w%f' -e modify,create,delete,attrib,close_write,move ./ | while read file
do
CREATE  /backup/  1.txt
	INO_EVENT=$(echo $file | awk '{print $1}')
	INO_FILE=$(echo $file | awk '{print $2}')        
	if [[ $INO_EVENT =~ 'CREATE' ]] || [[ $INO_EVENT =~ 'MODIFY' ]] || [[ $INO_EVENT =~ 'CLOSE_WRITE' ]] || [[ $INO_EVENT =~ 'MOVED_TO' ]]
	then                
		rsync -azcR --password-file=${rsync_passwd_file} ${INO_FILE} ${user}@${ip1}::${des}
	fi
	if [[ $INO_EVENT =~ 'DELETE' ]] || [[ $INO_EVENT =~ 'MOVED_FROM' ]]        
	then
		rsync -azR --delete --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip1}::${des} >/dev/null 2>&1        
	fi
	if [[ $INO_EVENT =~ 'ATTRIB' ]]
	then
	if [ ! -d "$INO_FILE" ]
	then
		rsync -azcR --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip1}::${des} >/dev/null 2>&1
	fi
	fi
done &
```
