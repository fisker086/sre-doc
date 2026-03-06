# 块存储RBD使用

## 一、介绍

>​    RBD(RADOS Block Devices)即为块存储设备，RBD 可以为 KVM、vmware 等虚拟化技术和云服务（如 OpenStack、kubernetes）提供高性能和无限可扩展性的存储后端，客户端基于 librbd 库即可将 RADOS 存储集群用作块设备，不过，用于 rbd 的存储池需要事先启用rbd 功能并进行初始化。

>​    Ceph 可以同时提供 RADOSGW(对象存储网关)、RBD(块存储)、Ceph FS(文件系统存储)，RBD 即 RADOS Block Device 的简称，RBD 块存储是常用的存储类型之一，RBD 块设备类似磁盘可以被挂载，RBD 块设备具有快照、多副本、克隆和一致性等特性，数据以条带化的方式存储在 Ceph 集群的多个 OSD 中。

>​    条带化技术就是一种自动的将 I/O 的负载均衡到多个物理磁盘上的技术，条带化技术就是将一块连续的数据分成很多小部分并把他们分别存储到不同磁盘上去。这就能使多个进程同时访问数据的多个不同部分而不会造成磁盘冲突，而且在需要对这种数据进行顺序访问的时候可以获得最大程度上的 I/O 并行能力，从而获得非常好的性能

## 二、使用

### 1、创建RBD

#### 1.格式

```bash
创建存储池命令格式：
ceph osd pool create <poolname> pg_num pgp_num {replicated|erasure}
```

#### 2.创建存储池

```bash
#创建存储池,指定 pg 和 pgp 的数量，pgp 是对存在于 pg 的数据进行组合存储，pgp 通常等于 pg 的值
root@ceph-node-01:~# ceph osd pool create myrbd1 64 64
pool 'myrbd1' created

```

#### 3.对存储池启用 RBD 功能

```bash
root@ceph-node-01:~# ceph osd pool --help
root@ceph-node-01:~# ceph osd pool application enable myrbd1 rbd
enabled application 'rbd' on pool 'myrbd1'
```

#### 4.通过RBD命令对存储池初始化

```bash
root@ceph-node-01:~# rbd -h
root@ceph-node-01:~# rbd pool init -p myrbd1
```

### 2、创建并验证 img

>​    不过，rbd 存储池并不能直接用于块设备，而是需要事先在其中按需创建映像（image），并把映像文件作为块设备使用，rbd 命令可用于创建、查看及删除块设备相在的映像（image），以及克隆映像、创建快照、将映像回滚到快照和查看快照等管理操作。

#### 1.创建镜像

```bash
root@ceph-node-01:~# rbd create myimg1 --size 5G --pool myrbd1
root@ceph-node-01:~# rbd create myimg2 --size 3G --pool myrbd1 --image-format 2 --image-feature layering
```

>`--image-format 2`指定镜像格式为2
>
>`--image-feature layering`启用镜像层次结构特性。

#### 2.列出所有img

```bash
root@ceph-node-01:~# rbd ls --pool myrbd1
myimg1
myimg2
```

#### 3.查看指定img信息

```bash
root@ceph-node-01:~# rbd --image myimg1 --pool myrbd1 info
rbd image 'myimg1':
        size 5 GiB in 1280 objects
        order 22 (4 MiB objects) #对象大小，每个对象是 2^22/1024/1024=4MiB
        snapshot_count: 0
        id: 39f0566f7917 #镜像 id
        block_name_prefix: rbd_data.39f0566f7917 #size 里面的 1280 个对象名称前缀
        format: 2 #镜像文件格式版本
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten #特性，layering 支持分层快照以写时复制
        op_features:
        flags:
        create_timestamp: Sun Apr 14 20:57:46 2024
        access_timestamp: Sun Apr 14 20:57:46 2024
        modify_timestamp: Sun Apr 14 20:57:46 2024
root@ceph-node-01:~# rbd --image myimg2 --pool myrbd1 info
rbd image 'myimg2':
        size 3 GiB in 768 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 39f69af8038c
        block_name_prefix: rbd_data.39f69af8038c
        format: 2
        features: layering
        op_features:
        flags:
        create_timestamp: Sun Apr 14 20:58:02 2024
        access_timestamp: Sun Apr 14 20:58:02 2024
        modify_timestamp: Sun Apr 14 20:58:02 2024
```

