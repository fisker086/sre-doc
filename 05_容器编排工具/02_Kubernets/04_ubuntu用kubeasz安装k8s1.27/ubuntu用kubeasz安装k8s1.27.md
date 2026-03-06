# ubuntu用kubeasz安装k8s1.27

## 一、生产服务器规格选择

### 1、master

> 8C8G60G 几百个pod
>
> 16C16G100G 上千个pod
>
> 32C32G   上万个pod

### 2、etcd

> 8C8G60G(高性能SSD)

### 3、node

>16C16G 80G(系统盘)、300G（数据盘）
>
>32C32G 80G(系统盘)、300G（数据盘）
>
>...

## 二、集群规划

### 1、架构简介

>k8s集群环境主要是kubernetes管理端服务(kube-apiserver、kube-controller-manager、kube-scheduler)的高可
>用实现，以及node节点上的(kubelet、kube-proxy)客户端服务的部署。

> Kubernetes设计架构：
>
> https://www.kubernetes.org.cn/kubernetes%E8%AE%BE%E8%AE%A1%E6%9E%B6%E6%9E%84

### 2、为什么要用二进制安装？

> 一些非二进制安装会有些未知问题
>
> kubeadm安装的k8s证书有效期默认是一年（365天）

### 3、服务器统计

| 类型            | 服务器IP地址             | 备注                                     |
| --------------- | ------------------------ | ---------------------------------------- |
| Ansible(2台)    | 172.31.7.101/102         | K8S集群部署服务器，可以和其他服务器混用  |
| K8S Master(3台) | 172.31.7.101/102/103     | K8s控制端，通过一个VIP做主备高可用       |
| Harbor(2台)     | 172.31.7.104/105         | 高可用镜像服务器                         |
| Etcd(最少3台)   | 172.31.7.106/107/108     | 保存k8s集群数据的服务器                  |
| Hproxy(2台)     | 172.31.7.109/110         | 高可用etcd代理服务器                     |
| Node节点(2-N台) | 172.31.7.111/112/113/xxx | 真正运行容器的服务器，高可用环境至少两台 |

### 4、服务器规划

> 服务器可以是私有云的虚拟机或物理机，也可以是公有云环境的虚拟机环境，如果是公司托管的IDC环境，可以直
> 接将harbor和node节点部署在物理机环境，master节点、etcd、负载均衡等可以是虚拟机。

|类型 |服务器IP |主机名 |VIP|
| ------------- | ----------------- | ------------- | ------- |
|模板机 |172.31.7.100 || |
|K8S Master1 |172.31.7.101 |k8s-master1 |172.31.7.188|
|K8S Master2 |172.31.7.102 |k8s-master2 |172.31.7.188|
|K8S Master3 |172.31.7.103 |k8s-master3 |172.31.7.188|
|Harbor1 |172.31.7.104 |k8s-harbor1| |
|Harbor2 |172.31.7.105 |k8s-harbor2| |
|etcd节点1 |172.31.7.106 |k8s-etcd1| |
|etcd节点2 |172.31.7.107 |k8s-etcd2| |
|etcd节点3 |172.31.7.108 |k8s-etcd3| |
|Haproxy1 |172.31.7.109 |k8s-ha1| |
|Haproxy2 |172.31.7.110 |k8s-ha2| |
|Node节点1 |172.31.7.111 |k8s-node1| |
|Node节点2 |172.31.7.112 |k8s-node2| |
|Node节点3 |172.31.7.113 |k8s-node3| |
|nfs-server |172.31.7.10 |nfs-server| |

## 三、基础环境准备(模板机制作)

> 系统主机名配置、IP配置、系统参数优化，以及依赖的负载均衡和Harbor部署

### 1、主机安装

> 建议自定义模板机进行克隆
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


# 重启ssh服务
```

#### 4.使用自己的shell工具连接

### 2、添加host解析

```bash
vim /etc/hosts
172.31.7.101  k8s-master1
172.31.7.102  k8s-master2
172.31.7.103  k8s-master3
172.31.7.104  k8s-harbor1
172.31.7.105  k8s-harbor2
172.31.7.106  k8s-etcd1
172.31.7.107  k8s-etcd2
172.31.7.108  k8s-etcd3
172.31.7.109  k8s-ha1
172.31.7.110  k8s-ha2
172.31.7.111  k8s-node1
172.31.7.112  k8s-node2
172.31.7.113  k8s-node3
172.31.7.10  nfs-server
```

### 3、添加DNS解析

```bash
vim /etc/resolv.conf
nameserver 114.114.114.114
```

### 4、开启命令行高亮

```bash
root@master01-virtual-machine:~# vim /root/.bashrc
force_color_prompt=yes   # 去掉这行注释

root@master01-virtual-machine:~# bash
```

或者

```bash
 vim /etc/bash.bashrc
...
force_color_prompt=yes  # 添加这行
```

### 5、修改网卡名称为eth*

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

### 6、配置apt源及安装常用组件

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

### 7、关闭防火墙

> Ubuntu中无selinux，防火墙服务为ufw，因此关闭防火墙并禁止防火墙开机自启动命令如下：

```bash
root@k8s-master-01:~# ufw disable
Firewall stopped and disabled on system startup
```

### 8、配置ntp时间同步

**推荐方式**

>vim /etc/systemd/timesyncd.conf

```bash
[Time]
NTP=ntp.aliyun.com
FallbackNTP=ntp.aliyun.com
#RootDistanceMaxSec=5
#PollIntervalMinSec=32
#PollIntervalMaxSec=2048
#ConnectionRetrySec=30
#SaveIntervalSec=60
```

```bash
sudo systemctl restart systemd-timesyncd
```

**ntp方式**

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
0 */1 * * * /usr/sbin/ntpdate ntp1.aliyun.com &>/dev/null && hwclock -w

# 检查结果
crontab -l
```



### 9、创建秘钥

```bash
rm -rf /root/.ssh

ssh-keygen

cd /root/.ssh/

mv id_rsa.pub authorized_keys
```

## 四、修改主机名（所有机器）

> 每个节点输入对应的命令

```bash
hostnamectl set-hostname k8s-master1
hostnamectl set-hostname k8s-master2
hostnamectl set-hostname k8s-master3
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node2
hostnamectl set-hostname k8s-node3
hostnamectl set-hostname k8s-etcd1
hostnamectl set-hostname k8s-etcd2
hostnamectl set-hostname k8s-etcd3
hostnamectl set-hostname k8s-ha1
hostnamectl set-hostname k8s-ha2
hostnamectl set-hostname k8s-harbor1
hostnamectl set-hostname k8s-harbor2
hostnamectl set-hostname nfs-server
```



## 五、负载均衡和高可用准备

> 172.31.7.109 172.31.7.110

### 1、keepalived高可用安装

> 没有具体的版本要求，支持高可用就行

#### 1.安装

```bash
apt update
apt install -y keepalived
```

#### 2.配置172.31.7.109

```bash
vim /etc/keepalived/keepalived.conf
global_defs {
    router_id k8s-ha1
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 50
    priority 150
    nopreempt
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.31.7.188 dev eth0 label eth0:1
        172.31.7.189 dev eth0 label eth0:2
        172.31.7.190 dev eth0 label eth0:3
    }
}
```

#### 3.配置172.31.7.110

```bash
vim /etc/keepalived/keepalived.conf
global_defs {
    router_id k8s-ha2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 50
    priority 100
    nopreempt
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.31.7.188 dev eth0 label eth0:1
        172.31.7.189 dev eth0 label eth0:2
        172.31.7.190 dev eth0 label eth0:3
    }
}
```

#### 4.配置说明

```bash
cat /etc/keepalived/keepalived.conf
global_defs {                   #全局配置
    router_id lb01              #标识身份->名称
}
 
vrrp_instance VI_1 {
    state MASTER                #标识角色状态
    interface eth0              #网卡绑定接口
    virtual_router_id 50        #虚拟路由id
    priority 150                #优先级
    nopreempt                   #非抢占式高可用
    advert_int 1                #监测间隔时间
    authentication {            #认证
        auth_type PASS          #认证方式
        auth_pass 1111          #认证密码
    }
    virtual_ipaddress {         
        10.0.0.3                #虚拟的VIP地址
    }
}

# 非抢占式注意点
	# 1、两个节点的state都必须配置为BACKUP
	# 2、两个节点都必须加上配置 nopreempt
	# 3、其中一个节点的优先级必须要高于另外一个节点的优先级。
	# 两台服务器都角色状态启用nopreempt后，必须修改角色状态统一为BACKUP，唯一的区分就是优先级。
```

