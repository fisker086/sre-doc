# SSH 远程管理服务

## 一、ssh简介

```bash
SSH是一个安全协议，在进行数据传输时，会对数据包进行加密处理，加密后在进行数据传输。确保了数据传输安全。那SSH服务主要功能有哪些呢？
1.提供远程连接服务器的服务
	1）linux远程连接协议： ssh服务  端口22
	2）windows远程连接：  RDP协议  端口3389
2.对传输的数据进行加密
```

```bash
#笔试题：请说明以下服务对应的端口号或者端口对应的服务
ssh				22
telnet			23
http			80
https			443
ftp				20 21
RDP				3389
mysql			3306
redis			6379
zabbix			10050 10051
elasticsearch	 9200 9300
rsync			873
rpcbind			111
```



## 二、ssh和telnet

### 1.使用telnet连接服务器

```bash
1.安装
[root@web01 ~]# yum install -y telnet-server

2.启动
[root@web01 ~]# systemctl start telnet.socket

3.创建普通用户并设置密码
[root@web01 ~]# useradd lhd
[root@web01 ~]# passwd lhd

4.测试连接
[C:\~]$ telnet 10.0.0.7 23
Connecting to 10.0.0.7:23...
Connection established.
To escape to local shell, press 'Ctrl+Alt+]'.

Kernel 3.10.0-957.el7.x86_64 on an x86_64
web01 login: lhd
Password: 123456
[lhd@web01 ~]$
```

### 2.ssh和telnet的区别

```bash
telnet：
	1.不支持root用户登录，只允许普通用户登录
	2.数据传输过程中明文的

ssh：
	1.支持root用户登录
	2.数据传输过程中时加密码
```



## 三、SSH相关命令

```bash
SSH有客户端与服务端，我们将这种模式称为C/S架构，ssh客户端支持Windows、Linux、Mac等平台。
在ssh客户端中包含 ssh|slogin远程登陆、scp远程拷贝、sftp文件传输、ssh-copy-id秘钥分发等应用程序
```

### 1.ssh命令

```bash
[root@web01 ~]# ssh root@172.16.1.31

#命令拆分
ssh				#命令
root			#连接远端服务器时使用的用户，远端服务器上真实存在的用户
				#如果连接时不指定用户，则使用当前服务器的当前用户连接的远端服务器上的相同用户
@				#分割符
172.16.1.31		 #远端服务器的IP地址
-p				#指定ssh服务端的端口
22				#ssh的端口
-o StrictHostKeyChecking=no		#首次连接不询问
```

### 2.xshell连接不上服务器怎么办？

```bash
1.查看网络
    ping 10.0.0.31
2.查网卡，网卡是否启动
3.查端口
    telnet 10.0.0.31 22
4.检查sshd服务是否启动
5.防火墙
6.虚拟机的虚拟网络编辑器
7.查看windows的网卡
```

### 3.scp命令

```bash
类似于rsync命令，远程拷贝，scp时全量，rsync时增量
scp 也支持推和拉
```

#### 1）scp推

```bash
#推送文件到远程目录
[root@web01 ~]# scp rewriteip.sh root@172.16.1.31:/tmp/
root@172.16.1.31's password: 1
rewriteip.sh                 100%  194   116.3KB/s   00:00

#推送目录到远端服务器
[root@web01 ~]# scp -r /etc root@172.16.1.31:/tmp/

#注意：
	1.与rsync不同，推送时不论加 / 或者不加，推送的都是目录
	2.如果想推送目录下的文件加 *
	[root@web01 ~]# scp -r /etc/* root@172.16.1.31:/tmp/

#命令拆分：
scp 				#命令
rewriteip.sh 		#要推送的文件或目录
root				#远程主机上的系统用户
@					#分隔符
172.16.1.31			 #远端主机的IP地址
:/tmp/				#远端主机保存文件的位置
```

#### 2）scp拉

```bash
[root@web01 ~]# scp root@172.16.1.31:/tmp/rewriteip.sh /tmp/

#注意：
	1.与rsync不同，推送时不论加 / 或者不加，拉取的都是目录
	2.如果想拉取目录下的文件加 *
```

#### 3）常用参数

