# ceph crush 进阶

## 一、介绍

### 1、ceph 集群中由 mon 服务器维护的的五种运行图

>Monitor map           #监视器运行图
>OSD map               #OSD 运行图
>PG map                 #PG 运行图
>Crush map             #(Controllers replication under scalable hashing #可控的、可复制的、可伸缩的一致性 hash 算法。crush 运行图,当新建存储池时会基于 OSD map 创建出新的 PG 组合列表用于存储数据
>MDS map               #cephfs metadata 运行图

>obj-->pg hash(oid)%pg=pgid
>Obj-->OSD crush 根据当前的 mon 运行图返回 pg 内的最新的 OSD 组合，数据即可开始往主的写然后往副本 OSD 同步

### 2、crush 算法针对目的节点的选择

>目前有 5 种算法来实现节点的选择，包括 Uniform、List、Tree、Straw、Straw2，早期版本使用的是 ceph 项目的发起者发明的算法 straw，目前已经发展社区优化的 straw2 版本。

### 3、straw(抽签算法)

>抽签是指挑取一个最长的签，而这个签值就是 OSD 的权重，让权重高的 OSD 分配较多的PG 以存储更多的数据

## 二、PG 与 OSD 映射调整

>默认情况下,crush 算法自行对创建的 pool 中的 PG 分配 OSD，但是可以手动基于权重设置crush 算法分配数据的倾向性，比如 1T 的磁盘权重是 1，2T 的就是 2，推荐使用相同大小的设备

### 1.查看当前状态

>weight 表示设备(device)的容量相对值，比如 1TB 对应 1.00，那么 500G 的 OSD 的 weight就应该是 0.5，weight 是基于磁盘空间分配 PG 的数量，让 crush 算法尽可能往磁盘空间大的 OSD 多分配 PG.往磁盘空间小的 OSD 分配较少的 PG

>Reweight 参数的目的是重新平衡 ceph 的 CRUSH 算法随机分配的 PG，默认的分配是概率上的均衡，即使 OSD 都是一样的磁盘空间也会产生一些 PG 分布不均匀的情况，此时可以通过调整 reweight 参数，让 ceph 集群立即重新平衡当前磁盘的 PG，以达到数据均衡分布的目的，REWEIGHT 是 PG 已经分配完成，要在 ceph 集群重新平衡 PG 的分布

```bash
ceph osd df
```

### 2.修改 WEIGHT 并验证

```bash
#修改某个指定 ID 的 osd 的权重，会触发数据的重新分配
ceph osd crush reweight osd.10 1.5

# 验证
ceph osd df
```

### 3.修改 REWEIGHT 并验证

>OSD 的 REWEIGHT 的值默认为 1，值可以调整，范围在 0~1 之间，值越低 PG 越小，如果调整了任何一个 OSD 的 REWEIGHT 值，那么 OSD 的 PG 会立即和其它 OSD 进行重新平衡，即数据的重新分配，用于当某个 OSD 的 PG 相对较多需要降低其 PG 数量的场景

```bash
ceph osd reweight 9 0.6
```

## 三、crush 运行图管理

>通过工具将 ceph 的 crush 运行图导出并进行编辑，然后导入

### 1、导出 crush 运行图

>导出的 crush 运行图为二进制格式，无法通过文本编辑器直接打开，需要使用 crushtool工具转换为文本格式后才能通过 vim 等文本编辑宫工具打开和编辑

```bash
mkdir /data/ceph -p
ceph osd getcrushmap -o /data/ceph/crushmap-v1
```

### 2、将运行图转换为文本

>导出的运行图不能直接编辑，需要转换为文本格式再进行查看与编辑

```bash
crushtool -d /data/ceph/crushmap-v1 > /data/ceph/crushmap-v1.txt
```

### 3、编辑文本