#### 5.启动并加入开机自启

```bash
systemctl restart keepalived.service
systemctl enable keepalived.service
```

#### 6.验证

### 2、haproxy负载均衡器安装

#### 1.安装

```bash
apt install -y haproxy
```

#### 2.配置

> 两台机器一致

```bash
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http
listen k8s-api-6443
  bind 172.31.7.188:6443
  mode tcp
  server master1 172.31.7.101:6443 check inter 3s fall 3 rise 1
  server master2 172.31.7.102:6443 check inter 3s fall 3 rise 1
  server master3 172.31.7.103:6443 check inter 3s fall 3 rise 1
```

#### 3.修改内核参数允许监听本机不存在的IP地址

```bash
vim /etc/sysctl.conf
...

net.ipv4.ip_nonlocal_bind = 1


# 重载内核
sysctl --system
```

#### 3.启动

```bash
systemctl restart haproxy.service
systemctl enable haproxy.service
```

## 六、harbor

### 1、安装docker

```bash
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
apt update
apt install -y docker-ce docker-ce-cli containerd.io
```

### 2、安装docker-compose

```bash
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.6.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

### 3、创建证书并分发

#### 1.配置https证书（仅在k8s-harbor1上执行）

```bash
	默认情况下，Harbor不附带证书。可以在没有安全性的情况下部署Harbor，以便您可以通过HTTP连接到它。但是，只有在没有连接到外部Internet的空白测试或开发环境中，才可以使用HTTP。在没有空隙的环境中使用HTTP会使您遭受中间人攻击。要配置HTTPS，必须创建SSL证书。您可以使用由受信任的第三方CA签名的证书，也可以使用自签名证书。
```



##### 1) 生成证书颁发机构证书

```bash
	在生产环境中，您应该从CA获得证书。在测试或开发环境中，您可以生成自己的CA。要生成CA证书，请运行以下命令。
```

###### ①生成证书私钥

````bash
mkdir /opt/cert
cd /opt/cert
openssl genrsa -out ca.key 4096
````



###### ②生成CA证书

```bash
openssl req -x509 -new -nodes -sha512 -days 7120 \
 -subj "/C=CN/ST=ShangHai/L=ShangHai/O=trevor/OU=Linux/CN=harbor.trevorwu.com" \
 -key ca.key \
 -out ca.crt
```



###### ③生成服务器证书

```bash
openssl genrsa -out harbor.trevorwu.com.key 4096
```



###### ④生成证书签名请求

```bash
openssl req -sha512 -new \
 -subj "/C=CN/ST=ShangHai/L=ShangHai/O=trevor/OU=Linux/CN=harbor.trevorwu.com" \
 -key harbor.trevorwu.com.key \
 -out harbor.trevorwu.com.csr
```



##### 2）生成x509 va扩展文件

###### ①域名版

```bash
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.trevorwu.com
DNS.2=k8s-harbor1
DNS.3=k8s-harbor2
EOF
```



###### ②IP版

```bash
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = IP:192.168.15.110
EOF
```



##### 3）使用v3.ext文件生成证书

```bash
openssl x509 -req -sha512 -days 7120 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.local.csr \
    -out harbor.local.crt
```



#### 2.提供证书给Harbor和Docker

```bash
生成ca.crt后，harbor.trevorwu.com.crt和harbor.trevorwu.com.key文件，必须将它们提供给Harbor和Docker，然后重启和启动他们。
```

##### 1) 将服务器证书和密钥复制到Harbor主机上的certficates文件夹中

```bash
openssl x509 -inform PEM -in harbor.trevorwu.com.crt -out harbor.trevorwu.com.cert

mkdir -pv /etc/docker/certs.d/harbor.trevorwu.com/

cp harbor.trevorwu.com.cert /etc/docker/certs.d/harbor.trevorwu.com/
cp harbor.trevorwu.com.key /etc/docker/certs.d/harbor.trevorwu.com/
cp ca.crt /etc/docker/certs.d/harbor.trevorwu.com/

# 复制证书到harbor2
for i in 172.31.7.105;
do 
ssh root@$i "mkdir -pv /etc/docker/certs.d/harbor.trevorwu.com/"
scp harbor.trevorwu.com.cert root@$i:/etc/docker/certs.d/harbor.trevorwu.com/
scp harbor.trevorwu.com.key root@$i:/etc/docker/certs.d/harbor.trevorwu.com/
scp ca.crt root@$i:/etc/docker/certs.d/harbor.trevorwu.com/
done
```

##### 2) 重启docker，加载证书

```bash
systemctl restart docker
```



##### 3) 复制Harbor证书

```bash
mkdir -p /service/harbor/cert 
cp harbor.trevorwu.com.crt  /service/harbor/cert 
cp harbor.trevorwu.com.key /service/harbor/cert 

# 复制证书到harbor2
for i in 172.31.7.105;
do 
ssh root@$i "mkdir -pv /service/harbor/harbor/cert"
scp harbor.trevorwu.com.crt root@$i:/service/harbor/cert 
scp harbor.trevorwu.com.key root@$i:/service/harbor/cert
done
```

### 4、nfs挂载

> 机器性能有限，这里不做详解
>
> 过程如下：
>
> 1.nfs服务端创建
>
> 2.harbor服务器集群“/data”挂载到nfs
>
> 3.haproxy负载harbor集群

### 5、安装Harbor

#### 1.下载

```bash
cd /service/harbor

wget https://github.com/goharbor/harbor/releases/download/v2.6.1/harbor-offline-installer-v2.6.1.tgz
```

#### 2.解压

```bash
tar xf harbor-offline-installer-v2.6.1.tgz
```

#### 6.修改harbor.yml配置

```bash
# 复制一份
cp harbor.yml.tmpl harbor.yml

harbor.local
...
hostname: harbor.trevorwu.com
...
certificate: /service/harbor/cert/harbor.trevorwu.com.crt 
private_key: /service/harbor/cert/harbor.trevorwu.com.key
...
```



### 7、生成配置文件

```
./prepare
```

### 8、清空生成配置的残留

```
docker-compose down
```

### 9、安装并启动

```
./install.sh --with-trivy  --with-chartmuseum
```



### 10、访问

```
URL:http://harbor.trevorwu.com

用户名：admin
密码：Harbor12345
```



### 11、docker测试登录

```
docker login harbor.trevorwu.com --username=admin --password=Harbor12345

WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```





## 七、kubeasz安装k8s

> https://github.com/easzlab/kubeasz

### 1、基础环境准备

> 服用172.31.7.101安装k8s

#### 1.安装ansible

```bash
apt install -y ansible
```

#### 2.配置互相秘钥验证

> 略，模板机制作中已经做好了

#### 3.制作python软连接

> etcd集群，k8s-master,k8s-node节点执行

```bash
which python3.10
ln -s /usr/bin/python3.10 /usr/bin/python
```

### 2、在部署节点编排k8s安装

#### 1.下载kubeasz项目源码、二进制及离线镜像

##### 1）下载工具脚本ezdown，并制定版本

```bash
mkdir k8s-install
cd k8s-install/
export release=3.6.1
wget https://github.com/easzlab/kubeasz/releases/download/${release}/ezdown
chmod +x ./ezdown
```

##### 2）下载kubeasz代码、二进制、默认容器镜像（更多关于ezdown的参数，运行./ezdown 查看）

```bash
# 海外环境
./ezdown -D
# 国内环境
#./ezdown -D -m standard
```

#### 2.创建集群配置实例

```bash
root@k8s-master1:~/k8s-install# ./ezdown -S
2022-10-19 21:22:06 INFO Action begin: start_kubeasz_docker
2022-10-19 21:22:06 INFO try to run kubeasz in a container
2022-10-19 21:22:06 DEBUG get host IP: 172.31.7.101
f1be87d9cb8a4092c1fbd1d359fdc96b47f00694d7900cdae855313809b386ec
2022-10-19 21:22:06 INFO Action successed: start_kubeasz_docker

root@k8s-master1:~/k8s-install# docker exec -it kubeasz ezctl new k8s-cluster1
2022-10-19 13:22:38 DEBUG generate custom cluster files in /etc/kubeasz/clusters/k8s-cluster1
2022-10-19 13:22:38 DEBUG set versions
2022-10-19 13:22:38 DEBUG cluster k8s-cluster1: files successfully created.
2022-10-19 13:22:38 INFO next steps 1: to config '/etc/kubeasz/clusters/k8s-cluster1/hosts'
2022-10-19 13:22:38 INFO next steps 2: to config '/etc/kubeasz/clusters/k8s-cluster1/config.yml'
```

> ​    然后根据提示配置'/etc/kubeasz/clusters/k8s-cluster1/hosts' 和 '/etc/kubeasz/clusters/k8s-cluster1/config.yml'：根据前面节点规划修改hosts 文件和其他集群层面的主要配置选项；其他集群组件等配置项可以在config.yml 文件中修改。

#### 3.更改配置

##### 1）hosts

```bash
# 'etcd' cluster should have odd member(s) (1,3,5,...)
[etcd]
192.168.0.13
192.168.0.14
192.168.0.15

