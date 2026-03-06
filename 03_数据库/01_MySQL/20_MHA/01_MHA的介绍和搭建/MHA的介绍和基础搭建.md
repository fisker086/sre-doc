# MHA的介绍和基础搭建

## 一、架构的工作原理

```mysql
1. 启动manager 程序: masterha_manger
2. 监控: masterha_master_monitor 每隔ping_interval检测主库心跳,一共检测四次.
3. 如果出现多次检查没有心跳,进入切换流程
4. 选主:  
           alive   数组: 存活的从节点的编号.
           lastest数组: 日志最新的从节点编号.  Master_Log_File ,Read_Master_Log_Pos
           pref    数组: 备选从节点的编号. candidate_master>=0
           bad    数组 : 不应该被选择的从节点编号.
                      binlog没开,no_master=1,日志差异100000000 pos
           0  手工切换时,人为指定的节点,被作为新主
          ① if     lastest &&  pref !& bad,被选择为新主 
          ② elif  lastest !& bad ,被选择为新主
          ③ elif  pref !& bad  ,被选择为新主
          ④ elif     alive   !& bad
          ⑤ 报错
           说明: 如果结果多个节点都满足,按照[serverN]中的N值来选主.
5.  数据补偿 
          a. 主库SSH能连
              save_binary_logs 远程保存缺失部分日志到从节点的/var/tmp/xxx,进行补偿
          b. ssh不能连
               从节点直接通过apply_diff_relay_logs,计算之间的差异并恢复

6.  切换主从关系masterha_master_switch
           解除老的主从关系 stop slave  ;reset slave;
           构建新的主从 change master to; start slave;
7.  VIP 应用透明 master_ip_failover
8.  故障通知 send_report
9.  将配置文件中的故障节点删掉masterha_conf_host
10. "自杀".
```



## 二、架构介绍

```mysql
1主2从，master：db01   slave：db02   db03 ）：

MHA 高可用方案软件构成
    Manager软件：选择一个从节点安装
    Node软件：所有节点都要安装
```



## 三、MHA软件

```mysql
Manager工具包主要包括以下几个工具：
masterha_manger             启动MHA 
masterha_check_ssh      	检查MHA的SSH配置状况 
masterha_check_repl         检查MySQL复制状况 
masterha_master_monitor     检测master是否宕机 
masterha_check_status       检测当前MHA运行状态 
masterha_master_switch  	控制故障转移（自动或者手动）
masterha_conf_host      	添加或删除配置的server信息

Node工具包主要包括以下几个工具：
这些工具通常由MHA Manager的脚本触发，无需人为操作
save_binary_logs            保存和复制master的二进制日志 
apply_diff_relay_logs       识别差异的中继日志事件并将其差异的事件应用于其他的
purge_relay_logs            清除中继日志（不会阻塞SQL线程）
```



## 四、产品经理角度，评估高可用软件设计

```bash
1、监控 
2、选主
3、数据补偿
4、故障转移
5、应用透明
6、自动提醒
7、自愈(待开发)
```



## 五、MHA架构搭建

### 1、规划

```mysql
主库: 
	51    node 
从库: 
    52      node
    53      node    manager
```



### 2、准备环境

一主两从GTID（略）



### 3、配置关键程序软连接（所有节点）

```mysql
ln -s /service/mysql/bin/mysqlbinlog /usr/bin/mysqlbinlog
ln -s /service/mysql/bin/mysql /usr/bin/mysql
```



### 4、配置各节点相互信任（密钥对）

```mysql
#db01创建秘钥对分发给从库
[root@db01 ~]# rm -rf /root/.ssh
[root@db01 ~]# ssh-keygen		交互式直接全部回车
[root@db01 ~]# cd /root/.ssh/
[root@db01 ~/.ssh]# mv id_rsa.pub authorized_keys
[root@db01 ~/.ssh]# scp  -r  /root/.ssh  10.0.0.52:/root
[root@db01 ~/.ssh]# scp  -r  /root/.ssh  10.0.0.53:/root

各个节点相互验证
db01:
ssh 10.0.0.51 date
ssh 10.0.0.52 date
ssh 10.0.0.53 date
db02:
ssh 10.0.0.51 date
ssh 10.0.0.52 date
ssh 10.0.0.53 date
db03:
ssh 10.0.0.51 date
ssh 10.0.0.52 date
ssh 10.0.0.53 date
```



### 5、安装软件

#### 1）下载MHA软件

```bash
mha官网：https://code.google.com/archive/p/mysql-master-ha/
github下载地址：https://github.com/yoshinorim/mha4mysql-manager/wiki/Downloads

说明：
8.0的版本
	1.修改密码加密模式sha2--->native
	2.使用0.58的MHA版本
```



#### 2）所有节点安装Node软件依赖包

