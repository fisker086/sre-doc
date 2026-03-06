# Ceph常用命令

## 一、集群类

### 1、查看集群存储状态

```bash
root@ceph-node-01:~# ceph df
--- RAW STORAGE ---
CLASS     SIZE    AVAIL     USED  RAW USED  %RAW USED
hdd    2.7 TiB  2.7 TiB  224 MiB   224 MiB          0
TOTAL  2.7 TiB  2.7 TiB  224 MiB   224 MiB          0

--- POOLS ---
POOL                 ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
.mgr                  1    1  577 KiB        2  1.7 MiB      0    885 GiB
.rgw.root             2   32  2.6 KiB        6   72 KiB      0    885 GiB
default.rgw.log       3   32  3.6 KiB      209  408 KiB      0    885 GiB
default.rgw.control   4   32      0 B        8      0 B      0    885 GiB
default.rgw.meta      5   32    382 B        3   24 KiB      0    885 GiB
cephfs.ceph-fs.meta   6   16   13 KiB       22  130 KiB      0    885 GiB
cephfs.ceph-fs.data   7    1      0 B        0      0 B      0    885 GiB

```

### 2、查看集群存储状态详情

```bash
root@ceph-node-01:~# ceph df detail
--- RAW STORAGE ---
CLASS     SIZE    AVAIL     USED  RAW USED  %RAW USED
hdd    2.7 TiB  2.7 TiB  224 MiB   224 MiB          0
TOTAL  2.7 TiB  2.7 TiB  224 MiB   224 MiB          0

--- POOLS ---
POOL                 ID  PGS   STORED   (DATA)   (OMAP)  OBJECTS     USED   (DATA)  (OMAP)  %USED  MAX AVAIL  QUOTA OBJECTS  QUOTA BYTES  DIRTY  USED COMPR  UNDER COMPR
.mgr                  1    1  577 KiB  577 KiB      0 B        2  1.7 MiB  1.7 MiB     0 B      0    885 GiB            N/A          N/A    N/A         0 B          0 B
.rgw.root             2   32  2.6 KiB  2.6 KiB      0 B        6   72 KiB   72 KiB     0 B      0    885 GiB            N/A          N/A    N/A         0 B          0 B
default.rgw.log       3   32  3.6 KiB  3.6 KiB      0 B      209  408 KiB  408 KiB     0 B      0    885 GiB            N/A          N/A    N/A         0 B          0 B
default.rgw.control   4   32      0 B      0 B      0 B        8      0 B      0 B     0 B      0    885 GiB            N/A          N/A    N/A         0 B          0 B
default.rgw.meta      5   32    382 B    382 B      0 B        3   24 KiB   24 KiB     0 B      0    885 GiB            N/A          N/A    N/A         0 B          0 B
cephfs.ceph-fs.meta   6   16   13 KiB  5.8 KiB  7.5 KiB       22  130 KiB  108 KiB  22 KiB      0    885 GiB            N/A          N/A    N/A         0 B          0 B
cephfs.ceph-fs.data   7    1      0 B      0 B      0 B        0      0 B      0 B     0 B      0    885 GiB            N/A          N/A    N/A         0 B          0 B
```



## 二、OSD

### 1、只显示存储池

```bash
root@ceph-node-01:~# ceph osd pool ls
.mgr
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
cephfs.ceph-fs.meta
cephfs.ceph-fs.data
```

### 2、列出存储池并显示 id

```bash
root@ceph-node-01:~# ceph osd lspools
1 .mgr
2 .rgw.root
3 default.rgw.log
4 default.rgw.control
5 default.rgw.meta
6 cephfs.ceph-fs.meta
7 cephfs.ceph-fs.data
```

### 3、查看指定 pool 或所有的 pool 的状态

```bash
root@ceph-node-01:~# ceph osd pool stats cephfs.ceph-fs.data
pool cephfs.ceph-fs.data id 7
  nothing is going on

```

### 4、查看 osd 状态

```bash
root@ceph-node-01:~# ceph osd stat
3 osds: 3 up (since 4m), 3 in (since 2d); epoch: e185
```

### 5、显示 OSD 的底层详细信息

