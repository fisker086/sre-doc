# redis哈希数据结构

## 一、特点

```bash
key 	value 
stu 	id:101 name:hu age:24
```



## 二、应用场景

```bash
数据库缓存(每⼀个表⾥每⼀⾏都是⼀个键值对)
```



## 三、使用

### 1、HSET：写入数据

```bash
127.0.0.1:6379> HSET people name xiaowu sex man add shanghai
3
```



### 2、HGET：获取数据

```bash
127.0.0.1:6379> HGET people name
xiaowu
127.0.0.1:6379> HGET people sex
man
```



### 3、HSET：修改数据

```bash
127.0.0.1:6379> HSET people name wuxiaoyuan
0
127.0.0.1:6379> HGET people name
wuxiaoyuan
```



### 4、HDEL：删除数据

```bash
127.0.0.1:6379> HDEL people name
1
127.0.0.1:6379> HGET people name

127.0.0.1:6379> 
```



### 5、DEL：删除所有数据

```bash
127.0.0.1:6379> DEL people
1
127.0.0.1:6379> HGET people sex

127.0.0.1:6379> HGET people add

127.0.0.1:6379> 
```



### 6、HMGET：获取多个值

```bash
127.0.0.1:6379> HSET people name wuxiaoyuan sex man add shanghai zhuji linyi
4
127.0.0.1:6379> HMGET people name add zhuji
wuxiaoyuan
shanghai
linyi
127.0.0.1:6379> 
```



### 7、HGETALL：获取所有key和value

```bash
127.0.0.1:6379> HGETALL people
name
wuxiaoyuan
sex
man
add
shanghai
zhuji
linyi
```



### 8、HKEYS：获取所有key

```bash
127.0.0.1:6379> HKEYS people
name
sex
add
zhuji
```



### 9、HVALS：获取所有的value

```bash
127.0.0.1:6379> HVALS people
wuxiaoyuan
man
shanghai
linyi
```



### 10、计数

#### 1）HINCRBY：增加

```bash
127.0.0.1:6379> HINCRBY people id 1
1
127.0.0.1:6379> HINCRBY people id 1
2
127.0.0.1:6379> HINCRBY people id 1
3
127.0.0.1:6379> hget people id
3
```



#### 2）HINCRBY：递减

```bash
127.0.0.1:6379> HINCRBY people id -1
2
127.0.0.1:6379> HINCRBY people id -1
1
127.0.0.1:6379> hget people id
1
```



#### 3）HINCRBYFLOAT：小数点计算

```bash
127.0.0.1:6379> HINCRBYFLOAT people num 10.9
43.6
127.0.0.1:6379> HINCRBYFLOAT people num 10.9
54.5
127.0.0.1:6379> HINCRBYFLOAT people num 10.9
65.4
127.0.0.1:6379> HINCRBYFLOAT people num -10.9
54.5
127.0.0.1:6379> HINCRBYFLOAT people num -10.9
43.6
127.0.0.1:6379> HINCRBYFLOAT people num -10.9
32.7
127.0.0.1:6379> 
```

**特殊情况**

```bash
127.0.0.1:6379> HINCRBYFLOAT people num 10.9
130.8
127.0.0.1:6379> HINCRBYFLOAT people num 10.9
141.7
127.0.0.1:6379> HINCRBYFLOAT people num 10.9
152.59999999999999999
127.0.0.1:6379> HINCRBYFLOAT people num 10.9
163.49999999999999999			#精度不够
127.0.0.1:6379> 
```



### 11、HLEN：获取数据对（field）长度

```bash
127.0.0.1:6379> HLEN people
6
```



### 12、HSTRLEN：获取某个字段的长度

```bash
127.0.0.1:6379> hget people name
wuxiaoyuan
127.0.0.1:6379> HSTRLEN people name
10
```



### 13、EXPIRE：设置过期时间

```bash
127.0.0.1:6379> EXPIRE people 12345
1
127.0.0.1:6379> TTL people
12334
127.0.0.1:6379> TTL people
12333
127.0.0.1:6379> TTL people
12333
```