#### 4.镜像特性介绍

##### 1)layering

>支持镜像分层快照特性，用于快照及写时复制，可以对 image 创建快照并保护，然后从快照克隆出新的 image 出来，父子 image 之间采用 COW 技术，共享对象数据

##### 2)striping

>支持条带化 v2，类似 raid 0，只不过在 ceph 环境中的数据被分散到不同的对象中，可改善顺序读写场景较多情况下的性能。

##### 3)exclusive-lock

>支持独占锁，限制一个镜像只能被一个客户端使用。

##### 4)object-map

>支持对象映射(依赖 exclusive-lock),加速数据导入导出及已用空间统计等，此特性开启的时候，会记录 image 所有对象的一个位图，用以标记对象是否真的存在，在一些场景下可以加速 io。

##### 5)fast-diff

>快速计算镜像与快照数据差异对比(依赖 object-map)

##### 6)deep-flatten

>支持快照扁平化操作，用于快照管理时解决快照依赖关系等。

##### 7)journaling

>修改数据是否记录日志，该特性可以通过记录日志并通过日志恢复数据(依赖独占锁),开启此特性会增加系统磁盘 IO 使用

##### 8)默认特性

>jewel 默认开启的特性包括: layering/exlcusive lock/object map/fast diff/deep flatten

#### 5.禁用特性

```bash
#禁用指定存储池中指定镜像的特性:
$ rbd feature disable fast-diff --pool rbd-data1 --image data-img1
#验证镜像特性：
$ rbd --image data-img1 --pool rbd-data1 info
rbd image 'data-img1':
        size 3 GiB in 768 objects
        order 22 (4 MiB objects)
        id: d45b6b8b4567
        block_name_prefix: rbd_data.d45b6b8b4567
        format: 2
        features: layering, exclusive-lock, object-map #少了一个 fast-diff 特性
        op_features:
        flags: object map invalid
        create_timestamp: Mon Dec 14 12:35:44 2020
```

### 3、客户端使用块存储

#### 1.当前ceph状态

```bash
root@ceph-node-01:~# ceph df
--- RAW STORAGE ---
CLASS     SIZE    AVAIL     USED  RAW USED  %RAW USED
hdd    2.7 TiB  2.7 TiB  646 MiB   646 MiB       0.02
TOTAL  2.7 TiB  2.7 TiB  646 MiB   646 MiB       0.02

--- POOLS ---
POOL                 ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
.mgr                  1    1  577 KiB        2  1.7 MiB      0    885 GiB
.rgw.root             2   32  2.6 KiB        6   72 KiB      0    885 GiB
default.rgw.log       3   32  3.6 KiB      209  408 KiB      0    885 GiB
default.rgw.control   4   32      0 B        8      0 B      0    885 GiB
default.rgw.meta      5   32    382 B        3   24 KiB      0    885 GiB
cephfs.ceph-fs.meta   6   16  2.3 KiB       22   96 KiB      0    885 GiB
cephfs.ceph-fs.data   7    1      0 B        0      0 B      0    885 GiB
myrbd1                8   64    405 B        7   48 KiB      0    885 GiB
```

#### 2.在客户端安装 ceph-common，同步认证文件

```bash
apt-get install ceph-common -y

rsync -avz root@172.31.0.121:/etc/ceph/ceph.client.admin.keyring root@172.31.0.121:/etc/ceph/ceph.conf /etc/ceph/
```

#### 3.客户端映射 img

```bash
root@ubuntu-admin:/etc/ceph# rbd -p myrbd1 map myimg1
/dev/rbd0
root@ubuntu-admin:/etc/ceph# rbd -p myrbd1 map myimg2
/dev/rbd1
```

#### 4.客户端验证

