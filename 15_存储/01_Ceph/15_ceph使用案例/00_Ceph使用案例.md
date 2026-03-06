# Ceph使用案例

## 一、rbd 结合 k8s 提供存储卷及动态存储卷使用案例

>让 k8s 中的 pod 可以访问 ceph 中 rbd 提供的镜像作为存储设备，需要在 ceph 创建 rbd、并且让 k8s node 节点能够通过 ceph 的认证

>k8s 在使用 ceph 作为动态存储卷的时候，需要 kube-controller-manager 组件能够访问ceph，因此需要在包括 k8s master 及 node 节点在内的每一个 node 同步认证文件

### 1、创建初始化 rbd

```bash
# 创建新的 rbd
$ ceph osd pool create xiaowu-rbd-pool1 32 32

# 验证存储池,确认存储池已经存在
$ ceph osd pool ls

# 存储池启用 rbd
$ ceph osd pool application enable xiaowu-rbd-pool1 rbd

# 初始化 rbd
$ rbd pool init -p xiaowu-rbd-pool1
```

### 2、创建image

```bash
# 创建镜像
rbd create xiaowu-img-img1 --size 3G --pool xiaowu-rbd-pool1 --image-format 2 --image-feature layering

# 验证镜像
rbd ls --pool xiaowu-rbd-pool1

# 查看镜像信息
rbd --image xiaowu-img-img1 --pool xiaowu-rbd-pool1 info
```

### 3、客户端安装 ceph-common

>分别在 k8s master 与各 node 节点安装 ceph-common 组件包，如果是老版本ceph需要指定安装客户端版本

```bash
apt-get install ceph-common -y
```

### 4、创建 ceph 用户与授权

```bash
# 创建并授权
ceph auth get-or-create client.xiaowu mon 'allow r' osd 'allow * pool=xiaowu-rbd-pool1'

# 验证用户
ceph auth get client.xiaowu

# 导出用户信息至keyring文件
ceph auth get client.xiaowu -o ceph.client.xiaowu.keyring

# 同步认证文件到 k8s 各 master 及 node 节点
scp ceph.conf ceph.client.xiaowu.keyring root@k8s-master-01:/etc/ceph/
scp ceph.conf ceph.client.xiaowu.keyring root@k8s-node-01:/etc/ceph/
scp ceph.conf ceph.client.xiaowu.keyring root@k8s-node-02:/etc/ceph/

# k8s 各 master 及 node 节点验证用户权限
ceph --user xiaowu -s

# 验证镜像访问权限
rbd --user xiaowu ls --pool=xiaowu-rbd-pool1
```

### 5、k8s 节点配置主机名解析

>在 ceph.conf 配置文件中包含 ceph 主机的主机名，因此需要在 k8s 各 master 及 node 配置配置主机名解析

### 6、通过 keyring 文件挂载 rbd

>基于 ceph 提供的 rbd 实现存储卷的动态提供，由两种实现方式，一是通过宿主机的 keyring文件挂载 rbd，另外一个是通过将 keyring 中 key 定义为 k8s 中的 secret，然后 pod 通过secret 挂载 rbd

>**rbd（RADOS Block Device）块存储只能独占挂载**，也就是说只能一个客户端挂载使用，无论是真实的服务器还是k8s的pod。
>
>因此，在k8s中，挂载rbd的pod必须只能是1个副本的deployment或者也可以是多副本的statefulset。这种限制决定了在k8s中，rbd的读写模式不能也不支持ReadWriteMany。

#### 1.通过 keyring 文件直接挂载-busybox

##### 1）yaml文件内容

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox 
    command:
      - sleep
      - "3600"
    imagePullPolicy: Always 
    name: busybox
    #restartPolicy: Always
    volumeMounts:
    - name: rbd-data1
      mountPath: /data
  volumes:
    - name: rbd-data1
      rbd:
        monitors:
        - '172.31.6.101:6789'
        - '172.31.6.102:6789'
        - '172.31.6.103:6789'
        pool: xiaowu-rbd-pool1
        image: xiaowu-img-img1
        fsType: ext4
        readOnly: false
        user: xiaowu
        keyring: /etc/ceph/ceph.client.xiaowu.keyring
```

##### 2）查看是否挂载成功

```bash
# 进入pod
kubectl exec -it busybox sh

