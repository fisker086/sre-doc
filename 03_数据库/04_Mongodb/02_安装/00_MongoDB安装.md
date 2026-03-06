# MongoDB安装

## 一、获取MongoDB

>https://www.mongodb.com/try/download/community

## 二、安装MongoDB

### 1、上传软件并解压

```bash
[root@node01 nosql]# mkdir -p /mongodb && cd /mongodb
[root@node01 nosql]# tar xf  mongodb-linux-x86_64-rhel70-4.2.8.tgz
```

### 2、关闭THP

>root用户下

```bash
在vi /etc/rc.local最后添加如下代码

if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
	echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
		   echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi

[root@node01 app]# cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]
[root@node01 app]# cat /sys/kernel/mm/transparent_hugepage/defrag
always madvise [never]
```

#### 其他系统关闭参照官方文档

>https://docs.mongodb.com/manual/tutorial/transparent-huge-pages/

#### 关闭原因

>Transparent Huge Pages (THP) is a Linux memory management system 
>that reduces the overhead of Translation Lookaside Buffer (TLB) 
>lookups on machines with large amounts of memory by using larger memory pages.
>However, database workloads often perform poorly with THP, 
>because they tend to have sparse rather than contiguous memory access patterns. 
>You should disable THP on Linux machines to ensure best performance with MongoDB.

### 3、环境准备

#### 1.创建用户和组

```bash
useradd mongod
passwd mongod
```

#### 2.创建mongodb所需目录结构

```bash
mkdir -p /mongodb/conf
mkdir -p /mongodb/log
mkdir -p /mongodb/data
```

#### 3.修改权限

```bash
chown -R mongod:mongod /mongodb
```

#### 4.切换用户并设置环境变量

```bash
su - mongod
vi .bash_profile

export PATH=/mongodb/app/bin:$PATH
source .bash_profile
```

#### 5.启动数据库并初始化数据

```bash
su - mongod 
mongod --dbpath=/mongodb/data --logpath=/mongodb/log/mongodb.log --port=27017 --logappend --fork 

# 登录mongodb
$ mongo 
```

### 4、配置持久化

#### 1.旧的方式配置

```bash
vim /mongodb/conf/mongodb.conf
logpath=/mongodb/log/mongodb.log
dbpath=/mongodb/data
port=27017
logappend=true
fork=true


# 关闭mongodb
mongod -f /mongodb/conf/mongodb.conf --shutdown
使用配置文件启动mongodb
mongod -f /mongodb/conf/mongodb.conf
```

#### 2.yaml方式配置

```bash
--
NOTE：
YAML does not support tab characters for indentation: use spaces instead.

--系统日志有关  
systemLog:
   destination: file        
   path: "/mongodb/log/mongodb.log"    --日志位置
   logAppend: true					   --日志以追加模式记录
--数据存储有关   
storage:
   journal:
      enabled: true
   dbPath: "/mongodb/data"            --数据路径的位置
-- 进程控制  
processManagement:
   fork: true                         --后台守护进程
   pidFilePath: <string>			  --pid文件的位置，一般不用配置，可以去掉这行，自动生成到data中
    
--网络配置有关   
net:			
   bindIp: <ip>                       -- 监听地址，如果不配置这行是监听在0.0.0.0
   port: <port>						  -- 端口号,默认不配置端口号，是27017
   
-- 安全验证有关配置      
security:
  authorization: enabled              --是否打开用户名密码验证

------------------以下是复制集与分片集群有关----------------------  
replication:
 oplogSizeMB: <NUM>
 replSetName: "<REPSETNAME>"
 secondaryIndexPrefetch: "all"
 
sharding:
   clusterRole: <string>
   archiveMovedChunks: <boolean>
      
---for mongos only
replication:
   localPingThresholdMs: <int>

sharding:
   configDB: <string>
---
.........
```

#### 3.单实例yaml配置

```bash
vim /mongodb/conf/mongo.conf
systemLog:
   destination: file
   path: "/mongodb/log/mongodb.log"
   logAppend: true
storage:
   journal:
      enabled: true
   dbPath: "/mongodb/data/"
processManagement:
   fork: true
net:
   port: 27017
   bindIp: 10.0.51,127.0.0.1


mongod -f /mongodb/conf/mongo.conf --shutdown
mongod -f /mongodb/conf/mongo.conf  
```