```bash
root@ubuntu-admin:/etc/ceph# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 63.9M  1 loop /snap/core20/2105
loop1    7:1    0 40.4M  1 loop /snap/snapd/20671
loop2    7:2    0   87M  1 loop /snap/lxd/27037
loop3    7:3    0 63.9M  1 loop /snap/core20/2264
loop4    7:4    0 39.1M  1 loop /snap/snapd/21184
loop5    7:5    0   87M  1 loop /snap/lxd/27948
sda      8:0    0  200G  0 disk
├─sda1   8:1    0    1M  0 part
└─sda2   8:2    0  200G  0 part /
rbd0   252:0    0    5G  0 disk
rbd1   252:16   0    3G  0 disk
root@ubuntu-admin:/etc/ceph# fdisk -l /dev/rbd0
Disk /dev/rbd0: 5 GiB, 5368709120 bytes, 10485760 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 65536 bytes / 65536 bytes
root@ubuntu-admin:/etc/ceph# fdisk -l /dev/rbd1
Disk /dev/rbd1: 3 GiB, 3221225472 bytes, 6291456 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 65536 bytes / 65536 bytes
```

#### 5.客户端格式化磁盘并挂载使用

```bash
root@ubuntu-admin:/etc/ceph# mkfs.xfs /dev/rbd0
meta-data=/dev/rbd0              isize=512    agcount=8, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=0 inobtcount=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
root@ubuntu-admin:/etc/ceph# mkfs.xfs /dev/rbd1
meta-data=/dev/rbd1              isize=512    agcount=8, agsize=98304 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=0 inobtcount=0
data     =                       bsize=4096   blocks=786432, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
root@ubuntu-admin:/etc/ceph# mkdir /data0
root@ubuntu-admin:/etc/ceph# mkdir /data1
root@ubuntu-admin:/etc/ceph# mount /dev/rbd0 /data0
root@ubuntu-admin:/etc/ceph# mount /dev/rbd1 /data1
root@ubuntu-admin:/etc/ceph# cp /etc/passwd /data0/
root@ubuntu-admin:/etc/ceph# cp /etc/passwd /data1/
root@ubuntu-admin:/etc/ceph# df -TH
Filesystem     Type   Size  Used Avail Use% Mounted on
tmpfs          tmpfs  411M  1.1M  410M   1% /run
/dev/sda2      ext4   211G  8.6G  191G   5% /
tmpfs          tmpfs  2.1G     0  2.1G   0% /dev/shm
tmpfs          tmpfs  5.3M     0  5.3M   0% /run/lock
tmpfs          tmpfs  411M  4.1k  411M   1% /run/user/0
/dev/rbd0      xfs    5.4G   72M  5.3G   2% /data0
/dev/rbd1      xfs    3.3G   57M  3.2G   2% /data1
```

#### 6.客户端创建文件验证

```bash
root@ubuntu-admin:/etc/ceph# dd if=/dev/zero of=/data0/ceph-test-file bs=1MB count=300
300+0 records in
300+0 records out
300000000 bytes (300 MB, 286 MiB) copied, 0.10494 s, 2.9 GB/s
root@ubuntu-admin:/etc/ceph# ll /data0/
total 292980
drwxr-xr-x  2 root root        42 Apr 14 23:27 ./
drwxr-xr-x 22 root root      4096 Apr 14 23:08 ../
-rw-r--r--  1 root root 300000000 Apr 14 23:27 ceph-test-file
-rw-r--r--  1 root root      1905 Apr 14 23:11 passwd
```

#### 7.ceph验证数据

```bash
root@ubuntu-admin:/etc/ceph# ceph df
--- RAW STORAGE ---
CLASS     SIZE    AVAIL     USED  RAW USED  %RAW USED
hdd    2.7 TiB  2.7 TiB  1.6 GiB   1.6 GiB       0.06
TOTAL  2.7 TiB  2.7 TiB  1.6 GiB   1.6 GiB       0.06

--- POOLS ---
POOL                 ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
.mgr                  1    1  577 KiB        2  1.7 MiB      0    884 GiB
.rgw.root             2   32  2.6 KiB        6   72 KiB      0    884 GiB
default.rgw.log       3   32  3.6 KiB      209  408 KiB      0    884 GiB
default.rgw.control   4   32      0 B        8      0 B      0    884 GiB
default.rgw.meta      5   32    382 B        3   24 KiB      0    884 GiB
cephfs.ceph-fs.meta   6   16  2.3 KiB       22   96 KiB      0    884 GiB
cephfs.ceph-fs.data   7    1      0 B        0      0 B      0    884 GiB
myrbd1                8   64  291 MiB       96  873 MiB   0.03    884 GiB
```

#### 8.删除数据

