# k8s集群升级和kubeasz扩容k8s集群

> 版本升级要做好充足的测试，最好用velero复制一套环境
>
> 大版本升级要注意API版本变动

## 一、集群升级准备

### 1、获取新版本二进制文件

#### 1.下载

>https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.25.md#downloads-for-v1253

```bash
cd /usr/local/src
wget https://dl.k8s.io/v1.25.3/kubernetes.tar.gz
wget https://dl.k8s.io/v1.25.3/kubernetes-client-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.25.3/kubernetes-server-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.25.3/kubernetes-node-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src# ll
total 504368
drwxr-xr-x  3 root root      4096 10月 23 22:42 ./
drwxr-xr-x 10 root root      4096 8月  31 14:52 ../
-rw-r--r--  1 root root  30893214 10月 23 22:41 kubernetes-client-linux-amd64.tar.gz
-rw-r--r--  1 root root 124808203 10月 23 22:41 kubernetes-node-linux-amd64.tar.gz
-rw-r--r--  1 root root 332213446 10月 23 22:41 kubernetes-server-linux-amd64.tar.gz
-rw-r--r--  1 root root    532649 10月 23 22:41 kubernetes.tar.gz
drwxr-xr-x  3 root root      4096 10月 23 20:02 velero-v1.9.2-linux-amd64/
-rw-r--r--  1 root root  27992078 10月 23 20:02 velero-v1.9.2-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src# mv kubernetes.tar.gz kubernetes-1.25.3.tar.gz
root@k8s-master1:/usr/local/src# mv kubernetes-client-linux-amd64.tar.gz kubernetes-1.25.3-client-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src# mv kubernetes-node-linux-amd64.tar.gz kubernetes-1.25.3-node-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src# mv kubernetes-server-linux-amd64.tar.gz kubernetes-1.25.3-server-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src# ll
total 504368
drwxr-xr-x  3 root root      4096 10月 23 22:44 ./
drwxr-xr-x 10 root root      4096 8月  31 14:52 ../
-rw-r--r--  1 root root  30893214 10月 23 22:41 kubernetes-1.25.3-client-linux-amd64.tar.gz
-rw-r--r--  1 root root 124808203 10月 23 22:41 kubernetes-1.25.3-node-linux-amd64.tar.gz
-rw-r--r--  1 root root 332213446 10月 23 22:41 kubernetes-1.25.3-server-linux-amd64.tar.gz
-rw-r--r--  1 root root    532649 10月 23 22:41 kubernetes-1.25.3.tar.gz
drwxr-xr-x  3 root root      4096 10月 23 20:02 velero-v1.9.2-linux-amd64/
-rw-r--r--  1 root root  27992078 10月 23 20:02 velero-v1.9.2-linux-amd64.tar.gz
```

#### 2.解压

```bash
tar xf kubernetes-1.25.3-client-linux-amd64.tar.gz
tar xf kubernetes-1.25.3-node-linux-amd64.tar.gz
tar xf kubernetes-1.25.3-server-linux-amd64.tar.gz
tar xf kubernetes-1.25.3.tar.gz
```

#### 3.找到二进制文件

