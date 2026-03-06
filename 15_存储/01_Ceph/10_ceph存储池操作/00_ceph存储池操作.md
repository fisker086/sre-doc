# ceph 存储池操作

>存储池的管理通常保存创建、列出、重命名和删除等操作，管理工具使用 ceph osd pool
>的子命令及参数，比如 create/ls/rename/rm 等。
>http://docs.ceph.org.cn/rados/

## 一、存储池管理常用命令

### 1、创建

```bash
# 创建存储池命令格式：
# ceph osd pool create <poolname> pg_num pgp_num {replicated|erasure}

# 列出存储池
root@ceph-node-01:~# ceph osd pool ls #不带 pool ID
.mgr
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
cephfs.ceph-fs.meta
cephfs.ceph-fs.data
```

### 2、列出存储池

```bash
#不带 pool ID
root@ceph-node-01:~# ceph osd pool ls
.mgr
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
cephfs.ceph-fs.meta
cephfs.ceph-fs.data

#带 pool ID
root@ceph-node-01:~# ceph osd lspools
1 .mgr
2 .rgw.root
3 default.rgw.log
4 default.rgw.control
5 default.rgw.meta
6 cephfs.ceph-fs.meta
7 cephfs.ceph-fs.data
```

### 3、获取存储池的事件信息

```bash
root@ceph-node-01:~# ceph osd pool stats cephfs.ceph-fs.data
pool cephfs.ceph-fs.data id 7
  nothing is going on
```

### 4、重命名存储池

```bash
$ ceph osd pool rename old-name new-name
$ ceph osd pool rename myrbd1 myrbd2
```

### 5、显示存储池的用量信息

```bash
root@ceph-node-01:~# rados df
POOL_NAME               USED  OBJECTS  CLONES  COPIES  MISSING_ON_PRIMARY  UNFOUND  DEGRADED  RD_OPS       RD  WR_OPS       WR  USED COMPR  UNDER COMPR
.mgr                 1.7 MiB        2       0       6                   0        0         0     428  995 KiB     349  5.4 MiB         0 B          0 B
.rgw.root             72 KiB        6       0      18                   0        0         0      31   28 KiB       0      0 B         0 B          0 B
cephfs.ceph-fs.data      0 B        0       0       0                   0        0         0       0      0 B       0      0 B         0 B          0 B
cephfs.ceph-fs.meta  130 KiB       22       0      66                   0        0         0     158  167 KiB      25   23 KiB         0 B          0 B
default.rgw.control      0 B        8       0      24                   0        0         0       0      0 B       0      0 B         0 B          0 B
default.rgw.log      408 KiB      209       0     627                   0        0         0  174645  171 MiB  115744      0 B         0 B          0 B
default.rgw.meta      24 KiB        3       0       9                   0        0         0      55   43 KiB       0      0 B         0 B          0 B

total_objects    250
total_used       204 MiB
total_avail      2.7 TiB
total_space      2.7 TiB
```

### 6、删除

>如果把存储池删除会导致把存储池内的数据全部删除，因此 ceph 为了防止误删除存储池设置了两个机制来防止误删除操作。
>第一个机制是 NODELETE 标志，需要设置为 false 但是默认就是 false 了。

```bash
# 创建一个测试 pool
root@ceph-node-01:~# ceph osd pool create mypool2 32 32
pool 'mypool2' created

# 如果设置为了 true 就表示不能删除，可以使用 set 指令设置为 fasle
root@ceph-node-01:~# ceph osd pool get mypool2 nodelete
nodelete: false

# 如果设置为了 true 就表示不能删除，可以使用 set 指令设置为 fasle
ceph osd pool set mypool2 nodelete true

root@ceph-node-01:~# ceph osd pool set mypool2 nodelete true
set pool 10 nodelete to true
root@ceph-node-01:~# ceph osd pool get mypool2 nodelete
nodelete: true
root@ceph-node-01:~# ceph osd pool set mypool2 nodelete false
set pool 10 nodelete to false
root@ceph-node-01:~# ceph osd pool get mypool2 nodelete
nodelete: false
```

>第二个机制是集群范围的配置参数 mon allow pool delete，默认值为 false，即监视器不允许删除存储池，可以在特定场合使用 tell 指令临时设置为(true)允许删除，在删除指定的 pool之后再重新设置为 false。

