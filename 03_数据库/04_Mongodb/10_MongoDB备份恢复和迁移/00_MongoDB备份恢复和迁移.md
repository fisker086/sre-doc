# MongoDB备份恢复和迁移

## 一、备份恢复工具介绍

```bash
（1）**   mongoexport/mongoimport
（2）***** mongodump/mongorestore
```

## 二、备份工具区别在哪里？

```bash
mongoexport/mongoimport  导入/导出的是JSON格式或者CSV格式，
mongodump/mongorestore导入/导出的是BSON格式。

应用场景:
mongoexport/mongoimport:json csv 
1、异构平台迁移  mysql  <---> mongodb
2、同平台，跨大版本：mongodb 2  ----> mongodb 3

mongodump/mongorestore
日常备份恢复时使用.
```

## 三、导出工具mongoexport

### 1、介绍

>Mongodb中的mongoexport工具可以把一个collection导出成JSON格式或CSV格式的文件。
>可以通过参数指定导出的数据项，也可以根据指定的条件导出数据。
>（1）版本差异较大
>（2）异构平台数据迁移

### 2、帮助

```bash
mongoexport具体用法如下所示：

$ mongoexport --help  
参数说明：
-h:指明数据库宿主机的IP
-u:指明数据库的用户名
-p:指明数据库的密码
-d:指明数据库的名字
-c:指明collection的名字
-f:指明要导出那些列
-o:指明到要导出的文件名
-q:指明导出数据的过滤条件
--authenticationDatabase admin
```

### 3、用例

#### 1.单表备份至json格式

>注：备份文件的名字可以自定义，默认导出了JSON格式的数据。

```bash
use test
for(i=0;i<10000;i++){ db.log.insert({"uid":i,"name":"mongodb","age":6,"date":new Date()}); }

mongoexport -uroot -proot123 --port 27017 --authenticationDatabase admin -d test -c log -o /mongodb/log.json
```

#### 2.单表备份至csv格式

>如果我们需要导出CSV格式的数据，则需要使用--type=csv参数：

```bash
mongoexport -uroot -proot123 --port 27017 --authenticationDatabase admin -d test -c log --type=csv -f uid,name,age,date  -o /mongodb/log.csv
```

## 四、导入工具mongoimport

### 1、介绍

>Mongodb中的mongoimport工具可以把一个特定格式文件中的内容导入到指定的collection中。该工具可以导入JSON格式数据，也可以导入CSV格式数据。具体使用如下所示：

### 2、帮助

```bash
$ mongoimport --help
参数说明：
-h:指明数据库宿主机的IP
-u:指明数据库的用户名
-p:指明数据库的密码
-d:指明数据库的名字
-c:指明collection的名字
-f:指明要导入那些列
-j, --numInsertionWorkers=<number>  number of insert operations to run concurrently
```

### 3、用例

#### 1.恢复json格式表数据到log1

```bash
mongoimport -uroot -proot123 --port 27017 --authenticationDatabase admin -d test -c log1 /mongodb/log.json
```

#### 2.恢复csv格式的文件到log2

>上面演示的是导入JSON格式的文件中的内容，如果要导入CSV格式文件中的内容，则需要通过--type参数指定导入格式，具体如下所示：
>
>--headerline:指明第一行是列名，不需要导入。

##### 1）csv格式的文件头行，有列名字

```bash
mongoimport   -uroot -proot123 --port 27017 --authenticationDatabase admin   -d test -c log2 --type=csv --headerline --file  /mongodb/log.csv
```

##### 2）csv格式的文件头行，没有列名字

```bash
mongoimport   -uroot -proot123 --port 27017 --authenticationDatabase admin   -d test -c log3 -j 4 --type=csv -f id,name,age,date --file  /mongodb/log.csv
```

## 五、异构平台迁移案例

>mysql   -----> mongodb  
>world数据库下city表进行导出，导入到mongodb

### 1、mysql开启安全路径

```bash
vim /etc/my.cnf   --->添加以下配置
secure-file-priv=/data/backup/


--重启数据库生效
/etc/init.d/mysqld restart
```

### 2、导出mysql的city表数据

```mysql
select * from test.t100w into outfile '/tmp/t100w.csv' fields terminated by ','   ENCLOSED BY '"' ;
```

### 3、获取列信息

```mysql
mysql> select table_name,group_concat(column_name)  from information_schema.columns where table_schema='test' group by table_name order by null ;
+------------+---------------------------+
| TABLE_NAME | group_concat(column_name) |
+------------+---------------------------+
| t100w      | dt,id,k1,k2,num           |
+------------+---------------------------+
1 row in set (0.07 sec)
```

### 4、在mongodb中导入备份

