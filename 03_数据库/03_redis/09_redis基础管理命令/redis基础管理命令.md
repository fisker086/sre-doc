# Redis基础管理命令

## 一、key操作

### 1、KEYS * ：查询redis中所有键的名字

>⽣产中禁⽤[可能键值对很多, redis会崩溃, 耗费资源, ⽽且获 取不到信息] 

```bash
127.0.0.1:6379> KEYS *
myzety
mysetd
num
mysetb
mysetc
a
myset
myseta
```



**支持简单正则**

```bash
127.0.0.1:6379> KEYS m*
myzety
mysetd
mysetb
mysetc
myset
myseta
127.0.0.1:6379> KEYS *d
mysetd
127.0.0.1:6379> KEYS *e*
myzety
mysetd
mysetb
mysetc
myset
myseta
```



### 2、TYPE key判断键值类型

```bash
127.0.0.1:6379> TYPE num
set
127.0.0.1:6379> TYPE myset
set
127.0.0.1:6379> TYPE a
string
127.0.0.1:6379> TYPE mysety
none
```



### 3、RANDOMKEY：随机返回一个key

```bash
127.0.0.1:6379> RANDOMKEY
mysetd
```



### 4、DEL：删除一个key

```bash
127.0.0.1:6379> DEL a
1
```



### 5、EXISTS：判断一个key是否存在

> 1：存在，0：不存在。做判断, 如果多次没有发现, 就从MySQL调取出来缓冲到redis 

```bash
127.0.0.1:6379> EXISTS num
1
127.0.0.1:6379> EXISTS abc
0
```



### 6、RENAME：重命名一个key

```bash
127.0.0.1:6379> RENAME num nums
OK
127.0.0.1:6379> KEYS *
myzety
mysetd
mysetb
mysetc
nums
myset
myseta
```



## 二、过期时间

### 1、TTL：查看过期时间，单位秒

> PTTL：查看过期时间，单位毫秒
>
> 1秒=1000毫秒

```bash
127.0.0.1:6379> TTL myset
-1	# -1代表永不过期
127.0.0.1:6379> PTTL myset
-1
```



### 2、EXPIRE：控制键值对过期时间，单位秒

> PEXPIRE：使⽤毫秒设置过期时间

```bash
127.0.0.1:6379> EXPIRE myseta 1000
1
127.0.0.1:6379> TTL myseta
991

# 毫秒
127.0.0.1:6379> PEXPIRE myseta 10000000000000000
1
127.0.0.1:6379> TTL myseta
9999999999995
127.0.0.1:6379> PTTL myseta
9999999999990695
127.0.0.1:6379> 
```



### 3、PERSIST：设置永不过期

```bash
127.0.0.1:6379> PERSIST myseta
1
127.0.0.1:6379> TTL myseta
-1
127.0.0.1:6379> PTTL myseta
-1
```



## 三、Redis服务相关命令

### 1、INFO：查看状态命令

>相当于MySQL show status, 内存关注⽐较多; info memory; info replication; info cpu

```bash
127.0.0.1:6379> INFO cpu
# CPU
used_cpu_sys:9.580034
used_cpu_user:5.005455
used_cpu_sys_children:0.610758
used_cpu_user_children:0.026063
127.0.0.1:6379> INFO memory


# Memory
used_memory:885912
used_memory_human:865.15K
used_memory_rss:8200192
used_memory_rss_human:7.82M
used_memory_peak:906480
used_memory_peak_human:885.23K
used_memory_peak_perc:97.73%
used_memory_overhead:843744
used_memory_startup:802360
used_memory_dataset:42168
used_memory_dataset_perc:50.47%
allocator_allocated:903320
allocator_active:1171456
allocator_resident:3534848
total_system_memory:2076565504
total_system_memory_human:1.93G
used_memory_lua:37888
used_memory_lua_human:37.00K
used_memory_scripts:0
used_memory_scripts_human:0B
number_of_cached_scripts:0
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
allocator_frag_ratio:1.30
allocator_frag_bytes:268136
allocator_rss_ratio:3.02
allocator_rss_bytes:2363392
rss_overhead_ratio:2.32
rss_overhead_bytes:4665344
mem_fragmentation_ratio:9.71
mem_fragmentation_bytes:7355296
mem_not_counted_for_evict:0
mem_replication_backlog:0
mem_clients_slaves:0
mem_clients_normal:41008
mem_aof_buffer:0
mem_allocator:jemalloc-5.1.0
active_defrag_running:0
lazyfree_pending_objects:0


127.0.0.1:6379> INFO replication
# Replication
role:master
connected_slaves:0
master_replid:b59128be81acee9750032b9077cadc42b6292c1e
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
127.0.0.1:6379> 

```



