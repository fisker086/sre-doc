# ubuntu用kubeadm安装k8s1.24.3

## 一、简介

>​    Kubernetes有两种方式，第一种是二进制的方式，可定制但是部署复杂容易出错；第二种是kubeadm工具安装，部署简单，不可定制化。本次我们部署kubeadm版.
>​    服务器配置至少是2G2核的。如果不是则可以在集群初始化后面增加 --ignore-preflight-errors=NumCPU

## 二、部署规划

### 1、版本规划

| 类型           | 版本                      |
| -------------- | ------------------------- |
| 操作系统版本   | Ubuntu 20.04.4 LTS        |
| 容器运行时     | containerd.io 1.6.6及以上 |
| kubernetes版本 | 1.24.3                    |
| 网络插件       | calico                    |
|                |                           |

### 2、节点规划

| Hostname      | IP          | 配置        |
| ------------- | ----------- | ----------- |
| k8s-master-01 | 172.16.2.21 | 4C 8G 100G  |
| k8s-node-01   | 172.16.2.22 | 4C 16G 100G |
| k8s-node-02   | 172.16.2.23 | 4C 16G 100G |

## 三、主机安装并连接（三台机器都做）

### 1、主机安装

> 略，请部署好三台ubuntu20.04版本主机，并分配好IP

### 2、安装SSH、VIM

> 设置root账户密码并允许root远程ssh登录

#### 1.安装ssh vim

```bash
sudo apt install -y vim ssh
```

#### 2.设置root账户密码

```bash
sudo passwd root
```

#### 3.允许root账户登录

```bash
sudo cat /etc/ssh/sshd_config
#PermitRootLogin prohibit-password
PermitRootLogin yes     #添加这句话
```

#### 4.使用自己的shell工具连接

## 四、修改主机名及解析(三台机器都做)

### 1、修改主机名

> 每个节点输入对应的命令

```bash
hostnamectl set-hostname k8s-master-01
hostnamectl set-hostname k8s-node-01
hostnamectl set-hostname k8s-node-02
```

### 2、添加host解析

```bash
vim /etc/hosts
172.16.2.21  k8s-master-01 m1
172.16.2.22  k8s-node-01   n1
172.16.2.23  k8s-node-02   n2
```

### 3、添加DNS解析

```bash
vim /etc/resolv.conf
nameserver 114.114.114.114
```



## 五、主机优化（三台机器都做）

### 1、开启命令行高亮

```bash
root@master01-virtual-machine:~# vim /root/.bashrc
force_color_prompt=yes   # 去掉这行注释

root@master01-virtual-machine:~# bash
```

### 2、修改网卡名称为eth*

#### 1.修改

```bash
root@master01-virtual-machine:~# vim /etc/default/grub
...
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"  # 只修改这行
```

#### 2.更新配置

```bash
root@master01-virtual-machine:~# update-grub
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.15.0-55-generic
Found initrd image: /boot/initrd.img-4.15.0-55-generic
done


###重启
reboot
```

> 重启后可能导致网卡配置失效，需要重新配置网卡

### 3、配置apt源及安装常用组件

#### 1.配源

>https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/

```bash
root@master01-virtual-machine:~# vim /etc/apt/sources.list
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
```

#### 2.更新存储库索引

```
apt update
```

#### 3.升级可升级软件包

```bash
apt upgrade
```

#### 4.安装常用组件

```bash
apt-get install -y net-tools vim htop iftop bash-completion wget rsync tcpdump telnet lrzsz tree iotop unzip zip apt-transport-https ca-certificates curl software-properties-common gnupg-agent ntp ntpdate ntpstat
```

### 4、关闭防火墙

> Ubuntu中无selinux，防火墙服务为ufw，因此关闭防火墙并禁止防火墙开机自启动命令如下：

```bash
root@k8s-master-01:~# ufw disable
Firewall stopped and disabled on system startup
```

### 5、配置ntp时间同步

#### 1.停止ntpd服务

```bash
root@k8s-master-01:~# systemctl disable --now ntp
```

#### 2.手动同步

```bash
ntpdate ntp1.aliyun.com

# 系统时间写入硬件
sudo hwclock -w
```

#### 3.配置定时同步

```bash
crontab -e

#Timing synchronization time
0 */1 * * * /usr/sbin/ntpdate ntp1.aliyun.com &>/dev/null

# 检查结果
crontab -l
```

### 6、配置免密登录

> master创建秘钥分发给node

