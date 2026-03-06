# Atlas管理操作

## 一、连接管理接口

```bash
mysql -uuser -ppwd -h127.0.0.1 -P2345
mysql -uuser -ppwd -h 10.0.0.53 -P2345
```



## 二、打印帮助

```bash
db03 [(none)]>select * from help;
+----------------------------+---------------------------------------------------------+
| command                    | description                                             |
+----------------------------+---------------------------------------------------------+
| SELECT * FROM help         | shows this help                                         |
| SELECT * FROM backends     | lists the backends and their state                      |
| SET OFFLINE $backend_id    | offline backend server, $backend_id is backend_ndx's id |
| SET ONLINE $backend_id     | online backend server, ...                              |
| ADD MASTER $backend        | example: "add master 127.0.0.1:3306", ...               |
| ADD SLAVE $backend         | example: "add slave 127.0.0.1:3306", ...                |
| REMOVE BACKEND $backend_id | example: "remove backend 1", ...                        |
| SELECT * FROM clients      | lists the clients                                       |
| ADD CLIENT $client         | example: "add client 192.168.1.2", ...                  |
| REMOVE CLIENT $client      | example: "remove client 192.168.1.2", ...               |
| SELECT * FROM pwds         | lists the pwds                                          |
| ADD PWD $pwd               | example: "add pwd user:raw_password", ...               |
| ADD ENPWD $pwd             | example: "add enpwd user:encrypted_password", ...       |
| REMOVE PWD $pwd            | example: "remove pwd user", ...                         |
| SAVE CONFIG                | save the backends to config file                        |
| SELECT VERSION             | display the version of Atlas                            |
+----------------------------+---------------------------------------------------------+


#常用命令
SELECT * FROM help		#查看帮助
SELECT * FROM backends		#查看后端节点状态
SET OFFLINE $backend_id 	#临时下线某个jiedian
SET ONLINE $backend_id 		#上线某个节点

ADD MASTER $backend		#添加主节点		add master 127.0.0.1:3306
ADD SLAVE $backend		#添加从库节点		add slave 127.0.0.1:3306
REMOVE BACKEND $backend_id	#移除某个节点		remove backend 1

#用户管理
SELECT * FROM pwds		#查看当前用户信息
ADD PWD $pwd		#添加一个管理用户 	add pwd user:raw_password（明文密码）
ADD ENPWD $pwd		#添加一个管理用户	add enpwd user:encrypted_password（密文密码）
	获取密文密码方式
		[root@db03 ~]# /usr/local/mysql-proxy/bin/encrypt 123
        3yb5jEku5h4=
REMOVE PWD $pwd		#移除某个用户

SAVE CONFIG		#当前状态保存到配置文件当中

```



## 三、查询后端所有节点信息

```bash
db03 [(none)]>select * from backends;
+-------------+----------------+-------+------+
| backend_ndx | address        | state | type |
+-------------+----------------+-------+------+
|           1 | 10.0.0.55:3306 | up    | rw   |
|           2 | 10.0.0.51:3306 | up    | ro   |
|           3 | 10.0.0.53:3306 | up    | ro   |
+-------------+----------------+-------+------+
```



## 四、动态添加删除节点

```bash
REMOVE BACKEND 3;
```



## 五、动态添加节点

```bash
ADD SLAVE 10.0.0.53:3306;
```



## 六、保存配置到配置文件

```bash
SAVE CONFIG;
```



## 七、自动分表

**介绍**

```bash
	使用Atlas的分表功能时，首先需要在配置文件test.cnf设置tables参数。
	tables参数设置格式：数据库名.表名.分表字段.子表数量，
比如：
	你的数据库名叫school，表名叫stu，分表字段叫id，总共分为2张表，那么就写为school.stu.id.2，如果还有其他的分表，以逗号分隔即可。
```



## 八、关于读写分离建议

```bash
MySQL-Router    ---> MySQL官方
ProxySQL         --->Percona
Maxscale         ---> MariaDB
```

