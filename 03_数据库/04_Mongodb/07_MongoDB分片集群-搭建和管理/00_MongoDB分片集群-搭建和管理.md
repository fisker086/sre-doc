# MongoDB分片集群-搭建和管理

## 一、分片规划

>10个实例：38017-38026
>(1)configserver:
>3台构成的复制集（1主两从，不支持arbiter）38018-38020
>
>(2)shard节点：
>sh1：38021-23    （1主两从，其中一个节点为arbiter，复制集名字sh1）
>sh2：38024-26    （1主两从，其中一个节点为arbiter，复制集名字sh2）
>
>(3)mongos
>38017

## 二、配置过程

### 1、shard复制集配置

#### 1.创建数据存储目录

```bash
mkdir -p /mongodb/38021/conf  /mongodb/38021/log  /mongodb/38021/data
mkdir -p /mongodb/38022/conf  /mongodb/38022/log  /mongodb/38022/data
mkdir -p /mongodb/38023/conf  /mongodb/38023/log  /mongodb/38023/data
mkdir -p /mongodb/38024/conf  /mongodb/38024/log  /mongodb/38024/data
mkdir -p /mongodb/38025/conf  /mongodb/38025/log  /mongodb/38025/data
mkdir -p /mongodb/38026/conf  /mongodb/38026/log  /mongodb/38026/data
```

#### 2.sh1复制集配置准备

>vi /mongodb/38021/conf/mongodb.conf 

```bash
systemLog:
  destination: file
  path: /mongodb/38021/log/mongodb.log   
  logAppend: true
storage:
  journal:
    enabled: true
  dbPath: /mongodb/38021/data
  directoryPerDB: true
  #engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true
net:
  bindIp: 10.0.0.51,127.0.0.1
  port: 38021
replication:
  oplogSizeMB: 2048
  replSetName: sh1
sharding:
  clusterRole: shardsvr
processManagement: 
  fork: true
```

```bash
cp  /mongodb/38021/conf/mongodb.conf  /mongodb/38022/conf/
cp  /mongodb/38021/conf/mongodb.conf  /mongodb/38023/conf/

sed 's#38021#38022#g' /mongodb/38022/conf/mongodb.conf -i
sed 's#38021#38023#g' /mongodb/38023/conf/mongodb.conf -i
```

#### 3.sh2复制集配置准备

>vi /mongodb/38024/conf/mongodb.conf 

```bash
systemLog:
  destination: file
  path: /mongodb/38024/log/mongodb.log   
  logAppend: true
storage:
  journal:
    enabled: true
  dbPath: /mongodb/38024/data
  directoryPerDB: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true
net:
  bindIp: 10.0.0.51,127.0.0.1
  port: 38024
replication:
  oplogSizeMB: 2048
  replSetName: sh2
sharding:
  clusterRole: shardsvr
processManagement: 
  fork: true
```

```bash
cp  /mongodb/38024/conf/mongodb.conf  /mongodb/38025/conf/
cp  /mongodb/38024/conf/mongodb.conf  /mongodb/38026/conf/

sed 's#38024#38025#g' /mongodb/38025/conf/mongodb.conf -i
sed 's#38024#38026#g' /mongodb/38026/conf/mongodb.conf -i
```

#### 4.所有节点启动

```bash
mongod -f  /mongodb/38021/conf/mongodb.conf 
mongod -f  /mongodb/38022/conf/mongodb.conf 
mongod -f  /mongodb/38023/conf/mongodb.conf 
mongod -f  /mongodb/38024/conf/mongodb.conf 
mongod -f  /mongodb/38025/conf/mongodb.conf 
mongod -f  /mongodb/38026/conf/mongodb.conf  
```

#### 5.sh1和sh2复制集部署