```bash
root@ceph-node-01:~# ceph tell mon.* injectargs --mon-allow-pool-delete=true
mon.ceph-node-01: {}
mon.ceph-node-01: mon_allow_pool_delete = 'true'
mon.ceph-node-02: {}
mon.ceph-node-02: mon_allow_pool_delete = 'true'
mon.ceph-node-03: {}
mon.ceph-node-03: mon_allow_pool_delete = 'true'
root@ceph-node-01:~# ceph osd pool rm mypool2 mypool2 --yes-i-really-really-mean-it
pool 'mypool2' removed
root@ceph-node-01:~# ceph tell mon.* injectargs --mon-allow-pool-delete=false
mon.ceph-node-01: {}
mon.ceph-node-01: mon_allow_pool_delete = 'false'
mon.ceph-node-02: {}
mon.ceph-node-02: mon_allow_pool_delete = 'false'
mon.ceph-node-03: {}
mon.ceph-node-03: mon_allow_pool_delete = 'false'
```

### 7、存储池配额

>存储池可以设置两个配对存储的对象进行限制，一个配额是最大空间(max_bytes)，另外一个配额是对象最大数量(max_objects)。

```bash
root@ceph-node-01:~# ceph osd pool get-quota cephfs.ceph-fs.data
quotas for pool 'cephfs.ceph-fs.data':
  max objects: N/A
  max bytes  : N/A

#限制最大 1000 个对象
ceph osd pool set-quota mypool max_objects 1000

#限制最大 10737418240 字节
ceph osd pool set-quota mypool max_bytes 10737418240
```

## 二、存储池可用参数

### 1、副本数量

>存储池中的对象副本数,默认一主两个备 3 副本

```bash
root@ceph-node-01:~# ceph osd pool get cephfs.ceph-fs.data size
size: 3

root@ceph-node-01:~# ceph osd pool get cephfs.ceph-fs.data min_size
min_size: 2

# min_size：提供服务所需要的最小副本数，如果定义 size 为 3，min_size 也为 3，坏掉一个OSD，如果 pool 池中有副本在此块 OSD 上面，那么此 pool 将不提供服务，如果将 min_size定义为 2，那么还可以提供服务，如果提供为 1，表示只要有一块副本都提供服务。
```

### 2、查看当前pg数量

```bash
root@ceph-node-01:~# ceph osd pool get cephfs.ceph-fs.data pg_num
pg_num: 1
```

### 3、crush_rule：设置 crush 算法规则

```bash
root@ceph-node-01:~# ceph osd pool get cephfs.ceph-fs.data crush_rule
crush_rule: replicated_rule #默认为副本池
```

### 4、nodelete：控制是否可删除

```bash
ceph osd pool get cephfs.ceph-fs.data nodelete
```

### 5、nopgchange：控制是否可更改存储池的 pg num 和 pgp num

```bash
ceph osd pool get mypool nopgchange

#修改指定 pool 的 pg 数量
ceph osd pool set mypool pg_num 64


```

### 6、nosizechange：控制是否可以更改存储池的大小

```bash
root@ceph-node-01:~# ceph osd pool get-quota cephfs.ceph-fs.data
quotas for pool 'cephfs.ceph-fs.data':
  max objects: N/A
  max bytes  : N/A

#限制最大 1000 个对象
ceph osd pool set-quota mypool max_objects 1000

#限制最大 10737418240 字节
ceph osd pool set-quota mypool max_bytes 10737418240
```

### 7、noscrub 和 nodeep-scrub：控制是否不进行轻量扫描或是否深层扫描存储池，可临时解决高 I/O 问题

```bash
#查看当前是否关闭轻量扫描数据，默认为不关闭，即开启
ceph osd pool get mypool noscrub
noscrub: false

#可以修改某个指定的 pool 的轻量级扫描测量为 true，即不执行轻量级扫描
ceph osd pool set mypool noscrub true

#查看当前是否关闭深度扫描数据，默认为不关闭，即开启
ceph osd pool get mypool nodeep-scrub
nodeep-scrub: false

#可以修改某个指定的 pool 的深度扫描测量为 true，即不执行深度扫描
ceph osd pool set mypool nodeep-scrub true
set pool 1 nodeep-scrub to true 
```

### 8、scrub_min_interval、scrub_max_interval和deep_scrub_interval

>scrub_min_interval：集群存储池的最小清理时间间隔，默认值没有设置，可以通过配置文件中的 osd_scrub_min_interval 参数指定间隔时间
>
>scrub_max_interval：整理存储池的最大清理时间间隔，默认值没有设置，可以通过配置文件中的 osd_scrub_max_interval 参数指定。
>
>deep_scrub_interval：深层整理存储池的时间间隔，默认值没有设置，可以通过配置文件中的 osd_deep_scrub_interval 参数指定。