```bash
root@ubuntu-admin:/etc/ceph# ceph df
--- RAW STORAGE ---
CLASS     SIZE    AVAIL     USED  RAW USED  %RAW USED
hdd    2.7 TiB  2.7 TiB  1.6 GiB   1.6 GiB       0.06
TOTAL  2.7 TiB  2.7 TiB  1.6 GiB   1.6 GiB       0.06

--- POOLS ---
POOL                 ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
.mgr                  1    1  577 KiB        2  1.7 MiB      0    884 GiB
.rgw.root             2   32  2.6 KiB        6   72 KiB      0    884 GiB
default.rgw.log       3   32  3.6 KiB      209  408 KiB      0    884 GiB
default.rgw.control   4   32      0 B        8      0 B      0    884 GiB
default.rgw.meta      5   32    382 B        3   24 KiB      0    884 GiB
cephfs.ceph-fs.meta   6   16  2.3 KiB       22   96 KiB      0    884 GiB
cephfs.ceph-fs.data   7    1      0 B        0      0 B      0    884 GiB
myrbd1                8   64  291 MiB       96  873 MiB   0.03    884 GiB
```

>删除完成的数据只是标记为已经被删除，但是不会从块存储立即清空，因此在删除完成后使用 ceph df 查看并没有回收空间
>
>但是后期可以使用此空间，如果需要立即在系统层回收空间，需要执行以下命令：

```bash
# fstrim 命令来自于英文词组“filesystem trim”的缩写，其功能是回收文件系统中未使用的块资源。
root@ubuntu-admin:/etc/ceph# fstrim -v /data0
/data0: 5 GiB (5357813760 bytes) trimmed
```

**或者挂在的时候传参**

```bash
mount -t xfs -o discard /dev/rbd0 /data/
```

>- `mount`: 是Linux系统中用于挂载文件系统的命令。
>- `-t xfs`: 指定要挂载的文件系统类型为XFS。
>- `-o discard`: 启用了discard选项，这会告诉文件系统在删除文件时通知底层块设备释放不再使用的块，以进行TRIM操作。这对于SSD等固态存储设备有益，可以提高性能和延长设备寿命。
>- `/dev/rbd0`: 指定要挂载的RBD块设备的路径。在这里，它是`/dev/rbd0`。
>- `/data`: 指定挂载点，也就是RBD文件系统将被挂载到的目录。

### 4、客户端使用普通账户挂载并使用 RBD

#### 1.创建普通账户并授权

```bash
root@ubuntu-admin:/etc/ceph# ceph auth add client.xiaowu mon 'allow r' osd 'allow rwx pool=myrbd1'
added key for client.xiaowu
```

>- `ceph auth add`: 是用于向Ceph集群添加新认证的命令。
>- `client.xiaowu`: 指定了要添加的客户端名称。
>- `mon 'allow r'`: 分配了对监控器（mon）的只读权限（r权限）。这意味着`client.xiaowu`可以读取监控器的信息，但不能进行修改。
>- `osd 'allow rwx pool=myrbd1'`: 分配了对对象存储守护进程（osd）的读写执行权限（rwx权限），并指定了权限仅适用于名为`myrbd1`的池。这意味着`client.xiaowu`可以对`rbd-data1`池进行读、写和执行操作。

**验证**

```bash
root@ceph-node-01:/etc/ceph# ceph auth get client.xiaowu
[client.xiaowu]
        key = AQDY+htmn4tCARAADBAVfH8a2tOhbh+PV7SbSg==
        caps mon = "allow r"
        caps osd = "allow rwx pool=myrbd1"
```

#### 2.创建用 keyring 文件

```bash
root@ceph-node-01:/etc/ceph# ceph-authtool --create-keyring ceph.client.xiaowu.keyring
creating ceph.client.xiaowu.keyring

# 导出
root@ceph-node-01:/etc/ceph# ceph auth get client.xiaowu -o ceph.client.xiaowu.keyring
root@ceph-node-01:/etc/ceph# ll
total 28
drwxr-xr-x   2 root root 4096 Apr 14 23:51 ./
drwxr-xr-x 101 root root 4096 Apr 14 01:59 ../
-rw-------   1 root root  151 Apr 14 01:14 ceph.client.admin.keyring
-rw-------   1 root root  122 Apr 14 23:52 ceph.client.xiaowu.keyring
-rw-r--r--   1 root root  283 Apr 14 01:39 ceph.conf
-rw-r--r--   1 root root  595 Apr 14 01:12 ceph.pub
-rw-r--r--   1 root root   92 Feb  8 19:33 rbdmap
```

