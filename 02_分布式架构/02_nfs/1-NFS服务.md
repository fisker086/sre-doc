# NFS 服务

## 一、NFS介绍

### 1.使用NFS解决了什么

```bash
1.为了实现文件共享
2.为了多台服务器之间数据一致
```

### 2.NFS 原理

![nfs原理](nfs%E5%8E%9F%E7%90%86.png)



## 二、NFS实践

### 1.环境准备

| 主机  | IP          | 角色      |
| ----- | ----------- | --------- |
| web01 | 172.16.1.7  | NFS客户端 |
| nfs   | 172.16.1.31 | NFS服务端 |



### 2.服务端（172.16.1.31）

#### 1）关闭防火墙和selinux

#### 2）安装NFS和rpcbind

```bash
[root@nfs ~]# yum install -y nfs-utils rpcbind

#注意：
Centos6 需要安装rpcbind
Centos7 默认已经安装好了rpcbind，并且默认是开机自启动
```

#### 3）配置NFS

```bash
#NFS默认的配置文件是
[root@nfs ~]# ll /etc/exports
-rw-r--r--. 1 root root 0 Jun  7  2013 /etc/exports

#配置NFS
[root@nfs ~]# vim /etc/exports
/data 172.16.1.0/24(rw,sync,all_squash)
```

| 语法 | /data               | 172.16.1.0/24         | (rw,sync,all_squash) |
| ---- | ------------------- | --------------------- | -------------------- |
| 含义 | NFS服务端共享的目录 | NFS允许连接的客户端IP | 允许操作的权限       |

#### 4）创建共享目录

```bash
[root@nfs ~]# mkdir /data
```

#### 5）启动服务

```bash
#Centos7启动
[root@nfs ~]# systemctl start rpcbind nfs

#Centos6启动，一定要先启动rpcbind在启动nfs
[root@nfs ~]# /etc/init.d/rpcbind start
[root@nfs ~]# /etc/init.d/nfs start

[root@nfs ~]# service rpcbind start
[root@nfs ~]# service nfs start

#验证启动
[root@nfs ~]# netstat -lntp | grep rpc
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      5824/rpcbind
tcp        0      0 0.0.0.0:20048           0.0.0.0:*               LISTEN      10187/rpc.mountd
tcp        0      0 0.0.0.0:49150           0.0.0.0:*               LISTEN      10149/rpc.statd 
tcp6       0      0 :::111                  :::*                    LISTEN      5824/rpcbind
tcp6       0      0 :::20048                :::*                    LISTEN      10187/rpc.mountd
tcp6       0      0 :::46384                :::*                    LISTEN      10149/rpc.statd
```

#### 6）验证NFS配置

```bash
[root@nfs ~]# cat /var/lib/nfs/etab
/data	172.16.1.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=65534,anongid=65534,sec=sys,rw,secure,root_squash,all_squash)
```



### 3.客户端操作（172.16.1.7）

#### 1）关闭防火墙和selinux

#### 2）安装服务

```bash
[root@web01 ~]# yum install -y rpcbind nfs-utils
```

#### 3）查看挂载点

```bash
[root@web01 ~]# showmount -e 172.16.1.31
Export list for 172.16.1.31:
/data 172.16.1.0/24
```

#### 4）挂载

```bash
[root@web01 ~]# mount -t nfs 172.16.1.31:/data /backup/

#验证挂载
[root@web01 ~]# df -h
Filesystem         Size  Used Avail Use% Mounted on
/dev/sda3           18G  1.6G   17G   9% /
devtmpfs           476M     0  476M   0% /dev
tmpfs              487M     0  487M   0% /dev/shm
tmpfs              487M   14M  473M   3% /run
tmpfs              487M     0  487M   0% /sys/fs/cgroup
/dev/sda1         1014M  127M  888M  13% /boot
tmpfs               98M     0   98M   0% /run/user/0
172.16.1.31:/data   18G  1.6G   17G   9% /backup
```

#### 5）写入数据进行测试

```bash
#第一次写入测试
[root@web01 ~]# cd /backup/
[root@web01 backup]# touch 123.txt
touch: cannot touch ‘123.txt’: Permission denied    #没有权限

#授权目录
[root@nfs ~]# chown -R nfsnobody.nfsnobody /data/

#再次创建测试
[root@web01 backup]# touch 123.txt
[root@web01 backup]# ll
total 0
-rw-r--r--. 1 nfsnobody nfsnobody 0 Nov 20 09:26 123.txt

#服务端查看
[root@nfs ~]# ll /data/
total 0
-rw-r--r-- 1 nfsnobody nfsnobody 0 Nov 20 09:26 123.txt
```



## 三、NFS挂载与卸载

```bash
NFS客户端的配置步骤也十分简单。先使用showmount命令,查询NFS服务器的远程共享信息，其输出格式为“共享的目录名称 允许使用客户端地址(权限)”。

NFS挂载：客户端的目录仅仅是服务端共享目录的一个入口，可以简单理解为软连接，真正的数据全都是存储在服务端的目录，客户端写入的数据也是在服务端存储的
```