```bash
-P 指定端口，默认22端口可不写
-r 表示递归拷贝目录
-p 表示在拷贝文件前后保持文件或目录属性不变
-l 限制传输使用带宽(默认kb)

#推：将本地/tmp/xiaowu推送至远端服务器10.0.0.61的/tmp目录，使用对端的root用户
[root@m01 ~]# scp -P22 -rp /tmp/xiaowu xiaowu@10.0.0.61:/tmp

#拉：将远程10.0.0.61服务器/tmp/xiaowu文件拉取到本地/opt/目录下
[root@m01 ~]# scp -P22 -rp root@10.0.0.61:/tmp/xiaowu /opt/

#限速
[root@m01 ~]# scp /opt/1.txt root@172.16.1.31:/tmp
root@172.16.1.31 password: 
test                        100%  656MB  '83.9MB/s'   00:07 

#限速为8096kb，换算为MB，要除以 8096/8=1024KB=1MB
[root@m01 ~]# scp -rp -l 8096 /opt/1.txt root@172.16.1.31:/tmp
root@172.16.1.31s password: 
test                        7%   48MB   '1.0MB/s'   09:45
```

#### 4）总结

```bash
1.scp通过ssh协议加密方式进行文件或目录拷贝。
2.scp连接时的用户作为为拷贝文件或目录的权限。
3.scp支持数据推送和拉取，每次都是全量拷贝，效率较低。
```



### 4.sftp命令

#### 1）终端连接服务器

```bash
1.终端连接
[C:\~]$ sftp 10.0.0.31

2.下载文件
sftp:/data> get 2_nfs.jpg
Fetching /data/2_nfs.jpg to 2_nfs.jpg
sftp: received 29.7 KB in 0.03 seconds

3.上传文件
sftp:/data> put
#选择文件
```

#### 2）服务器之间连接

```bash
1.连接
    [root@web01 ~]# sftp 172.16.1.31
    root@172.16.1.31's password: 
    Connected to 172.16.1.31.
    sftp>

2.操作远程主机
    sftp> pwd
    Remote working directory: /root
    sftp> cd /data
    sftp> pwd
    Remote working directory: /data
    sftp> ls
    2_nfs.jpg  
    sftp> ls -l
    -rw-r--r--    1 www      www         30419 Nov 23 18:17 2_nfs.jpg

3.操作本机（在命令前面加一个 l ，表示localhost）
    sftp> lls -l
    total 8
    -rw-------. 1 root root 1588 Nov 17 12:11 anaconda-ks.cfg
    drwxr-xr-x. 2 root root    6 Nov 18 09:02 dir1
    drwxr-xr-x. 2 root root    6 Nov 18 09:02 dir2
    -rw-r--r--. 1 root root  194 Nov 17 12:29 rewriteip.sh

4.拉取命令
    sftp> get 2_nfs.jpg 
    Fetching /data/2_nfs.jpg to 2_nfs.jpg
    /data/2_nfs.jpg                                                                                      100%   30KB   3.1MB/s   00:00    
    sftp> lls -l
    total 40
    -rw-r--r--. 1 root root 30419 Nov 24 10:22 2_nfs.jpg

    #指定目录拉取
    sftp> get 2_nfs.jpg /opt
    Fetching /data/2_nfs.jpg to /opt/2_nfs.jpg
    /data/2_nfs.jpg                                                                                      100%   30KB  10.9MB/s   00:00    
    sftp> lls -l /opt
    total 32
    -rw-r--r--. 1 root root 30419 Nov 24 10:22 2_nfs.jpg

5.推送命令
    sftp> put 2_nfs.jpg
    Uploading 2_nfs.jpg to /data/2_nfs.jpg
    2_nfs.jpg
```



#### 3）文件传输工具

```bash
1.xftp
2.filezilla
3.flashfxp
```

#### 4）命令

```bash
1.sz/rz
	1）不能上传大于4G的文件
	2）不能断点续传
	3）不能上传文件夹

2.sftp
	1）能上传大于4G的文件
	2）支持断点续传
	3）可以上传文件夹
```



## 四、SSH验证方式

### 1.基于账户密码远程登录

```bash
#知道服务器的IP地址，端口，系统用户，密码，即可通过ssh客户端命令登陆远程主机。
[root@web01 ~]# ssh root@172.16.1.31 -p 22
root@172.16.1.31's password: 1
Last login: Tue Nov 24 09:57:59 2020 from 10.0.0.1

#设置密码
1.如果太难，会记不住
2.如果太简单，容易破解
3.每台服务器密码不一样
4.密码是动态的
5.密码三个月一修改
6.密码错误三次锁定用户
7.密码是没有规律的
```

