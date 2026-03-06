# Zookeeper集群运行

## 一、Zookeeper介绍

### 1、概述

> ​    ZooKeeper 是一个开源的分布式协调服务，是 Hadoop，HBase 和其他分布式框架使用的有组织服务的标准。
> 分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。

> ​    ZooKeeper是一个开源的分布式应用程序协调服务，是Google的Chubby一个开源的实现。ZooKeeper为分布式应用提供一致性服务，提供的功能包括：分布式同步（Distributed Synchronization）、命名服务（Naming Service）、集群维护（Group Maintenance）、分布式锁（Distributed Lock）等，简化分布式应用协调及其管理的难度，提供高性能的分布式服务。

> ​    ZooKeeper本身可以以单机模式安装运行，不过它的长处在于通过分布式ZooKeeper集群（一个Leader，多个Follower），基于一定的策略来保证ZooKeeper集群的稳定性和可用性，从而实现分布式应用的可靠性。

> ​    ZooKeeper在提供分布式锁等服务的时候需要过半数的节点可用。另外高可用的诉求来说节点的个数必须>1，所以ZooKeeper集群需要是>1的奇数节点。例如3、5、7等等。


### 2、集群角色说明

> ZooKeeper主要有领导者（Leader）、跟随者（Follower）和观察者（Observer）三种角色。

| 角色               | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| 领导者（Leader）   | 为客户端提供读和写的服务，负责投票的发起和决议，更新系统状态 |
| 跟随者（Follower） | 为客户端提供读服务，如果是写服务则转发给Leader，在选举过程中参与投票 |
| 观察者（Observer） | 为客户端提供读服务器，如果是写服务则转发给Leader。不参与选举过程中的投票，也不参与“过半写成功策略”，在不影响性能的情况下提升集群的读性能。此角色与zookeeper3.3系列新增的角色。 |



### 3、应用场景

```bash
分布式协调
分布式锁
元数据/配置信息管理
HA 高可用性
发布/订阅
负载均衡
Master 选举
```

#### 1.分布式协调

> ​    这个其实是 zookeeper 很经典的一个用法，简单来说，就好比，你 A 系统发送个请求到 mq，然后 B 系统消息消费之后处理了。那 A 系统如何知道 B 系统的处理结果？用 zookeeper 就可以实现分布式系统之间的协调工作。A 系统发送请求之后可以在 zookeeper 上对某个节点的值注册个监听器，一旦 B 系统处理完了就修改 zookeeper 那个节点的值，A 系统立马就可以收到通知，完美解决。

![2分布式协调](2分布式协调.png)



#### 2.分布式锁

> ​    举个栗子。对某一个数据连续发出两个修改操作，两台机器同时收到了请求，但是只能一台机器先执行完另外一个机器再执行。那么此时就可以使用 zookeeper 分布式锁，一个机器接收到了请求之后先获取 zookeeper 上的一把分布式锁，就是可以去创建一个 znode，接着执行操作；然后另外一个机器也尝试去创建那个 znode，结果发现自己创建不了，因为被别人创建了，那只能等着，等第一个机器执行完了自己再执行。

![3分布式锁](3分布式锁.png)



#### 3.元数据/配置信息管理

> ​    zookeeper 可以用作很多系统的配置信息的管理，比如 kafka、storm 等等很多分布式系统都会选用 zookeeper 来做一些元数据、配置信息的管理，包括 dubbo 注册中心不也支持 zookeeper 么？


![4元数据：配置信息管理](4元数据：配置信息管理.png)

#### 4.HA高可用性

> ​    这个应该是很常见的，比如 hadoop、hdfs、yarn 等很多大数据系统，都选择基于 zookeeper 来开发 HA 高可用机制，就是一个重要进程一般会做主备两个，主进程挂了立马通过 zookeeper 感知到切换到备用进程。

![5HA高可用性](5HA高可用性.png)

## 二、Zookeeper运行

### 1、上传镜像到harbor

```bash
docker pull zookeeper:3.7.0
docker tag zookeeper:3.7.0 harbor.local/zookeeper/zookeeper:3.7.0
docker push harbor.local/zookeeper/zookeeper:3.7.0
```

### 2、创建namespace

> 01-zookeeper-ns.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata: 
  name: zookeeper
```

### 3、nfs创建挂载目录

```bash
root@k8s-nfs-server:~# showmount -e
Export list for k8s-nfs-server:
/k8s-data *

mkdir -p /k8s-data/zookeeper/zookeeper-datadir-1
mkdir -p /k8s-data/zookeeper/zookeeper-datadir-2
mkdir -p /k8s-data/zookeeper/zookeeper-datadir-3
```

### 4、创建静态存储

> 不要和动态存储使用同一个nfs-server，会报错

#### 1.创建pv

> 02-zookeeper-persistentvolume.yaml

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zookeeper-datadir-pv-1
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.16.3.21
    path: /k8s-data/zookeeper/zookeeper-datadir-1

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zookeeper-datadir-pv-2
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.16.3.21
    path: /k8s-data/zookeeper/zookeeper-datadir-2

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zookeeper-datadir-pv-3
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.16.3.21
    path: /k8s-data/zookeeper/zookeeper-datadir-3
```