# master node(s), set unique 'k8s_nodename' for each node
# CAUTION: 'k8s_nodename' must consist of lower case alphanumeric characters, '-' or '.',
# and must start and end with an alphanumeric character
[kube_master]
192.168.0.13 k8s_nodename='k8smaster1'
192.168.0.14 k8s_nodename='k8smaster2'
192.168.0.15 k8s_nodename='k8smaster3'

# work node(s), set unique 'k8s_nodename' for each node
# CAUTION: 'k8s_nodename' must consist of lower case alphanumeric characters, '-' or '.',
# and must start and end with an alphanumeric character
[kube_node]
192.168.0.16 k8s_nodename='k8snode1'
192.168.0.17 k8s_nodename='k8snode2'

# [optional] harbor server, a private docker registry
# 'NEW_INSTALL': 'true' to install a harbor server; 'false' to integrate with existed one
[harbor]
#192.168.0.8 NEW_INSTALL=false

# [optional] loadbalance for accessing k8s from outside
[ex_lb]
#192.168.0.6 LB_ROLE=backup EX_APISERVER_VIP=192.168.0.250 EX_APISERVER_PORT=8443
#192.168.0.7 LB_ROLE=master EX_APISERVER_VIP=192.168.0.250 EX_APISERVER_PORT=8443

# [optional] ntp server for the cluster
[chrony]
#192.168.0.1

[all:vars]
# --------- Main Variables ---------------
# Secure port for apiservers
SECURE_PORT="6443"

# Cluster container-runtime supported: docker, containerd
# if k8s version >= 1.24, docker is not supported
CONTAINER_RUNTIME="containerd"

# Network plugins supported: calico, flannel, kube-router, cilium, kube-ovn
CLUSTER_NETWORK="calico"

# Service proxy mode of kube-proxy: 'iptables' or 'ipvs'
PROXY_MODE="ipvs"

# K8S Service CIDR, not overlap with node(host) networking
SERVICE_CIDR="10.100.0.0/16"

# Cluster CIDR (Pod CIDR), not overlap with node(host) networking
CLUSTER_CIDR="10.200.0.0/16"

# NodePort Range
NODE_PORT_RANGE="0-32767"

# Cluster DNS Domain
CLUSTER_DNS_DOMAIN="k8s.dudewu.top"

# -------- Additional Variables (don't change the default value right now) ---
# Binaries Directory
bin_dir="/opt/kube/bin"

# Deploy Directory (kubeasz workspace)
base_dir="/etc/kubeasz"

# Directory for a specific cluster
cluster_dir="{{ base_dir }}/clusters/k8s-cluster1"

# CA and other components cert/key Directory
ca_dir="/etc/kubernetes/ssl"

# Default 'k8s_nodename' is empty
k8s_nodename=''

# Default python interpreter
ansible_python_interpreter=/usr/bin/python3
```

##### 2)config.yml

```bash
############################
# prepare
############################
# 可选离线安装系统软件包 (offline|online)
INSTALL_SOURCE: "online"

# 可选进行系统安全加固 github.com/dev-sec/ansible-collection-hardening
OS_HARDEN: false


############################
# role:deploy
############################
# default: ca will expire in 100 years
# default: certs issued by the ca will expire in 50 years
CA_EXPIRY: "876000h"
CERT_EXPIRY: "876000h"

# kubeconfig 配置参数
CLUSTER_NAME: "cluster1"
CONTEXT_NAME: "context-{{ CLUSTER_NAME }}"

# k8s version
K8S_VER: "1.25.1"

############################
# role:etcd
############################
# 设置不同的wal目录，可以避免磁盘io竞争，提高性能
ETCD_DATA_DIR: "/var/lib/etcd"
ETCD_WAL_DIR: ""


############################
# role:runtime [containerd,docker]
############################
# ------------------------------------------- containerd
# [.]启用容器仓库镜像
ENABLE_MIRROR_REGISTRY: true

# [containerd]基础容器镜像
SANDBOX_IMAGE: "easzlab.io.local:5000/easzlab/pause:3.8"

# [containerd]容器持久化存储目录
CONTAINERD_STORAGE_DIR: "/var/lib/containerd"

# ------------------------------------------- docker
# [docker]容器存储目录
DOCKER_STORAGE_DIR: "/var/lib/docker"

# [docker]开启Restful API
ENABLE_REMOTE_API: false

# [docker]信任的HTTP仓库
INSECURE_REG: '["http://easzlab.io.local:5000","172.31.7.119",“172.31.7.104”]'


############################
# role:kube-master
############################
# k8s 集群 master 节点证书配置，可以添加多个ip和域名（比如增加公网ip和域名）
MASTER_CERT_HOSTS:
  - "172.31.7.188"
  - "k8s.trevorwu.io"
  #- "www.test.com"

# node 节点上 pod 网段掩码长度（决定每个节点最多能分配的pod ip地址）
# 如果flannel 使用 --kube-subnet-mgr 参数，那么它将读取该设置为每个节点分配pod网段
# https://github.com/coreos/flannel/issues/847
NODE_CIDR_LEN: 16


############################
# role:kube-node
############################
# Kubelet 根目录
KUBELET_ROOT_DIR: "/var/lib/kubelet"

# node节点最大pod 数
MAX_PODS: 500

# 配置为kube组件（kubelet,kube-proxy,dockerd等）预留的资源量
# 数值设置详见templates/kubelet-config.yaml.j2
KUBE_RESERVED_ENABLED: "no"

# k8s 官方不建议草率开启 system-reserved, 除非你基于长期监控，了解系统的资源占用状况；
# 并且随着系统运行时间，需要适当增加资源预留，数值设置详见templates/kubelet-config.yaml.j2
# 系统预留设置基于 4c/8g 虚机，最小化安装系统服务，如果使用高性能物理机可以适当增加预留
# 另外，集群安装时候apiserver等资源占用会短时较大，建议至少预留1g内存
SYS_RESERVED_ENABLED: "no"


############################
# role:network [flannel,calico,cilium,kube-ovn,kube-router]
############################
# ------------------------------------------- flannel
# [flannel]设置flannel 后端"host-gw","vxlan"等
FLANNEL_BACKEND: "vxlan"
DIRECT_ROUTING: false

# [flannel]
flannel_ver: "v0.19.2"

# ------------------------------------------- calico
# [calico] IPIP隧道模式可选项有: [Always, CrossSubnet, Never],跨子网可以配置为Always与CrossSubnet(公有云建议使用always比较省事，其他的话需要修改各自公有云的网络配置，具体可以参考各个公有云说明)
# 其次CrossSubnet为隧道+BGP路由混合模式可以提升网络性能，同子网配置为Never即可.
CALICO_IPV4POOL_IPIP: "Always"

# [calico]设置 calico-node使用的host IP，bgp邻居通过该地址建立，可手工指定也可以自动发现
IP_AUTODETECTION_METHOD: "can-reach={{ groups['kube_master'][0] }}"

# [calico]设置calico 网络 backend: brid, vxlan, none
CALICO_NETWORKING_BACKEND: "brid"

# [calico]设置calico 是否使用route reflectors
# 如果集群规模超过50个节点，建议启用该特性
CALICO_RR_ENABLED: false

# CALICO_RR_NODES 配置route reflectors的节点，如果未设置默认使用集群master节点
# CALICO_RR_NODES: ["192.168.1.1", "192.168.1.2"]
CALICO_RR_NODES: []

# [calico]更新支持calico 版本: ["3.19", "3.23"]
calico_ver: "v3.23.3"

# [calico]calico 主版本
calico_ver_main: "{{ calico_ver.split('.')[0] }}.{{ calico_ver.split('.')[1] }}"

# ------------------------------------------- cilium
# [cilium]镜像版本
cilium_ver: "1.12.2"
cilium_connectivity_check: true
cilium_hubble_enabled: false
cilium_hubble_ui_enabled: false

# ------------------------------------------- kube-ovn
# [kube-ovn]选择 OVN DB and OVN Control Plane 节点，默认为第一个master节点
OVN_DB_NODE: "{{ groups['kube_master'][0] }}"