### 2.基于秘钥远程登录

```bash
默认情况下，通过ssh客户端命令登陆远程服务器，需要提供远程系统上的帐号与密码，但为了降低密码泄露的机率和提高登陆的方便性，建议使用密钥验证方式。
```

#### 1）原理

![ssh服务原理](./ssh%E6%9C%8D%E5%8A%A1%E5%8E%9F%E7%90%86.jpg)

#### 2）生成密钥对

```bash
[root@m01 ~]# ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:OKxLhAZ0qD/LXHzGUByirfRI5k1YRCCMT8lK8sLIk10 root@m01
The key's randomart image is:
+---[RSA 2048]----+
|+oo*=...         |
|+=== Eo          |
|O=O +.           |
|=@oO.. .         |
| oB.+o+ S        |
| .o.o.+.         |
| o +oo           |
|  +. .           |
|    .            |
+----[SHA256]-----+

```

#### 3）将公钥发送至要免密登录的服务器

##### 1> 手动复制公钥

```bash
1.查看公钥
	[root@m01 ~]# cat .ssh/id_rsa.pub

2.在其他服务器创建文件，将内容粘贴进去
	[root@nfs ~]# mkdir .ssh
	[root@nfs ~]# vim .ssh/authorized_keys
	
3.授权文件
    [root@nfs ~]# chmod 700 .ssh/
    [root@nfs ~]# chmod 600 .ssh/authorized_keys
    
4.测试连接
	#首次连接需要记录服务器信息到 .ssh/known_hosts
	[root@m01 ~]# ssh 172.16.1.31
    The authenticity of host '172.16.1.31 (172.16.1.31)' can't be established.
    ECDSA key fingerprint is SHA256:sYhpMuszVGaHSeWKyLXMGQQ72f/6KxyExWabnY/cz6w.
    ECDSA key fingerprint is MD5:bc:9c:0b:45:b5:27:71:cd:da:02:68:c0:48:71:9d:69.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '172.16.1.31' (ECDSA) to the list of known hosts.
    Last login: Tue Nov 24 10:37:03 2020 from 172.16.1.7
    [root@nfs ~]# 
    
    #再一次连接
    [root@m01 ~]# ssh 172.16.1.31
    Last login: Tue Nov 24 11:00:39 2020 from 172.16.1.61
    [root@nfs ~]#
```

##### 2> 使用命令推送公钥

```bash
#推送公钥到 172.16.1.7
[root@m01 ~]# ssh-copy-id -i .ssh/id_rsa.pub root@172.16.1.7
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: ".ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@172.16.1.7's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@172.16.1.7'"
and check to make sure that only the key(s) you wanted were added.

[root@m01 ~]# 

#连接测试
[root@m01 ~]# ssh 172.16.1.7
Last login: Tue Nov 24 09:02:26 2020 from 10.0.0.1
[root@web01 ~]#
```



## 五、SSH免密场景

```bash
实践场景，用户通过Windows/MAC/Linux客户端连接跳板机免密码登录，跳板机连接后端无外网的Linux主机实现免密登录
实践多用户登陆一台跳板机免密码
实践跳板机登陆多台服务器免密码
```

### 1.powershell免密连接跳板机

```bash
1.打开powershell
2.执行ssh-keygen
3.windows下找到公钥 C:\Users\Administrator.DESKTOP-7PQVV6E\.ssh\id_rsa.pub
4.将公钥内容复制到 m01 跳板机上
	[root@m01 ~]# vim .ssh/authorized_keys
	[root@m01 ~]# chmod 600 .ssh/authorized_keys
```

### 2.xshell免密登录跳板机

```bash
1.打开xshell，工具栏中的工具，点击新建用户密钥
2.选择加密文件类型和位数，下一步
3.生成密钥对成功，下一步
4.给密钥配置信息，名字，密码...，完成

5.密钥属性或者工具栏用户密钥管理者
6.选择密钥对，点击属性，查看公钥
7.将公钥内容复制到 m01 跳板机上
    [root@m01 ~]# vim .ssh/authorized_keys
    [root@m01 ~]# chmod 600 .ssh/authorized_keys
8.连接时使用密钥连接
```

