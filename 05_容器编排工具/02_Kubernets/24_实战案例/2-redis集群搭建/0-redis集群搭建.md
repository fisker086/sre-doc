# redis集群搭建

## 一、镜像准备

```bash
docker pull redis:5.0.5
docker tag redis:5.0.5 harbor.local/redis/redis:5.0.5
docker push harbor.local/redis/redis:5.0.5
```

## 二、Redis运行

### 1、创建namespace

> 01-redis-cluster-ns.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata: 
  name: redis-cluster
```

### 2、nfs创建挂载目录

```bash
root@k8s-nfs-server:~# showmount -e
Export list for k8s-nfs-server:
/k8s-data *

mkdir -p /k8s-data/redis/redis-datadir-0
mkdir -p /k8s-data/redis/redis-datadir-1
mkdir -p /k8s-data/redis/redis-datadir-2
mkdir -p /k8s-data/redis/redis-datadir-3
mkdir -p /k8s-data/redis/redis-datadir-4
mkdir -p /k8s-data/redis/redis-datadir-5
```

### 3、创建pv

>02-redis-cluster-pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv0
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.16.3.21
    path: /k8s-data/redis/redis-datadir-0
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv1
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.16.3.21
    path: /k8s-data/redis/redis-datadir-1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv2
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.16.3.21
    path: /k8s-data/redis/redis-datadir-2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv3
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.16.3.21
    path: /k8s-data/redis/redis-datadir-3
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv4
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.16.3.21
    path: /k8s-data/redis/redis-datadir-4
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv5
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.16.3.21
    path: /k8s-data/redis/redis-datadir-5
```

```bash
kubectl apply  -f 02-redis-cluster-pv.yaml
```

### 4、部署redis集群

#### 1.准备配置文件

> redis.conf

```bash
appendonly yes
cluster-enabled yes
cluster-config-file /var/lib/redis/nodes.conf
cluster-node-timeout 5000
dir /var/lib/redis
port 6379

requirepass 123456

save 60 1000
stop-writes-on-bgsave-error no
rdbcompression no
dbfilename dump.rdb
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

### 2.创建configmap

```bash
kubectl create configmap redis-conf --from-file=redis.conf -n redis-cluster
```

## 四、创建集群

>03-redis-cluster.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: redis-cluster
  labels:
    app: redis
spec:
  selector:
    app: redis
    appCluster: redis-cluster
  ports:
  - name: redis
    port: 6379
  clusterIP: None
---
apiVersion: v1
kind: Service
metadata:
  name: redis-access
  namespace: redis-cluster
  labels:
    app: redis
spec:
  selector:
    app: redis
    appCluster: redis-cluster
  ports:
  - name: redis-access
    protocol: TCP
    port: 6379
    targetPort: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: redis-cluster
spec:
  serviceName: redis
  replicas: 6
  selector:
    matchLabels:
      app: redis
      appCluster: redis-cluster
  template:
    metadata:
      labels:
        app: redis
        appCluster: redis-cluster
    spec:
      # 终止删除时间
      terminationGracePeriodSeconds: 20
      # 亲和，倾向于redis集群运行在同一个机器上
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
      containers:
      - name: redis
        image: harbor.local/redis/redis:5.0.5
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
        resources:
          requests:
            cpu: "500m"
            memory: "500Mi"
        ports:
        - containerPort: 6379
          name: redis
          protocol: TCP
        - containerPort: 16379
          name: cluster
          protocol: TCP
        volumeMounts:
        - name: conf
          mountPath: /etc/redis
        - name: data
          mountPath: /var/lib/redis
      volumes:
      - name: conf
        configMap:
          name: redis-conf
          items:
          - key: redis.conf
            path: redis.conf
  # pvc模板，前提有多余的pv，或者有动态存储
  volumeClaimTemplates:
  - metadata:
      name: data
      namespace: redis-cluster
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```

```yaml
 kubectl apply -f 03-redis-cluster.yaml
```

## 五、集群初始化

> 初始化只需要初始化一次，redis 4及之前的版本需要使用redis-tribe工具进行初始化，redis 5开始使用redis-cli。

### 1、5.0以前

#### 1.创建临时容器