### 1.注意事项

```bash
1.挂载目录后，原来文件下的内容不会丢失，仅仅是被遮盖住，取消挂载后仍然存在
2.取消挂载时不要在挂载的目录下面操作，否则会提示忙碌，切换到其他目录再进行卸载
3.挂载是如果在挂载的目录下，还是可以看到挂载前目录下的文件，需要重新进入目录才会显示挂载后目录的内容
```

### 2.挂载

#### 1）客户端安装

```bash
1.rpcbind:
	为了连接服务端的进程
	
2.nfs-utils
	为了使用showmount命令
```

#### 2）客户端查看挂载点

```bash
[root@web01 ~]# showmount -e 172.16.1.31
Export list for 172.16.1.31:
/data 172.16.1.0/24
```

#### 3）挂载命令

```bash
[root@web01 ~]# mount -t nfs 172.16.1.31:/data /backup
mount 			#挂载命令
-t 				#指定挂载的文件类型
nfs 			#nfs文件类型
172.16.1.31		 #服务端的IP地址
:/data 			#服务端提供的可挂载目录
/backup			#本地要挂载到服务端的目录

#挂在后查看挂载
[root@web01 ~]# df -h | grep /backup
172.16.1.31:/data   18G  1.6G   17G   9% /backup
```

### 3.卸载

```bash
#卸载的两种方式
[root@web01 ~]# umount /backup 
[root@web01 ~]# umount 172.16.1.31:/data

#强制取消挂载
[root@web01 ~]# umount -lf /backup
```

### 4.开机挂载

```bash
#编辑fstab文件
[root@web01 ~]# vim /etc/fstab
172.16.1.31:/data /backup nfs defaults 0 0

#验证fstab是否写正确
[root@web01 ~]# mount -a
```



## 四、NFS配置详解

```bash
[root@nfs ~]# cat /etc/exports
/data 172.16.1.0/24(rw,sync,all_squash)
```

| nfs共享参数    | 参数作用                                                     |
| -------------- | ------------------------------------------------------------ |
| rw             | 读写权限  (常用)                                             |
| ro             | 只读权限  (不常用)                                           |
| root_squash    | 当NFS客户端以root管理员访问时，映射为NFS服务器的匿名用户  (不常用) |
| no_root_squash | 当NFS客户端以root管理员访问时，映射为NFS服务器的root管理员  (不常用) |
| all_squash     | 无论NFS客户端使用什么账户访问，均映射为NFS服务器的匿名用户  (常用) |
| no_all_squash  | 无论NFS客户端使用什么账户访问，都不进行压缩  (不常用)        |
| sync           | 同时将数据写入到内存与硬盘中，保证不丢失数据  (常用)         |
| async          | 优先将数据保存到内存，然后再写入硬盘；这样效率更高，但可能会丢失数据  (不常用) |
| anonuid        | 配置all_squash使用,指定NFS的用户UID,必须存在系统  (常用)     |
| anongid        | 配置all_squash使用,指定NFS的用户UID,必须存在系统  (常用)     |



## 五、NFS案例

### 1.环境准备

| 主机  | IP          | 身份      |
| ----- | ----------- | --------- |
| web01 | 10.0.0.7    | NFS客户端 |
| web02 | 10.0.0.8    | NFS客户端 |
| nfs   | 172.16.1.31 | NFS服务端 |

### 2.web端安装http和php

```bash
[root@web01 ~]# yum install -y httpd php
[root@web02 ~]# yum install -y httpd php
```

### 3.上传代码

```bash
[root@web01 ~]# rz
[root@web01 ~]# ll
-rw-r--r--  1 root root    26995 Aug 23 10:35 kaoshi.zip
```

### 4.解压代码

```bash
#找到httpd服务的站点目录
[root@web01 ~]# rpm -ql httpd | grep html
/var/www/html

#解压代码至站点目录
[root@web01 ~]# unzip kaoshi.zip -d /var/www/html/
```

### 5.启动httpd

```bash
[root@web01 ~]# systemctl start httpd

#查看启动
[root@web01 ~]# netstat -lntp | grep 80
tcp6       0      0 :::80                   :::*                    LISTEN      7530/httpd         

[root@web01 ~]# ps -ef | grep httpd
root       7530      1  0 10:39 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache     7531   7530  0 10:39 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache     7532   7530  0 10:39 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache     7533   7530  0 10:39 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache     7534   7530  0 10:39 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache     7535   7530  0 10:39 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
root       7543   7262  0 10:40 pts/0    00:00:00 grep --color=auto httpd
```



### 6.访问页面测试

#### 1）访问

```bash
http://10.0.0.7/
http://10.0.0.8/
```

![web页面访问](web%E9%A1%B5%E9%9D%A2%E8%AE%BF%E9%97%AE.png)

#### 2）测试上传作业

