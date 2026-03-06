# Keepalived高可用

## 一、高可用介绍

### 1.什么是高可用

```bash
一般是指2台机器启动着完全相同的业务系统，当有一台机器down机了，另外一台服务器就能快速的接管，对于访问的用户是无感知的。
```

### 2.常用的工具

```bash
1.硬件通常使用 F5
2.软件通常使用 keepalived
```

### 3.keepalived是如何实现高可用的？

#### 1）涉及名词

```bash
keepalived软件是基于VRRP协议实现的，VRRP虚拟路由冗余协议，主要用于解决单点故障问题

ARP广播
VRRP协议
vip负责IP漂移
vmac负责通知ARP广播修改mac地址
```

#### 2）例子

```bash
比如公司的网络是通过网关进行上网的，那么如果该路由器故障了，网关无法转发报文了，此时所有人都无法上网了，怎么办？

通常做法是给路由器增加一台备节点，但是问题是，如果我们的主网关master故障了，用户是需要手动指向backup的，如果用户过多修改起来会非常麻烦。

问题一：假设用户将指向都修改为backup路由器，那么master路由器修好了怎么办？
问题二：假设Master网关故障，我们将backup网关配置为master网关的ip是否可以？

其实是不行的，因为PC第一次通过ARP广播寻找到Master网关的MAC地址与IP地址后，会将信息写到ARP的缓存表中，那么PC之后连接都是通过那个缓存表的信息去连接，然后进行数据包的转发，即使我们修改了IP但是Mac地址是唯一的，pc的数据包依然会发送给master。（除非是PC的ARP缓存表过期，再次发起ARP广播的时候才能获取新的backup对应的Mac地址与IP地址）

如何才能做到出现故障自动转移，此时VRRP就出现了，我们的VRRP其实是通过软件或者硬件的形式在Master和Backup外面增加一个虚拟的MAC地址（VMAC）与虚拟IP地址（VIP），那么在这种情况下，PC请求VIP的时候，无论是Master处理还是Backup处理，PC仅会在ARP缓存表中记录VMAC与VIP的信息。
```



### 4.高可用keepalived核心概念

```bash
1.如何确定谁是主节点谁是备节点（选举投票，优先级）
2.如果Master故障，Backup自动接管，那么Master恢复后会夺权吗（抢占试、非抢占式）
3.如果两台服务器都认为自己是Master会出现什么问题（脑裂）
```



## 二、keepalived搭建

### 1.环境准备

| 主机  | IP          | 身份              |
| ----- | ----------- | ----------------- |
| lb01  | 10.0.0.4    | keepalived master |
| lb02  | 10.0.0.5    | keepalived backup |
| web01 | 172.16.1.7  | web端             |
| web02 | 172.16.1.8  | web端             |
| db01  | 172.16.1.51 | 数据库            |
|       | 10.0.0.3    | VIP               |

### 2.保证七层负载均衡完全一致

```bash
[root@lb01 conf.d]# scp linux.wp.com.conf 172.16.1.5:/etc/nginx/conf.d/
[root@lb01 conf.d]# scp -r /etc/nginx/ssl_key 172.16.1.5:/etc/nginx/
```

### 3.安装keepalived

```bash
[root@lb01 ~]# yum install -y keepalived
[root@lb02 ~]# yum install -y keepalived
```

### 4.配置keepalived

#### 1）查找配置文件

```bash
[root@lb01 ~]# rpm -qc keepalived
/etc/keepalived/keepalived.conf
```

#### 2）配置主节点的配置文件

```bash
[root@lb01 ~]# vim /etc/keepalived/keepalived.conf
global_defs {
    router_id lb01
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 50
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.0.0.3
    }
}


[root@lb01 ~]# vim /etc/keepalived/keepalived.conf
#全局配置
global_defs {
	#身份识别
    router_id lb01
}

#配置VRRP协议
vrrp_instance VI_1 {
	#状态，MASTER和BACKUP
    state MASTER
    #绑定网卡
    interface eth0
    #虚拟路由标示，可以理解为分组
    virtual_router_id 50
    #优先级
    priority 100
    #监测心跳间隔时间
    advert_int 1
    #配置认证
    authentication {
    	#认证类型
        auth_type PASS
        #认证的密码
        auth_pass 1111
    }
    #设置VIP
    virtual_ipaddress {
    	#虚拟的VIP地址
        10.0.0.3
    }
}
```

#### 3）配置备节点

```bash
[root@lb02 ~]# vim /etc/keepalived/keepalived.conf
global_defs {
    router_id lb02
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 50
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.0.0.3
    }
}
```

### 5.启动服务

```bash
[root@lb01 ~]# tail -f /var/log/messages
[root@lb01 ~]# systemctl start keepalived.service

[root@lb02 ~]# tail -f /var/log/messages
[root@lb02 ~]# systemctl start keepalived.service
```

### 6.keepalived开启日志

```bash
#配置keepalived
[root@lb01 ~]# vim /etc/sysconfig/keepalived
KEEPALIVED_OPTIONS="-D -d -S 0"

#配置rsyslog抓取日志
[root@lb01 ~]# vim /etc/rsyslog.conf
local0.*		/var/log/keepalived.log

#重启服务
[root@lb01 ~]# systemctl restart keepalived rsyslog
```



## 三、keepalived的抢占式与非抢占式

### 1.两个节点都启动的情况

```bash
#两个节点都启动时，由于节点1优先级高于节点2，所以只有节点1上有VIP
[root@lb01 ~]# ip addr | grep 10.0.0.3
    inet 10.0.0.3/32 scope global eth0
    
[root@lb02 ~]# ip addr | grep 10.0.0.3
```

### 2.停止主节点

```bash
[root@lb01 ~]# systemctl stop keepalived.service 
[root@lb01 ~]# ip addr | grep 10.0.0.3

#由于节点1keepalived挂掉，节点2会自动接管节点1的工作，即VIP
[root@lb02 ~]# ip addr | grep 10.0.0.3
    inet 10.0.0.3/32 scope global eth0
```

### 3.重新启动主节点

```bash
#启动主节点
[root@lb01 ~]# systemctl start keepalived
[root@lb01 ~]# ip addr | grep 10.0.0.3
    inet 10.0.0.3/32 scope global eth0

#由于节点1优先级高于节点2，所以当节点1恢复时，会将VIP抢占回来
```

### 4.配置非抢占式

#### 1）主节点配置

```bash
[root@lb01 ~]# vim /etc/keepalived/keepalived.conf 
... ...
vrrp_instance VI_1 {
    state BACKUP
    #开启非抢占式
    nopreempt
    priority 100
... ...
}

[root@lb01 ~]# systemctl restart keepalived
```

#### 2）备节点配置

```bash
[root@lb02 ~]# vim /etc/keepalived/keepalived.conf 
... ...
vrrp_instance VI_1 {
    state BACKUP
    nopreempt
    priority 90
... ...
}

[root@lb02 ~]# systemctl restart keepalived.service
```

#### 3）配置

```bash
1.两个节点的state都必须配置为BACKUP
2.两个节点都必须加上配置 nopreempt
3.其中一个节点的优先级必须要高于另外一个节点的优先级。

两台服务器都角色状态启用nopreempt后，必须修改角色状态统一为BACKUP，唯一的区分就是优先级。
```