# 在pod里面查看是否挂载成功
df
```

#### 2.通过 keyring 文件直接挂载-nginx

##### 1）删除busybox释放rbd占用

##### 2）yaml文件内容

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels: #rs or deployment
      app: ng-deploy-80
  template:
    metadata:
      labels:
        app: ng-deploy-80
    spec:
      containers:
      - name: ng-deploy-80
        image: nginx
        ports:
        - containerPort: 80

        volumeMounts:
        - name: rbd-data1
          mountPath: /data
      volumes:
        - name: rbd-data1
          rbd:
            monitors:
            - '172.31.6.101:6789'
            - '172.31.6.102:6789'
            - '172.31.6.103:6789'
            pool: xiaowu-rbd-pool1
            image: xiaowu-img-img1
            fsType: ext4
            readOnly: false
            user: xiaowu
            keyring: /etc/ceph/ceph.client.xiaowu.keyring
```

##### 3）查看是否挂载成功

```bash
# 进入pod
kubectl exec -it nginx-deployment-************* sh

# 在pod里面查看是否挂载成功
df
```

#### 3.宿主机验证 rbd

>rbd 在 pod 里面看是挂载到了 pod，但是由于 pod 是使用的宿主机内核，因此实际是在宿主机挂载的

```bash
# 查看pod所在的node
kubectl get pod nginx-deployment-************* -o wide

# 到宿主机所在的 node 验证 rbd 挂载
rbd showmapped
```

### 7、通过 secret 挂载 rbd

#### 1.创建 secret

>首先要创建 secret，secret 中主要就是包含 ceph 中被授权用户 keyrin 文件中的 key，需要将 key 内容通过 base64 编码后即可创建 secret

```bash
# 将 key 进行编码
ceph auth print-key client.xiaowu
AQB4L79g/he7HBAAvJQ7sI3zdSsTUL21Nx6zLQ==


ceph auth print-key client.xiaowu | base64
QVFCNEw3OWcvaGU3SEJBQXZKUTdzSTN6ZFNzVFVMMjFOeDZ6TFE9PQ==
```

#### 2.secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-xiaowu
type: "kubernetes.io/rbd"
data:
  key: QVFDMVowbGlUYWxZQXhBQUt0Mzhpengxc0YrREVhZllOSkVuMkE9PQ== 
```

#### 3.pod.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels: #rs or deployment
      app: ng-deploy-80
  template:
    metadata:
      labels:
        app: ng-deploy-80
    spec:
      containers:
      - name: ng-deploy-80
        image: nginx
        ports:
        - containerPort: 80

        volumeMounts:
        - name: rbd-data1
          mountPath: /usr/share/nginx/html/rbd
      volumes:
        - name: rbd-data1
          rbd:
            monitors:
            - '172.31.6.101:6789'
            - '172.31.6.102:6789'
            - '172.31.6.103:6789'
            pool: xiaowu-rbd-pool1
            image: xiaowu-img-img1
            fsType: ext4
            readOnly: false
            user: xiaowu
            secretRef:
              name: ceph-secret-xiaowu
```

#### 4.验证挂载方式同上

### 8、动态存储卷供给-需要使用二进制安装 k8s

>https://github.com/kubernetes/kubernetes/issues/38923
>存储卷可以通过 kube-controller-manager 组件动态创建，适用于有状态服务需要多个存储卷的场合。
>将 ceph admin 用户 key 文件定义为 k8s secret，用于 k8s 调用 ceph admin 权限动态创建存储卷，即不再需要提前创建好 image 而是 k8s 在需要使用的时候再调用 ceph 创建。

#### 1.获取 admin 用户 secret值

```bash
# 获取 admin 用户 secret
ceph auth print-key client.admin | base64
```

#### 2.secret.yaml

```bash
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-admin
type: "kubernetes.io/rbd"
data:
  key: QVFBTkJVQmlRc3MzR3hBQWtqR3hCVlk0Y1VyRy9waS9TWlFqd3c9PQ==
```

#### 3.创建存储类

>创建动态存储类，为 pod 提供动态 pvc

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-storage-class-xiaowu
  annotations:
    storageclass.kubernetes.io/is-default-class: "false" #设置为默认存储类
provisioner: kubernetes.io/rbd
parameters:
  monitors: 172.31.6.101:6789,172.31.6.102:6789,172.31.6.103:6789
  adminId: admin
  adminSecretName: ceph-secret-admin
  adminSecretNamespace: default 
  pool: xiaowu-rbd-pool1
  userId: xiaowu
  userSecretName: ceph-secret-xiaowu