```bash
kubectl run -it ubuntu1804 --image=ubuntu:18.04 --restart=Never -n redis-cluster bash
root@ubuntu:/# apt update
root@ubuntu1804:/# apt install  python2.7 python-pip redis-tools dnsutils iputils-ping net-tools -y

root@ubuntu1804:/# pip install --upgrade pip

root@ubuntu1804:/# pip install redis-trib==0.5.1
```

#### 2.创建集群

```bash
root@ubuntu1804:/# redis-trib.py create \
  `dig +short redis-0.redis.redis-cluster.svc.xiaowurobot.local`:6379 \
  `dig +short redis-1.redis.redis-cluster.svc.xiaowurobot.local`:6379 \
  `dig +short redis-2.redis.redis-cluster.svc.xiaowurobot.local`:6379
```

#### 3.slave加入集群

```bash
# 将redis-3加入redis-0：
root@ubuntu1804:/#  redis-trib.py replicate \
  --master-addr `dig +short redis-0.redis.redis-cluster.svc.xiaowurobot.local`:6379 \
  --slave-addr `dig +short redis-3.redis.redis-cluster.svc.xiaowurobot.local`:6379
  
# 将redis-4加入redis-1：
root@ubuntu1804:/# redis-trib.py replicate \
  --master-addr `dig +short redis-1.redis.redis-cluster.svc.xiaowurobot.local`:6379 \
  --slave-addr `dig +short redis-4.redis.redis-cluster.svc.xiaowurobot.local`:6379
  
# 将redis-5加入redis-2：
root@ubuntu1804:/# redis-trib.py replicate \
  --master-addr `dig +short redis-2.redis.redis-cluster.svc.xiaowurobot.local`:6379 \
  --slave-addr `dig +short redis-5.redis.redis-cluster.svc.xiaowurobot.local`:6379
```

### 2、5.0以后

```bash
kubectl -n redis-cluster exec -it redis-0 bashred

redis-cli --cluster create \
10.200.107.227:6379 \
10.200.36.91:6379 \
10.200.169.155:6379 \
10.200.36.92:6379 \
10.200.107.228:6379 \
10.200.169.156:6379 --cluster-replicas 1 -a 123456
```



## 六、验证

````bash
root@k8s-master1:~# kubectl exec -it redis-0 bash -n redis-cluster
root@redis-0:/data# redis-cli -c
127.0.0.1:6379> auth 123456
OK
127.0.0.1:6379> CLUSTER INFO
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:133
cluster_stats_messages_pong_sent:124
cluster_stats_messages_sent:257
cluster_stats_messages_ping_received:119
cluster_stats_messages_pong_received:133
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:257


127.0.0.1:6379> CLUSTER NODES
eeeb6995e0249a3c117111661fe7014d58749356 10.200.151.214:6379@16379 slave 6b556ef41754e60797234a3b0151c77aa3ca6ab9 0 1642743571441 1 connected
c777e1fc7d1975f45093a2309429ef881a6caf57 10.200.218.115:6379@16379 slave d494fc79c6e8b34a280ae37d6ed3c74ae54b924b 0 1642743572446 4 connected
d494fc79c6e8b34a280ae37d6ed3c74ae54b924b 10.200.218.106:6379@16379 master - 0 1642743572547 2 connected 0-5461
6b556ef41754e60797234a3b0151c77aa3ca6ab9 10.200.151.222:6379@16379 master - 0 1642743572446 1 connected 5462-10922
9b89c3b016a1d85ebe1973942dcdb84d1c84a0f3 10.200.218.50:6379@16379 slave e3adff86d91c5aaa2c03ff88ee5cd54ad8c18547 0 1642743571541 5 connected
e3adff86d91c5aaa2c03ff88ee5cd54ad8c18547 10.200.218.42:6379@16379 myself,master - 0 1642743571000 5 connected 10923-16383

测试在master写入数据:
root@k8s-master1:~# kubectl exec -it redis-2 bash -n redis-cluster
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@redis-2:/data# redis-cli -c
127.0.0.1:6379> set key1 valye1
OK

在slave验证数据：
root@k8s-master1:~# kubectl exec -it redis-5 bash -n redis-cluster
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@redis-5:/data# redis-cli 
127.0.0.1:6379> KEYS *
1) "key1"
````