### 2、CLIENT LIST：查看当前正在连接的会话

**先新连接个会话**

```bash
[root@xiaowu ~]# redis-cli -h 172.16.1.130 --raw
172.16.1.130:6379> AUTH 123
OK
```

```bash
127.0.0.1:6379> CLIENT LIST
id=4 addr=127.0.0.1:48952 fd=8 name= age=11617 idle=3023 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 obl=0 oll=0 omem=0 tot-mem=20504 events=r cmd=keys user=default
id=5 addr=127.0.0.1:48954 fd=9 name= age=1556 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=26 qbuf-free=32742 argv-mem=10 obl=0 oll=0 omem=0 tot-mem=61466 events=r cmd=client user=default
127.0.0.1:6379> CLIENT LIST
id=4 addr=127.0.0.1:48952 fd=8 name= age=11768 idle=3174 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 obl=0 oll=0 omem=0 tot-mem=20504 events=r cmd=keys user=default
id=5 addr=127.0.0.1:48954 fd=9 name= age=1707 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=26 qbuf-free=32742 argv-mem=10 obl=0 oll=0 omem=0 tot-mem=61466 events=r cmd=client user=default
id=6 addr=172.16.1.130:46068 fd=10 name= age=76 idle=69 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 obl=0 oll=0 omem=0 tot-mem=20504 events=r cmd=auth user=default
127.0.0.1:6379> 
```



### 3、CLIENT KILL：强制关闭会话

>轻易不要做, 除⾮卡死

```bash
127.0.0.1:6379> CLIENT KILL 172.16.1.130:46068
OK
```

**会话端，退出登录状态，需要重新登录**

```bash
172.16.1.130:6379> AUTH 123
OK
172.16.1.130:6379> KEYS *
myzety
mysetd
mysetb
mysetc
nums
myset
myseta

# kill后 

172.16.1.130:6379> KEYS *
NOAUTH Authentication required.
172.16.1.130:6379> AUTH 123
OK
172.16.1.130:6379> KEYS *
myzety
mysetd
mysetb
mysetc
nums
myset
myseta
```



### 4、DBSIZE：判断键值对个数

```bash
127.0.0.1:6379> DBSIZE
7
```



### 5、CONFIG RESETSTAT：重置统计