```bash
root@ceph-node-01:~# ceph osd dump
epoch 185
fsid b227ebe2-f9b8-11ee-826d-15b492408c47
created 2024-04-13T17:11:42.938628+0000
modified 2024-04-16T13:36:14.436745+0000
flags sortbitwise,recovery_deletes,purged_snapdirs,pglog_hardlimit
crush_version 11
full_ratio 0.95
backfillfull_ratio 0.9
nearfull_ratio 0.85
require_min_compat_client luminous
min_compat_client jewel
require_osd_release reef
stretch_mode_enabled false
pool 1 '.mgr' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 1 pgp_num 1 autoscale_mode on last_change 21 flags hashpspool stripe_width 0 pg_num_max 32 pg_num_min 1 application mgr read_balance_score 3.00
pool 2 '.rgw.root' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode on last_change 38 lfor 0/0/32 flags hashpspool stripe_width 0 application rgw read_balance_score 1.31
pool 3 'default.rgw.log' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode on last_change 100 lfor 0/0/34 flags hashpspool stripe_width 0 application rgw read_balance_score 1.13
pool 4 'default.rgw.control' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode on last_change 38 lfor 0/0/34 flags hashpspool stripe_width 0 application rgw read_balance_score 1.13
pool 5 'default.rgw.meta' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode on last_change 38 lfor 0/0/36 flags hashpspool stripe_width 0 pg_autoscale_bias 4 application rgw read_balance_score 1.22
pool 6 'cephfs.ceph-fs.meta' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 16 pgp_num 16 autoscale_mode on last_change 130 lfor 0/0/103 flags hashpspool stripe_width 0 pg_autoscale_bias 4 pg_num_min 16 recovery_priority 5 application cephfs read_balance_score 1.50
pool 7 'cephfs.ceph-fs.data' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 1 pgp_num 1 autoscale_mode on last_change 92 flags hashpspool,bulk stripe_width 0 application cephfs read_balance_score 3.00
max_osd 3
osd.0 up   in  weight 1 up_from 176 up_thru 176 down_at 175 last_clean_interval [149,165) [v2:192.168.31.121:6802/1395579241,v1:192.168.31.121:6803/1395579241] [v2:192.168.31.121:6804/1395579241,v1:192.168.31.121:6805/1395579241] exists,up 57196899-3c08-4e76-900d-c20165dc1374
osd.1 up   in  weight 1 up_from 176 up_thru 176 down_at 175 last_clean_interval [149,165) [v2:192.168.31.122:6800/2257202174,v1:192.168.31.122:6801/2257202174] [v2:192.168.31.122:6802/2257202174,v1:192.168.31.122:6803/2257202174] exists,up 84c907c5-cd98-41e3-bf9f-dcd4d561906e
osd.2 up   in  weight 1 up_from 175 up_thru 176 down_at 174 last_clean_interval [147,165) [v2:192.168.31.123:6802/2375931830,v1:192.168.31.123:6803/2375931830] [v2:192.168.31.123:6804/2375931830,v1:192.168.31.123:6805/2375931830] exists,up 4f80fe2f-26be-4f61-9215-7537dc308aa8
blocklist 192.168.31.121:0/1009565350 expires 2024-04-17T13:32:23.343602+0000
blocklist 192.168.31.121:0/1916861071 expires 2024-04-17T13:32:23.343602+0000
blocklist 192.168.31.121:0/2506391152 expires 2024-04-17T13:32:23.343602+0000
blocklist 192.168.31.121:6810/2650024984 expires 2024-04-17T13:32:23.343602+0000
blocklist 192.168.31.121:0/3741783376 expires 2024-04-17T13:32:23.343602+0000
blocklist 192.168.31.122:6800/3680641927 expires 2024-04-16T15:53:23.917982+0000
blocklist 192.168.31.122:6801/3680641927 expires 2024-04-16T15:53:23.917982+0000
blocklist 192.168.31.123:6800/3045589790 expires 2024-04-17T13:32:15.747240+0000
blocklist 192.168.31.123:6801/3045589790 expires 2024-04-17T13:32:15.747240+0000
blocklist 192.168.31.121:6811/2650024984 expires 2024-04-17T13:32:23.343602+0000
```

### 6、显示 OSD 和节点的对应关系

```bash
root@ceph-node-01:~# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME              STATUS  REWEIGHT  PRI-AFF
-1         2.72910  root default
-3         0.90970      host ceph-node-01
 0    hdd  0.90970          osd.0              up   1.00000  1.00000
-5         0.90970      host ceph-node-02
 1    hdd  0.90970          osd.1              up   1.00000  1.00000
-7         0.90970      host ceph-node-03
 2    hdd  0.90970          osd.2              up   1.00000  1.00000
```



## 三、pg

### 1、查看 pg 状态

```bash
root@ceph-node-01:~# ceph pg stat
146 pgs: 146 active+clean; 589 KiB data, 224 MiB used, 2.7 TiB / 2.7 TiB avail
```

## 四、查看 mon

### 1、查看 mon 节点状态

```bash
root@ceph-node-01:~# ceph mon stat
e3: 3 mons at {ceph-node-01=[v2:192.168.31.121:3300/0,v1:192.168.31.121:6789/0],ceph-node-02=[v2:192.168.31.122:3300/0,v1:192.168.31.122:6789/0],ceph-node-03=[v2:192.168.31.123:3300/0,v1:192.168.31.123:6789/0]} removed_ranks: {} disallowed_leaders: {}, election epoch 24, leader 0 ceph-node-01, quorum 0,1,2 ceph-node-01,ceph-node-02,ceph-node-03
```

### 2、查看 mon 节点的 dump 信息

```bash
root@ceph-node-01:~# ceph mon dump
epoch 3
fsid b227ebe2-f9b8-11ee-826d-15b492408c47
last_changed 2024-04-13T17:39:38.687975+0000
created 2024-04-13T17:11:41.357975+0000
min_mon_release 18 (reef)
election_strategy: 1
0: [v2:192.168.31.121:3300/0,v1:192.168.31.121:6789/0] mon.ceph-node-01
1: [v2:192.168.31.122:3300/0,v1:192.168.31.122:6789/0] mon.ceph-node-02
2: [v2:192.168.31.123:3300/0,v1:192.168.31.123:6789/0] mon.ceph-node-03
dumped monmap epoch 3
```