### 3.巡检脚本

```bash
#通过跳板机获取所有机器的load，CPU，Memory等信息(思考:如果服务器数量多，如何并发查看和分发数据)
[root@m01 ~]# cat all.sh 
#!/usr/bin/bash

for i in 31 41
do
    echo "#########172.16.1.$i#####"
    ssh root@172.16.1.$i "$1"
done
```

### 4.跳板机脚本

```bash
[root@m01 ~]# cat jump.sh 
#!/bin/bash
#jumpserver
lb01=10.0.0.5
lb02=10.0.0.6
web01=10.0.0.7
web02=10.0.0.8
web03=10.0.0.9
nfs=10.0.0.31
backup=10.0.0.41
db01=10.0.0.51
m01=10.0.0.61
zabbix=10.0.0.71

menu(){
        cat <<-EOF
        +-------------------------+
        |     1) lb01             |
        |     2) lb02             |
        |     3) web01            |
        |     4) web02            |
        |     5) web03            |
        |     6) nfs              |
        |     7) backup           |
        |     8) db01             |
        |     9) m01              |
        |     10) zabbix          |
        |     h) help             |
        +-------------------------+
EOF
}
#菜单函数
menu

#连接函数
connect(){
  ping -c 1 -w 1 $1 &>/dev/null
  if [ $? -eq 0 ];then
    ssh root@$1
  else
    echo -e "\033[5;4;40;31m 别连了,我的哥,$2:$1机器都没开!!!\033[0m"
  fi
}

#控制不让输入ctrl+c,z
trap "" HUP INT TSTP
while true
do
    read -p "请输入要连接的主机编号：" num
    case $num in
            1|lb01)
              connect $lb01 lb01
                    ;;
            2|lb02)
              connect $lb02 lb02
                    ;;
            3|web01)
              connect $web01 web01
                    ;;
            4|web02)
              connect $web02 web02
                    ;;
            5|web03)
                  connect $web03 web03
                    ;;
            6|nfs)
              connect $nfs nfs
                    ;;
            7|backup)
                  connect $backup backup
                    ;;
            8|db01)
                   connect $db01 db01
                    ;;
            9|m01)
                    connect $m01 m01
                    ;;
            10|zabbix)
                    connect $zabbix zabbix
                    ;;
            h|help)
                    clear
                    menu
                    ;;
            求求你放过我)
                    break
                    ;;
    esac
done
```



## 六、SSH安全优化

### 1.优化内容

```bash
#SSH作为远程连接服务，通常我们需要考虑到该服务的安全，所以需要对该服务进行安全方面的配置。
1.更改远程连接登陆的端口
2.禁止ROOT管理员直接登录
3.密码认证方式改为密钥认证
4.重要服务不使用公网IP地址
5.使用防火墙限制来源IP地址
```

### 2.优化的配置

```bash
[root@m01 ~]# vim /etc/ssh/sshd_config
#修改ssh服务的端口
Port 1748
#禁止使用root登录服务器
PermitRootLogin no
#禁止使用密码登录服务器
PasswordAuthentication no

UseDNS                  no      # 禁止ssh进行dns反向解析，影响ssh连接效率参数
GSSAPIAuthentication    no      # 禁止GSS认证，减少连接时产生的延迟
```



## 七、扩展

### 1.免交互expect

#### 1）安装expect

```bash
[root@m01 ~]# yum install -y expect
```

#### 2）编写expect脚本

```bash
[root@m01 ~]# vim xuanjian.exp
#!/usr/bin/expect
set ip 10.0.0.51
set pass 123456
set timeout 30
spawn ssh root@$ip
expect {
        "(yes/no)" {send "yes\r"; exp_continue}
        "password:" {send "$pass\r"}
}
expect "root@*"  {send "df -h\r"}
expect "root@*"  {send "exit\r"}
expect eof
```



### 2.免交互sshpass

#### 1）安装sshpass

```bash
[root@m01 ~]# yum install -y sshpass
```

#### 2）使用sshpass命令

```bash
[root@m01 ~]# sshpass -p 123456 ssh root@10.0.0.51

[option]
-p：指定密码
-f：从文件中取密码
-e：从环境变量中取密码
-P：设置密码提示
```