```bash
mongo --port 38021

use  admin
config = {_id: 'sh1', members: [
                          {_id: 0, host: '10.0.0.51:38021'},
                          {_id: 1, host: '10.0.0.51:38022'},
                          {_id: 2, host: '10.0.0.51:38023',"arbiterOnly":true}]
           }

rs.initiate(config)


mongo --port 38024 

use admin
config = {_id: 'sh2', members: [
                          {_id: 0, host: '10.0.0.51:38024'},
                          {_id: 1, host: '10.0.0.51:38025'},
                          {_id: 2, host: '10.0.0.51:38026',"arbiterOnly":true}]
           }

rs.initiate(config)
```

### 2、config节点配置

#### 1.目录创建

```bash
mkdir -p /mongodb/38018/conf  /mongodb/38018/log  /mongodb/38018/data
mkdir -p /mongodb/38019/conf  /mongodb/38019/log  /mongodb/38019/data
mkdir -p /mongodb/38020/conf  /mongodb/38020/log  /mongodb/38020/data
```

#### 2.修改配置文件

>vi /mongodb/38018/conf/mongodb.conf

```bash
systemLog:
  destination: file
  path: /mongodb/38018/log/mongodb.conf
  logAppend: true
storage:
  journal:
    enabled: true
  dbPath: /mongodb/38018/data
  directoryPerDB: true
  #engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true
net:
  bindIp: 10.0.0.51,127.0.0.1
  port: 38018
replication:
  oplogSizeMB: 2048
  replSetName: configReplSet
sharding:
  clusterRole: configsvr
processManagement: 
  fork: true
```

```bash
cp /mongodb/38018/conf/mongodb.conf /mongodb/38019/conf/
cp /mongodb/38018/conf/mongodb.conf /mongodb/38020/conf/

sed 's#38018#38019#g' /mongodb/38019/conf/mongodb.conf -i
sed 's#38018#38020#g' /mongodb/38020/conf/mongodb.conf -i
```

#### 3.启动节点

```bash
mongod -f /mongodb/38018/conf/mongodb.conf 
mongod -f /mongodb/38019/conf/mongodb.conf 
mongod -f /mongodb/38020/conf/mongodb.conf 
```

#### 4.配置复制集部署

```bash
mongo --port 38018

use  admin

 config = {_id: 'configReplSet', members: [
                          {_id: 0, host: '10.0.0.51:38018'},
                          {_id: 1, host: '10.0.0.51:38019'},
                          {_id: 2, host: '10.0.0.51:38020'}]
           }
rs.initiate(config) 
```

>注：configserver 可以是一个节点，官方建议复制集。configserver不能有arbiter。
>新版本中，要求必须是复制集。
>注：mongodb 3.4之后，虽然要求config server为replica set，但是不支持arbiter

### 3、mongos节点配置

#### 1.创建目录

```bash
mkdir -p /mongodb/38017/conf  /mongodb/38017/log 
```

#### 2.配置文件

>vi /mongodb/38017/conf/mongos.conf

```bash
systemLog:
  destination: file
  path: /mongodb/38017/log/mongos.log
  logAppend: true
net:
  bindIp: 10.0.0.51,127.0.0.1
  port: 38017
sharding:
  configDB: configReplSet/10.0.0.51:38018,10.0.0.51:38019,10.0.0.51:38020
processManagement: 
  fork: true
```

#### 3.启动mongos

```bash
mongos -f /mongodb/38017/conf/mongos.conf
```

### 4、分片集群操作

>连接到其中一个mongos（10.0.0.51），做以下配置

#### 1.连接到mongs的admin数据库

```bash
# su - mongod
$ mongo 10.0.0.51:38017/admin
```

#### 2.添加分片

```bash
db.runCommand( { addshard : "sh1/10.0.0.51:38021,10.0.0.51:38022,10.0.0.51:38023",name:"shard1"} )
db.runCommand( { addshard : "sh2/10.0.0.51:38024,10.0.0.51:38025,10.0.0.51:38026",name:"shard2"} )
```

#### 3.列出分片

```bash
mongos> db.runCommand( { listshards : 1 } )
```

#### 4.整体状态查看