```bash
root@k8s-master1:/usr/local/src# cd kubernetes/server/bin/
root@k8s-master1:/usr/local/src/kubernetes/server/bin# ll
total 1025848
drwxr-xr-x 2 root root      4096 10月 12 19:18 ./
drwxr-xr-x 3 root root      4096 10月 12 19:22 ../
-rwxr-xr-x 1 root root  54652928 10月 12 19:18 apiextensions-apiserver*
-rwxr-xr-x 1 root root  43802624 10月 12 19:18 kubeadm*
-rwxr-xr-x 1 root root  48877568 10月 12 19:18 kube-aggregator*
-rwxr-xr-x 1 root root 123850752 10月 12 19:18 kube-apiserver*
-rw-r--r-- 1 root root         8 10月 12 19:16 kube-apiserver.docker_tag
-rw------- 1 root root 129094656 10月 12 19:16 kube-apiserver.tar
-rwxr-xr-x 1 root root 113205248 10月 12 19:18 kube-controller-manager*
-rw-r--r-- 1 root root         8 10月 12 19:16 kube-controller-manager.docker_tag
-rw------- 1 root root 118449664 10月 12 19:16 kube-controller-manager.tar
-rwxr-xr-x 1 root root  45015040 10月 12 19:18 kubectl*
-rwxr-xr-x 1 root root  53140008 10月 12 19:18 kubectl-convert*
-rwxr-xr-x 1 root root 114237464 10月 12 19:18 kubelet*
-rwxr-xr-x 1 root root   1536000 10月 12 19:18 kube-log-runner*
-rwxr-xr-x 1 root root  41168896 10月 12 19:17 kube-proxy*
-rw-r--r-- 1 root root         8 10月 12 19:16 kube-proxy.docker_tag
-rw------- 1 root root  63284736 10月 12 19:16 kube-proxy.tar
-rwxr-xr-x 1 root root  46690304 10月 12 19:18 kube-scheduler*
-rw-r--r-- 1 root root         8 10月 12 19:16 kube-scheduler.docker_tag
-rw------- 1 root root  51934208 10月 12 19:16 kube-scheduler.tar
-rwxr-xr-x 1 root root   1458176 10月 12 19:18 mounter*
```

#### 4.确定版本

```bash
root@k8s-master1:/usr/local/src/kubernetes/server/bin# ./kubectl version
WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.3", GitCommit:"434bfd82814af038ad94d62ebe59b133fcb50506", GitTreeState:"clean", BuildDate:"2022-10-12T10:57:26Z", GoVersion:"go1.19.2", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.7
Server Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.1", GitCommit:"e4d4e1ab7cf1bf15273ef97303551b279f0920a9", GitTreeState:"clean", BuildDate:"2022-09-14T19:42:30Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"linux/amd64"}
root@k8s-master1:/usr/local/src/kubernetes/server/bin# ./kube-controller-manager --version
Kubernetes v1.25.3
root@k8s-master1:/usr/local/src/kubernetes/server/bin# ./kube-apiserver --version
Kubernetes v1.25.3
root@k8s-master1:/usr/local/src/kubernetes/server/bin# ./kube-scheduler --version
Kubernetes v1.25.3
root@k8s-master1:/usr/local/src/kubernetes/server/bin# ./kube-proxy --version
Kubernetes v1.25.3
root@k8s-master1:/usr/local/src/kubernetes/server/bin# ./kubelet --version
Kubernetes v1.25.3
```

#### 5.复制二进制文件到kubeasz下

```bash
rsync -avz kubectl kube-controller-manager kube-apiserver kube-scheduler kube-proxy kubelet /etc/kubeasz/bin/
```

#### 6.验证拷贝过去的版本

```bash
root@k8s-master1:/usr/local/src/kubernetes/server/bin# /etc/kubeasz/bin/kubectl version
WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.3", GitCommit:"434bfd82814af038ad94d62ebe59b133fcb50506", GitTreeState:"clean", BuildDate:"2022-10-12T10:57:26Z", GoVersion:"go1.19.2", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.7
Server Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.1", GitCommit:"e4d4e1ab7cf1bf15273ef97303551b279f0920a9", GitTreeState:"clean", BuildDate:"2022-09-14T19:42:30Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"linux/amd64"}
root@k8s-master1:/usr/local/src/kubernetes/server/bin# /etc/kubeasz/bin/kube-controller-manager --version
Kubernetes v1.25.3
root@k8s-master1:/usr/local/src/kubernetes/server/bin# /etc/kubeasz/bin/kube-apiserver --version
Kubernetes v1.25.3
root@k8s-master1:/usr/local/src/kubernetes/server/bin# /etc/kubeasz/bin/kube-scheduler --version
Kubernetes v1.25.3
root@k8s-master1:/usr/local/src/kubernetes/server/bin# /etc/kubeasz/bin/kube-proxy --version
Kubernetes v1.25.3
root@k8s-master1:/usr/local/src/kubernetes/server/bin# /etc/kubeasz/bin/kubelet --version
Kubernetes v1.25.3
```

## 二、master升级

### 1、node节点负载均衡摘掉master01

> 所有node节点执行
>
> master2的是master1升级之后执行的命令，没有特殊标注的命令几乎和master1一致

#### 1.注释其中一个master