```bash
vim /data/ceph/crushmap-v1.txt
# begin crush map #可调整的 crush map 参数
tunable choose_local_tries 0
...

# devices #当前的设备列表
device 0 osd.0 class hdd
device 1 osd.1 class hdd


# types #当前支持的 bucket 类型
type 0 osd #osd 守护进程，对应到一个磁盘设备
type 1 host #一个主机
type 2 chassis #刀片服务器的机箱
type 3 rack #包含若干个服务器的机柜/机架
type 4 row #包含若干个机柜的一排机柜
type 5 pdu #机柜的接入电源插座
type 6 pod #一个机房中的若干个小房间
type 7 room #包含若干机柜的房间，一个数据中心有好多这样的房间组成
type 8 datacenter #一个数据中心或 IDC
type 9 region #一个区域，比如 AWS 宁夏中卫数据中心
type 10 root #bucket 分层的最顶部，根


# buckets
host ceph-node1 { 类型 Host 名称为 ceph-node1
    id -3 # do not change unnecessarily #ceph 生成的 OSD ID，非必要不要改
    id -4 class hdd # do not change unnecessarily
    # weight 0.488
    alg straw2 #crush 算法，管理 OSD 角色
    hash 0 # rjenkins1 #使用是哪个 hash 算法，0 表示选择 rjenkins1 这种 hash
    算法
    item osd.0 weight 0.098 #osd0 权重比例，crush 会自动根据磁盘空间计算，不同的磁盘空间的权重不一样
    item osd.1 weight 0.098
    item osd.2 weight 0.098
    item osd.3 weight 0.098
    item osd.4 weight 0.098
}
...


root default { #根的配置
    id -1 # do not change unnecessarily
    id -2 class hdd # do not change unnecessarily
    # weight 3.256
    alg straw2
    hash 0 # rjenkins1
    item ceph-node1 weight 0.488
    item ceph-node2 weight 0.488
    item ceph-node3 weight 0.488
    item ceph-node4 weight 0.488
}

# rules
    rule replicated_rule { #副本池的默认配置
    id 0
    type replicated
    min_size 1
    max_size 10 #默认最大副本为 10
    step take default #基于 default 定义的主机分配 OSD
    step chooseleaf firstn 0 type host #选择主机，故障域类型为主机
    step emit #弹出配置即返回给客户端
}
rule erasure-code { #纠删码池的默认配置
    id 1
    type erasure
    min_size 3
    max_size 4
    step set_chooseleaf_tries 5
    step set_choose_tries 100
    step take default
    step chooseleaf indep 0 type host
    step emit
}
```

### 4、将文本转换为 crush 格式

```bash
crushtool -c /data/ceph/crushmap-v1.txt -o /data/ceph/crushmap-v2
```

### 5、导入新的 crush

>导入的运行图会立即覆盖原有的运行图并立即生效

```bash
ceph osd setcrushmap -i /data/ceph/crushmap-v2
```

### 6、验证 crush 运行图是否生效

```bash
ceph osd crush rule dump
```

## 四、crush 数据分类管理

>Ceph crush 算法分配的 PG 的时候可以将 PG 分配到不同主机的 OSD 上，以实现以主机为单位的高可用，这也是默认机制，但是无法保证不同 PG 位于不同机柜或者机房的主机，如果要实现基于机柜或者是更高级的 IDC 等方式的数据高可用，而且也不能实现 A 项目的数据在 SSD，B 项目的数据在机械盘,如果想要实现此功能则需要导出 crush 运行图并手动编辑，之后再导入并覆盖原有的 crush 运行图

### 1、导出 cursh 运行图

```bash
ceph osd getcrushmap -o /data/ceph/crushmap-v3
```

### 2、将运行图转换为文本

```bash
crushtool -d /data/ceph/crushmap-v3 > /data/ceph/crushmap-v3.txt
```

### 3、添加自定义配置