```bash
yum install perl-DBD-MySQL -y
rpm -ivh mha4mysql-node-0.57-0.el7.noarch.rpm
```



#### 3）在主库创建mha需要的用户

```bash
 grant all privileges on *.* to mha@'10.0.0.%' identified by 'mha';
```



#### 4）db03安装Manager

```bash
yum install -y perl-Config-Tiny epel-release perl-Log-Dispatch perl-Parallel-ForkManager perl-Time-HiRes
[root@db03 ~]# rpm -ivh mha4mysql-manager-0.57-0.el7.noarch.rpm 
```





### 6、配置文件准备

```bash
1、创建配置文件目录
 mkdir -p /etc/mha

2、创建日志目录
 mkdir -p /var/log/mha/app1

3、编辑mha配置文件
vim /etc/mha/app1.cnf
[server default]
manager_log=/var/log/mha/app1/manager        
manager_workdir=/var/log/mha/app1            
master_binlog_dir=/service/mysql/binlog       
user=mha                                   
password=mha                               
ping_interval=2
repl_password=123
repl_user=repl
ssh_user=root                               
[server1]                                   
hostname=10.0.0.51
port=3306                                  
[server2]            
hostname=10.0.0.52
port=3306
[server3]
hostname=10.0.0.53
port=3306
```

```bash
[server default]						#总配置文件
manager_log=/var/log/mha/app1/manager   #manager主程序日志
manager_workdir=/var/log/mha/app1       #工作目录
master_binlog_dir=/service/mysql/binlog       	#主库binlog位置点
user=mha                            	#mha工作用户       
password=mha                            #mha工作密码
ping_interval=2							#判断间隔期
repl_password=123						#主从复制用户
repl_user=repl							#主从复制密码
ssh_user=root          			        #本地root互信名             


#各个节点
[server1]                                   
hostname=10.0.0.51
port=3306                                  
[server2]            
hostname=10.0.0.52
port=3306
[server3]
hostname=10.0.0.53
port=3306
```



### 7、状态检查

#### 1）互信检查

```bash
[root@db03 ~]# masterha_check_ssh  --conf=/etc/mha/app1.cnf 

Fri Apr 19 16:39:34 2019 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Fri Apr 19 16:39:34 2019 - [info] Reading application default configuration from /etc/mha/app1.cnf..
Fri Apr 19 16:39:34 2019 - [info] Reading server configuration from /etc/mha/app1.cnf..
Fri Apr 19 16:39:34 2019 - [info] Starting SSH connection tests..
Fri Apr 19 16:39:35 2019 - [debug] 
Fri Apr 19 16:39:34 2019 - [debug]  Connecting via SSH from root@10.0.0.51(10.0.0.51:22) to root@10.0.0.52(10.0.0.52:22)..
Fri Apr 19 16:39:34 2019 - [debug]   ok.
Fri Apr 19 16:39:34 2019 - [debug]  Connecting via SSH from root@10.0.0.51(10.0.0.51:22) to root@10.0.0.53(10.0.0.53:22)..
Fri Apr 19 16:39:35 2019 - [debug]   ok.
Fri Apr 19 16:39:36 2019 - [debug] 
Fri Apr 19 16:39:35 2019 - [debug]  Connecting via SSH from root@10.0.0.52(10.0.0.52:22) to root@10.0.0.51(10.0.0.51:22)..
Fri Apr 19 16:39:35 2019 - [debug]   ok.
Fri Apr 19 16:39:35 2019 - [debug]  Connecting via SSH from root@10.0.0.52(10.0.0.52:22) to root@10.0.0.53(10.0.0.53:22)..
Fri Apr 19 16:39:35 2019 - [debug]   ok.
Fri Apr 19 16:39:37 2019 - [debug] 
Fri Apr 19 16:39:35 2019 - [debug]  Connecting via SSH from root@10.0.0.53(10.0.0.53:22) to root@10.0.0.51(10.0.0.51:22)..
Fri Apr 19 16:39:35 2019 - [debug]   ok.
Fri Apr 19 16:39:35 2019 - [debug]  Connecting via SSH from root@10.0.0.53(10.0.0.53:22) to root@10.0.0.52(10.0.0.52:22)..
Fri Apr 19 16:39:36 2019 - [debug]   ok.
Fri Apr 19 16:39:37 2019 - [info] All SSH connection tests passed successfully.
```



#### 2）主从状态检查

