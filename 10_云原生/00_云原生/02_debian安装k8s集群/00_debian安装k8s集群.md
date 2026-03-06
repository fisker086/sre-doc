# Debian12安装k8s集群

## 一、debian系统安装成模板机-略

### 1、确认网卡地址

```bash
root@debian-template:~# cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto ens18
iface ens18 inet static
    address 192.168.6.100/24
    gateway 192.168.6.1
```

## 二、debian系统优化

### 1、安装ssh vim

#### 1.安装

```bash
apt install -y vim ssh
```

#### 2.允许root账户登录

```bash
cat /etc/ssh/sshd_config
#PermitRootLogin prohibit-password
PermitRootLogin yes     #添加这句话


# 重启ssh服务

```

### 2、配置host解析

```bash
vim /etc/hosts
192.168.6.1 immortalwrt
192.168.6.20 tyan-s8030
192.168.6.21 pve

192.168.6.100 debian-template

192.168.6.101 k8s-master-01
192.168.6.102 k8s-master-02
192.168.6.103 k8s-master-03

192.168.6.104 k8s-node-01
192.168.6.105 k8s-node-02
192.168.6.106 k8s-node-03
192.168.6.107 k8s-node-04

192.168.6.190 pve-win
```

### 3、配置apt源及安装常用组件

#### 1.配源

>https://mirrors.tuna.tsinghua.edu.cn/help/debian/

```bash
vim /etc/apt/sources.list

# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware

# 以下安全更新软件源包含了官方源与镜像站配置，如有需要可自行修改注释切换
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security bookworm-security main contrib non-free non-free-firmware
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian-security bookworm-security main contrib non-free non-free-firmware
```

#### 2.更新存储库索引

```bash
apt update
```

#### 3.升级可升级软件包

```bash
apt upgrade
```

#### 4.安装常用组件

```bash
apt-get install -y net-tools vim htop iftop bash-completion wget rsync tcpdump telnet lrzsz tree iotop unzip zip apt-transport-https ca-certificates curl software-properties-common gnupg-agent
```

### 4、配置时间同步和时区

#### 1.配置时间服务器

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
systemctl restart systemd-timesyncd
```

#### 2.配置时区

```bash
# timedatectl
               Local time: Wed 2025-07-30 19:56:06 CST
           Universal time: Wed 2025-07-30 11:56:06 UTC
                 RTC time: Wed 2025-07-30 11:56:06
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no

# timedatectl set-timezone Asia/Shanghai
```

### 5、创建秘钥

```bash
rm -rf /root/.ssh

ssh-keygen -t ed25519

cd /root/.ssh/

cp id_ed25519.pub authorized_keys
```

## 三、克隆虚拟机-服务器准备

### 1、克隆虚拟机

>调整虚拟机的的cpu、内存、磁盘等资源规格

### 2、硬盘直通

```bash
qm set 104 -sata0 /dev/disk/by-id/scsi-35000c50083f9cb77
qm set 105 -sata0 /dev/disk/by-id/scsi-35000c50095ffb553
qm set 106 -sata0 /dev/disk/by-id/scsi-35000c50096073373
```

### 3、pve显卡直通



### 4、修改主机名和IP地址

```bash
hostnamectl set-hostname k8s-master-01 && sed -i s#192.168.6.100#192.168.6.101# /etc/network/interfaces && systemctl restart networking.service
hostnamectl set-hostname k8s-master-02 && sed -i s#192.168.6.101#192.168.6.102# /etc/network/interfaces && systemctl restart networking.service
hostnamectl set-hostname k8s-master-03 && sed -i s#192.168.6.101#192.168.6.103# /etc/network/interfaces && systemctl restart networking.service

hostnamectl set-hostname k8s-node-01 && sed -i s#192.168.6.100#192.168.6.104# /etc/network/interfaces && systemctl restart networking.service
hostnamectl set-hostname k8s-node-02 && sed -i s#192.168.6.104#192.168.6.105# /etc/network/interfaces && systemctl restart networking.service
hostnamectl set-hostname k8s-node-03 && sed -i s#192.168.6.104#192.168.6.106# /etc/network/interfaces && systemctl restart networking.service
```

## 四、kubeasz安装k8s

详细过程请看

>https://md2web.gmbaifa.online/#sort=./05_%E5%AE%B9%E5%99%A8%E7%BC%96%E6%8E%92%E5%B7%A5%E5%85%B7&doc=02_Kubernets/04_ubuntu%E7%94%A8kubeasz%E5%AE%89%E8%A3%85k8s1.27/ubuntu%E7%94%A8kubeasz%E5%AE%89%E8%A3%85k8s1.27.md

















