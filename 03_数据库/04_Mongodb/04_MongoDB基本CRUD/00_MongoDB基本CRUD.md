# MongoDB 基本CRUD

## 一、帮助和通用方法

### 1、获取帮助

#### 1.help

```bash
help
> help
	db.help()                    help on db methods
	db.mycoll.help()             help on collection methods
	sh.help()                    sharding helpers
	rs.help()                    replica set helpers
```

#### 2.db.help();

```bash
db.help();
DB methods:
	db.adminCommand(nameOrDocument) - switches to 'admin' db, and runs command [just calls db.runCommand(...)]
	db.aggregate([pipeline], {options}) - performs a collectionless aggregation on this database; returns a cursor
	db.auth(username, password)
	db.cloneDatabase(fromhost) - will only function with MongoDB 4.0 and below
```

#### 3.db.t1.help();

```bash
db.t1.help();
DBCollection help
	db.t1.find().help() - show DBCursor help
	db.t1.bulkWrite( operations, <optional params> ) - bulk execute write operations, optional parameters are: w, wtimeout, j
	db.t1.count( query = {}, <optional params> ) - count the number of documents that matches the query, optional parameters are: limit, skip, hint, maxTimeMS
```

#### 4.使用键盘的tab键可以提示

```bash
db.[TAB][TAB]
db.t1.[TAB][TAB]
```

### 2、常用操作

#### 1.查看当前db版本

```bash
db.version()
```

#### 2.显示当前数据库

```bash
db
db.getName()
```

#### 3.查询所有数据库

```bash
show dbs
```

#### 4.切换数据库

```bash
use local
```

#### 5.显示当前数据库状态

```bash
db.stats()
```

#### 6.查看当前数据库的连接机器地址

```bash
db.getMongo()
```

#### 7.指定数据库进行连接：（默认连接本机test数据库）

```bash
mongo 192.168.1.24/admin
```

### 3、库和表的操作

#### 1.建库

```bash
use test
```

#### 2.删除

```bash
db.dropDatabase()
```

#### 3.创建集合（表）

##### 1）方法一

```bash
use app

app> db.createCollection('a')
app> db.createCollection('b')
```

##### 2）方法二

```bash
admin> use app
switched to db app

app> db.c.insert({username:"mongodb"})
WriteResult({ "nInserted" : 1 })
app> show collections

app> db.c.find()
{ "_id" : ObjectId("5743c9a9bf72d9f7b524713d"), "username" : "mongodb" }
```

#### 4.删除集合

```bash
app> use app
switched to db app
app> db.log.drop()
```

#### 5.重命名集合

```bash
db.log.renameCollection("log1")
```

## 二、使用 insert 完成插入操作

```bash
操作格式：
db.<集合>.insertOne(<JSON对象>)
db.<集合>.insertMany([<JSON 1>, <JSON 2>, …<JSON n>])


示例：
db.fruit.insertOne({name: "apple"})
db.fruit.insertMany([
    {name: "apple"},
    {name: "pear"},
    {name: "orange"}
])


批量插入数据： 
for(i=0;i<10000;i++){ db.log.insert({"uid":i,"name":"mongodb","age":6,"date":new Date()}); }
```

## 三、使用 find 查询文档

### 1、简单使用

```bash
# 关于 find:
find 是 MongoDB 中查询数据的基本指令，相当于 SQL 中的 SELECT 。
find 返回的是游标。
# find 示例：
db.movies.find( { "year" : 1975 } ) //单条件查询

db.movies.find( { "year" : 1989, "title" : "Batman" } ) //多条件and查询

db.movies.find( { $and : [ {"title" : "Batman"}, { "category" : "action" }] } ) // and的另一种形式

db.movies.find( { $or: [{"year" : 1989}, {"title" : "Batman"}] } ) //多条件or查询

db.movies.find( { "title" : /^B/} ) //按正则表达式查找
```

### 2、查询条件对照表

| SQL    | MSQL           |
| ------ | -------------- |
| a = 1  | {a: 1}         |
| a <> 1 | {a: {$ne: 1}}  |
| a > 1  | {a: {$gt: 1}}  |
| a >= 1 | {a: {$gte: 1}} |
| a < 1  | {a: {$lt: 1}}  |
| a <= 1 | {a: {$lte: 1}} |

### 3、查询逻辑对照表

| SQL             | MSQL                                   |
| --------------- | -------------------------------------- |
| a = 1 AND b = 1 | {a: 1, b: 1}或{$and: [{a: 1}, {b: 1}]} |
| a = 1 OR b = 1  | {$or: [{a: 1}, {b: 1}]}                |
| a IS NULL       | {a: {$exists: false}}                  |
| a IN (1, 2, 3)  | {a: {$in: [1, 2, 3]}}                  |

### 4、查询逻辑运算符

```bash
● $lt: 存在并小于
● $lte: 存在并小于等于
● $gt: 存在并大于
● $gte: 存在并大于等于
● $ne: 不存在或存在但不等于
● $in: 存在并在指定数组中
● $nin: 不存在或不在指定数组中
● $or: 匹配两个或多个条件中的一个
● $and: 匹配全部条件
```

### 5、使用 find 搜索子文档

```bash
find 支持使用“field.sub_field”的形式查询子文档。假设有一个文档：

db.fruit.insertOne({
    name: "apple",
    from: {
        country: "China",
        province: "Guangdong"
    }
})

正确写法： 
db.fruit.find( { "from.country" : "China" } )
```

### 6、使用 find 搜索数组