#### 3.传输客户端keyring文件

```bash
root@ubuntu-admin:/etc/ceph# rsync -avz root@172.31.0.121:/etc/ceph/ceph.client.xiaowu.keyring /etc/ceph/
receiving incremental file list
ceph.client.xiaowu.keyring

sent 43 bytes  received 241 bytes  568.00 bytes/sec
total size is 122  speedup is 0.43
```

#### 4.验证文件有效性

```bash
root@ubuntu-admin:/etc/ceph# ceph --user xiaowu -s
  cluster:
    id:     b227ebe2-f9b8-11ee-826d-15b492408c47
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ceph-node-01,ceph-node-02,ceph-node-03 (age 22h)
    mgr: ceph-node-01.doogpo(active, since 22h), standbys: ceph-node-02.nlocus
    mds: 1/1 daemons up, 2 standby
    osd: 3 osds: 3 up (since 21h), 3 in (since 21h)
    rgw: 3 daemons active (3 hosts, 1 zones)

  data:
    volumes: 1/1 healthy
    pools:   8 pools, 210 pgs
    objects: 346 objects, 19 MiB
    usage:   763 MiB used, 2.7 TiB / 2.7 TiB avail
    pgs:     210 active+clean
```

#### 5.卸载admin磁盘挂载和映射

```bash
root@ubuntu-admin:/etc/ceph# umount /data0
root@ubuntu-admin:/etc/ceph# umount /data1
root@ubuntu-admin:/etc/ceph# df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           392M  1.1M  391M   1% /run
/dev/sda2       196G  8.0G  178G   5% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           392M  4.0K  392M   1% /run/user/0

root@ubuntu-admin:/etc/ceph# rbd -p myrbd1 unmap myimg1
root@ubuntu-admin:/etc/ceph# rbd -p myrbd1 unmap myimg2
root@ubuntu-admin:/etc/ceph# fdisk -l | grep rbd
```

#### 6.使用普通用户挂在

```bash
root@ubuntu-admin:/etc/ceph# rbd --user xiaowu -p myrbd1 map myimg1
/dev/rbd0
rbd: --user is deprecated, use --id
root@ubuntu-admin:/etc/ceph# rbd --id xiaowu -p myrbd1 map myimg2
/dev/rbd1
root@ubuntu-admin:/etc/ceph# fdisk -l | grep rbd
Disk /dev/rbd0: 5 GiB, 5368709120 bytes, 10485760 sectors
Disk /dev/rbd1: 3 GiB, 3221225472 bytes, 6291456 sector
```

#### 7.再次挂载使用

```bash
root@ubuntu-admin:/etc/ceph# mount /dev/rbd0 /data0
root@ubuntu-admin:/etc/ceph# mount /dev/rbd1 /data1
root@ubuntu-admin:/etc/ceph# ll /data0/
total 8
drwxr-xr-x  2 root root   20 Apr 14 23:29 ./
drwxr-xr-x 22 root root 4096 Apr 14 23:08 ../
-rw-r--r--  1 root root 1905 Apr 14 23:11 passwd
root@ubuntu-admin:/etc/ceph# ll /data1/
total 8
drwxr-xr-x  2 root root   20 Apr 14 23:11 ./
drwxr-xr-x 22 root root 4096 Apr 14 23:08 ../
-rw-r--r--  1 root root 1905 Apr 14 23:11 passwd
```

### 5、验证ceph内核模块

>挂载 rbd 之后系统内核会自动加载 libceph.ko 模块