```bash
mongos> sh.status();
```

## 三、使用分片集群

### 1、RANGE分片配置及测试

>test库下的vast大表进行手工分片

#### 1.激活数据库分片功能

```bash
mongo --port 38017 admin
admin>  ( { enablesharding : "数据库名称" } )

eg：
admin> db.runCommand( { enablesharding : "test" } )
```

#### 2.指定分片建对集合分片

```bash
eg：范围片键
--创建索引
use test
> db.vast.ensureIndex( { id: 1 } )

--开启分片
use admin
> db.runCommand( { shardcollection : "test.vast",key : {id: 1} } )
```

#### 3.集合分片验证

```bash
admin> use test

test> for(i=1;i<500000;i++){ db.vast.insert({"id":i,"name":"shenzheng","age":70,"date":new Date()}); }

test> db.vast.stats()
```

#### 4.分片结果测试

```bash
shard1:
mongo --port 38021
db.vast.count();


shard2:
mongo --port 38024
db.vast.count();
```

### 2、Hash分片

>对test库下的vast大表进行hash

#### 1.对于test开启分片功能

```bash
mongo --port 38017 admin
use admin
admin> db.runCommand( { enablesharding : "test" } )
```

#### 2.对于test库下的vast表建立hash索引

```bash
use test
test> db.vast.ensureIndex( { id: "hashed" } )
```

#### 3.开启分片

```bash
use admin
admin > sh.shardCollection( "test.vast", { id: "hashed" } )
```

#### 4.录入10w行数据测试

```bash
use test
for(i=1;i<100000;i++){ db.vast.insert({"id":i,"name":"shenzheng","age":70,"date":new Date()}); }
```

#### 5.hash分片结果测试

```bash
mongo --port 38021
use test
db.vast.count();

mongo --port 38024
use test
db.vast.count();
```

## 四、分片的管理

### 1、判断是否Shard集群

```bash
admin> db.runCommand({ isdbgrid : 1})
```

### 2、列出所有分片信息

```bash
admin> db.runCommand({ listshards : 1})
```

### 3、列出开启分片的数据库

```bash
admin> use config

config> db.databases.find( { "partitioned": true } )
或者：
config> db.databases.find() //列出所有数据库分片情况
```

### 4、查看分片的片键

```bash
config> db.collections.find().pretty()
{
	"_id" : "test.vast",
	"lastmodEpoch" : ObjectId("58a599f19c898bbfb818b63c"),
	"lastmod" : ISODate("1970-02-19T17:02:47.296Z"),
	"dropped" : false,
	"key" : {
		"id" : 1
	},
	"unique" : false
}
```

### 5、查看分片的详细信息

```bash
admin> db.printShardingStatus()
或
admin> sh.status()
```

### 6、删除分片节点（谨慎）

>注意：删除操作一定会立即触发blancer。

#### 1.确认blance是否在工作

```bash
sh.getBalancerState()
```

#### 2.删除shard2节点(谨慎)

```bash
mongos> db.runCommand( { removeShard: "shard2" } )
```

## 五、balancer操作

### 1、介绍

>mongos的一个重要功能，自动巡查所有shard节点上的chunk的情况，自动做chunk迁移。
>什么时候工作？
>1、自动运行，会检测系统不繁忙的时候做迁移
>2、在做节点删除的时候，立即开始迁移工作
>3、balancer只能在预设定的时间窗口内运行

### 2、有需要时可以关闭和开启blancer（备份的时候）

```bash
mongos> sh.stopBalancer()
mongos> sh.startBalancer()
```

### 3、自定义 自动平衡进行的时间段

>https://docs.mongodb.com/manual/tutorial/manage-sharded-cluster-balancer/#schedule-the-balancing-window

>// connect to mongos

```bash
use config
sh.setBalancerState( true )
db.settings.update({ _id : "balancer" }, { $set : { activeWindow : { start : "3:00", stop : "5:00" } } }, true )

sh.getBalancerWindow()
sh.status()
```