#### 2.创建pvc

>03-zookeeper-persistentvolumeclaim.yaml

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: zookeeper-datadir-pvc-1
  namespace: zookeeper
spec:
  accessModes:
    - ReadWriteOnce
  volumeName: zookeeper-datadir-pv-1
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: zookeeper-datadir-pvc-2
  namespace: zookeeper
spec:
  accessModes:
    - ReadWriteOnce
  volumeName: zookeeper-datadir-pv-2
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: zookeeper-datadir-pvc-3
  namespace: zookeeper
spec:
  accessModes:
    - ReadWriteOnce
  volumeName: zookeeper-datadir-pv-3
  resources:
    requests:
      storage: 10Gi
```

### 5、创建集群

>04-zookeeper-cluster.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
  namespace: zookeeper
spec:
  type: NodePort
  ports:
    - name: client
      port: 2181
      nodePort: 32180
  selector:
    app: zookeeper
---
apiVersion: v1
kind: Service
metadata:
  name: zookeeper1
  namespace: zookeeper
spec:
  type: NodePort
  ports:
    - name: client
      port: 2181
      nodePort: 32181
    - name: followers
      port: 2888
    - name: election
      port: 3888
  selector:
    app: zookeeper
    server-id: "1"
---
apiVersion: v1
kind: Service
metadata:
  name: zookeeper2
  namespace: zookeeper
spec:
  type: NodePort
  ports:
    - name: client
      port: 2181
      nodePort: 32182
    - name: followers
      port: 2888
    - name: election
      port: 3888
  selector:
    app: zookeeper
    server-id: "2"
---
apiVersion: v1
kind: Service
metadata:
  name: zookeeper3
  namespace: zookeeper
spec:
  type: NodePort
  ports:
    - name: client
      port: 2181
      nodePort: 32183
    - name: followers
      port: 2888
    - name: election
      port: 3888
  selector:
    app: zookeeper
    server-id: "3"
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: zookeeper1-deployment
  namespace: zookeeper
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
        server-id: "1"
    spec:
      containers:
        - name: server
          image: harbor.local/zookeeper/zookeeper:3.7.0
          imagePullPolicy: Always
          env:
            - name: ZOO_MY_ID
              value: "1"
            - name: ZOO_SERVERS
              value: "quorumListenOnAllIPs:true server.1=zookeeper1:2888:3888;2181 server.2=zookeeper2:2888:3888;2181 server.3=zookeeper3:2888:3888;2181"
          ports:
            - containerPort: 2181
            - containerPort: 2888
            - containerPort: 3888
          volumeMounts:
          - mountPath: "/data"
            name: zookeeper-datadir-pvc-1
      volumes:
        - name: zookeeper-datadir-pvc-1
          persistentVolumeClaim:
            claimName: zookeeper-datadir-pvc-1
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: zookeeper2-deployment
  namespace: zookeeper
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
        server-id: "2"
    spec:
      containers:
        - name: server
          image: harbor.local/zookeeper/zookeeper:3.7.0
          imagePullPolicy: Always
          env:
            - name: ZOO_MY_ID
              value: "2"
            - name: ZOO_SERVERS
              value: "quorumListenOnAllIPs:true server.1=zookeeper1:2888:3888;2181 server.2=zookeeper2:2888:3888;2181 server.3=zookeeper3:2888:3888;2181"
          ports:
            - containerPort: 2181
            - containerPort: 2888
            - containerPort: 3888
          volumeMounts:
          - mountPath: "/data"
            name: zookeeper-datadir-pvc-2
      volumes:
        - name: zookeeper-datadir-pvc-2
          persistentVolumeClaim:
            claimName: zookeeper-datadir-pvc-2
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: zookeeper3-deployment
  namespace: zookeeper
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
        server-id: "3"
    spec:
      containers:
        - name: server
          image: harbor.local/zookeeper/zookeeper:3.7.0
          imagePullPolicy: Always
          env:
            - name: ZOO_MY_ID
              value: "3"
            - name: ZOO_SERVERS
              value: "quorumListenOnAllIPs:true server.1=zookeeper1:2888:3888;2181 server.2=zookeeper2:2888:3888;2181 server.3=zookeeper3:2888:3888;2181"
          ports:
            - containerPort: 2181
            - containerPort: 2888
            - containerPort: 3888
          volumeMounts:
          - mountPath: "/data"
            name: zookeeper-datadir-pvc-3
      volumes:
        - name: zookeeper-datadir-pvc-3
          persistentVolumeClaim:
            claimName: zookeeper-datadir-pvc-3
```