![上传测试](%E4%B8%8A%E4%BC%A0%E6%B5%8B%E8%AF%95.png)

#### 3）上传文件失败

```bash
#显示上传文件成功，但是服务器上没有文件
[root@web01 html]# ll /var/www/html/upload
ls: cannot access /var/www/html/upload: No such file or directory
#为什么找这个文件，因为代码里写的上传文件地址就是这个目录

#原因：代码不严谨，没有判断目录权限，目录权限不足，导致上传失败
#授权
[root@web01 html]# chown -R apache.apache /var/www/html/

#重新上传成功
```

#### 4）测试（没有挂载）

```bash
在10.0.0.7服务器上传 1_test_nfs.gif
在10.0.0.8服务器上传 2_nfs.jpg

#访问
http://10.0.0.7/upload/1_test_nfs.gif	 访问成功
http://10.0.0.8/upload/1_test_nfs.gif	 访问失败
http://10.0.0.8/upload/2_nfs.jpg		访问成功
http://10.0.0.7/upload/2_nfs.jpg		访问失败

#在没有挂载的情况下，文件无法实现共享，在哪台机器上传就只能在哪台机器访问
```

### 7.挂载

#### 1）web端挂载目录

```bash
1.先同步多台web的文件
[root@web01 html]# rsync -avz upload/ 172.16.1.31:/data
[root@web02 html]# rsync -avz upload/ 172.16.1.31:/data

2.找到需要挂载的目录
/var/www/html/upload

3.挂载
[root@web01 html]# mount -t nfs 172.16.1.31:/data /var/www/html/upload
[root@web02 html]# mount -t nfs 172.16.1.31:/data /var/www/html/upload
```

#### 2）再次访问测试（挂载后）

```bash
#访问
http://10.0.0.7/upload/1_test_nfs.gif	 访问成功
http://10.0.0.8/upload/1_test_nfs.gif	 访问成功
http://10.0.0.8/upload/2_nfs.jpg		访问成功
http://10.0.0.7/upload/2_nfs.jpg		访问成功
```



## 六、统一用户

### 1.服务器创建统一用户

```bash
[root@web01 ~]# groupadd www -g 666
[root@web01 ~]# useradd www -u 666 -g 666

[root@web02 ~]# groupadd www -g 666
[root@web02 ~]# useradd www -u 666 -g 666

[root@nfs ~]# groupadd www -g 666
[root@nfs ~]# useradd www -u 666 -g 666

[root@backup ~]# groupadd www -g 666
[root@backup ~]# useradd www -u 666 -g 666
```

### 2.需要修改用户的服务

```bash
httpd
nfs
rsync
```

### 3.修改httpd的用户

```bash
#找到配置文件
[root@web01 ~]# rpm -qc httpd
/etc/httpd/conf/httpd.conf

#修改配置文件
[root@web01 ~]# vim /etc/httpd/conf/httpd.conf
User www
Group www

#重启服务
[root@web01 ~]# systemctl restart httpd

#确认启动用户
[root@web01 ~]# ps -ef | grep httpd
root       7768      1  1 11:49 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
www        7769   7768  0 11:49 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
www        7770   7768  0 11:49 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
www        7771   7768  0 11:49 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
www        7772   7768  0 11:49 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
www        7773   7768  0 11:49 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
```

### 4.修改nfs服务的用户

```bash
[root@nfs ~]# vim /etc/exports
/data 172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)

#授权/data目录
[root@nfs ~]# chown -R www.www /data/

#重启服务
[root@nfs ~]# systemctl restart nfs

#验证启动用户
[root@nfs ~]# cat /var/lib/nfs/etab 
/data	172.16.1.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=666,anongid=666,sec=sys,rw,secure,root_squash,all_squash)
```

### 5.修改rsync用户

```bash
#修改配置
[root@backup ~]# vim /etc/rsyncd.conf
uid = www
gid = www

#重启服务
[root@backup ~]# systemctl restart rsyncd

#目录重新授权
[root@backup ~]# chown -R www.www /backup/
```

### 6.测试架构

```bash
1.两台web服务器
2.一台nfs服务器挂载web服务器的文件目录
3.一台backup服务器实时同步nfs挂载目录下的内容
```



## 七、NFS小结

### 1.NFS存储优点

```bash
1.NFS文件系统简单易用、方便部署、数据可靠、服务稳定、满足中小企业需求
2.NFS文件系统内存放的数据都在文件系统之上，所有数据都是能看得见
```

### 2.NFS存储局限

```bash
1.存在单点故障, 如果构建高可用维护麻烦  web -> nfs -> backup
2.NFS数据明文, 并不对数据做任何校验
3.客户端挂载NFS服务没有密码验证, 安全性一般(内网使用)
```

### 3.NFS应用建议

```bash
1.生产场景应将静态数据尽可能往前端推, 减少后端存储压力
2.必须将存储里的静态资源通过CDN缓存 jpg\png\mp4\avi\css\js
3.如果没有缓存或架构本身历史遗留问题太大, 在多存储也无用
```
