# etcd管理及备份

## 一、简介

### 1、介绍

>​    etcd是CoreOS团队于2013年6月发起的开源项目，它的目标是构建一个高可用的分布式键值(key-value)数据库。etcd内部采用raft协议作为一致性算法，etcd基于Go语言实现
>
>​    和redis存储方式不同，etcd的key是访问k8s某个资源的路径(类似k8s不同名称空间之间访问)

### 2、相关网站

>官方网站：https://etcd.io/
>github地址：https://github.com/etcd-io/etcd
>https://etcd.io/docs/v3.4/op-guide/hardware/ #官方硬件推荐
>
>​     etcd虽然不是k8s集群组件，但是他是集群的核心
>
>​        生产CPU最低4G
>
>​        生产内存最低8G
>
>​        硬盘使用固态
>
>​     node
>
>​        系统盘：机械
>
>​        数据盘：ssd
>
>​    master
>
>​        cpu足够强

### 3、属性

>完全复制：集群中的每个节点都可以使用完整的存档
>高可用性：Etcd可用于避免硬件的单点故障或网络问题
>一致性：每次读取都会返回跨多主机的最新写入
>简单：包括一个定义良好、面向用户的API（gRPC）
>安全：实现了带有可选的客户端证书身份验证的自动化TLS
>快速：每秒10000次写入的基准速度
>可靠：使用Raft算法实现了存储的合理分布Etcd的工作原理

## 二、service参数

### 1、默认参数解读

```bash
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd #工作目录
ExecStart=/usr/local/bin/etcd \ #二进制文件路径
  --name=etcd-172.31.7.106 \ #当前node 名称
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls=https://172.31.7.106:2380 \ #通告自己的集群端口
  --listen-peer-urls=https://172.31.7.106:2380 \ #集群之间通讯端口
  --listen-client-urls=https://172.31.7.106:2379,http://127.0.0.1:2379 \ #客户端访问地址
  --advertise-client-urls=https://172.31.7.106:2379 \ #通告自己的客户端端口
  --initial-cluster-token=etcd-cluster-0 \ #创建集群使用的token，一个集群内的节点保持一致
  --initial-cluster=etcd-172.31.7.106=https://172.31.7.106:2380,etcd-172.31.7.107=https://172.31.7.107:2380,etcd-172.31.7.108=https://172.31.7.108:2380 \ #集群所有的节点信息
  --initial-cluster-state=new \  #新建集群的时候的值为new,如果是已经存在的集群为existing
  --data-dir=/var/lib/etcd \ #数据目录路径
  --wal-dir= \
  --snapshot-count=50000 \ # 快照
  --auto-compaction-retention=1 \ # 压缩周期，单位小时。第一次压缩等待1小。以后每1时*10%=6分钟压缩一次，建议改成10
  --auto-compaction-mode=periodic \ # 周期性压缩，不建议开启压缩，占用CPU资源
  --max-request-bytes=10485760 \ # request size limit（请求的最大字节数，默认一个key最大1.5Mib，官方推荐最大10Mib）
  --quota-backend-bytes=8589934592 # storage size limit（磁盘存储空间大小限制，默认2G。这个值超过8G会有警告信息）
Restart=always
RestartSec=15
LimitNOFILE=65536
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

> 数据写入机制：（类似bin-log）
>
> wal-预写日志：在插入数据的时候，先写完日志，然后在写数据。如果日志没有写入成功就相当于数据未写入完成

## 三、命令使用

>etcd有多个不同的API访问版本，v1版本已经废弃，etcd v2 和 v3 本质上是共享同一套 raft 协议代码的两个独立的应用，接口不一样，存储不一样，
>数据互相隔离。也就是说如果从 Etcd v2 升级到 Etcd v3，原来v2 的数据还是只能用 v2 的接口访问，v3 的接口创建的数据也只能访问通过 v3 的接
>口访问。



```bash
WARNING:
    Environment variable ETCDCTL_API is not set; defaults to etcdctl v2. #默认使用V2版本
    Set environment variable ETCDCTL_API=3 to use v3 API or ETCDCTL_API=2 to use v2 API. #设置API版本