```bash
find 支持对数组中的元素进行搜索。假设有一个文档：

db.fruit.insert([
    { "name" : "Apple", color: ["red", "green" ] },
    { "name" : "Mango", color: ["yellow", "green"] }
])

查看单个条件： 
db.fruit.find({color: "red"})

查询多个条件： 
db.fruit.find({$or: [{color: "red"}, {color: "yellow"}]} )
```

### 7、使用 find 搜索数组中的对象

```bash
考虑以下文档，在其中搜索

db.movies.insertOne( {
    "title" : "Raiders of the Lost Ark",
    "filming_locations" : [ 
        { "city" : "Los Angeles", "state" : "CA", "country" : "USA" },
        { "city" : "Rome", "state" : "Lazio", "country" : "Italy" },
        { "city" : "Florence", "state" : "SC", "country" : "USA" }
    ]
})

// 查找城市是 Rome 的记录
db.movies.find({"filming_locations.city": "Rome"})
```

```bash
在数组中搜索子对象的多个字段时，如果使用 $elemMatch，它表示必须是同一个子对象满足多个条件。

考虑以下两个查询：
db.getCollection('movies').find({
    "filming_locations.city": "Rome",
    "filming_locations.country": "USA"
})

db.movies.insertOne( {
    "title" : "11111",
    "filming_locations" : [ 
        { "city" : "bj", "state" : "CA", "country" : "CHN" },
        { "city" : "Rome", "state" : "Lazio", "country" : "Italy" },
        { "city" : "tlp", "state" : "SC", "country" : "USA" }
    ] 
})


db.getCollection('movies').find({
    "filming_locations": {
        $elemMatch:{"city":"bj", "country": "CHN"}
    }
})
```

### 8、控制 find 返回的字段

```bash
find 可以指定只返回指定的字段；
● _id字段必须明确指明不返回，否则默认返回；
● 在 MongoDB 中我们称这为投影（projection）；
● db.movies.find({},{"_id":0, title:1})
```

![image-20240613153720194](./image-20240613153720194.png)

## 四、使用 remove 删除文档

```bash
remove 命令需要配合查询条件使用；
● 匹配查询条件的的文档会被删除；
● 指定一个空文档条件会删除所有文档；
● 以下示例：
db.testcol.remove( { a : 1 } ) // 删除a 等于1的记录
db.testcol.remove( { a : { $lt : 5 } } ) // 删除a 小于5的记录
db.testcol.remove( { } ) // 删除所有记录
db.testcol.remove() //报错
```

## 五、使用 update 更新文档

### 1、简单使用

```bash
Update 操作执行格式：db.<集合>.update(<查询条件>, <更新字段>)
● 以以下数据为例：
db.fruit.insertMany([
    {name: "apple"},
    {name: "pear"},
    {name: "orange"}
])

db.fruit.updateOne({name: "apple"}, {$set: {from: "China"}})
```

![image-20240613154354496](./image-20240613154354496.png)

### 2、注意事项

```bash
● 使用 updateOne 表示无论条件匹配多少条记录，始终只更新第一条；
● 使用 updateMany 表示条件匹配多少条就更新多少条；
● updateOne/updateMany 方法要求更新条件部分必须具有以下之一，否则将报错：
	• $set/$unset
	• $push/$pushAll/$pop
	• $pull/$pullAll
	• $addToSet
● // 报错
db.fruit.updateOne({name: "apple"}, {from: "China"})
```

### 3、使用 update 更新数组

```bash
● $push: 增加一个对象到数组底部
● $pushAll: 增加多个对象到数组底部
● $pop: 从数组底部删除一个对象
● $pull: 如果匹配指定的值，从数组中删除相应的对象
● $pullAll: 如果匹配任意的值，从数据中删除相应的对象
● $addToSet: 如果不存在则增加一个值到数组
```

## 六、使用 drop 删除集合

```bash
● 使用 db.<集合>.drop() 来删除一个集合
● 集合中的全部文档都会被删除
● 集合相关的索引也会被删除
db.colToBeDropped.drop()
```

## 七、 使用 dropDatabase 删除数据库

```bash
● 使用 db.dropDatabase() 来删除数据库
● 数据库相应文件也会被删除，磁盘空间将被释放
use tempDB
db.dropDatabase()
show collections // No collections
show dbs // The db is gone
```

## 八、通过python操作mongodb

### 1、安装 Python MongoDB 驱动程序

```bash
在 Python 中使用 MongoDB 之前必须先安装用于访问数据库的驱动程序：
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum install -y python3
pip3 install pymongo
在 python 交互模式下导入 pymongo，检查驱动是否已正确安装：
import pymongo 
pymongo.version
```

### 2、创建连接

#### 1.确定连接串

```bash
使用驱动连接到 MongoDB 集群只需要指定 MongoDB 连接字符串即可。其基本格式可以参考文档: Connection String URI Format 最简单的形式是
mongodb://数据库服务器主机地址：端口号
如：mongodb://127.0.0.1:27017
```

#### 2.初始化数据库连接

```bash
from pymongo import MongoClient 
uri = "mongodb://root:root123@10.0.0.51:27017"
client = MongoClient(uri) 
client
```

#### 3.数据库操作：插入数据

```bash
初始化数据库和集合
db = client["eshop"]
user_coll = db["users"]
插入一条新的用户数据
new_user = {"username": "nina", "password": "xxxx", "email": 
"123456@qq.com "}
result = user_coll.insert_one(new_user)
result
```