```bash
root@k8s-node1:~# cat /etc/kube-lb/conf/kube-lb.conf
user root;
worker_processes 1;

error_log  /etc/kube-lb/logs/error.log warn;

events {
    worker_connections  3000;
}

stream {
    upstream backend {
#        server 172.31.7.101:6443    max_fails=2 fail_timeout=3s;
        server 172.31.7.102:6443    max_fails=2 fail_timeout=3s;
    }

    server {
        listen 127.0.0.1:6443;
        proxy_connect_timeout 1s;
        proxy_pass backend;
    }
}
```

#### 2.重启kube-lb

```bash
systemctl reload kube-lb.service
```

### 2、master 停止服务

```bash
systemctl stop kube-apiserver.service kube-controller-manager.service kube-scheduler.service kube-proxy.service kubelet.service
```

### 3、替换命令

```bash
root@k8s-master1:~# cd /etc/kubeasz/bin/
rsync -avz kubectl kube-controller-manager kube-apiserver kube-scheduler kube-proxy kubelet /usr/local/bin/
```

> master2

```bash
cd /etc/kubeasz/bin/
rsync -avz kubectl kube-controller-manager kube-apiserver kube-scheduler kube-proxy kubelet root@172.31.7.102:/usr/local/bin/
```

### 4、验证版本

```bash
kubectl version
kube-controller-manager --version
kube-apiserver --version
kube-scheduler --version
kube-proxy --version
kubelet --version
```

### 5、启动服务

```bash
systemctl start kube-apiserver.service kube-controller-manager.service kube-scheduler.service kube-proxy.service kubelet.service
```

### 6、验证

```bash
root@k8s-master1:/etc/kubeasz/bin# kubectl get nodes
NAME           STATUS                     ROLES    AGE   VERSION
172.31.7.101   Ready,SchedulingDisabled   master   4d    v1.25.3
172.31.7.102   Ready,SchedulingDisabled   master   4d    v1.25.1
172.31.7.111   Ready                      <none>   4d    v1.25.1
172.31.7.112   Ready                      <none>   4d    v1.25.1
```

### 7、按照上面的方式升级其他master节点

## 三、node升级

### 1、驱逐node1节点

```bash
root@k8s-master1:~# kubectl drain 172.31.7.111 --force --ignore-daemonsets --delete-emptydir-data
node/172.31.7.111 already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/calico-node-rqx6p
evicting pod velero-system/velero-5f674d5cc8-gqfv5
evicting pod kubernetes-dashboard/dashboard-metrics-scraper-884c978b-2p5v8
evicting pod kubernetes-dashboard/kubernetes-dashboard-99cd8855c-599x8
pod/velero-5f674d5cc8-gqfv5 evicted
pod/kubernetes-dashboard-99cd8855c-599x8 evicted
pod/dashboard-metrics-scraper-884c978b-2p5v8 evicted
node/172.31.7.111 drained
```

### 2、查看

```bash
root@k8s-master1:~# kubectl get pod -A  -o wide
NAMESPACE              NAME                                       READY   STATUS    RESTARTS      AGE     IP               NODE           NOMINATED NODE   READINESS GATES
default                net-test3                                  1/1     Running   1 (56m ago)   82m     10.200.169.145   172.31.7.112   <none>           <none>
default                net-test4                                  1/1     Running   4 (56m ago)   4d      10.200.169.148   172.31.7.112   <none>           <none>
kube-system            calico-kube-controllers-6d5cf54455-64jf9   1/1     Running   5 (56m ago)   4d      172.31.7.112     172.31.7.112   <none>           <none>
kube-system            calico-node-csktv                          1/1     Running   4 (56m ago)   4d      172.31.7.112     172.31.7.112   <none>           <none>
kube-system            calico-node-jv25x                          1/1     Running   4 (56m ago)   4d      172.31.7.102     172.31.7.102   <none>           <none>
kube-system            calico-node-nvwq5                          1/1     Running   4 (56m ago)   4d      172.31.7.101     172.31.7.101   <none>           <none>
kube-system            calico-node-rqx6p                          1/1     Running   4 (56m ago)   4d      172.31.7.111     172.31.7.111   <none>           <none>
kube-system            coredns-8496f84465-frjtm                   1/1     Running   3 (56m ago)   2d21h   10.200.169.146   172.31.7.112   <none>           <none>
kube-system            coredns-8496f84465-zpvjf                   1/1     Running   3 (56m ago)   3d1h    10.200.169.147   172.31.7.112   <none>           <none>
kubernetes-dashboard   dashboard-metrics-scraper-884c978b-b968t   1/1     Running   0             2m16s   10.200.169.149   172.31.7.112   <none>           <none>
kubernetes-dashboard   kubernetes-dashboard-99cd8855c-frl44       1/1     Running   0             2m16s   10.200.169.150   172.31.7.112   <none>           <none>
velero-system          velero-5f674d5cc8-lldsw                    1/1     Running   0             2m16s   10.200.169.151   172.31.7.112   <none>           <none>
root@k8s-master1:~# kubectl get nodes
NAME           STATUS                     ROLES    AGE   VERSION
172.31.7.101   Ready,SchedulingDisabled   master   4d    v1.25.3
172.31.7.102   Ready,SchedulingDisabled   master   4d    v1.25.3
172.31.7.111   Ready,SchedulingDisabled   <none>   4d    v1.25.1
172.31.7.112   Ready                      <none>   4d    v1.25.1
```

