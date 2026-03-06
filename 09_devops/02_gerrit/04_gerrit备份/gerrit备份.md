# Rsync备份gerrrit

## 一、了解需求

```bash
客户端需求:
1.客户端将本地数据进行推送至备份服务器
2.客户端每天凌晨3点定时执行该脚本

服务端需求:
1.服务端部署rsync，用于接收客户端推送过来的备份数据
2.服务端仅保留7天的备份数据,其余的全部删除
```



## 二、客户端需求

```bash
apt-get install -y rsync
```



### 1、尝试获取信息

```bash
#获取主机名
root@gerrit:~# hostname
gerrit

#获取主机ip
root@gerrit:~# hostname -I
172.16.3.50 172.17.0.1 
root@gerrit:~# hostname -I |awk '{print $1}'   #排除容器网桥
172.16.3.50

# 获取当前时间
root@gerrit:~# date +%F
2022-04-19
```

### 2、编写脚本

```bash
 vim gerrit_rsync_client.sh 
 #!/bin/bash
 #1、定义变量
 DIR=/home/gerrit2/gerrit_site
 HOSTNAME=$(hostname)
 IP=$(hostname -I | awk '{print $1}')
 DATE=$(date +%F)
 SRC=${HOSTNAME}_${IP}_${DATE} 
 
 #2、推送文件
 export RSYNC_PASSWORD=123456
 rsync -az $DIR/ gerrit-backup@172.16.0.6::gerrit-backup/$SRC
```

### 3、将脚本加入定时任务

```bash
[root@web01 ~]# crontab -e
#每天凌晨3点执行备份脚本
0 3 * * * root /bin/bash /home/gerrit2/gerrit_rsync_client.sh
```



## 三、服务端需求

### 1、部署rsync服务端

```bash
#下载rsync
apt-get install -y rsync

#创建用户
id rsync
#没有就创建
useradd -M -s /sbin/nologin rsync

#创建密码文件（一定不能有空格）
echo "gerrit-backup:123456" >> /etc/rsync.passwd
chmod 600 /etc/rsync.passwd

#创建备份目录
mkdir /gerrit-backup

#授权备份目录
chown -R rsync.rsync /gerrit-backup
```

### 2、编辑配置文件

```bash
vim /etc/rsyncd.conf

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
secrets file = /etc/rsync.passwd
log file = /var/log/rsyncd.log
#####################################
[gerrit-backup]
auth users = gerrit-backup
comment = This is gerrit-backup!
path = /gerrit-backup

systemctl restart rsync.service 
netstat -lntup
systemctl enable rsync.service 
```

### 3、服务端脚本

```bash
 vim gerrit_rsync_server.sh 
#!/bin/bash
#1.定义变量
DIR=/gerrit-backup
#2.删除7天之前的数据
find $DIR/ -type d -mtime +7 | xargs rm -rf
```

### 4、将服务脚本加入定时任务

```bash
crontab -e
30 7 * * * /bin/bash /gerrit-backup/gerrit_rsync_server.sh 
```