```bash
mongoimport -uroot -proot123 --port 27017 --authenticationDatabase admin -d test -c t100w  --type=csv -f id,num，k1,k2,,dt --file  /tmp/t100w.csv

use world
db.t100w.find({});
```

## 六、如何将MySQL大量表迁移到MongoDB

### 1、批量从MySQL导出多张表

```bash
mysqldump   --fields-terminated-by ',' --fields-enclosed-by '"'  world -T /tmp/
cd /data/backup
rm -rf /data/backup/*.sql
find ./  -name "*.txt" | awk -F "." '{print $2}' | xargs -i -t mv ./{}.txt  ./{}.csv
```

### 2、拼接语句

```mysql
select concat("mongoimport -uroot -proot123 --port 27017 --authenticationDatabase admin -d ",table_schema, " -c  ",table_name ," --type=csv "," -f ", group_concat(column_name) ," --file  /data/backup/",table_name ,".csv")
from information_schema.columns where table_schema='world' group by table_name;
```

### 3、导入数据

```bash
[mongod@db01 backup]$ ll
total 256
-rwxrwxrwx 1 mysql mysql 184355 Jul 22 13:45 city.csv
-rwxrwxrwx 1 mysql mysql  38659 Jul 22 13:45 country.csv
-rwxrwxrwx 1 mysql mysql  26106 Jul 22 13:45 countrylanguage.csv
-rwxrwxrwx 1 mysql mysql    656 Jul 22 13:45 import.sh
[mongod@db01 backup]$ sh import.sh 
```

## 七、mongodump和mongorestore

### 1、介绍

>mongodump能够在Mongodb运行时进行备份，它的工作原理是对运行的Mongodb做查询，然后将所有查到的文档写入磁盘。但是存在的问题时使用mongodump产生的备份不一定是数据库的实时快照，如果我们在备份时对数据库进行了写入操作，则备份出来的文件可能不完全和Mongodb实时数据相等。另外在备份时可能会对其它客户端性能产生不利的影响。

### 2、mongodump参数

```bash
$ mongodump --help
参数说明：
-h:指明数据库宿主机的IP
-u:指明数据库的用户名
-p:指明数据库的密码
-d:指明数据库的名字
-c:指明collection的名字
-o:指明到要导出的文件名
-q:指明导出数据的过滤条件
-j, --numParallelCollections=  number of collections to dump in parallel (4 by default)
--oplog  备份的同时备份oplog
```

### 3、mongodump和mongorestore基本使用

```bash
全库备份
mkdir /mongodb/backup -p 
mongodump  -uroot -proot123 --port 27017 --authenticationDatabase admin -o /mongodb/backup

--备份world库
$ mongodump   -uroot -proot123 --port 27017 --authenticationDatabase admin -d test -o /mongodb/backup/

--备份test库下的log集合
$ mongodump   -uroot -proot123 --port 27017 --authenticationDatabase admin -d test -c log -o /mongodb/backup/

 --压缩备份
$ mongodump   -uroot -proot123 --port 27017 --authenticationDatabase admin -d abc -o /mongodb/backup/ --gzip
 mongodump   -uroot -proot123 --port 27017 --authenticationDatabase admin -o /mongodb/backup/ --gzip
$ mongodump   -uroot -proot123 --port 27017 --authenticationDatabase admin -d app -c vast -o /mongodb/backup/ --gzip

--全备中恢复单库
$ mongorestore   -uroot -proot123 --port 27017 --authenticationDatabase admin -d world1  /mongodb/backup/world

--全备中恢复单表
[mongod@db01 backup]$ mongorestore   -uroot -proot123 --port 27017 --authenticationDatabase admin -d a -c t1   /mongodb/backup/world/t5.bson.gz --gzip 

--drop表示恢复的时候把之前的集合drop掉(危险)
$ mongorestore  -uroot -proot123 --port 27017 --authenticationDatabase admin -d test --drop  /mongodb/backup/test
```

### 4、mongodump和mongorestore高级企业应用（--oplog）

>注意：这是replica set模式专用
>--oplog
> use oplog for taking a point-in-time snapshot

#### 1.oplog介绍

>在replica set中oplog是一个定容集合（capped collection），它的默认大小是磁盘空间的5%（可以通过--oplogSizeMB参数修改）.
>
>位于local库的db.oplog.rs，有兴趣可以看看里面到底有些什么内容。
>其中记录的是整个mongod实例一段时间内数据库的所有变更（插入/更新/删除）操作。
>当空间用完时新记录自动覆盖最老的记录。
>其覆盖范围被称作oplog时间窗口。需要注意的是，因为oplog是一个定容集合，所以时间窗口能覆盖的范围会因为你单位时间内的更新次数不同而变化。

#### 2.当前的oplog时间窗口预计值