### 3、停止node服务

```bash
systemctl stop  kubelet.service kube-proxy.service
```

### 4、拷贝二进制文件

```bash
cd /etc/kubeasz/bin/
rsync -avz kubectl kube-proxy kubelet root@172.31.7.111:/usr/local/bin/
```

### 5、启动服务

```bash
systemctl start kubelet.service kube-proxy.service
```

### 6、验证

```bash
root@k8s-master1:/etc/kubeasz/bin# kubectl get nodes
NAME           STATUS                     ROLES    AGE   VERSION
172.31.7.101   Ready,SchedulingDisabled   master   4d    v1.25.3
172.31.7.102   Ready,SchedulingDisabled   master   4d    v1.25.3
172.31.7.111   Ready,SchedulingDisabled   <none>   4d    v1.25.3
172.31.7.112   Ready                      <none>   4d    v1.25.1
```

### 7、取消驱逐加入调度

```bash

root@k8s-master1:/etc/kubeasz/bin# kubectl uncordon 172.31.7.111
node/172.31.7.111 uncordoned
root@k8s-master1:/etc/kubeasz/bin# kubectl get nodes
NAME           STATUS                     ROLES    AGE   VERSION
172.31.7.101   Ready,SchedulingDisabled   master   4d    v1.25.3
172.31.7.102   Ready,SchedulingDisabled   master   4d    v1.25.3
172.31.7.111   Ready                      <none>   4d    v1.25.3
172.31.7.112   Ready                      <none>   4d    v1.25.1
```

### 8、其他node节点同上操作

## 四、添加node节点

### 1、添加

```bash
root@k8s-master1:/etc/kubeasz# ./ezctl add-node k8s-cluster1  172.31.7.113
```

### 2、验证

```bash
root@k8s-master1:/etc/kubeasz# kubectl get nodes
NAME           STATUS                     ROLES    AGE     VERSION
172.31.7.101   Ready,SchedulingDisabled   master   4d1h    v1.25.3
172.31.7.102   Ready,SchedulingDisabled   master   4d1h    v1.25.3
172.31.7.111   Ready                      <none>   4d      v1.25.3
172.31.7.112   Ready                      <none>   4d      v1.25.3
172.31.7.113   Ready                      node     2m48s   v1.25.3
```

## 五、添加master节点

### 1、添加

```bash
./ezctl add-master k8s-cluster1  172.31.7.103
```

### 2、验证

```bash
root@k8s-master1:/etc/kubeasz# kubectl get nodes
NAME           STATUS                     ROLES    AGE     VERSION
172.31.7.101   Ready,SchedulingDisabled   master   4d1h    v1.25.3
172.31.7.102   Ready,SchedulingDisabled   master   4d1h    v1.25.3
172.31.7.103   Ready,SchedulingDisabled   master   77s     v1.25.3
172.31.7.111   Ready                      <none>   4d      v1.25.3
172.31.7.112   Ready                      <none>   4d      v1.25.3
172.31.7.113   Ready                      node     9m29s   v1.25.3
```