```

#### 4.创建基于存储类的 PVC

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-storage-class-xiaowu 
  resources:
    requests:
      storage: '5Gi'
```

#### 5.ceph 验证是否自动创建 image

```bash
rbd ls --pool xiaowu-rbd-pool1
```

#### 6.运行单机 mysql 并验证

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6.46
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: xiaowu123456
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-data-pvc
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: mysql-service-label 
  name: mysql-service
spec:
  type: NodePort
  ports:
  - name: http
    port: 3306
    protocol: TCP
    targetPort: 3306
    nodePort: 33306
  selector:
    app: mysql
```

## 二、cephfs 使用案例

### 1、k8s集群使用-手动挂载

>k8s 中的 pod 挂载 ceph 的 cephfs 共享存储，实现业务中数据共享、持久化、高性能、高可用的目的

#### 1.创建 secret

>创建对应fs账户或者使用admin账户，步骤同上

#### 2.deployment.yaml准备

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels: #rs or deployment
      app: ng-deploy-80
  template:
    metadata:
      labels:
        app: ng-deploy-80
    spec:
      containers:
      - name: ng-deploy-80
        image: nginx
        ports:
        - containerPort: 80

        volumeMounts:
        - name: magedu-staticdata-cephfs 
          mountPath: /usr/share/nginx/html/cephfs
      volumes:
        - name: magedu-staticdata-cephfs
          cephfs:
            monitors:
            - '172.31.6.101:6789'
            - '172.31.6.102:6789'
            - '172.31.6.103:6789'
            path: /
            user: admin
            secretRef:
              name: ceph-secret-admin

---
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: ng-deploy-80-service-label
  name: ng-deploy-80-service
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 33380
  selector:
    app: ng-deploy-80
```

#### 3.验证方式同fs验证方式一样

### 2、nfs-ganesha-ceph应用

```bash
apt -y install nfs-ganesha-ceph
```

```yaml
root@lb-kubeasz:~# cat /etc/ganesha/ganesha.conf
# create new
NFS_CORE_PARAM {
    # disable NLM
    Enable_NLM = false;
    # disable RQUOTA (not suported on CephFS)
    Enable_RQUOTA = false;
    # NFS protocol
    Protocols = 3,4;
}
NFS_KRB5 {
    Active_krb5 = false;
}
EXPORT_DEFAULTS {
    # default access mode
    Access_Type = RW;
}
EXPORT {
    # uniq ID
    Export_Id = 101;
    # mount path of CephFS
    Path = "/";
    FSAL {
        name = CEPH;
        # hostname or IP address of this Node
        hostname="192.168.6.108";
        Filesystem = "myfs";
        User_ID = "fnosfs";
    }
    # setting for root Squash
    Squash="No_root_squash";
    # NFSv4 Pseudo path
    Pseudo="/cephfs";
    # allowed security options
    SecType = "sys";
}
LOG {
    # default log level
    Default_Log_Level = INFO;
}
```

```bash
systemctl restart nfs-ganesha
```

### 3、ceph-fuse挂载

#### 1.凭证和集群信息准备

```bash
root@lb-kubeasz:/# ll /etc/ceph/      
total 16
drwxr-xr-x  2 root root 4096 Aug 31 00:44 ./
drwxr-xr-x 82 root root 4096 Aug 31 00:06 ../
-rw-r--r--  1 root root  161 Aug 30 23:18 ceph.client.fnosfs.keyring
-rw-r--r--  1 root root  141 Aug 31 00:44 ceph.conf
root@lb-kubeasz:/# cat /etc/ceph/ceph.conf
[global]
mon_host = 192.168.6.104:6789,192.168.6.105:6789,192.168.6.106:6789

[client.fnosfs]
keyring = /etc/ceph/ceph.client.fnosfs.keyring
root@lb-kubeasz:/# cat /etc/ceph/ceph.client.fnosfs.keyring 
[client.fnosfs]
        key = Axxxxxxxxxxxxxxxxxxg==
        caps mds = "allow rw"
        caps mon = "allow r"
        caps osd = "allow rwx"
```

#### 2.system准备

```bash
root@lb-kubeasz:/# cat /etc/systemd/system/ceph-fuse-docker.service
[Unit]
Description=CephFS fuse mount for /docker-service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/ceph-fuse --name client.fnosfs --client_mountpoint=/docker-service --foreground /docker-service
ExecStop=/bin/fusermount -u /docker-service
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

