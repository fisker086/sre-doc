# GTID复制

## 一、介绍

```mysql
1/GTID(Global Transaction ID)是对于一个已提交事务的唯一编号，并且是一个全局(主从复制)唯一的编号。

它的官方定义如下：
	GTID = source_id ：transaction_id
	7E11FA47-31CA-19E1-9E56-C43AA21293967:29

什么是sever_uuid，和Server-id 区别？

	核心特性: 全局唯一,具备幂等性
```



## 二、作用

```mysql
1、主要保证主从复制中的高级的特性
2、GTID：5.6版本出现没有默认开启，5.7中及时不开启也有匿名的GTID记录。
3、提供了dump传输并行，SQL线程并发。
4、5.7.17+的版本以后几乎都是GTID模式了。
```



## 三、GTID核心参数

```mysql
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1

gtid-mode=on                        --启用gtid类型，否则就是普通的复制架构
enforce-gtid-consistency=true               --强制GTID的一致性
log-slave-updates=1                 --slave更新是否记入日志
```



## 四、GTID配置过程

### 1、环境准备

```mysql
ip：51 52 53
hostname：db01 db02 db03
防火墙关闭
能够实现远程xshell连接
```



### 2、清理环境

```bash
#三个节点都做
pkill mysqld
rm -rf /service/mysql/data/*
rm -rf /service/mysql/binlog/mysql-bin.*
mkdir /service/mysql/data/
mkdir /service/mysql/binlog/
```



### 3、准备配置文件

#### 1）主库

```bash
cat > /etc/my.cnf <<EOF
[mysqld]
basedir=/service/mysql/
datadir=/service/mysql/data
socket=/tmp/mysql.sock
server_id=51
port=3306
secure-file-priv=/tmp
autocommit=0
log_bin=/service/mysql/binlog/mysql-bin
binlog_format=row
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
[mysql]
prompt=db01 [\\d]>
EOF
```



#### 2）从库db02

```bash
cat > /etc/my.cnf <<EOF
[mysqld]
basedir=/service/mysql
datadir=/service/mysql/data
socket=/tmp/mysql.sock
server_id=52
port=3306
secure-file-priv=/tmp
autocommit=0
log_bin=/service/mysql/binlog/mysql-bin
binlog_format=row
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
[mysql]
prompt=db02 [\\d]>
EOF
```



#### 3）从库db03

```bash
cat > /etc/my.cnf <<EOF
[mysqld]
basedir=/service/mysql
datadir=/service/mysql/data
socket=/tmp/mysql.sock
server_id=53
port=3306
secure-file-priv=/tmp
autocommit=0
log_bin=/service/mysql/binlog/mysql-bin
binlog_format=row
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
[mysql]
prompt=db03 [\\d]>
EOF
```



### 4、初始化数据(三库)

```mysql
mysqld --initialize-insecure --user=mysql --basedir=/service/mysql  --datadir=/service/mysql/data 
```



### 5、启动数据库并加入开机自启（三库）

```bash
systemctl start mysqld.service 
systemctl enable mysqld.service 
```



### 6、构建主从

#### 4）主库：

```mysql
grant replication slave  on *.* to repl@'10.0.0.%' identified by '123';
```



#### 5）从库：

```mysql
change master to 
master_host='10.0.0.51',
master_user='repl',
master_password='123' ,
MASTER_AUTO_POSITION=1;

start slave;
```



## 五、GTID复制和传统复制的区别

```bash
CHANGE MASTER TO
MASTER_HOST='10.0.0.51',
MASTER_USER='repl',
MASTER_PASSWORD='123',
MASTER_PORT=3307,
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=444,
MASTER_CONNECT_RETRY=10;

change master to 
master_host='10.0.0.51',
master_user='repl',
master_password='123' ,
MASTER_AUTO_POSITION=1;
start slave;

（0）在主从复制环境中，主库发生过的事务，在全局都是由唯一GTID记录的，更方便Failover
（1）额外功能参数（3个）
（2）change master to 的时候不再需要binlog 文件名和position号,MASTER_AUTO_POSITION=1;
（3）在复制过程中，从库不再依赖master.info文件，而是直接读取最后一个relaylog的 GTID号
（4） mysqldump备份时，默认会将备份中包含的事务操作，以以下方式
    SET @@GLOBAL.GTID_PURGED='8c49d7ec-7e78-11e8-9638-000c29ca725d:1';
    告诉从库，我的备份中已经有以上事务，你就不用运行了，直接从下一个GTID开始请求binlog就行。
```



## 六、GTID优缺点

```bash
#GTID优点(新特性):
1.GTID会开启多个SQL线程，每一个库，开启一个SQL线程
2.如果开启row:只保存改变数据的列，节省网络资源，磁盘占用空间和内存使用
3.GTID会把主从相关的信息，记录到表中，增强使用性
4.支持延时复制
5.传统的主从复制，show master status; 记录:binlog_file  binlog_pos
  基于GTID的主从复制，不需要记录这两个值，直接自动查找
6.保证事务全局统一
7.截取日志更加方便。跨多文件，判断起点终点更加方便。
8.判断主从工作状态更加方便
9.传输日志可以并发传输。
10。主从复制更加方便

#缺点:
1.mysqldump 备份起来很麻烦，需要额外加参数 --set-gtid=on
2.如果主从复制遇到了错误，SQL停了，跳过错误，GTID无法跳过错误
```



## 七、GTID从库误写入操作处理

```bash
查看监控信息:
Last_SQL_Error: Error 'Can't create database 'xiaowu'; database exists' on query. Default database: 'xiaowu'. Query: 'create database xiaowu'

Retrieved_Gtid_Set: 71bfa52e-4aae-11e9-ab8c-000c293b577e:1-3
Executed_Gtid_Set:  71bfa52e-4aae-11e9-ab8c-000c293b577e:1-2,
7ca4a2b7-4aae-11e9-859d-000c298720f6:1

注入空事物的方法：

stop slave;
set gtid_next='99279e1e-61b7-11e9-a9fc-000c2928f5dd:3';
begin;commit;
set gtid_next='AUTOMATIC';
    
这里的xxxxx:N 也就是你的slave sql thread报错的GTID，或者说是你想要跳过的GTID。
最好的解决方案：重新构建主从环境
```

