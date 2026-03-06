# ProxySQL读写分离

## 一、介绍

```bash
ProxySQL是基于MySQL的一款开源的中间件的产品，是一个灵活的MySQL代理层，可以实现读写分离，支持 Query路由功能，支持动态指定某个SQL进行缓存，支持动态加载配置信息（无需重启 ProxySQL 服务），支持故障切换和SQL的过滤功能。 
相关 ProxySQL 的网站：
https://www.proxysql.com/
https://github.com/sysown/proxysql/wiki
```

## 二、安装

### 1、下载proxySQL

```bash
https://proxysql.com/
https://github.com/sysown/proxysql/releases

https://github.com/sysown/proxysql/releases/download/v2.3.2/proxysql_2.3.2-ubuntu18_amd64.deb
```

### 2、安装

```bash
dpkg -i proxysql_2.3.2-ubuntu18_amd64.deb
```

## 三、操作

```bash
版本：sudo proxysql --version
启动：sudo service proxysql start
暂停：sudo service proxysql stop
重启：sudo service proxysql restart
状态：sudo service proxysql status
```

## 四、端口

```bash
客户端：6033端口
管理端：6032端口
```

## 五、配置文件

> 不推荐使用

```bash
/etc/proxysql.cnf
```

## 六、控制台及介绍

>上述之所以不推荐，是因为我们可以通过ProxySQL控制台在线修改配置，无需重启，立即生效。

### 1、登录

```bash
mysql -uadmin -padmin -h127.0.0.1 -P6032 --prompt='Admin> ' --default-auth=mysql_native_password
```

### 2、ProxySQL中管理结构自带的系统库

>在ProxySQL，6032端口共五个库： main、disk、stats 、monitor、stats_history

#### 1.main

```bash
mysql_servers: 后端可以连接 MySQL 服务器的列表 
	mysql_users:   配置后端数据库的账号和监控的账号。 
	mysql_query_rules: 指定 Query 路由到后端不同服务器的规则列表。
	mysql_replication_hostgroups : 节点分组配置信息
注： 表名以 runtime_开头的表示ProxySQL 当前运行的配置内容，不能直接修改。不带runtime_是下文图中Mem相关的配置。
```

#### 2.disk

```bash
持久化的磁盘的配置
```

#### 3.stats

```bash
统计信息的汇总
```

#### 4.monitor

```bash
监控的收集信息，比如数据库的健康状态等
```

#### 5.stats_history

```bash
ProxySQL 收集的有关其内部功能的历史指标
```

### 3、ProxySQL管理接口的多层配置关系

#### 1.整套配置系统分为三大层

```bash
顶层   RUNTIME 
中间层 MEMORY  （主要修改的配置表）
持久层 DISK 和 CFG FILE 
```

```bash
RUNTIME ： 
	代表 ProxySQL 当前正在使用的配置，无法直接修改此配置，必须要从下一层 （MEM层）“load” 进来。 
	
MEMORY： 
	MEMORY 层上面连接 RUNTIME 层，下面disk持久层。这层可以在线操作 ProxySQL 配置，随便修改，不会影响生产环境。确认正常之后在加载达到RUNTIME和持久化的磁盘上。修改方法： insert、update、delete、select。
	
DISK和CONFIG FILE：
	持久化配置信息。重启时，可以从磁盘快速加载回来。不建议直接修改此配置文件，建议从上一层(MEM层)“SAVE”进来。
```

### 4、在不同层次件移动配置

```bash
LOAD xxxx  TO RUNTIME;
SAVE xxxx  TO DISK;
```

>为了将配置持久化到磁盘或者应用到 runtime，在管理接口下有一系列管理命令来实现它们。

#### 1.user相关配置

```bash
##  MEM 加载到runtime
LOAD MYSQL USERS TO RUNTIME;

##  runtime 保存至 MEM
SAVE MYSQL USERS TO MEMORY;

## disk 加载到 MEM
LOAD MYSQL USERS FROM DISK;

## MEM  到 disk 
SAVE MYSQL USERS TO DISK;

## CFG 到 MEM
LOAD MYSQL USERS FROM CONFIG
```

#### 2.server 相关配置

```bash
##  MEM 加载到runtime
LOAD MYSQL SERVERS TO RUNTIME;

##  runtime 保存至 MEM
SAVE MYSQL SERVERS TO MEMORY;

## disk 加载到 MEM
LOAD MYSQL SERVERS FROM DISK;

## MEM  到 disk 
SAVE MYSQL SERVERS TO DISK;

## CFG 到 MEM
LOAD MYSQL SERVERS FROM CONFIG
```

#### 3.mysql query rules配置

```bash
##  MEM 加载到runtime
LOAD MYSQL QUERY RULES TO RUNTIME;

##  runtime 保存至 MEM
SAVE MYSQL QUERY RULES TO MEMORY;

## disk 加载到 MEM
LOAD MYSQL QUERY RULES FROM DISK;

## MEM  到 disk 
SAVE MYSQL QUERY RULES TO DISK;

## CFG 到 MEM
LOAD MYSQL QUERY RULES FROM CONFIG
```

#### 4.MySQL variables配置

```bash
##  MEM 加载到runtime
LOAD MYSQL VARIABLES TO RUNTIME;

##  runtime 保存至 MEM
SAVE MYSQL VARIABLES TO MEMORY;

## disk 加载到 MEM
LOAD MYSQL VARIABLES FROM DISK;

## MEM  到 disk 
SAVE MYSQL VARIABLES TO DISK;

## CFG 到 MEM
LOAD MYSQL VARIABLES FROM CONFIG
```