root@k8s-etcd1:~# ETCDCTL_API=3 etcdctl member list
```

### 1、命令使用帮助

```bash
root@k8s-etcd1:~# ETCDCTL_API=3 etcdctl --help
root@k8s-etcd1:~# ETCDCTL_API=3 etcdctl member --help
NAME:
        member - Membership related commands

USAGE:
        etcdctl member <subcommand> [flags]

API VERSION:
        3.5


COMMANDS:
        add     Adds a member into the cluster
        list    Lists all members in the cluster
        promote Promotes a non-voting member in the cluster
        remove  Removes a member from the cluster
        update  Updates a member in the cluster

OPTIONS:
  -h, --help[=false]    help for member

GLOBAL OPTIONS:
      --cacert=""                               verify certificates of TLS-enabled secure servers using this CA bundle
      --cert=""                                 identify secure client using this TLS certificate file
      --command-timeout=5s                      timeout for short running command (excluding dial timeout)
      --debug[=false]                           enable client-side debug logging
      --dial-timeout=2s                         dial timeout for client connections
  -d, --discovery-srv=""                        domain name to query for SRV records describing cluster endpoints
      --discovery-srv-name=""                   service name to query when using DNS discovery
      --endpoints=[127.0.0.1:2379]              gRPC endpoints
      --hex[=false]                             print byte strings as hex encoded strings
      --insecure-discovery[=true]               accept insecure SRV records describing cluster endpoints
      --insecure-skip-tls-verify[=false]        skip server certificate verification (CAUTION: this option should be enabled only for testing purposes)
      --insecure-transport[=true]               disable transport security for client connections
      --keepalive-time=2s                       keepalive time for client connections
      --keepalive-timeout=6s                    keepalive timeout for client connections
      --key=""                                  identify secure client using this TLS key file
      --password=""                             password for authentication (if this option is used, --user option shouldn't include password)
      --user=""                                 username[:password] for authentication (prompt if password is not supplied)
  -w, --write-out="simple"                      set the output format (fields, json, protobuf, simple, table)
```

### 2、查看集群节点

```bash
root@k8s-etcd1:~# ETCDCTL_API=3 etcdctl member list
360a6fc679b60f19, started, etcd-172.31.7.107, https://172.31.7.107:2380, https://172.31.7.107:2379, false
b786e8f07f81b519, started, etcd-172.31.7.108, https://172.31.7.108:2380, https://172.31.7.108:2379, false
e7e623d2f4a7c9e4, started, etcd-172.31.7.106, https://172.31.7.106:2380, https://172.31.7.106:2379, false
```

### 3、查看etcd所有节点状态

```bash
root@k8s-etcd1:~# export NODE_IPS="172.31.7.106 172.31.7.107 172.31.7.108"
root@k8s-etcd1:~# for ip in ${NODE_IPS};
do
     ETCDCTL_API=3 /usr/local/bin/etcdctl --endpoints=https://${ip}:2379 --cacert=/etc/kubernetes/ssl/ca.pem --cert=/etc/kubernetes/ssl/etcd.pem --key=/etc/kubernetes/ssl/etcd-key.pem endpoint health;
done

# 结果
https://172.31.7.106:2379 is healthy: successfully committed proposal: took = 8.945324ms
https://172.31.7.107:2379 is healthy: successfully committed proposal: took = 9.3052ms
https://172.31.7.108:2379 is healthy: successfully committed proposal: took = 9.031669ms
```

### 4、显示成员信息

```bash
root@k8s-etcd1:~# ETCDCTL_API=3 /usr/local/bin/etcdctl --write-out=table member list --endpoints=https://172.31.7.106:2379 --cacert=/etc/kubernetes/ssl/ca.pem --cert=/etc/kubernetes/ssl/etcd.pem --key=/etc/kubernetes/ssl/etcd-key.pem

# 结果
+------------------+---------+-------------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |       NAME        |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+-------------------+---------------------------+---------------------------+------------+
| 360a6fc679b60f19 | started | etcd-172.31.7.107 | https://172.31.7.107:2380 | https://172.31.7.107:2379 |      false |
| b786e8f07f81b519 | started | etcd-172.31.7.108 | https://172.31.7.108:2380 | https://172.31.7.108:2379 |      false |
| e7e623d2f4a7c9e4 | started | etcd-172.31.7.106 | https://172.31.7.106:2380 | https://172.31.7.106:2379 |      false |
+------------------+---------+-------------------+---------------------------+---------------------------+------------+
```

### 5、以表格方式显示节点详细状态

```bash
root@k8s-etcd1:~# export NODE_IPS="172.31.7.106 172.31.7.107 172.31.7.108"
root@k8s-etcd1:~# for ip in ${NODE_IPS}; do ETCDCTL_API=3 /usr/local/bin/etcdctl --write-out=table endpoint status --endpoints=https://${ip}:2379 --cacert=/etc/kubernetes/ssl/ca.pem --cert=/etc/kubernetes/ssl/etcd.pem --key=/etc/kubernetes/ssl/etcd-key.pem; done

# 结果
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://172.31.7.106:2379 | e7e623d2f4a7c9e4 |   3.5.4 |  3.5 MB |     false |      false |        11 |     108904 |             108904 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://172.31.7.107:2379 | 360a6fc679b60f19 |   3.5.4 |  3.4 MB |      true |      false |        11 |     108904 |             108904 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://172.31.7.108:2379 | b786e8f07f81b519 |   3.5.4 |  3.4 MB |     false |      false |        11 |     108904 |             108904 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

```

### 6、查看etcd数据信息

#### 1.以路径的方式所有key信息

```bash
ETCDCTL_API=3 etcdctl get / --prefix --keys-only
```

#### 2.查看对应资源的信息

```bash
ETCDCTL_API=3 etcdctl get / --prefix --keys-only | grep pod
ETCDCTL_API=3 etcdctl get / --prefix --keys-only | grep namespaces
ETCDCTL_API=3 etcdctl get / --prefix --keys-only | grep deployment
ETCDCTL_API=3 etcdctl get / --prefix --keys-only | grep calico
```

### 7、etcd增删该查

```bash
#添加数据
root@etcd1:~# ETCDCTL_API=3 /usr/bin/etcdctl put /name "tom"
OK

#查询数据
root@etcd1:~# ETCDCTL_API=3 /usr/bin/etcdctl get /name
/name
tom

#改动数据，#直接覆盖就是更新数据
root@etcd1:~# ETCDCTL_API=3 /usr/bin/etcdctl put /name "jack"
OK

#验证改动
root@etcd1:~# ETCDCTL_API=3 /usr/bin/etcdctl get /name
/name
jack

#删除数据
root@etcd1:~# ETCDCTL_API=3 /usr/bin/etcdctl del /name
1
root@etcd1:~# ETCDCTL_API=3 /usr/bin/etcdctl get /name
```

### 8、集群碎片整理

> 一般很少使用，
>
> 可以写个定时任务，每周，每月整理一下

```bash
ETCDCTL_API=3 /usr/local/bin/etcdctl defrag --cluster --endpoints=https://172.31.7.106:2379 --cacert=/etc/kubernetes/ssl/ca.pem --cert=/etc/kubernetes/ssl/etcd.pem --key=/etc/kubernetes/ssl/etcd-key.pem
```

## 四、etcd数据watch机制

>基于不断监看数据，发生变化就主动触发通知客户端，Etcd v3 的watch机制支持watch某个固定的key，也支持watch一个范围

```bash
# 在etcd node1上watch一个key，没有此key也可以执行watch，后期可以再创建：
root@k8s-etcd1:~# ETCDCTL_API=3 /usr/bin/etcdctl watch /data
#在etcd node2修改数据，验证etcd node1是否能够发现数据变化
root@k8s-etcd2:~# ETCDCTL_API=3 /usr/bin/etcdctl put /data "data v1"
OK
root@k8s-etcd2:~# ETCDCTL_API=3 /usr/bin/etcdctl put /data "data v2"
OK


#验证etcd node1：
root@k8s-etcd1:~# ETCDCTL_API=3 /usr/bin/etcdctl watch /data
PUT
/data
data v1
PUT
/data
data v2
```

## 五、etcd单节点 V3 API版本数据备份与恢复

> WAL是write ahead log的缩写，顾名思义，也就是在执行真正的写操作之前先写一个日志，预写日志。
> wal: 存放预写式日志,最大的作用是记录了整个数据变化的全部历程。在etcd中，所有数据的修改在提交前，都要先写入到WAL中。

### 1、备份

```bash
root@k8s-etcd2:~# ETCDCTL_API=3 etcdctl snapshot save snapshot.db
Snapshot saved at snapshot.db
```

### 2、恢复

```bash
root@k8s-etcd2:~# ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir=/opt/etcd-testdir #将数据恢复到一个新的不存在的目录中
```

> 修改service参数重启etcd

### 3、自动备份数据

```bash
root@k8s-etcd2:~# mkdir /data/etcd-backup-dir/ -p
root@k8s-etcd2:~# cat script.sh
#!/bin/bash
source /etc/profile
DATE=`date +%Y-%m-%d_%H-%M-%S`
ETCDCTL_API=3 /usr/bin/etcdctl snapshot save /data/etcd-backup-dir/etcd-snapshot-${DATE}.db
```

## 六、etcd集群 V3 API版本数据备份与恢复（kubeasz安装的）

> 全量备份，全量恢复

### 1、备份恢复方式

```bash
# 使用帮助
# root@k8s-master1:/etc/kubeasz# ./ezctl --help
Usage: ezctl COMMAND [args]
-------------------------------------------------------------------------------------
Cluster setups:
    list                             to list all of the managed clusters
    checkout    <cluster>            to switch default kubeconfig of the cluster
    new         <cluster>            to start a new k8s deploy with name 'cluster'
    setup       <cluster>  <step>    to setup a cluster, also supporting a step-by-step way
    start       <cluster>            to start all of the k8s services stopped by 'ezctl stop'
    stop        <cluster>            to stop all of the k8s services temporarily
    upgrade     <cluster>            to upgrade the k8s cluster
    destroy     <cluster>            to destroy the k8s cluster
    backup      <cluster>            to backup the cluster state (etcd snapshot)
    restore     <cluster>            to restore the cluster state from backups
    start-aio                        to quickly setup an all-in-one cluster with 'default' settings

Cluster ops:
    add-etcd    <cluster>  <ip>      to add a etcd-node to the etcd cluster
    add-master  <cluster>  <ip>      to add a master node to the k8s cluster
    add-node    <cluster>  <ip>      to add a work node to the k8s cluster
    del-etcd    <cluster>  <ip>      to delete a etcd-node from the etcd cluster
    del-master  <cluster>  <ip>      to delete a master node from the k8s cluster
    del-node    <cluster>  <ip>      to delete a work node from the k8s cluster

Extra operation:
    kcfg-adm    <cluster>  <args>    to manage client kubeconfig of the k8s cluster

Use "ezctl help <command>" for more information about a given command.


# 备份
 ./ezctl backup k8s-cluster01
 
# 恢复，恢复期间，api-server、调度、控制等读写etcd的服务不可用，建议在业务低峰期间操作
./ezctl restore k8s-cluster01
```

### 2、备份原理

```bash
root@k8s-master1:/etc/kubeasz/playbooks# cat 94.backup.yml
# cluster-backup playbook
# read the guide: 'op/cluster_restore.md'

- hosts:
  - localhost
  tasks:
  # step1: find a healthy member in the etcd cluster
  - name: set NODE_IPS of the etcd cluster
    set_fact: NODE_IPS="{% for host in groups['etcd'] %}{{ host }} {% endfor %}"

  - name: get etcd cluster status
    shell: 'for ip in {{ NODE_IPS }};do \
              ETCDCTL_API=3 {{ base_dir }}/bin/etcdctl \
              --endpoints=https://"$ip":2379 \
              --cacert={{ cluster_dir }}/ssl/ca.pem \
              --cert={{ cluster_dir }}/ssl/etcd.pem \
              --key={{ cluster_dir }}/ssl/etcd-key.pem \
              endpoint health; \
            done'
    register: ETCD_CLUSTER_STATUS
    ignore_errors: true

  - debug: var="ETCD_CLUSTER_STATUS"

  - name: get a running ectd node
    shell: 'echo -e "{{ ETCD_CLUSTER_STATUS.stdout }}" \
             "{{ ETCD_CLUSTER_STATUS.stderr }}" \
             |grep "is healthy"|sed -n "1p"|cut -d: -f2|cut -d/ -f3'
    register: RUNNING_NODE

  - debug: var="RUNNING_NODE.stdout"

  - name: get current time
    shell: "date +'%Y%m%d%H%M'"
    register: timestamp

  # step2: backup data on the healthy member
  - name: make a backup on the etcd node
    shell: "mkdir -p /etcd_backup && cd /etcd_backup && \
        ETCDCTL_API=3 {{ bin_dir }}/etcdctl snapshot save snapshot_{{ timestamp.stdout }}.db"
    args:
      warn: false
    delegate_to: "{{ RUNNING_NODE.stdout }}"

  - name: fetch the backup data
    fetch:
      src: /etcd_backup/snapshot_{{ timestamp.stdout }}.db
      dest: "{{ cluster_dir }}/backup/"
      flat: yes
    delegate_to: "{{ RUNNING_NODE.stdout }}"

  - name: update the latest backup
    shell: 'cd {{ cluster_dir }}/backup/ && /bin/cp -f snapshot_{{ timestamp.stdout }}.db snapshot.db'
```

### 3、恢复原理

```bash
root@k8s-master1:/etc/kubeasz/roles# cat cluster-restore/tasks/main.yml
- name: 停止ectd 服务
  service: name=etcd state=stopped

- name: 清除etcd 数据目录
  file: name={{ ETCD_DATA_DIR }}/member state=absent

- name: 生成备份目录
  file: name=/etcd_backup state=directory

- name: 准备指定的备份etcd 数据
  copy:
    src: "{{ cluster_dir }}/backup/{{ db_to_restore }}"
    dest: "/etcd_backup/snapshot.db"

- name: 清理上次备份恢复数据
  file: name=/etcd_backup/etcd-{{ inventory_hostname }}.etcd state=absent

- name: etcd 数据恢复
  shell: "cd /etcd_backup && \
        ETCDCTL_API=3 {{ bin_dir }}/etcdctl snapshot restore snapshot.db \
        --name etcd-{{ inventory_hostname }} \
        --initial-cluster {{ ETCD_NODES }} \
        --initial-cluster-token etcd-cluster-0 \
        --initial-advertise-peer-urls https://{{ inventory_hostname }}:2380"

- name: 恢复数据至etcd 数据目录
  shell: "cp -rf /etcd_backup/etcd-{{ inventory_hostname }}.etcd/member {{ ETCD_DATA_DIR }}/"

- name: 重启etcd 服务
  service: name=etcd state=restarted

- name: 以轮询的方式等待服务同步完成
  shell: "systemctl is-active etcd.service"
  register: etcd_status
  until: '"active" in etcd_status.stdout'
  retries: 8
  delay: 8
```

## 七、恢复流程

> 当etcd集群宕机数量超过集群总节点数一半以上的时候(如总数为三台宕机两台)，就会导致整个集群宕机，后期需要重新恢复数据，则恢复流程如下

```bash
1、恢复服务器系统
2、重新部署ETCD集群
3、停止kube-apiserver/controller-manager/scheduler/kubelet/kube-proxy
4、停止ETCD集群
5、各ETCD节点恢复同一份备份数据
6、启动各节点并验证ETCD集群
7、启动kube-apiserver/controller-manager/scheduler/kubelet/kube-proxy
8、验证k8s master状态及pod数据
```

## 八、ETCD集群节点添加与删除

```bash
./ezctl add-etcd 集群名称 主机IP
./ezctl del-etcd 集群名称 主机IP
```