> 命令用于重置 [INFO](https://www.runoob.com/redis/server-info.html) 命令中的某些统计数据，包括：

```bash
Keyspace hits (键空间命中次数)
Keyspace misses (键空间不命中次数)
Number of commands processed (执行命令的次数)
Number of connections received (连接服务器的次数)
Number of expired keys (过期key的数量)
Number of rejected connections (被拒绝的连接数量)
Latest fork(2) time(最后执行 fork(2) 的时间)
The aof_delayed_fsync counter(aof_delayed_fsync 计数器的值)
```



重置前

```bash
redis 127.0.0.1:6379> INFO
# Server
redis_version:2.5.3
redis_git_sha1:d0407c2d
redis_git_dirty:0
arch_bits:32
multiplexing_api:epoll
gcc_version:4.6.3
process_id:11095
run_id:ef1f6b6c7392e52d6001eaf777acbe547d1192e2
tcp_port:6379
uptime_in_seconds:6
uptime_in_days:0
lru_clock:1205426

# Clients
connected_clients:1
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0

# Memory
used_memory:331076
used_memory_human:323.32K
used_memory_rss:1568768
used_memory_peak:293424
used_memory_peak_human:286.55K
used_memory_lua:16384
mem_fragmentation_ratio:4.74
mem_allocator:jemalloc-2.2.5

# Persistence
loading:0
aof_enabled:0
changes_since_last_save:0
bgsave_in_progress:0
last_save_time:1333260015
last_bgsave_status:ok
bgrewriteaof_in_progress:0

# Stats
total_connections_received:1
total_commands_processed:0
instantaneous_ops_per_sec:0
rejected_connections:0
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0

# Replication
role:master
connected_slaves:0

# CPU
used_cpu_sys:0.01
used_cpu_user:0.00
used_cpu_sys_children:0.00
used_cpu_user_children:0.00

# Keyspace
db0:keys=20,expires=0
```



**执行命令**

```bash
redis 127.0.0.1:6379> CONFIG RESETSTAT
OK
```



**重置后**

```bash
redis 127.0.0.1:6379> INFO
# Server
redis_version:2.5.3
redis_git_sha1:d0407c2d
redis_git_dirty:0
arch_bits:32
multiplexing_api:epoll
gcc_version:4.6.3
process_id:11095
run_id:ef1f6b6c7392e52d6001eaf777acbe547d1192e2
tcp_port:6379
uptime_in_seconds:134
uptime_in_days:0
lru_clock:1205438

# Clients
connected_clients:1
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0

# Memory
used_memory:331076
used_memory_human:323.32K
used_memory_rss:1568768
used_memory_peak:330280
used_memory_peak_human:322.54K
used_memory_lua:16384
mem_fragmentation_ratio:4.74
mem_allocator:jemalloc-2.2.5

# Persistence
loading:0
aof_enabled:0
changes_since_last_save:0
bgsave_in_progress:0
last_save_time:1333260015
last_bgsave_status:ok
bgrewriteaof_in_progress:0

# Stats
total_connections_received:0
total_commands_processed:1
instantaneous_ops_per_sec:0
rejected_connections:0
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0

# Replication
role:master
connected_slaves:0

# CPU
used_cpu_sys:0.05
used_cpu_user:0.02
used_cpu_sys_children:0.00
used_cpu_user_children:0.00

# Keyspace
db0:keys=20,expires=0
```



### 6、CONFIG GET：命令用于获取 redis 服务的配置参数

```bash
127.0.0.1:6379> CONFIG GET databases
databases
16
127.0.0.1:6379> CONFIG GET bind
bind
127.0.0.1 172.16.1.130
```



### 7、CONFIG SET：动态临时修改服务配置

```bash
127.0.0.1:6379> CONFIG GET slowlog-max-len
slowlog-max-len
128
127.0.0.1:6379> CONFIG SET slowlog-max-len 130
OK
127.0.0.1:6379> CONFIG GET slowlog-max-len
slowlog-max-len
130
```



### 8、CONFIG REWRITE：当前redis写入对应的配置文件

**修改前**

```bash
[root@xiaowu ~]# cat /usr/local/redis/conf/redis.conf|grep slowlog-max-len
slowlog-max-len 128
```

**修改**

```bash
127.0.0.1:6379> CONFIG REWRITE
```

**修改后**

```bash
[root@xiaowu ~]# cat /usr/local/redis/conf/redis.conf|grep slowlog-max-len
slowlog-max-len 130
```



### 9、FLUSHALL：清空整个 Redis 服务器的数据(删除所有数据库的所有 key )。

```bash
127.0.0.1:6379> KEYS *
myzety
mysetd
mysetb
mysetc
nums
myset
myseta
127.0.0.1:6379> FLUSHALL
OK
127.0.0.1:6379> KEYS *

```



### 10、FLUSHDB：清空当前库键值对，非常危险

```bash
127.0.0.1:6379> FLUSHDB
OK
# 没数据做演示了
```



### 11、SELECT：进入指定库

>redis默认16个库0-15, 当前库就是0号, select 0进⼊0号, 每⼀个库都是单独隔离空间 

```bash
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> SELECT 1 
```



### 12、MONITOR：实时监控命令

>监控实时命令, 默认记录到屏幕上, 只记录命令 

```bash
127.0.0.1:6379> MONITOR
OK
```

**客户端操作**

```bash
172.16.1.130:6379> KEYS *

172.16.1.130:6379> 
```

**显示**

```bash
127.0.0.1:6379> MONITOR
OK
1619972259.019898 [0 172.16.1.130:46070] "KEYS" "*"
```



**将监控数据指向文件夹**

```bash
redis-cli -h IP -p 6379 monitor >> /tm p/monitor.log &
```

ps：mysql做审计可以使⽤general log, 或者atlas来做; 企业版也有audit 



### 13、SHUTDOWN：关闭redis服务

```bash
127.0.0.1:6379> SHUTDOWN
not connected> 

```