```bash
root@ubuntu-admin:/etc/ceph# lsmod | grep ceph
libceph               450560  1 rbd
libcrc32c              16384  4 btrfs,xfs,raid456,libceph
root@ubuntu-admin:/etc/ceph# modinfo libceph
filename:       /lib/modules/5.15.0-102-generic/kernel/net/ceph/libceph.ko
license:        GPL
description:    Ceph core library
author:         Patience Warnick <patience@newdream.net>
author:         Yehuda Sadeh <yehuda@hq.newdream.net>
author:         Sage Weil <sage@newdream.net>
srcversion:     0ED50A7A4F60A156737704A
depends:        libcrc32c
retpoline:      Y
intree:         Y
name:           libceph
vermagic:       5.15.0-102-generic SMP mod_unload modversions
sig_id:         PKCS#7
signer:         Build time autogenerated kernel key
sig_key:        2E:23:16:41:BE:E2:75:E4:E8:36:37:AE:9D:F0:67:4D:5A:8B:52:0F
sig_hashalgo:   sha512
signature:      3D:57:67:43:2A:41:36:6A:1A:87:01:9F:99:7C:27:B8:DA:1F:AB:59:
                55:CF:7B:CC:99:22:90:4F:FF:15:65:C2:58:09:94:0D:85:9A:64:77:
                B7:E1:0A:1D:46:98:AE:56:BD:D0:9B:FF:36:54:72:1D:70:27:ED:72:
                8E:53:99:E6:D8:2F:73:65:0B:DA:76:34:86:B5:3F:D6:03:70:15:AB:
                D4:4B:23:8F:E3:82:0E:8E:A8:3F:46:EA:C1:A1:62:F3:F8:2A:F1:A3:
                7A:5D:DA:08:C8:63:06:25:80:5C:AD:2B:0A:50:6F:3C:54:64:A7:89:
                C4:42:97:6D:E2:89:75:B6:66:2F:86:DC:08:FC:EE:C2:3F:ED:53:78:
                D6:E0:0B:31:47:14:63:24:2B:08:BB:2C:23:9C:0D:F0:52:A1:EF:0B:
                D6:FC:20:3B:82:10:57:C9:6E:C0:5D:D6:C5:B0:35:FA:C4:7C:51:A0:
                31:25:23:6C:CE:0D:02:A3:E3:D7:D7:C7:BA:66:BC:65:AE:AA:DE:E4:
                C6:A6:C3:CD:A9:26:12:05:D2:CB:2A:05:1F:E5:C7:58:CE:BD:DA:BC:
                BB:D2:66:32:FC:ED:FC:A5:CD:5A:16:CB:31:A7:F0:6B:8A:4A:0F:08:
                95:6A:43:47:75:94:6E:A5:8F:C6:96:A7:74:3D:3D:6C:4C:76:34:E6:
                B1:F4:B8:2E:C8:3F:22:C1:BC:FA:8C:B1:A6:D7:F9:6A:7E:C7:04:38:
                F4:FB:D6:3B:6D:3B:F4:C7:5C:85:16:9B:38:1C:96:66:08:E6:F9:E9:
                EE:A7:4F:06:42:CF:9B:37:11:75:0D:B6:D1:E6:19:DC:55:AF:09:2F:
                DF:30:28:65:5E:01:FF:B3:8E:1B:B9:C9:CF:29:DE:26:9D:25:2A:5E:
                91:0E:AF:28:BA:DD:3E:46:9E:2D:90:49:9A:09:1A:C5:52:52:ED:96:
                1D:5C:90:26:D8:3D:B0:CF:93:60:28:E4:13:AD:AF:52:D5:FD:54:A1:
                80:45:37:11:2A:4B:76:14:22:79:4E:0C:2B:6B:6B:A3:C0:63:B9:42:
                E6:28:81:4D:0C:35:2D:A9:3E:95:83:D8:13:07:8D:07:9C:68:B6:FF:
                E9:C2:F6:F5:AF:6D:9C:37:76:5D:43:7B:6E:8F:4C:28:4E:2A:6D:28:
                72:9C:46:DA:06:AA:BF:39:AA:23:2D:66:F9:54:4C:5D:F5:16:F0:34:
                8D:6A:C1:9A:2A:5A:56:8A:6D:D0:73:45:01:29:0A:97:76:21:17:53:
                16:95:75:4B:3D:20:8B:F1:FF:B4:80:0D:0C:3C:01:A6:02:35:1E:C3:
                42:28:4E:6E:F9:AD:8E:F5:31:23:46:C2
```

### 6、rbd镜像空间拉伸

>可以扩展空间，不建议缩小空间

#### 1.当前 rbd 镜像空间大小

```bash
root@ubuntu-admin:/etc/ceph# rbd ls --pool myrbd1 -l
NAME    SIZE   PARENT  FMT  PROT  LOCK
myimg1  5 GiB            2        excl
myimg2  3 GiB            2

```