```bash
cat /data/ceph/crushmap-v3.txt
# begin crush map
tunable choose_local_tries 0 # 本地选择尝试次数设置为0，意味着不进行本地选择尝试。
tunable choose_local_fallback_tries 0 # 本地回退选择尝试次数设置为0，意味着不进行本地回退选择尝试。
tunable choose_total_tries 50 # 总选择尝试次数设置为50，这是在选择算法中尝试的总次数。
tunable chooseleaf_descend_once 1 # 选择leaf（例如主机或OSD）时，设置为1表示只下降一次
tunable chooseleaf_vary_r 1 # 选择leaf时，允许变化的r（代表副本数或权重）
tunable chooseleaf_stable 1 # 设置为1表示在选择leaf时优先选择稳定的权重
tunable straw_calc_version 1 # 选择算法版本设置为1，这是Straw算法的版本
tunable allowed_bucket_algs 54 # 允许的桶算法设置为54，这指定了在CRUSH映射中可以使用的桶算法
# devices
device 0 osd.0 class hdd
device 1 osd.1 class hdd
device 2 osd.2 class hdd
device 3 osd.3 class hdd
device 4 osd.4 class hdd
device 5 osd.5 class hdd
device 6 osd.6 class hdd
device 7 osd.7 class hdd
device 8 osd.8 class hdd
device 9 osd.9 class hdd
device 10 osd.10 class hdd
device 11 osd.11 class hdd
device 12 osd.12 class hdd
device 13 osd.13 class hdd
device 14 osd.14 class hdd
device 15 osd.15 class hdd
device 16 osd.16 class hdd
device 17 osd.17 class hdd
device 18 osd.18 class hdd
device 19 osd.19 class hdd
# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 zone
type 10 region
type 11 root
# buckets
host ceph-node1 {
    id -3 # do not change unnecessarily
    id -4 class hdd # do not change unnecessarily
    # weight 0.490
    alg straw2
    hash 0 # rjenkins1
    item osd.0 weight 0.098
    item osd.1 weight 0.098
    item osd.2 weight 0.098
    item osd.3 weight 0.098
    item osd.4 weight 0.098
}
host ceph-node2 {
    id -5 # do not change unnecessarily
    id -6 class hdd # do not change unnecessarily
    # weight 0.490
    alg straw2
    hash 0 # rjenkins1
    item osd.5 weight 0.098
    item osd.6 weight 0.098
    item osd.7 weight 0.098
    item osd.8 weight 0.098
    item osd.9 weight 0.098
}
host ceph-node3 {
    id -7 # do not change unnecessarily
    id -8 class hdd # do not change unnecessarily
    # weight 1.792
    alg straw2
    hash 0 # rjenkins1
    item osd.10 weight 1.400
    item osd.11 weight 0.098
    item osd.12 weight 0.098
    item osd.13 weight 0.098
    item osd.14 weight 0.098
}
host ceph-node4 {
    id -9 # do not change unnecessarily
    id -10 class hdd # do not change unnecessarily
    # weight 0.490
    alg straw2
    hash 0 # rjenkins1
    item osd.15 weight 0.098
    item osd.16 weight 0.098
    item osd.17 weight 0.098
    item osd.18 weight 0.098
    item osd.19 weight 0.098
}
root default {
    id -1 # do not change unnecessarily
    id -2 class hdd # do not change unnecessarily
    # weight 3.255
    alg straw2
    hash 0 # rjenkins1
    item ceph-node1 weight 0.488
    item ceph-node2 weight 0.488
    item ceph-node3 weight 1.791
    item ceph-node4 weight 0.488
}
#xiaowu ssh node
host ceph-ssdnode1 {
    id -103 # do not change unnecessarily
    id -104 class hdd # do not change unnecessarily
    # weight 0.098
    alg straw2
    hash 0 # rjenkins1
    item osd.0 weight 0.098
}
host ceph-ssdnode2 {
    id -105 # do not change unnecessarily
    id -106 class hdd # do not change unnecessarily
    # weight 0.098
    alg straw2
    hash 0 # rjenkins1
    item osd.5 weight 0.098
}
host ceph-ssdnode3 {
    id -107 # do not change unnecessarily
    id -108 class hdd # do not change unnecessarily
    # weight 0.098
    alg straw2
    hash 0 # rjenkins1
    item osd.10 weight 0.098
}
host ceph-ssdnode4 {
    id -109 # do not change unnecessarily
    id -110 class hdd # do not change unnecessarily
    # weight 0.098
    alg straw2
    hash 0 # rjenkins1
    item osd.15 weight 0.098
}
#xiaowu bucket
root ssd {
    id -127 # do not change unnecessarily
    id -11 class hdd # do not change unnecessarily
    # weight 1.952
    alg straw
    hash 0 # rjenkins1
    item ceph-ssdnode1 weight 0.488
    item ceph-ssdnode2 weight 0.488
    item ceph-ssdnode3 weight 0.488
    item ceph-ssdnode4 weight 0.488
}
#xiaowu rules
rule xiaowu_ssd_rule {
    id 20
    type replicated
    min_size 1
    max_size 5
    step take ssd
    step chooseleaf firstn 0 type host
    step emit
}
# rules
rule replicated_rule {
    id 0
    type replicated
    min_size 1
    max_size 10
    step take default
    step chooseleaf firstn 0 type host
    step emit
}
rule erasure-code {
    id 1
    type erasure
    min_size 3
    max_size 4
    step set_chooseleaf_tries 5
    step set_choose_tries 100
    step take default
    step chooseleaf indep 0 type host
    step emit
}
# end crush map
```

### 4、转换为 crush 二进制格式

```bash
crushtool -c /data/ceph/crushmap-v3.txt -o /data/ceph/crushmap-v4
```

### 5、导入新的 crush 运行图

```bash
ceph osd setcrushmap -i /data/ceph/crushmap-v4
```

### 6、验证 crush 运行图是否生效

```bash
ceph osd crush rule dump
```

### 7、测试创建存储池

```bash
ceph osd pool create xiaowu-ssdpool 32 32 xiaowu_ssd_rule
```

### 8、验证 pgp 状态

```bash
ceph pg ls-by-pool xiaowu-ssdpool | awk '{print $1,$2,$15}'
```
