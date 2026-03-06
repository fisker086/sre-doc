**登录规范**

```bash
redis-cli -h IP -p PORT --raw
  -h		# 主机IP
  -p		# 端口
  --raw 	# 避免中文乱码
```



## 一、string字符串数据结构

### 1、特点

```bash
key value 
num 10
```



### 2、使用场景

```bash
	1、会话缓存(登录⽹站会话, 缓存的保留, 使⽤本地cookie, 或者memcache, redis记录hash值, 存 键值对) 
	2、计数器(直播类平台点击量, 关注量), 如果存到MySQL看不到实时场景 及时更新, 快速反馈
```



### 3、使用

#### 1.SET：添加数据

```bash
127.0.0.1:6379> set name 小武
OK
```



#### 2.GET：查看数据

```bash
127.0.0.1:6379> get name
小武
```



#### 3.SET：修改数据

```bash
127.0.0.1:6379> set name 待手绾青丝的笔记
OK
127.0.0.1:6379> get name
待手绾青丝的笔记
```



#### 4.DEL：删除

```bash
127.0.0.1:6379> del name
1
127.0.0.1:6379> get name

```



#### 5.EXISTS：判断一个key是否存在

```bash
127.0.0.1:6379> EXISTS name
0
127.0.0.1:6379> set name 待手绾青丝的笔记
OK
127.0.0.1:6379> EXISTS name
1
```



#### 6.过期时间

##### 1）TTL：查看过期时间

```bash
127.0.0.1:6379> TTL name
-1		# 永久不过期
127.0.0.1:6379> TTL name
-2		# 已经过期了
127.0.0.1:6379> TTL name
5		# 剩下5秒钟就过期了
```



##### 2）SETEX：设置一个单位以秒的过期的数据

> 设置一个name xiaowu过期时间10秒

```bash
127.0.0.1:6379> set name xiaowu ex 10
# 等价于
127.0.0.1:6379> setex name 10 xiaowu
```



##### 3）PSETEX：设置一个以毫秒为单位的过期时间

> 设置一个name xiaowu过期时间10000毫秒

```bash
127.0.0.1:6379> set name xiaowu px 10000
# 等价于
127.0.0.1:6379> PSETEX name 10000 xiaowu
```



#### 7.key存在性判断

##### 1）SETNX：一个key不存在则创建，存在则忽略

```bash
127.0.0.1:6379> del name
0
127.0.0.1:6379> set name xiaowu nx
OK
127.0.0.1:6379> get name
xiaowu
127.0.0.1:6379> set name wuxiaoyuan nx

127.0.0.1:6379> get name
xiaowu
# 等价于
127.0.0.1:6379> del name
1
127.0.0.1:6379> SETNX name xiaowu
1
127.0.0.1:6379> get name
xiaowu
127.0.0.1:6379> SETNX name wuxiaoyuan
0
127.0.0.1:6379> get name
xiaowu
```



##### 2）SET [key] [value] xx：一个key存在则更新，不存在则忽略

```bash
# 有则更新
127.0.0.1:6379> set name xiaowu
OK
127.0.0.1:6379> set name wuxiaoyuan xx
OK
127.0.0.1:6379> get name
wuxiaoyuan

# 无则跳过
127.0.0.1:6379> del name
1
127.0.0.1:6379> get name

127.0.0.1:6379> set name wuxiaoyuan xx

127.0.0.1:6379> get name

127.0.0.1:6379> 
```



#### 8.MSET：设置多个值

```bash
127.0.0.1:6379> MSET a 1 b 2 c 3
OK
127.0.0.1:6379> KEYS *			#生产上不建议使用，容易造成宕机
b
a
c
127.0.0.1:6379> get a
1
127.0.0.1:6379> get b
2
127.0.0.1:6379> get c
3
```



#### 9.GETSET：查看原value并更新value

```bash
127.0.0.1:6379> set name xiaowu
OK
127.0.0.1:6379> GETSET name wuxiaoyuan
xiaowu
127.0.0.1:6379> get name
wuxiaoyuan
```



#### 10.SETRANGE：按照下标去更新

```bash
127.0.0.1:6379> get name
wuxiaoyuan
127.0.0.1:6379> SETRANGE name 2 111
# 从第二个字符开始向后覆盖111
10
127.0.0.1:6379> get name
wu111oyuan
```



#### 11.获取多个key值

```bash
127.0.0.1:6379> KEYS *	#查看所有的key
b
name
a
c
127.0.0.1:6379> MGET get a

1
127.0.0.1:6379> MGET get name	#查看key的所有的值

wu111oyuan
```



#### 12.截取

```bash
127.0.0.1:6379> get name
wu111oyuan
127.0.0.1:6379> GETRANGE name 3 -1
11oyuan
127.0.0.1:6379> 

# -1 表示最后一个字符， -2 表示倒数第二个，以此类推。3表示第三个。遵循左开右闭的原则
```



#### 13.计数

> redis当中的计数器是具有原子性的，初始值是0，默认步长为1

##### 1）INCR：递增

```bash
127.0.0.1:6379> INCR num
1
127.0.0.1:6379> INCR num
2
127.0.0.1:6379> INCR num
3
127.0.0.1:6379> INCR num
4
127.0.0.1:6379> INCR num
5
```



##### 2）DECR：递减

```bash
127.0.0.1:6379> DECR num
4
127.0.0.1:6379> DECR num
3
127.0.0.1:6379> DECR num
2
127.0.0.1:6379> DECR num
1
127.0.0.1:6379> DECR num
0
127.0.0.1:6379> DECR num
-1
127.0.0.1:6379> DECR num
-2
```



##### 3）INCRby：指定增加的步长

```bash
127.0.0.1:6379> INCRBY num 2
0
127.0.0.1:6379> INCRBY num 2
2
127.0.0.1:6379> INCRBY num 2
4
127.0.0.1:6379> INCRBY num 2
6
127.0.0.1:6379> INCRBY num 2
8
127.0.0.1:6379> INCRBY num 2
10
```



##### 4）DECRBY：指定递减的步长

```bash
127.0.0.1:6379> DECRBY num 2
8
127.0.0.1:6379> DECRBY num 2
6
127.0.0.1:6379> DECRBY num 2
4
127.0.0.1:6379> DECRBY num 2
2
127.0.0.1:6379> DECRBY num 2
0
```



#### 14.APPEND：追加

```bash
127.0.0.1:6379> set name xiaowu
OK
127.0.0.1:6379> get name
xiaowu
127.0.0.1:6379> APPEND name 666
9
127.0.0.1:6379> get name
xiaowu666
```