#### 2.命令

```bash
root@ubuntu-admin:/etc/ceph# rbd resize --pool myrbd1 --image myimg1 --size 8G
Resizing image: 100% complete...done.
root@ubuntu-admin:/etc/ceph# rbd ls --pool myrbd1 -l
NAME    SIZE   PARENT  FMT  PROT  LOCK
myimg1  8 GiB            2        excl
myimg2  3 GiB            2
```

#### 3.客户端验证

```bash
root@ubuntu-admin:/etc/ceph# fdisk -l /dev/rbd0
Disk /dev/rbd0: 8 GiB, 8589934592 bytes, 16777216 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 65536 bytes / 65536 bytes
root@ubuntu-admin:/etc/ceph# df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           392M  1.1M  391M   1% /run
/dev/sda2       196G  8.0G  178G   5% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           392M  4.0K  392M   1% /run/user/0
/dev/rbd0       5.0G   69M  5.0G   2% /data0
/dev/rbd1       3.0G   54M  3.0G   2% /data1
```

#### 4.容量重新识别

```bash
# resize2fs /dev/rbd0 #在 node 节点对磁盘重新识别
# xfs_growfs /data0/ #在 node 挂载点对挂载点识别
root@ubuntu-admin:/etc/ceph# resize2fs /dev/rbd0
resize2fs 1.46.5 (30-Dec-2021)
resize2fs: Bad magic number in super-block while trying to open /dev/rbd0
Couldn't find valid filesystem superblock.
root@ubuntu-admin:/etc/ceph# xfs_growfs /data0
meta-data=/dev/rbd0              isize=512    agcount=8, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=0 inobtcount=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 1310720 to 2097152
root@ubuntu-admin:/etc/ceph# df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           392M  1.1M  391M   1% /run
/dev/sda2       196G  8.0G  178G   5% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           392M  4.0K  392M   1% /run/user/0
/dev/rbd0       8.0G   90M  8.0G   2% /data0
/dev/rbd1       3.0G   54M  3.0G   2% /data1
```

#### 5.设置开机自动挂载

>/etc/rc.d/rc.local

```bash
rbd --user xiaowu -p myrbd1 map myimg2
mount /dev/rbd0 /data0/

chmod a+x /etc/rc.d/rc.local

reboot
```

**验证**

```bash
root@ubuntu-admin:/etc/ceph# rbd showmapped
id  pool    namespace  image   snap  device
0   myrbd1             myimg1  -     /dev/rbd0
1   myrbd1             myimg2  -     /dev/rbd1
root@ubuntu-admin:/etc/ceph# df -TH
Filesystem     Type   Size  Used Avail Use% Mounted on
tmpfs          tmpfs  411M  1.1M  410M   1% /run
/dev/sda2      ext4   211G  8.6G  191G   5% /
tmpfs          tmpfs  2.1G     0  2.1G   0% /dev/shm
tmpfs          tmpfs  5.3M     0  5.3M   0% /run/lock
tmpfs          tmpfs  411M  4.1k  411M   1% /run/user/0
/dev/rbd0      xfs    8.6G   95M  8.5G   2% /data0
/dev/rbd1      xfs    3.3G   57M  3.2G   2% /data1
```

### 7、卸载rbd镜像

```bash
root@ubuntu-admin:/etc/ceph# umount /data0
root@ubuntu-admin:/etc/ceph# umount /data1
root@ubuntu-admin:/etc/ceph# rbd --user xiaowu -p myrbd1 unmap myimg1
rbd: --user is deprecated, use --id
root@ubuntu-admin:/etc/ceph# rbd --id xiaowu -p myrbd1 unmap myimg2
```

### 8、删除rbd镜像

>镜像删除后数据也会被删除而且是无法恢复，因此在执行删除操作的时候要慎重。

```bash
root@ubuntu-admin:/etc/ceph# rbd help rm
usage: rbd rm [--pool <pool>] [--namespace <namespace>] [--image <image>]
              [--no-progress]
              <image-spec>

Delete an image.

Positional arguments
  <image-spec>         image specification
                       (example: [<pool-name>/[<namespace>/]]<image-name>)

Optional arguments
  -p [ --pool ] arg    pool name
  --namespace arg      namespace name
  --image arg          image name
  --no-progress        disable progress output

root@ubuntu-admin:/etc/ceph# rbd rm --pool myrbd1 --image myimg1
Removing image: 100% complete...done.
root@ubuntu-admin:/etc/ceph# rbd rm --pool myrbd1 --image myimg2
Removing image: 100% complete...done.
```