#### 5.总结

```bash
日常配置其实大部分时间在MEM配置，然后load到RUNTIME，然后SAVE到DIsk。cfg很少使用。
例如 ： 
load xxx to runtime;
save xxx to disk;
```

```bash
注意：
	只有load到 runtime 状态时才会验证配置。在保MEM或disk时，都不会发生任何警告或错误。当load到 runtime 时，如果出现错误，将恢复为之前保存得状态，这时可以去检查错误日志。
```

## 七、ProxySQL应用——基于SQL的读写分离

### 1、从库设定read_only参数

```mysql
set global read_only=1;
set global super_read_only=1;
```

### 2、在mysql_replication_hostgroup表中，配置读写组编号

#### 1.登录控制台

```bash
mysql -uadmin -padmin -h127.0.0.1 -P6032 --prompt='Admin> ' --default-auth=mysql_native_password
```

#### 2.配置读写组编号

```mysql
insert into 
mysql_replication_hostgroups 
(writer_hostgroup, reader_hostgroup, comment) 
values (10,20,'proxy');
```

#### 3.读取到runtime运行

```bash
load mysql servers to runtime;
```

#### 4.查看结果

```mysql
select * from main.mysql_replication_hostgroups\G
*************************** 1. row ***************************
writer_hostgroup: 10
reader_hostgroup: 20
      check_type: read_only
         comment: proxy
1 row in set (0.00 sec)
```

#### 5.确认后保存并永久生效

```bash
save mysql servers to disk;
```

#### 6.说明

```bash
ProxySQL 会根据server 的read_only 的取值将服务器进行分组。 read_only=0 的server，master被分到编号为10的写组，read_only=1 的server，slave则被分到编号20的读组。所以需要将从库设置：
set global read_only=1;
```

### 3、添加主机当ProxySQL

#### 1.添加

```mysql
insert into mysql_servers(hostgroup_id,hostname,port) values (10,'172.16.0.25',3306);
insert into mysql_servers(hostgroup_id,hostname,port) values (20,'172.16.0.26',3306);
insert into mysql_servers(hostgroup_id,hostname,port) values (20,'172.16.0.6',3306);
```

#### 2.查看

```mysql
select * from main.mysql_servers;
+--------------+-------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname    | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+-------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 10           | 172.16.0.25 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 20           | 172.16.0.26 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 20           | 172.16.0.6  | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+-------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+

```

#### 3.立即生效并永久生效

```mysql
load mysql servers to runtime;
save mysql servers to disk;
```

### 4、创建监控用户，并开启监控

#### 1.主库创建监控用户

```mysql
create user monitor@'%' identified with mysql_native_password  by 'proxysqlmonitor';
grant replication client on *.* to monitor@'%';

flush privileges;
```

#### 2.ProxySQL修改variables表

```mysql
set mysql-monitor_username='monitor';
set mysql-monitor_password='proxysqlmonitor';
```

或者

```mysql
UPDATE global_variables SET variable_value='monitor'
WHERE variable_name='mysql-monitor_username';
UPDATE global_variables SET variable_value='proxysqlmonitor'
WHERE variable_name='mysql-monitor_password';
```

#### 3.立即并永久生效

```bash
load mysql variables to runtime;
save mysql variables to disk;
```

#### 4.查询监控日志

```mysql
select * from mysql_server_connect_log;
select * from mysql_server_ping_log; 
select * from mysql_server_read_only_log;
select * from mysql_server_replication_lag_log;
```

### 5、配置应用用户

#### 1.主库创建应用账户

```mysql
create user mrobot@'%' identified with mysql_native_password  by 'root1qaz@WSX';
grant all on *.* to mrobot@'%';

flush privileges;
```

#### 2.ProxySQL添加应用账户

```mysql
insert into mysql_users(username,password,default_hostgroup) values('mrobot','root1qaz@WSX',10);
```

#### 3.保存并立即生效

```mysql
load mysql users to runtime;
save mysql users to disk;
```

> 早起版本要开启事务持续化

```mysql
update mysql_users set transaction_persistent=1 where username='root';
load mysql users to runtime;
save mysql users to disk;
```

### 6、编写读写规则

#### 1.插入规则

```mysql
insert into mysql_query_rules(rule_id,active,match_pattern,destination_hostgroup,apply) values (1,1,'^select.*for update$',10,1);
insert into mysql_query_rules(rule_id,active,match_pattern,destination_hostgroup,apply) values (2,1,'^select',20,1);
```

#### 2.保存并立即生效

```mysql
load mysql query rules to runtime;
save mysql query rules to disk;
```

#### 3.注意

```bash
select … for update规则的rule_id必须要小于普通的select规则的rule_id，ProxySQL是根据rule_id的顺序进行规则匹配。
```

### 7、测试读写分离

随便找台有mysql客户端的机器

#### 1.测试读操作

```bash
mysql -umrobot -proot1qaz@WSX -h 172.16.0.6 -P 6033 -e "select @@server_id;"
```

#### 2.测试写操作

```bash
mysql -umrobot -proot1qaz@WSX -h 172.16.0.6 -P 6033 -e "begin;select @@server_id;commit;"
```