```bash
[root@db03 ~]# masterha_check_repl  --conf=/etc/mha/app1.cnf



Fri Mar 19 13:40:11 2021 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Fri Mar 19 13:40:11 2021 - [info] Reading application default configuration from /etc/mha/app1.cnf..
Fri Mar 19 13:40:11 2021 - [info] Reading server configuration from /etc/mha/app1.cnf..
Fri Mar 19 13:40:11 2021 - [info] MHA::MasterMonitor version 0.57.
Fri Mar 19 13:40:12 2021 - [info] GTID failover mode = 1
Fri Mar 19 13:40:12 2021 - [info] Dead Servers:
Fri Mar 19 13:40:12 2021 - [info] Alive Servers:
Fri Mar 19 13:40:12 2021 - [info]   10.0.0.51(10.0.0.51:3306)
Fri Mar 19 13:40:12 2021 - [info]   10.0.0.52(10.0.0.52:3306)
Fri Mar 19 13:40:12 2021 - [info]   10.0.0.53(10.0.0.53:3306)
Fri Mar 19 13:40:12 2021 - [info] Alive Slaves:
Fri Mar 19 13:40:12 2021 - [info]   10.0.0.52(10.0.0.52:3306)  Version=5.7.28-log (oldest major version between slaves) log-bin:enabled
Fri Mar 19 13:40:12 2021 - [info]     GTID ON
Fri Mar 19 13:40:12 2021 - [info]     Replicating from 10.0.0.51(10.0.0.51:3306)
Fri Mar 19 13:40:12 2021 - [info]   10.0.0.53(10.0.0.53:3306)  Version=5.7.28-log (oldest major version between slaves) log-bin:enabled
Fri Mar 19 13:40:12 2021 - [info]     GTID ON
Fri Mar 19 13:40:12 2021 - [info]     Replicating from 10.0.0.51(10.0.0.51:3306)
Fri Mar 19 13:40:12 2021 - [info] Current Alive Master: 10.0.0.51(10.0.0.51:3306)
Fri Mar 19 13:40:12 2021 - [info] Checking slave configurations..
Fri Mar 19 13:40:12 2021 - [info]  read_only=1 is not set on slave 10.0.0.52(10.0.0.52:3306).
Fri Mar 19 13:40:12 2021 - [info]  read_only=1 is not set on slave 10.0.0.53(10.0.0.53:3306).
Fri Mar 19 13:40:12 2021 - [info] Checking replication filtering settings..
Fri Mar 19 13:40:12 2021 - [info]  binlog_do_db= , binlog_ignore_db= 
Fri Mar 19 13:40:12 2021 - [info]  Replication filtering check ok.
Fri Mar 19 13:40:12 2021 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Fri Mar 19 13:40:12 2021 - [info] Checking SSH publickey authentication settings on the current master..
Fri Mar 19 13:40:13 2021 - [info] HealthCheck: SSH to 10.0.0.51 is reachable.
Fri Mar 19 13:40:13 2021 - [info] 
10.0.0.51(10.0.0.51:3306) (current master)
 +--10.0.0.52(10.0.0.52:3306)
 +--10.0.0.53(10.0.0.53:3306)

Fri Mar 19 13:40:13 2021 - [info] Checking replication health on 10.0.0.52..
Fri Mar 19 13:40:13 2021 - [info]  ok.
Fri Mar 19 13:40:13 2021 - [info] Checking replication health on 10.0.0.53..
Fri Mar 19 13:40:13 2021 - [info]  ok.
Fri Mar 19 13:40:13 2021 - [warning] master_ip_failover_script is not defined.
Fri Mar 19 13:40:13 2021 - [warning] shutdown_script is not defined.
Fri Mar 19 13:40:13 2021 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.

```

### 8、状态检查无误，开启MHA

```bash
[root@db03 ~]# nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover  < /dev/null> /var/log/mha/app1/manager.log 2>&1 &
```



### 9、确认MHA启动成功

```bash
[root@db03 ~]# masterha_check_status --conf=/etc/mha/app1.cnf
app1 (pid:2526) is running(0:PING_OK), master:10.0.0.51
```



### 10、Manager额外参数介绍

```bash
说明：
主库宕机谁来接管？
1. 所有从节点日志都是一致的，默认会以配置文件的顺序去选择一个新主。
2. 从节点日志不一致，自动选择最接近于主库的从库
3. 如果对于某节点设定了权重（candidate_master=1），权重节点会优先选择。
但是此节点日志量落后主库100M日志的话，也不会被选择。可以配合check_repl_delay=0，关闭日志量的检查，强制选择候选节点。

(1)  ping_interval=1
#设置监控主库，发送ping包的时间间隔，尝试三次没有回应的时候自动进行failover
(2) candidate_master=1
#设置为候选master，如果设置该参数以后，发生主从切换以后将会将此从库提升为主库，即使这个主库不是集群中事件最新的slave
(3)check_repl_delay=0
#默认情况下如果一个slave落后master 100M的relay logs的话，
MHA将不会选择该slave作为一个新的master，因为对于这个slave的恢复需要花费很长时间，通过设置check_repl_delay=0,MHA触发切换在选择一个新的master的时候将会忽略复制延时，这个参数对于设置了candidate_master=1的主机非常有用，因为这个候选主在切换的过程中一定是新的master
```



