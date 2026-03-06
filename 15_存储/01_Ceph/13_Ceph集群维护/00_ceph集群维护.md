# Ceph集群维护

## 一、集群开机和关机

### 1、需求

> 机房停电，或者需要搬迁，需要所有ceph节点关机

### 2、关机

#### 1.关闭存储客户端停止读写数据

#### 2.先在admin节点执行以下命令关闭集群流量

	ceph osd set noout
	ceph osd set norecover
	ceph osd set norebalance
	ceph osd set nobackfill
	ceph osd set nodown
	ceph osd set pause

#### 3.关闭服务

```bash
如果使用了 RGW，关闭 RGW
关闭 cephfs 元数据服务
关闭 ceph OSD
关闭 ceph manager
关闭 ceph monitor
```

### 3、开机

#### 1.启动服务

```bash
启动 ceph monitor
启动 ceph manager
启动 ceph OSD
关闭 cephfs 元数据服务
启动 RGW
```

#### 2.去除标签

```bash
ceph osd unset noout
ceph osd unset norecover
ceph osd unset norebalance
ceph osd unset nobackfill
ceph osd unset nodown
ceph osd unset pause
```

#### 3.启动存储客户端

## 二、增加服务器

### 1、主机初始化配置好ssh免密等等

### 2、docker安装

```bash
export DOWNLOAD_URL="https://mirrors.tuna.tsinghua.edu.cn/docker-ce"
curl -fsSL https://get.docker.com/ | sh
```

### 3、准备ceph配置文件存储地址

```bash
mkdir -p /etc/cephadmin
mkdir -p /etc/ceph
```

### 4、cephadm标记其他主机到集群

#### 1)添加公钥

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub ceph-node-04
```

#### 2)添加_admin标签

```bash
ceph orch host add ceph-node-02 192.168.31.124 _admin
```

### 5、新增添加monitor

```bash
ceph orch daemon add mon ceph-node-04:192.168.31.124
```

### 6、添加新增节点硬盘-OSD

#### 1.查看各个节点可用磁盘

```bash
root@ceph-node-01:~# ceph orch device ls
```

#### 2.关闭自动发现磁盘自动创建OSD

```bash
root@ceph-node-01:~# ceph orch apply osd --all-available-devices --unmanaged=true
Scheduled osd.all-available-devices update...
```

#### 3.清除磁盘GPT

```bash
root@ceph-node-04:~# gdisk /dev/sdb
GPT fdisk (gdisk) version 1.0.8

The protective MBR's 0xEE partition is oversized! Auto-repairing.

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): x

Expert command (? for help): z
About to wipe out GPT on /dev/sdb. Proceed? (Y/N): y
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
Blank out MBR? (Y/N): y
```

#### 4.手动添加硬盘

```bash
root@ceph-node-01:~# ceph orch daemon add osd ceph-node-04:/dev/sdb
Created osd(s) 2 on host 'ceph-node-04'
```

**验证**

```bash
root@ceph-node-01:~# ceph status
```

### 7、所有节点添加RGW

```bash
ceph orch apply rgw ceph-rgw  --placement="4" --port=80
```

### 8、所有节点添加MDS

```bash
ceph fs volume create ceph-fs --placement="4"
```

## 三、删除服务器

### 1、删除OSD

#### 1.踢出osd

```bash
ceph osd out 1
```

#### 2.等待时间，让服务器进行数据传输。

>监控磁盘IO，3分钟内没有读写后继续操作

#### 3.删除osd

```bash
ceph osd rm 1
```

### 2、删除服务器

>停止服务器之前要把服务器的 OSD 先停止并从 ceph 集群删除

#### 1.删除所有osd

#### 2.node节点下线

```bash
ceph osd crush rm ceph-node1
```

### 3、重新调整RGW数量

```bash
ceph orch apply rgw ceph-rgw  --placement="3" --port=80
```

### 4、重新调整MDS数量

```bash
ceph fs volume create ceph-fs --placement="3"
```

## 四、ceph的配置文件了解

>ceph 的主配置文件是/etc/ceph/ceph.conf，ceph 服务在启动时会检查 ceph.conf，分号;和#在配置文件中都是注释，ceph.conf 主要由以下配置段组成

### 1、配置文件格式

```bash
[global] #全局配置
[osd] #osd 专用配置，可以使用 osd.N，来表示某一个 OSD 专用配置，N 为 osd 的编号，如 0、2、1 等。
[mon] #mon 专用配置，也可以使用 mon.A 来为某一个 monitor 节点做专用配置，其中 A为该节点的名称，ceph-monitor-2、ceph-monitor-1 等，使用命令 ceph mon dump 可以获取节点的名称
[client] #客户端专用配置
```

### 2、ceph 文件的加载顺序

```bash
$CEPH_CONF 环境变量
-c 指定的位置
/etc/ceph/ceph.conf
~/.ceph/ceph.conf
./ceph.conf
```

### 3、ceph node 的默认配置

```bash
ceph daemon osd.3 config show | grep scrub
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



