```bash
rm -rf /root/.ssh

ssh-keygen

cd /root/.ssh/

mv id_rsa.pub authorized_keys

rsync -avz  /root/.ssh  n1:/root
rsync -avz  /root/.ssh  n2:/root
```

### 7、关闭swap分区

```bash
# 关闭swap分区
swapoff -a 

# 注释swap分区
vim /etc/fstab
```

### 8、安装ipvs模块

#### 1.安装

```bash
apt install ipvsadm ipset sysstat conntrack -y
```

#### 2.加载ipvs模块

```bash
cat >> /etc/modules-load.d/ipvs.conf <<EOF 
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF

systemctl restart systemd-modules-load.service

lsmod | grep -e ip_vs -e nf_conntrack
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 155648  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          139264  1 ip_vs
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
nf_defrag_ipv4         16384  1 nf_conntrack
libcrc32c              16384  4 nf_conntrack,btrfs,raid456,ip_vs
```

### 9、内核参数优化

```bash
cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720

net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp.keepaliv.probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp.max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp.max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.top_timestamps = 0
net.core.somaxconn = 16384

net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
net.ipv6.conf.all.forwarding = 0
EOF

# 立即生效
sysctl --system
```

## 六、安装Containerd（三台机器都做）

### 1、删除可能存在残留

```bash
sudo apt-get remove -y docker docker-engine docker.io containerd runc
```

### 2、安装依赖

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common
```

### 3、信任GPG公钥

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

### 4、添加软件仓库

```bash
sudo add-apt-repository "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 5、安装

```bash
sudo apt-get -y update
sudo apt-get -y install containerd.io
```

### 6、确认版本可用

```bash
root@k8s-master-01:~# containerd --version
containerd containerd.io 1.6.6 10c12954828e7c7c9b6e0ea9b0c02b01407d3ae1
root@k8s-master-01:~# ctr --version
ctr containerd.io 1.6.6
root@k8s-master-01:~# ctr version
Client:
  Version:  1.6.6
  Revision: 10c12954828e7c7c9b6e0ea9b0c02b01407d3ae1
  Go version: go1.17.11

Server:
  Version:  1.6.6
  Revision: 10c12954828e7c7c9b6e0ea9b0c02b01407d3ae1
  UUID: cf75a2de-b75b-4e7f-ac61-53948a2682b6
```

### 7、生成配置文件

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

### 8、修改配置文件

> ​    由于一些众所周周知的原因，从谷歌无法拉取镜像，所以我们需要修改镜像仓库，由于配置文件太长，我这里只把需要修改的地方贴出来

```bash
vim  /etc/containerd/config.toml
...
# 将sandbox_image后面的谷歌仓库改为阿里云仓库地址
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"
# 将原来默认的false改为true，如果不修改可能会报warning提示cgroup控制器有问题（需要和kubelet的控制器保持一致）
systemd_cgroup = true
#如果不修改这个，后面可能无法正常拉取镜像
runtime_type = "io.containerd.runtime.v1.linux"
```

### 9、配置需要的模块

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

> 加载模块设置开机自启

```bash
systemctl restart systemd-modules-load.service
systemctl enable systemd-modules-load.service
```

### 10、配置containerd所需内核

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# 加载内核
sysctl --system
```



### 11、重启并加入开机自启

```bash
systemctl daemon-reload
systemctl enable --now containerd && systemctl status containerd
```

## 七、安装kubeadm、kubelet、kubectl（三台机器都执行）

> 使用阿里云源
>
> https://developer.aliyun.com/mirror/kubernetes?spm=a2c6h.13651102.0.0.7df41b11wHmwqe

### 1、导入GPG

```bash
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
```

### 2、配源

```bash
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
```

### 3、安装

```bash
apt-get update
apt-cache madison kubelet
apt-get install -y kubelet=1.24.3-00 kubeadm=1.24.3-00 kubectl=1.24.3-00
```

### 4、设置tab补全

```bash
apt install -y bash-completion
source /usr/share/bash-completion/bash_completion

source <(kubeadm completion bash)
source <(kubectl completion bash)
source <(crictl completion bash)


echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "source <(kubeadm completion bash)" >> ~/.bashrc
echo "source <(crictl completion bash)" >> ~/.bashrc
echo "source /usr/share/bash-completion/bash_completion" >> ~/.bashrc
```

### 5、修改crictl配置

```bash
vim /etc/crictl.yaml

runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 10
debug: false
pull-image-on-create: false
disable-pull-on-run: false
```

> 重启服务查看是否正常

```bash
systemctl daemon-reload && systemctl restart containerd
crictl images
```



## 八、k8s集群安装(仅master节点)

### 1、默认配置文件输出

```bash
kubeadm config print init-defaults > init.yaml
```

### 2、修改默认配置文件

```bash
root@k8s-master-01:~# cat init.yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.16.2.21  # 这个地址需要修改为master节点地址
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: k8s-master-01 # 需要和master的hostname匹配，否则会报错
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers # 修改为阿里云仓库，否则拉不下来镜像
kind: ClusterConfiguration
kubernetesVersion: 1.24.3 # 版本需要对应好，否则可能有兼容性问题
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12 # 子网可以自己喜欢，这个是后期分配给pod的
scheduler: {}
```

### 3、根据配置文件初始化集群

```bash
kubeadm init --config=init.yaml
```

### 4、建立用户集群权限

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#如果是root用户，则可以使用：
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```

### 5、node节点加入集群(node节点操作)

```bash
kubeadm join 172.16.2.21:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:3184df9e0dd5ce340e3133f45dg4dg5d4g5d4gfed81a86c8d2c1453
        
# 重新生成token（master节点）
kubeadm token create --print-join-command 
```

### 6、安装kubernetes网络插件

> kubernetes网络插件有很多，比如flannel、calico等等，具体区别可以自行查询，本次我选用的是[calico](https://projectcalico.docs.tigera.io/)

#### 1.下载配置

```bash
wget http://down.i4t.com/k8s1.24/calico.yaml
```

#### 2.修改配置

```bash
vim calicao.yaml
            - name: CALICO_IPV4POOL_CIDR
              value: "10.96.0.0/12"
...

            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            # 加上后面这句
            - name: IP_AUTODETECTION_METHOD
              value: "interface=eth0"
```

#### 3.集群应用

```bash
kubectl apply -f calico.yaml
```

## 九、验证

### 1、检查集群状态

```bash
root@k8s-master-01:~# kubectl get nodes
NAME            STATUS   ROLES           AGE   VERSION
k8s-master-01   Ready    control-plane   41m   v1.24.3
k8s-node-01     Ready    <none>          37m   v1.24.3
k8s-node-02     Ready    <none>          37m   v1.24.3
root@k8s-master-01:~# kubectl get pods -n kube-system -o wide
NAME                                       READY   STATUS    RESTARTS        AGE     IP              NODE            NOMINATED NODE   READINESS GATES
calico-kube-controllers-56cdb7c587-csw5b   1/1     Running   3 (5m51s ago)   15m     10.99.98.129    k8s-node-02     <none>           <none>
calico-node-2jc79                          1/1     Running   0               7m54s   172.16.2.21     k8s-master-01   <none>           <none>
calico-node-g68tv                          1/1     Running   1 (6m16s ago)   7m54s   172.16.2.23     k8s-node-02     <none>           <none>
calico-node-wpxtj                          1/1     Running   0               7m54s   172.16.2.22     k8s-node-01     <none>           <none>
calico-typha-6775694657-7fp84              1/1     Running   0               15m     172.16.2.23     k8s-node-02     <none>           <none>
coredns-74586cf9b6-mb67k                   1/1     Running   0               41m     10.110.107.66   k8s-master-01   <none>           <none>
coredns-74586cf9b6-r9z82                   1/1     Running   0               41m     10.110.107.65   k8s-master-01   <none>           <none>
etcd-k8s-master-01                         1/1     Running   0               41m     172.16.2.21     k8s-master-01   <none>           <none>
kube-apiserver-k8s-master-01               1/1     Running   0               41m     172.16.2.21     k8s-master-01   <none>           <none>
kube-controller-manager-k8s-master-01      1/1     Running   3 (7m35s ago)   41m     172.16.2.21     k8s-master-01   <none>           <none>
kube-proxy-4gsf2                           1/1     Running   0               37m     172.16.2.23     k8s-node-02     <none>           <none>
kube-proxy-57ftg                           1/1     Running   0               41m     172.16.2.21     k8s-master-01   <none>           <none>
kube-proxy-g57m7                           1/1     Running   0               37m     172.16.2.22     k8s-node-01     <none>           <none>
kube-scheduler-k8s-master-01               1/1     Running   3 (7m35s ago)   41m     172.16.2.21     k8s-master-01   <none>           <none>
```

### 2、验证DNS

```bash
root@k8s-master-01:~# kubectl run test -it --rm --image=busybox:1.28.3
If you don't see a command prompt, try pressing enter.
/ # nslookup kubernetes
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```