```bash
$ ceph osd pool get mypool scrub_min_interval
Error ENOENT: option 'scrub_min_interval' is not set on pool 'mypool'

$ ceph osd pool get mypool scrub_max_interval
Error ENOENT: option 'scrub_max_interval' is not set on pool 'mypool'

$ ceph osd pool get mypool deep_scrub_interval
Error ENOENT: option 'deep_scrub_interval' is not set on pool 'mypool'



```

## 三、存储池默认配置

>ceph daemon osd.3 config show | grep scrub

```bash
"mds_max_scrub_ops_in_progress": "5",
"mon_scrub_inject_crc_mismatch": "0.000000",
"mon_scrub_inject_missing_keys": "0.000000",
"mon_scrub_interval": "86400",
"mon_scrub_max_keys": "100",
"mon_scrub_timeout": "300",
"mon_warn_not_deep_scrubbed": "0",
"mon_warn_not_scrubbed": "0",
"osd_debug_deep_scrub_sleep": "0.000000",
"osd_deep_scrub_interval": "604800.000000", #定义深度清洗间隔，604800 秒=7天
"osd_deep_scrub_keys": "1024",
"osd_deep_scrub_large_omap_object_key_threshold": "200000",
"osd_deep_scrub_large_omap_object_value_sum_threshold": "1073741824",
"osd_deep_scrub_randomize_ratio": "0.150000",
"osd_deep_scrub_stride": "524288",
"osd_deep_scrub_update_digest_min_age": "7200",
"osd_max_scrubs": "1", #定义一个 ceph OSD daemon 内能够同时进行 scrubbing的操作数
"osd_op_queue_mclock_scrub_lim": "0.001000",
"osd_op_queue_mclock_scrub_res": "0.000000",
"osd_op_queue_mclock_scrub_wgt": "1.000000",
"osd_requested_scrub_priority": "120",
"osd_scrub_auto_repair": "false",
"osd_scrub_auto_repair_num_errors": "5",
"osd_scrub_backoff_ratio": "0.660000",
"osd_scrub_begin_hour": "0",
"osd_scrub_begin_week_day": "0",
"osd_scrub_chunk_max": "25",
"osd_scrub_chunk_min": "5",
"osd_scrub_cost": "52428800",
"osd_scrub_during_recovery": "false",
"osd_scrub_end_hour": "24",
"osd_scrub_end_week_day": "7",
"osd_scrub_interval_randomize_ratio": "0.500000",
"osd_scrub_invalid_stats": "true", #定义 scrub 是否有效
"osd_scrub_load_threshold": "0.500000",
"osd_scrub_max_interval": "604800.000000", #定义最大执行 scrub 间隔，604800秒=7 天
"osd_scrub_max_preemptions": "5",
"osd_scrub_min_interval": "86400.000000", #定义最小执行普通 scrub 间隔，86400秒=1 天
"osd_scrub_priority": "5",
"osd_scrub_sleep": "0.000000",
```

## 四、存储池快照

>快照用于读存储池中的数据进行备份与还原，创建快照需要占用的磁盘空间会比较大，取决于存储池中的数据大小，使用以下命令创建快照：

### 1、创建快照

```bash
$ ceph osd pool ls

#命令 1：ceph osd pool mksnap {pool-name} {snap-name}
$ ceph osd pool mksnap mypool mypool-snap
created pool mypool snap mypool-snap


#命令 2: rados -p {pool-name} mksnap {snap-name}
$ rados -p mypool mksnap mypool-snap2
created pool mypool snap mypool-snap2
```

### 2、验证快照

```bash
$ rados lssnap -p mypool
1 mypool-snap 2020.11.03 16:12:56
2 mypool-snap2 2020.11.03 16:13:40
2 snaps
```

### 3、回滚快照

>测试上传文件后创建快照，然后删除文件再还原文件,基于对象还原
>
>rados rollback <obj-name> <snap-name> roll back object to snap <snap-name>