### 9、rbd 镜像回收站机制

>删除的镜像数据无法恢复，但是还有另外一种方法可以先把镜像移动到回收站，后期确认删除的时候再从回收站删除即可。

```bash
# 查看帮助
rbd help trash
# 查看状态
rbd status --pool rbd-data1 --image data-img2
# 将进行移动到回收站：
rbd trash move --pool rbd-data1 --image data-img2
# 查看回收站的镜像
rbd trash list --pool rbd-data1
# 从回收站删除镜像
rbd trash remove --pool rbd-data1 --image data-img2 --image-id d42b6b8b4567
# 还原镜像
bd trash restore --pool rbd-data1 --image data-img2 --image-id d42b6b8b4567
```

### 10、镜像快照

#### 1.创建快照

```bash
# rbd help snap
    snap create (snap add) #创建快照
    snap limit clear #清除镜像的快照数量限制
    snap limit set #设置一个镜像的快照上限
    snap list (snap ls) #列出快照
    snap protect #保护快照被删除
    snap purge #删除所有未保护的快照
    snap remove (snap rm) #删除一个快照
    snap rename #重命名快照
    snap rollback (snap revert) #还原快照
    snap unprotect #允许一个快照被删除(取消快照保护
```

```bash
#创建快照
rbd snap create --pool rbd-data1 --image data-img2 --snap img2-snap-20201215
# 验证快照
rbd snap list --pool rbd-data1 --image data-img2
```

#### 2.删除数据还原快照

```bash
#客户端删除数据
[root@ceph-client2 ~]# rm -rf /data/passwd
#验证数据
[root@ceph-client2 ~]# ll /data/
total 1544
drwx------ 2 root root 16384 Dec 15 11:17 lost+found
-rw------- 1 root root 1563590 Dec 15 14:27 messages
#卸载 rbd
[root@ceph-client2 ~]# umount /data
[root@ceph-client2 ~]# rbd unmap /dev/rbd0
#回滚命令:
[cephadmin@ceph-deploy ceph-cluster]$ rbd help snap rollback
usage: rbd snap rollback [--pool <pool>] [--image <image>] [--snap <snap>] [--no-progress] <snap-spec>
#回滚快照
[cephadmin@ceph-deploy ceph-cluster]$ rbd snap rollback --pool rbd-data1 --image data-img2 --snap img2-snap-20201215
Rolling back to snapshot: 100% complete...done.
```

#### 3.客户端验证快照

```bash
#客户端映射 rbd
[root@ceph-client2 ~]# rbd --user xiaowu -p rbd-data1 map data-img2
/dev/rbd0
#客户端挂载 rbd
[root@ceph-client2 ~]# mount /dev/rbd0 /data/
#客户端验证数据
[root@ceph-client2 ~]# ll /data/
total 1548
drwx------ 2 root root 16384 Dec 15 11:17 lost+found
-rw------- 1 root root 1563590 Dec 15 14:27 messages
-rw-r--r-- 1 root root 923 Dec 15 11:28 passwd
```

#### 4.删除指定快照

```bash
#删除指定快照
[cephadmin@ceph-deploy ceph-cluster]$ rbd snap remove --pool rbd-data1 --image data-img2 --snap img2-snap-20201215
Removing snap: 100% complete...done.
#验证快照是否删除
[cephadmin@ceph-deploy ceph-cluster]$ rbd snap list --pool rbd-data1 --image data-img2
```

#### 5.快照数量限制

```bash
#设置与修改快照数量限制
[cephadmin@ceph-deploy ceph-cluster]$ rbd snap limit set --pool rbd-data1 --image data-img2 --limit 30
[cephadmin@ceph-deploy ceph-cluster]$ rbd snap limit set --pool rbd-data1 --image data-img2 --limit 20
[cephadmin@ceph-deploy ceph-cluster]$ rbd snap limit set --pool rbd-data1 --image data-img2 --limit 15
#清除快照数量限制
[cephadmin@ceph-deploy ceph-cluster]$ rbd snap limit clear --pool rbd-data1 --image data-img
```