# [kube-ovn]离线镜像tar包
kube_ovn_ver: "v1.5.3"

# ------------------------------------------- kube-router
# [kube-router]公有云上存在限制，一般需要始终开启 ipinip；自有环境可以设置为 "subnet"
OVERLAY_TYPE: "full"

# [kube-router]NetworkPolicy 支持开关
FIREWALL_ENABLE: true

# [kube-router]kube-router 镜像版本
kube_router_ver: "v0.3.1"
busybox_ver: "1.28.4"


############################
# role:cluster-addon
############################
# 生产部署建议打开，为了学习手动安装dns组件
# coredns 自动安装
dns_install: "no"
corednsVer: "1.9.3"
ENABLE_LOCAL_DNS_CACHE: false
dnsNodeCacheVer: "1.22.8"
# 设置 local dns cache 地址
LOCAL_DNS_CACHE: "169.254.20.10"

# metric server 自动安装
metricsserver_install: "no"
metricsVer: "v0.5.2"

# dashboard 自动安装
dashboard_install: "no"
dashboardVer: "v2.6.1"
dashboardMetricsScraperVer: "v1.0.8"

# prometheus 自动安装
prom_install: "no"
prom_namespace: "monitor"
prom_chart_ver: "39.11.0"

# nfs-provisioner 自动安装
nfs_provisioner_install: "no"
nfs_provisioner_namespace: "kube-system"
nfs_provisioner_ver: "v4.0.2"
nfs_storage_class: "managed-nfs-storage"
nfs_server: "192.168.1.10"
nfs_path: "/data/nfs"

# network-check 自动安装
network_check_enabled: false
network_check_schedule: "*/5 * * * *"

############################
# role:harbor
############################
# harbor version，完整版本号
HARBOR_VER: "v2.1.3"
HARBOR_DOMAIN: "harbor.easzlab.io.local"
HARBOR_TLS_PORT: 8443

# if set 'false', you need to put certs named harbor.pem and harbor-key.pem in directory 'down'
HARBOR_SELF_SIGNED_CERT: true

# install extra component
HARBOR_WITH_NOTARY: false
HARBOR_WITH_TRIVY: false
HARBOR_WITH_CLAIR: false
HARBOR_WITH_CHARTMUSEUM: true
```

##### 3）关闭ansible负载均衡、时间服务器初始化

```bash
vim /etc/kubeasz/playbooks/01.prepare.yml
# [optional] to synchronize system time of nodes with 'chrony'
- hosts:
  - kube_master
  - kube_node
  - etcd
  - ex_lb   # 删除这行
  - chrony  # 删除这行
  roles:
  - { role: os-harden, when: "OS_HARDEN|bool" }
  - { role: chrony, when: "groups['chrony']|length > 0" }

# to create CA, kubeconfig, kube-proxy.kubeconfig etc.
- hosts: localhost
  roles:
  - deploy

# prepare tasks for all nodes
- hosts:
  - kube_master
  - kube_node
  - etcd
  roles:
  - prepare

```

#### 4.安装

```bash
#建议配置命令alias，方便执行
echo "alias dk='docker exec -it kubeasz'" >> /root/.bashrc
source /root/.bashrc

# 一键安装，等价于执行docker exec -it kubeasz ezctl setup k8s-01 all
# dk ezctl setup k8s-01 all

# 或者分步安装，具体使用 dk ezctl help setup 查看分步安装帮助信息
dk ezctl setup k8s-cluster1 01
dk ezctl setup k8s-cluster1 02
dk ezctl setup k8s-cluster1 03
dk ezctl setup k8s-cluster1 04    # 这个node不是ready状态不要慌，先执行05 06
dk ezctl setup k8s-cluster1 05    # 这个node不是ready状态不要慌，先执行06
dk ezctl setup k8s-cluster1 06
```

#### 5.测试验证

```bash
kubectl run net-test1 --image=centos:7.9.2009 sleep 3600000
kubectl run net-test2 --image=centos:7.9.2009 sleep 3600000
kubectl run net-test3 --image=centos:7.9.2009 sleep 3600000
kubectl run net-test4 --image=centos:7.9.2009 sleep 3600000

root@k8s-master1:/etc/kubeasz/playbooks# kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
net-test1   1/1     Running   0          2m24s   10.200.36.65     172.31.7.111   <none>           <none>
net-test2   1/1     Running   0          96s     10.200.36.66     172.31.7.111   <none>           <none>
net-test3   1/1     Running   0          91s     10.200.169.129   172.31.7.112   <none>           <none>
net-test4   1/1     Running   0          82s     10.200.169.130   172.31.7.112   <none>           <none>

# 进入其中一个容器
root@k8s-master1:/etc/kubeasz/playbooks# kubectl exec -it net-test1 bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
[root@net-test1 /]# ping 10.200.169.130
[root@net-test1 /]# ping 114.114.114.114
```

### 3、containerd接入Harbor

> 所有k8s节点执行

#### 1.创建证书存放目录

```bash
mkdir -p /etc/containerd/certs.d/harbor.trevorwu.com
```

#### 2.查看是否有配置文件，无则创建

```bash
 cat /etc/containerd/config.toml
 
 # 创建