```bash
#上传文件
$ rados -p mypool put testfile /etc/hosts

#验证文件
$ rados -p mypool ls
msg1
testfile
my.conf

#创建快照
$ ceph osd pool mksnap mypool
mypool-snapshot001
created pool mypool snap mypool-snapshot001

#验证快照
$ rados lssnap -p mypool
3 mypool-snap 2020.11.04 14:11:41
4 mypool-snap2 2020.11.04 14:11:49
5 mypool-conf-bak 2020.11.04 14:18:41
6 mypool-snapshot001 2020.11.04 14:38:50
4 snaps

#删除文件
$ rados -p mypool rm testfile

#删除文件后，无法再次删除文件，提升文件不存在
$ rados -p mypool rm testfile
error removing mypool>testfile: (2) No such file or directory

#通过快照还原某个文件
$ rados rollback -p mypool testfile
mypool-snapshot001
rolled back pool mypool to snapshot mypool-snapshot001

#再次执行删除就可以执行成功
$ rados -p mypool rm testfile
```

### 4、删除快照

>ceph osd pool rmsnap <poolname> <snap>

```bash
$ rados lssnap -p mypool
3 mypool-snap 2020.11.04 14:11:41
4 mypool-snap2 2020.11.04 14:11:49
5 mypool-conf-bak 2020.11.04 14:18:41
6 mypool-snapshot001 2020.11.04 14:38:50
4 snaps

$ ceph osd pool rmsnap mypool mypool-snap
removed pool mypool snap mypool-snap

$ rados lssnap -p mypool
4 mypool-snap2 2020.11.04 14:11:49
5 mypool-conf-bak 2020.11.04 14:18:41
6 mypool-snapshot001 2020.11.04 14:38:50
3 snaps
```

## 五、数据压缩

>如果使用 bulestore 存储引擎，ceph 支持称为”实时数据压缩”即边压缩边保存数据的功能，该功能有助于节省磁盘空间，可以在 BlueStore OSD 上创建的每个 Ceph 池上启用或禁用压缩，以节约磁盘空间，默认没有开启压缩，需要后期配置并开启

### 1、启用压缩并指定压缩算法

```bash
#默认算法为 snappy
$ ceph osd pool set <pool name> compression_algorithm snappy
```

>snappy：该配置为指定压缩使用的算法默认为 sanppy，还有 none、zlib、lz4、zstd 和 snappy等算法，zstd 压缩比好，但消耗 CPU，lz4 和 snappy 对 CPU 占用较低，不建议使用 zlib。

### 2、指定压缩模式

```bash
ceph osd pool set <pool name> compression_mode aggressive
```

>aggressive：压缩的模式，有 none、aggressive、passive 和 force，默认 none。
>none：从不压缩数据。
>passive：除非写操作具有可压缩的提示集，否则不要压缩数据。
>aggressive：压缩数据，除非写操作具有不可压缩的提示集。
>force：无论如何都尝试压缩数据，即使客户端暗示数据不可压缩也会压缩，也就是在所有情况下都使用压缩。

### 3、存储池压缩设置参数

>compression_algorithm #压缩算法
>
>compression_mode #压缩模式
>
>compression_required_ratio #压缩后与压缩前的压缩比，默认为.875
>
>compression_max_blob_size： #大于此的块在被压缩之前被分解成更小的 blob(块)，此设置将覆盖 bluestore 压缩 max blob*的全局设置。
>
>compression_min_blob_size：#小于此的块不压缩, 此设置将覆盖 bluestore 压缩 min blob*的全局设置

### 4、全局压缩选项

> 这些可以配置到 ceph.conf 配置文件，作用于所有存储池

```bash
bluestore_compression_algorithm #压缩算法
bluestore_compression_mode #压缩模式
bluestore_compression_required_ratio #压缩后与压缩前的压缩比，默认为.875
bluestore_compression_min_blob_size #小于它的块不会被压缩,默认 0
bluestore_compression_max_blob_size #大于它的块在压缩前会被拆成更小的块,默认 0
bluestore_compression_min_blob_size_ssd #默认 8k
bluestore_compression_max_blob_size_ssd #默认 64k
bluestore_compression_min_blob_size_hdd #默认 128k
bluestore_compression_max_blob_size_hdd #默认 512k
```

### 5、命令行修改

```bash
#修改压缩算法
$ ceph osd pool set mypool compression_algorithm snappy
set pool 2 compression_algorithm to snappy

$ ceph osd pool get mypool compression_algorithm
compression_algorithm: snappy


#修改压缩模式：
$ ceph osd pool set mypool compression_mode passive
set pool 2 compression_mode to passive

$ ceph osd pool get mypool compression_mode
compression_mode: passive
```