```bash
mongod -f /mongodb/28017/conf/mongod.conf 
 mongod -f /mongodb/28018/conf/mongod.conf 
 mongod -f /mongodb/28019/conf/mongod.conf 
 mongod -f /mongodb/28020/conf/mongod.conf 

 use local 
 db.oplog.rs.find().pretty()


 	"ts" : Timestamp(1553597844, 1),
	"op" : "n"
	"o"  :

    "i": insert
    "u": update
    "d": delete
    "c": db cmd

test:PRIMARY> rs.printReplicationInfo()
configured oplog size:   1561.5615234375MB <--集合大小
log length start to end: 423849secs (117.74hrs) <--预计窗口覆盖时间
oplog first event time:  Wed Sep 09 2015 17:39:50 GMT+0800 (CST)
oplog last event time:   Mon Sep 14 2015 15:23:59 GMT+0800 (CST)
now:                     Mon Sep 14 2015 16:37:30 GMT+0800 (CST)
```

#### 3.oplog备份与恢复

>实现热备，在备份时使用--oplog选项
>注：为了演示效果我们在备份过程，模拟数据插入

##### 1）准备测试数据

```bash
use test

for(var i = 1 ;i < 100; i++) {
    db.foo.insert({a:i});
}

my_repl:PRIMARY> db.oplog.rs.find({"op":"i"}).pretty()
```

##### 2）oplog 配合mongodump实现热备

>作用介绍：--oplog 会记录备份过程中的数据变化。会以oplog.bson保存下来

```bash
mongodump --port 28017 --oplog -o /mongodb/backup
```

##### 3）恢复

```bash
mongorestore  --port 28017 --oplogReplay /mongodb/backup
```

#### 4.oplog高级应用类似于binlog应用

##### 1）背景

>每天0点全备，oplog恢复窗口为48小时
>某天，上午10点world.city 业务表被误删除。
>
>恢复思路：
>	0、停应用
>	2、找测试库
>	3、恢复昨天晚上全备
>	4、截取全备之后到world.city误删除时间点的oplog，并恢复到测试库
>	5、将误删除表导出，恢复到生产库

##### 2）恢复步骤

###### ①模拟原始数据

```bash
mongo --port 28017
use wo
for(var i = 1 ;i < 20; i++) {
    db.ci.insert({a: i});
}
```

###### ②全备

>--oplog功能:在备份同时,将备份过程中产生的日志进行备份
>文件必须存放在/mongodb/backup下,自动命令为oplog.bson

```bash
rm -rf /mongodb/backup/*
mongodump --port 28017 --oplog -o /mongodb/backup
```

###### ③再次模拟数据

```bash
db.ci1.insert({id:1})
db.ci2.insert({id:2})
```

###### ④上午10点：删除wo库下的ci表

>10:00时刻,误删除

```bash
db.ci.drop()
show tables;
```

###### ⑤备份现有的oplog.rs表

```bash
mongodump --port 28017 -d local -c oplog.rs  -o /mongodb/backup
```

###### ⑥截取oplog并恢复到drop之前的位置

```bash
更合理的方法：登陆到原数据库
[mongod@db03 local]$ mongo --port 28017
my_repl:PRIMARY> use local
db.oplog.rs.find({op:"c"}).pretty();
{
	"ts" : Timestamp(1600489082, 1),
	"t" : NumberLong(1),
	"h" : NumberLong(0),
	"v" : 2,
	"op" : "c",
	"ns" : "wo.$cmd",
	"ui" : UUID("875f2a41-57e3-4b6b-b738-469fad032b18"),
	"o2" : {
		"numRecords" : 19
	},
	"wall" : ISODate("2020-09-19T04:18:02.717Z"),
	"o" : {
		"drop" : "ci"
	}
}

获取到oplog误删除时间点位置:
1600489082, 1
```

###### ⑦恢复备份+应用oplog

```bash
[mongod@db03 backup]$ cd /mongodb/backup/local/
[mongod@db03 local]$ ls
oplog.rs.bson  oplog.rs.metadata.json

[mongod@db03 local]$ cp oplog.rs.bson ../oplog.bson 
rm -rf /mongodb/backup/local/

mongorestore --port 28017  --oplogReplay --oplogLimit "1600489082:1"  --drop   /mongodb/backup/
```

## 八、分片集群的备份思路

### 1、需要备份的数据

>shard、configserver 

### 2、痛点

>chunk 迁移 ，关闭或者调整balancer时间窗口
>备份出来的数据时间不一致。

### 3、备份方式

#### 1.Ops Manager

>监控免费
>备份、管理、审计是商用功能

#### 2.脚本

>balancer  关闭---> 
>同一时刻config、shard其中一个节点脱离集群--->
>开始备份节点数据 --->
>把节点恢复到集群