containerd config default > /etc/containerd/config.toml
```

#### 3.拉取证书

```bash
rsync -avz root@172.31.7.104:/etc/docker/certs.d/harbor.trevorwu.com/* /etc/containerd/certs.d/harbor.trevorwu.com/
```

#### 4.创建软连接（好像不需要执行）

```bash
ln -s /etc/containerd/certs.d/harbor.trevorwu.com/ca.crt /etc/ssl/certs/
ln -s /etc/containerd/certs.d/harbor.trevorwu.com/harbor.trevorwu.com.cert /etc/ssl/certs/
ln -s /etc/containerd/certs.d/harbor.trevorwu.com/harbor.trevorwu.com.key /etc/ssl/certs/
```

#### 5.更新（好像不需要执行）

```bash
update-ca-certificates
```

#### 6.安装nerdctl

```bash
https://github.com/containerd/nerdctl/releases
tar xf nerdctl-0.23.0-linux-amd64.tar.gz
mv nerdctl ../bin/
```

#### 7.修改contanerd配置文件

> root@k8s-node1:~# cat /etc/containerd/config.toml

```bash
disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
temp = ""
version = 2

[cgroup]
  path = ""

[debug]
  address = ""
  format = ""
  gid = 0
  level = ""
  uid = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216
  tcp_address = ""
  tcp_tls_ca = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0

[metrics]
  address = ""
  grpc_histogram = false

[plugins]

  [plugins."io.containerd.gc.v1.scheduler"]
    deletion_threshold = 0
    mutation_threshold = 100
    pause_threshold = 0.02
    schedule_delay = "0s"
    startup_delay = "100ms"

  [plugins."io.containerd.grpc.v1.cri"]
    device_ownership_from_security_context = false
    disable_apparmor = false
    disable_cgroup = false
    disable_hugetlb_controller = true
    disable_proc_mount = false
    disable_tcp_service = true
    enable_selinux = false
    enable_tls_streaming = false
    enable_unprivileged_icmp = false
    enable_unprivileged_ports = false
    ignore_image_defined_volumes = false
    max_concurrent_downloads = 3
    max_container_log_line_size = 16384
    netns_mounts_under_state_dir = false
    restrict_oom_score_adj = false
    sandbox_image = "easzlab.io.local:5000/easzlab/pause:3.8"
    selinux_category_range = 1024
    stats_collect_period = 10
    stream_idle_timeout = "4h0m0s"
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    systemd_cgroup = false
    tolerate_missing_hugetlb_controller = true
    unset_seccomp_profile = ""

    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/usr/local/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = "/etc/cni/net.d/10-default.conf"
      max_conf_num = 1

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      disable_snapshot_annotations = true
      discard_unpacked_layers = false
      ignore_rdt_not_enabled_errors = false
      no_pivot = false
      snapshotter = "overlayfs"

      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        base_runtime_spec = ""
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          base_runtime_spec = ""
          container_annotations = []
          pod_annotations = []
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = true

      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        base_runtime_spec = ""
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime.options]

    [plugins."io.containerd.grpc.v1.cri".image_decryption]
      key_model = "node"

    [plugins."io.containerd.grpc.v1.cri".registry]

      [plugins."io.containerd.grpc.v1.cri".registry.auths]

      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."easzlab.io.local:5000"]
          [plugins."io.containerd.grpc.v1.cri".registry.configs."easzlab.io.local:5000".tls]
            insecure_skip_verify = true
        [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.trevorwu.com"]
          [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.trevorwu.com".tls] #harbor 证书认证配置
            ca_file   = "/etc/containerd/certs.d/harbor.trevorwu.com/ca.crt" #CA 证书
            cert_file = "/etc/containerd/certs.d/harbor.trevorwu.com/harbor.trevorwu.com.cert"  #harbor证书
            key_file  = "/etc/containerd/certs.d/harbor.trevorwu.com/harbor.trevorwu.com.key"  #harbor私钥
          [plugin."io.containerd.grpc.v1.cri".registry.configs."harbor.trevorwu.com".auth]  #配置凭据
            username = "admin"  #Harbor用户名
            password = "Harbor12345" #Harbor密码

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."easzlab.io.local:5000"]
          endpoint = ["http://easzlab.io.local:5000"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://docker.mirrors.ustc.edu.cn", "http://hub-mirror.c.163.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."gcr.io"]
          endpoint = ["https://gcr.mirrors.ustc.edu.cn"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["https://gcr.mirrors.ustc.edu.cn/google-containers/"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
          endpoint = ["https://quay.mirrors.ustc.edu.cn"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbor.trevorwu.com"]
          endpoint = ["https://harbor.trevorwu.com"] #Harbor服务地址

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""

  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"

  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"

  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"

  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false

  [plugins."io.containerd.runtime.v1.linux"]
    no_shim = false
    runtime = "runc"
    runtime_root = ""
    shim = "containerd-shim"
    shim_debug = false

  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]

  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]

  [plugins."io.containerd.snapshotter.v1.aufs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.btrfs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.devmapper"]
    async_remove = false
    base_image_size = ""
    pool_name = ""
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.native"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.overlayfs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.zfs"]
    root_path = ""

[proxy_plugins]

[stream_processors]

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar"

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar+gzip"

[timeouts]
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"

[ttrpc]
  address = ""
  gid = 0
  uid = 0
```

#### 8.重载

```bash
systemctl daemon-reload && systemctl restart containerd.service
systemctl status containerd.service
```

#### 9.登录

```bash
root@k8s-master1:/etc/containerd# nerdctl login harbor.trevorwu.com
Enter Username: admin
Enter Password:
WARNING: Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```



## 八、如何获取k8s相关的yaml文件？

### 1、去官网下载

> 根据自己的版本下载

> https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.25.md#downloads-for-v1251

#### 1.下载压缩包

> 下载完后为了方便区分可以加上版本号，然后上传到master1的/usr/local/src下

![1-获取yaml方式1](1-获取yaml方式1.png)

![2-获取yaml方式2](2-获取yaml方式2.png)

### 2、解压

```bash
root@k8s-master1:/usr/local/src# ll
total 476916
drwxr-xr-x  2 root root      4096 10月 20 21:03 ./
drwxr-xr-x 10 root root      4096 8月  31 14:52 ../
-rw-r--r--  1 root root    516068 10月 20 21:03 kubernete-1.25.1.tar.gz
-rw-r--r--  1 root root  30881786 10月 20 21:03 kubernetes-1.25.1-client-linux-amd64.tar.gz
-rw-r--r--  1 root root 124784595 10月 20 21:03 kubernetes-1.25.1-node-linux-amd64.tar.gz
-rw-r--r--  1 root root 332157931 10月 20 21:03 kubernetes1.25.1-server-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src# tar xf kubernete-1.25.1.tar.gz
root@k8s-master1:/usr/local/src# tar xf kubernetes-1.25.1-client-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src# tar xf kubernetes-1.25.1-node-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src# tar xf kubernetes1.25.1-server-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src# ll
total 476920
drwxr-xr-x  3 root root      4096 10月 20 21:03 ./
drwxr-xr-x 10 root root      4096 8月  31 14:52 ../
-rw-r--r--  1 root root    516068 10月 20 21:03 kubernete-1.25.1.tar.gz
drwxr-xr-x 10 root root      4096 9月  15 04:06 kubernetes/
-rw-r--r--  1 root root  30881786 10月 20 21:03 kubernetes-1.25.1-client-linux-amd64.tar.gz
-rw-r--r--  1 root root 124784595 10月 20 21:03 kubernetes-1.25.1-node-linux-amd64.tar.gz
-rw-r--r--  1 root root 332157931 10月 20 21:03 kubernetes1.25.1-server-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src# cd kubernetes/
root@k8s-master1:/usr/local/src/kubernetes# ll
total 37204
drwxr-xr-x 10 root root     4096 9月  15 04:06 ./
drwxr-xr-x  3 root root     4096 10月 20 21:03 ../
drwxr-xr-x  2 root root     4096 9月  15 04:06 addons/
drwxr-xr-x  3 root root     4096 9月  15 04:04 client/
drwxr-xr-x  9 root root     4096 9月  15 04:11 cluster/
drwxr-xr-x  2 root root     4096 9月  15 04:11 docs/
drwxr-xr-x  3 root root     4096 9月  15 04:11 hack/
-rw-r--r--  1 root root 38041246 9月  15 04:06 kubernetes-src.tar.gz
drwxr-xr-x  4 root root     4096 9月  15 04:06 LICENSES/
drwxr-xr-x  3 root root     4096 9月  15 04:04 node/
-rw-r--r--  1 root root     4407 9月  15 04:11 README.md
drwxr-xr-x  3 root root     4096 9月  15 04:04 server/
-rw-r--r--  1 root root        8 9月  15 04:11 version
```

### 3、内部文件

#### 1.脚本

```bash
root@k8s-master1:/usr/local/src/kubernetes# ll cluster/
total 112
drwxr-xr-x  9 root root  4096 9月  15 04:11 ./
drwxr-xr-x 10 root root  4096 9月  15 04:06 ../
drwxr-xr-x 18 root root  4096 9月  15 04:11 addons/
-rwxr-xr-x  1 root root 17970 9月  15 04:11 common.sh*
drwxr-xr-x  6 root root  4096 9月  15 04:11 gce/
-rwxr-xr-x  1 root root  9457 9月  15 04:11 get-kube-binaries.sh*
-rwxr-xr-x  1 root root  9003 9月  15 04:11 get-kube.sh*
drwxr-xr-x  5 root root  4096 9月  15 04:11 images/
-rwxr-xr-x  1 root root  2868 9月  15 04:11 kubectl.sh*
-rwxr-xr-x  1 root root  1112 9月  15 04:11 kube-down.sh*
drwxr-xr-x  5 root root  4096 9月  15 04:11 kubemark/
-rwxr-xr-x  1 root root  2483 9月  15 04:11 kube-up.sh*
-rwxr-xr-x  1 root root  1414 9月  15 04:11 kube-util.sh*
drwxr-xr-x  2 root root  4096 9月  15 04:11 log-dump/
-rw-r--r--  1 root root   305 9月  15 04:11 OWNERS
drwxr-xr-x  2 root root  4096 9月  15 04:11 pre-existing/
-rw-r--r--  1 root root   331 9月  15 04:11 README.md
drwxr-xr-x  2 root root  4096 9月  15 04:11 skeleton/
-rwxr-xr-x  1 root root  7476 9月  15 04:11 validate-cluster.sh*
```

#### 2.插件yaml文件

```bash
root@k8s-master1:/usr/local/src/kubernetes# ll cluster/addons
total 80
drwxr-xr-x 18 root root 4096 9月  15 04:11 ./
drwxr-xr-x  9 root root 4096 9月  15 04:11 ../
drwxr-xr-x  2 root root 4096 9月  15 04:11 addon-manager/
drwxr-xr-x  3 root root 4096 9月  15 04:11 calico-policy-controller/
drwxr-xr-x  3 root root 4096 9月  15 04:11 cluster-loadbalancing/
drwxr-xr-x  3 root root 4096 9月  15 04:11 device-plugins/
drwxr-xr-x  5 root root 4096 9月  15 04:11 dns/
drwxr-xr-x  2 root root 4096 9月  15 04:11 dns-horizontal-autoscaler/
drwxr-xr-x  3 root root 4096 9月  15 04:11 fluentd-gcp/
drwxr-xr-x  3 root root 4096 9月  15 04:11 ip-masq-agent/
drwxr-xr-x  2 root root 4096 9月  15 04:11 kube-proxy/
drwxr-xr-x  3 root root 4096 9月  15 04:11 metadata-agent/
drwxr-xr-x  3 root root 4096 9月  15 04:11 metadata-proxy/
drwxr-xr-x  2 root root 4096 9月  15 04:11 metrics-server/
drwxr-xr-x  5 root root 4096 9月  15 04:11 node-problem-detector/
-rw-r--r--  1 root root  104 9月  15 04:11 OWNERS
drwxr-xr-x  8 root root 4096 9月  15 04:11 rbac/
-rw-r--r--  1 root root 1655 9月  15 04:11 README.md
drwxr-xr-x  8 root root 4096 9月  15 04:11 storage-class/
drwxr-xr-x  4 root root 4096 9月  15 04:11 volumesnapshots/
```

#### 3.二进制

```bash
root@k8s-master1:/usr/local/src/kubernetes# ll server/bin/
total 1025756
drwxr-xr-x 2 root root      4096 9月  15 04:06 ./
drwxr-xr-x 3 root root      4096 9月  15 04:04 ../
-rwxr-xr-x 1 root root  54640640 9月  15 04:06 apiextensions-apiserver*
-rwxr-xr-x 1 root root  43798528 9月  15 04:06 kubeadm*
-rwxr-xr-x 1 root root  48861184 9月  15 04:06 kube-aggregator*
-rwxr-xr-x 1 root root 123834368 9月  15 04:06 kube-apiserver*
-rw-r--r-- 1 root root         8 9月  15 04:05 kube-apiserver.docker_tag
-rw------- 1 root root 129078272 9月  15 04:05 kube-apiserver.tar
-rwxr-xr-x 1 root root 113205248 9月  15 04:06 kube-controller-manager*
-rw-r--r-- 1 root root         8 9月  15 04:05 kube-controller-manager.docker_tag
-rw------- 1 root root 118449152 9月  15 04:05 kube-controller-manager.tar
-rwxr-xr-x 1 root root  45015040 9月  15 04:06 kubectl*
-rwxr-xr-x 1 root root  53135912 9月  15 04:06 kubectl-convert*
-rwxr-xr-x 1 root root 114229208 9月  15 04:06 kubelet*
-rwxr-xr-x 1 root root   1536000 9月  15 04:06 kube-log-runner*
-rwxr-xr-x 1 root root  41168896 9月  15 04:06 kube-proxy*
-rw-r--r-- 1 root root         8 9月  15 04:05 kube-proxy.docker_tag
-rw------- 1 root root  63284736 9月  15 04:05 kube-proxy.tar
-rwxr-xr-x 1 root root  46690304 9月  15 04:06 kube-scheduler*
-rw-r--r-- 1 root root         8 9月  15 04:05 kube-scheduler.docker_tag
-rw------- 1 root root  51934208 9月  15 04:05 kube-scheduler.tar
-rwxr-xr-x 1 root root   1458176 9月  15 04:06 mounter*
```

## 九、安装coredns

### 1、复制coredns.yaml文件

```bash
mkdir /root/k8s-install.yaml.d
cp /usr/local/src/kubernetes/cluster/addons/dns/coredns/coredns.yaml.base /root/k8s-install.yaml.d/
cd /root/k8s-install.yaml.d/
mv coredns.yaml.base coredns.yaml
```

### 2、修改

#### 1.查看部署k8s时的域名和coredns的service地址

```bash
root@k8s-master1:~# cat /etc/kubeasz/clusters/k8s-cluster1/hosts | grep  DOMAIN
CLUSTER_DNS_DOMAIN="trevorwu.local"

root@k8s-master1:~/k8s-install.yaml.d# kubectl exec -it net-test1 cat /etc/resolv.conf
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
search default.svc.trevorwu.local svc.trevorwu.local trevorwu.local
nameserver 10.100.0.2
options ndots:5
```

#### 2.修改yaml文件

```bash
vim coredns.yaml
...
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes trevorwu.local in-addr.arpa ip6.arpa {  # 配置k8s集群域名
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . 114.114.114.114 {   # 配置自己解析不了的域名去哪解析
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
...
      containers:
      - name: coredns
        image: coredns/coredns:1.9.3   # 换成hub.docker上面的镜像
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 200Mi    # 内存限制大小
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
...
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.100.0.2   # 上面获取的容器nameserver的地址
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
```

### 3、运行

```bash
root@k8s-master1:~/k8s-install.yaml.d# kubectl apply -f coredns.yaml
root@k8s-master1:~/k8s-install.yaml.d# kubectl get pods -n kube-system -o wide
NAME                                       READY   STATUS    RESTARTS      AGE   IP               NODE           NOMINATED NODE   READINESS GATES
calico-kube-controllers-6d5cf54455-64jf9   1/1     Running   1 (85m ago)   22h   172.31.7.112     172.31.7.112   <none>           <none>
calico-node-csktv                          1/1     Running   1 (85m ago)   22h   172.31.7.112     172.31.7.112   <none>           <none>
calico-node-jv25x                          1/1     Running   1 (86m ago)   22h   172.31.7.102     172.31.7.102   <none>           <none>
calico-node-nvwq5                          1/1     Running   1 (86m ago)   22h   172.31.7.101     172.31.7.101   <none>           <none>
calico-node-rqx6p                          1/1     Running   1 (85m ago)   22h   172.31.7.111     172.31.7.111   <none>           <none>
coredns-8496f84465-zpvjf                   1/1     Running   0             15s   10.200.169.133   172.31.7.112   <none>           <none>
root@k8s-master1:~/k8s-install.yaml.d# kubectl get svc -n kube-system -o wide
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE     SELECTOR
kube-dns   ClusterIP   10.100.0.2   <none>        53/UDP,53/TCP,9153/TCP   5m11s   k8s-app=kube-dns
```

### 4、验证

```bash
root@k8s-master1:~/k8s-install.yaml.d# kubectl exec -it net-test1 ping baidu.com
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
PING baidu.com (39.156.66.10) 56(84) bytes of data.
64 bytes from 39.156.66.10 (39.156.66.10): icmp_seq=1 ttl=127 time=32.1 ms
64 bytes from 39.156.66.10 (39.156.66.10): icmp_seq=2 ttl=127 time=34.3 ms

# 同个名称空间的资源可以直接ping
root@k8s-master1:~/k8s-install.yaml.d# kubectl exec -it net-test1 ping kubernetes
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
PING kubernetes.default.svc.trevorwu.local (10.100.0.1) 56(84) bytes of data.
64 bytes from kubernetes.default.svc.trevorwu.local (10.100.0.1): icmp_seq=1 ttl=64 time=0.036 ms
64 bytes from kubernetes.default.svc.trevorwu.local (10.100.0.1): icmp_seq=2 ttl=64 time=0.049 ms
64 bytes from kubernetes.default.svc.trevorwu.local (10.100.0.1): icmp_seq=3 ttl=64 time=0.077 ms

# 不同名称空间ping格式：名称.名称空间.类型.api-server域名
root@k8s-master1:~/k8s-install.yaml.d# kubectl exec -it net-test1 ping kube-dns.kube-system.svc.trevorwu.local
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
PING kube-dns.kube-system.svc.trevorwu.local (10.100.0.2) 56(84) bytes of data.
64 bytes from kube-dns.kube-system.svc.trevorwu.local (10.100.0.2): icmp_seq=1 ttl=64 time=0.025 ms
64 bytes from kube-dns.kube-system.svc.trevorwu.local (10.100.0.2): icmp_seq=2 ttl=64 time=0.059 ms

root@k8s-master1:~/k8s-install.yaml.d# kubectl exec -it net-test1 yum install -y dns-utils
root@k8s-master1:~/k8s-install.yaml.d# kubectl exec net-test1 nslookup kubernetes
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
Server:         10.100.0.2
Address:        10.100.0.2#53

Name:   kubernetes.default.svc.trevorwu.local
Address: 10.100.0.1
```

### 5、多副本

>DNS响应变慢
>
>   1.副本数太少
>
>   2.资源不够
>
>   3.开dns缓存

```bash
root@k8s-master1:~/k8s-install.yaml.d# kubectl edit  deployments coredns  -n kube-system

apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"addonmanager.kubernetes.io/mode":"Reconcile","k8s-app":"kube-dns","kubernetes.io/cluster-service":"true","kubernetes.io/name":"CoreDNS"},"name":"coredns","namespace":"kube-system"},"spec":{"selector":{"matchLabels":{"k8s-app":"kube-dns"}},"strategy":{"rollingUpdate":{"maxUnavailable":1},"type":"RollingUpdate"},"template":{"metadata":{"labels":{"k8s-app":"kube-dns"}},"spec":{"affinity":{"podAntiAffinity":{"preferredDuringSchedulingIgnoredDuringExecution":[{"podAffinityTerm":{"labelSelector":{"matchExpressions":[{"key":"k8s-app","operator":"In","values":["kube-dns"]}]},"topologyKey":"kubernetes.io/hostname"},"weight":100}]}},"containers":[{"args":["-conf","/etc/coredns/Corefile"],"image":"coredns/coredns:1.9.3","imagePullPolicy":"IfNotPresent","livenessProbe":{"failureThreshold":5,"httpGet":{"path":"/health","port":8080,"scheme":"HTTP"},"initialDelaySeconds":60,"successThreshold":1,"timeoutSeconds":5},"name":"coredns","ports":[{"containerPort":53,"name":"dns","protocol":"UDP"},{"containerPort":53,"name":"dns-tcp","protocol":"TCP"},{"containerPort":9153,"name":"metrics","protocol":"TCP"}],"readinessProbe":{"httpGet":{"path":"/ready","port":8181,"scheme":"HTTP"}},"resources":{"limits":{"memory":"200Mi"},"requests":{"cpu":"100m","memory":"70Mi"}},"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"add":["NET_BIND_SERVICE"],"drop":["all"]},"readOnlyRootFilesystem":true},"volumeMounts":[{"mountPath":"/etc/coredns","name":"config-volume","readOnly":true}]}],"dnsPolicy":"Default","nodeSelector":{"kubernetes.io/os":"linux"},"priorityClassName":"system-cluster-critical","securityContext":{"seccompProfile":{"type":"RuntimeDefault"}},"serviceAccountName":"coredns","tolerations":[{"key":"CriticalAddonsOnly","operator":"Exists"}],"volumes":[{"configMap":{"items":[{"key":"Corefile","path":"Corefile"}],"name":"coredns"},"name":"config-volume"}]}}}}
  creationTimestamp: "2022-10-20T13:39:44Z"
  generation: 3
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
  name: coredns
  namespace: kube-system
  resourceVersion: "17213"
  uid: c66ad408-f114-4adc-8109-936563d1ca93
spec:
  progressDeadlineSeconds: 600
  replicas: 2   # 改成2
 ....
```

### 6、coredns插件

```bash
vim coredns.yaml
...
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors    # 错误信息的标准输出
        health {   # 在coredns的http://localhost:8080/health端口提供coredns服务的健康报告
            lameduck 5s
        }
        ready  # 监听8181端口，当coredns的插件已经就绪后访问该接口会返回200 OK
        kubernetes trevorwu.local in-addr.arpa ip6.arpa {  # 配置k8s集群域名，coredns基于k8s service name进行dns查询并返回查询记录给客户端
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153   # coredns的监控数据已prometheus的key-value的格式在http://localhost:9153/metrics URL上呈现
        forward . 114.114.114.114 {   # 配置自己解析不了的域名去哪解析
            max_concurrent 1000
        }
        cache 30    # service缓存，单位秒
        loop    # 防止DNS解析死循环，终止coredns进程（k8s会重启）
        reload   # 检测yaml文件是否更改，如果更改，两分钟后会自动加载
        loadbalance # 轮训解析DNS记录
    }
    baidu.com {         # 额外的域名解析
       forward . 172.16.16.1:53
    }
```

## 十、Dashboard

### 1、官方

> https://github.com/kubernetes/dashboard/releases

```bash
wget http://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
mv recommended.yaml dashboard-v2.7.0.yaml
```

#### 1.containerd拉取镜像

> 后面可以用docker制作镜像，上传镜像。上传到harbor然后k8s去拉取镜像

```bash
docker pull kubernetesui/dashboard:v2.7.0
docker pull kubernetesui/metrics-scraper:v1.0.8
```

#### 2.上传到harbor

> harbor需要创建镜像仓库，这里不细讲

```bash
docker tag kubernetesui/dashboard:v2.7.0 harbor.trevorwu.com/k8s.1.25.1/k8s-dashboard:v2.7.0
docker tag kubernetesui/metrics-scraper:v1.0.8 harbor.trevorwu.com/k8s.1.25.1/k8s-dashboard-metrics-scraper:v2.7.0
docker push harbor.trevorwu.com/k8s.1.25.1/k8s-dashboard:v2.7.0
docker push harbor.trevorwu.com/k8s.1.25.1/k8s-dashboard-metrics-scraper:v2.7.0
```

#### 3.查看镜像仓库是否有镜像

![3.harbor-dashboard镜像](3.harbor-dashboard镜像.png)

#### 4.修改yaml文件里面的镜像源

```bash
sed -i 's#kubernetesui/dashboard:v2.7.0#harbor.trevorwu.com/k8s.1.25.1/k8s-dashboard:v2.7.0#' dashboard-v2.7.0.yaml
sed -i 's#kubernetesui/metrics-scraper:v1.0.8#harbor.trevorwu.com/k8s.1.25.1/k8s-dashboard-metrics-scraper:v2.7.0#' dashboard-v2.7.0.yaml
```

#### 5.创建应用

```bash
kubectl apply -f dashboard-v2.7.0.yaml
```

#### 6.发现没有暴露端口

```bash
root@k8s-master1:~/k8s-install.yaml.d# kubectl get pod -n  kubernetes-dashboard -o wide
NAME                                       READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
dashboard-metrics-scraper-884c978b-2p5v8   1/1     Running   0          5m44s   10.200.36.72   172.31.7.111   <none>           <none>
kubernetes-dashboard-99cd8855c-599x8       1/1     Running   0          5m44s   10.200.36.71   172.31.7.111   <none>           <none>
root@k8s-master1:~/k8s-install.yaml.d# kubectl get svc -n  kubernetes-dashboard -o wide
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
dashboard-metrics-scraper   ClusterIP   10.100.54.231   <none>        8000/TCP   18m   k8s-app=dashboard-metrics-scraper
kubernetes-dashboard        ClusterIP   10.100.15.137   <none>        443/TCP    18m   k8s-app=kubernetes-dashboard
```

#### 7.修改配置暴露端口

> vim dashboard-v2.7.0.yaml

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30004
  selector:
    k8s-app: kubernetes-dashboard
```

#### 8.验证

```bash
root@k8s-master1:~/k8s-install.yaml.d# kubectl get svc -n  kubernetes-dashboard -o wide
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE   SELECTOR
dashboard-metrics-scraper   ClusterIP   10.100.54.231   <none>        8000/TCP        22m   k8s-app=dashboard-metrics-scraper
kubernetes-dashboard        NodePort    10.100.15.137   <none>        443:30004/TCP   22m   k8s-app=kubernetes-dashboard
```

#### 9.创建管理员账号

> vim admin-user.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

```bash
kubectl apply -f admin-user.yaml
```

#### 10.获取创建账号的token

```bash
root@k8s-master1:~/k8s-install.yaml.d# kubectl -n kubernetes-dashboard create token admin-user
eyJhbGciOiJSUzI1NiIsImtpZCI6InJ3ZTNmYkxaZnRMd2pVbEN6dXhmVzZibGRJWklLamluV3dTTEc4Y0VtY1EifQ.eyJhdWQiOlsiYXBpIiwiaXN0aW8tY2EiXSwiZXhwIjoxNjY2MzY1MDEzLCJpYXQiOjE2NjYzNjE0MTMsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2YyIsImt1YmVybmV0ZXMuaW8iOnsibmFtZXNwYWNlIjoia3ViZXJuZXRlcy1kYXNoYm9hcmQiLCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiYWRtaW4tdXNlciIsInVpZCI6IjdmZGVhNDNkLTJkMmYtNGU1NC1hNzM1LTAwYjQwNjg0NWE0NyJ9fSwibmJmIjoxNjY2MzYxNDEzLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.jlBV1yHODloLFGGn_2uLxFPT7P0BCbCWjhzrdDvWYIcPfT3csY3DlNEDICugVGi622eNdYswunMOW9HxQCq1-3l0Hp2EJECGJg1m-ud5tJ93PH4ZafVVM1onk1ApcCsicXIWWpTo27IRSHDV2qiGq6EL6eB-TrF0X8-nX8vPrRoKRMg6k_N_ZogD1OFBTmavljGj4m7u9xI7bDKW3zKpd6R4Os7NRc48ALHSUgyISLLqfAQJEe1jkmD1yDWQq95ILN5SDyquuA3JfrhASpDlvd5vI_X1L3KTkUKaI30HLRz01iGJ1pJorQ0sK7KUiSO4iUDFS90Lm5qbnhImutPynQ
```

#### 11.访问并登陆

> https://172.31.7.101:30004/#/login

### 2、rancher

> https://rancher.com/quick-start/

```bash
root@k8s-master1:~# sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
root@k8s-master1:~# docker logs adafaf8a2b8 2>&1 | grep "Bootstrap Password:"
2021/09/10 11:45:46 [INFO] Bootstrap Password: jrxfp749g55h2gnw8rakdsjdskflqsb5m2r
```

### 3、kuboard

>https://kuboard.cn/support/#kuboard-%E4%BB%8B%E7%BB%8D #kuboard-介绍
>https://kuboard.cn/install/v3/install.html#%E5%85%BC%E5%AE%B9%E6%80%A7 #kuboard与kubernetes兼容性
>https://kuboard.cn/install/v3/install-built-in.html#%E9%83%A8%E7%BD%B2%E8%AE%A1%E5%88%92 #安装教程

```bash
root@k8s-master1:~# sudo docker run -d \
--restart=unless-stopped \
--name=kuboard \
-p 80:80/tcp \
-p 10081:10081/tcp \
-e KUBOARD_ENDPOINT="http://172.31.7.101:80" \
-e KUBOARD_AGENT_SERVER_TCP_PORT="10081" \
-v /root/kuboard-data:/data \
swr.cn-east-2.myhuaweicloud.com/kuboard/kuboard:v3
在浏览器输入 http://your-host-ip:80 即可访问 Kuboard v3.x 的界面，登录方式：
用户名： admin
密 码： Kuboard123
```

### 4、kubesphere

> 1.23版本之前可以

> 推荐

#### 1.准备nfs服务器

> 克隆模板机然后添加一块硬盘

```bash
root@nfs-server:~# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0    7:0    0     4K  1 loop /snap/bare/5
loop1    7:1    0    62M  1 loop /snap/core20/1611
loop2    7:2    0  91.7M  1 loop /snap/gtk-common-themes/1535
loop3    7:3    0 346.3M  1 loop /snap/gnome-3-38-2004/115
loop4    7:4    0    47M  1 loop /snap/snapd/16292
loop5    7:5    0  54.2M  1 loop /snap/snap-store/558
sda      8:0    0    60G  0 disk
├─sda1   8:1    0   512M  0 part /boot/efi
├─sda2   8:2    0     1K  0 part
└─sda5   8:5    0  59.5G  0 part /
sdb      8:16   0    60G  0 disk
sr0     11:0    1  1024M  0 rom

```

#### 2.安装NFS软件，即是客户端也是服务器端

```bash
apt-get install xfsprogs nfs-kernel-server -y
```

#### 3.硬盘格式化挂载

##### 1）创建挂载点

```bash
mkdir /kubesphare_data
```

##### 2）格式化硬盘

```bash
mkfs.xfs /dev/sdb
```

##### 3）开机自动挂载

```bash
vim /etc/fstab
# 在文件最后添加此行内容
/dev/sdb                /kubesphare_data               xfs     defaults        0 0
```

##### 4）挂载生效

```bash
mount -a
```

##### 5）验证

```bash
root@nfs-server:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
...
/dev/sdb         60G  461M   60G   1% /kubesphare_data
```

#### 4.nfs配置

>vim /etc/exports

```bash
/kubesphare_data     *(rw,insecure,sync,no_root_squash,no_subtree_check)
```

**生效并加入开机自启**

```bash
systemctl restart nfs-server
systemctl enable nfs-server
```

#### 5.验证

##### 1）本地

```bash
root@nfs-server:~# showmount -e
Export list for nfs-server:
/kubesphare_data *
```

##### 2）k8s-master

```bash
root@k8s-master1:~/k8s-install.yaml.d# showmount -e 172.31.7.10
Export list for 172.31.7.10:
/kubesphare_data *
```

##### 3）k8s-node

```bash
root@k8s-node1:/etc/containerd# showmount -e 172.31.7.10
Export list for 172.31.7.10:
/kubesphare_data *
```

#### 6.部署动态存储供给

##### 1）在k8s master节点获取NFS后端存储动态供给配置资源清单文件

```bash
for file in class.yaml deployment.yaml rbac.yaml; 
do 
    wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/$file; 
done
```

##### 2）修改

> class.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
mountOptions:
  - vers=4.1
  - noresvport
  - noatime
parameters:
  mountOptions: "vers=4.1,noresvport,noatime"
  archiveOnDelete: "false" #archiveOnDelete定义为false时，删除NFS Server中对应的目录，为true则保留；
```

>deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: dyrnq/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 172.31.7.10
            - name: NFS_PATH
              value: /kubesphare_data
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.31.7.10
            path: /kubesphare_data
```

> rbac.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

##### 3）应用

```bash
kubectl apply -f rbac.yaml
kubectl apply -f class.yaml
kubectl apply -f deployment.yaml
```

#### 7.设置成默认存储类

```bash
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### 8.查看

```bash
root@k8s-master1:~/k8s-install.yaml.d/nfs-client# kubectl get sc
NAME                   PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   fuseim.pri/ifs   Delete          Immediate           false                  6m39s
```

#### 9.测试用例验证动态供给是否可用

> nginx.yaml

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs-client"
      resources:
        requests:
          storage: 1Gi
```

```bash
kubectl apply -f nginx.yaml
root@k8s-master1:~/k8s-install.yaml.d/nfs-client# kubectl get pod
NAME                                     READY   STATUS    RESTARTS       AGE
net-test3                                1/1     Running   2 (5h2m ago)   2d1h
net-test4                                1/1     Running   2 (5h2m ago)   2d1h
nfs-client-provisioner-57cf4c85b-99pbf   1/1     Running   0              3m16s
web-0                                    1/1     Running   0              85m
web-1                                    1/1     Running   0              2m10s
root@k8s-master1:~/k8s-install.yaml.d/nfs-client# kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    pvc-d91f82f3-742a-401d-aca5-1e09b419d4bb   1Gi        RWO            nfs-client     86m
www-web-1   Bound    pvc-06c80434-4870-4425-bbe6-eddb64b7378f   1Gi        RWO            nfs-client     2m49s
```





#### 11.Kubesphere部署

![image-20220809220111447](image-20220809220111447.png)

##### 1）下载yaml

~~~bash
wget https://github.com/kubesphere/ks-installer/releases/download/v3.3.0/kubesphere-installer.yaml
wget https://github.com/kubesphere/ks-installer/releases/download/v3.3.0/cluster-configuration.yaml
~~~

##### 2）部署

~~~bash
root@k8s-master1:~/k8s-install.yaml.d/kubesphere# kubectl apply -f kubesphere-installer.yaml
customresourcedefinition.apiextensions.k8s.io/clusterconfigurations.installer.kubesphere.io created
namespace/kubesphere-system created
serviceaccount/ks-installer created
clusterrole.rbac.authorization.k8s.io/ks-installer created
clusterrolebinding.rbac.authorization.k8s.io/ks-installer created
deployment.apps/ks-installer created
root@k8s-master1:~/k8s-install.yaml.d/kubesphere# kubectl apply -f cluster-configuration.yaml
clusterconfiguration.installer.kubesphere.io/ks-installer created
~~~

##### 3）查看密码

~~~powershell
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
~~~



~~~powershell
**************************************************
Waiting for all tasks to be completed ...
task network status is successful  (1/4)
task openpitrix status is successful  (2/4)
task multicluster status is successful  (3/4)
task monitoring status is successful  (4/4)
**************************************************
Collecting installation results ...
#####################################################
###              Welcome to KubeSphere!           ###
#####################################################

Console: http://192.168.10.141:30880
Account: admin
Password: P@88w0rd

NOTES：
  1. After you log into the console, please check the
     monitoring status of service components in
     "Cluster Management". If any service is not
     ready, please wait patiently until all components
     are up and running.
  2. Please change the default password after login.

#####################################################
~~~





![image-20220809230741623](image-20220809230741623.png)



![image-20220809230815158](image-20220809230815158.png)





![image-20220809230848565](image-20220809230848565.png)





![image-20220809230905838](image-20220809230905838.png)



![image-20220809230934784](image-20220809230934784.png)



#### 12.Kubesphere开启devops功能

![image-20220809231058295](image-20220809231058295.png)



![image-20220809231146157](image-20220809231146157.png)



![image-20220809231209668](image-20220809231209668.png)



![image-20220809231245060](image-20220809231245060.png)



![image-20220809232511974](image-20220809232511974.png)

## 十一、calico手动安装

>kubeasz中calico版本过低与系统内核不匹配导致服务启动失败

### 1、获取yaml

```bash
https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
```

### 2、修改pod网段

```bash
            - name: CALICO_IPV4POOL_CIDR
              value: "10.201.0.0/16"
```

![image-20240821170552837](./image-20240821170552837.png)

### 3、应用

```bash
kubectl delete -f calico.yaml
kubectl delete -f apply.yaml
```

