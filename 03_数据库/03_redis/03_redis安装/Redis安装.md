# Redis安装

## 一、容器安装

```bash
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: redis-deployment
spec:
  selector:
    matchLabels:
      app: redis
      deploy: redis
  template:
    metadata:
      labels:
        app: redis
        deploy: redis
    spec:
      containers:
        - name: redis
          image: redis:6.0.9
---
kind: Service
apiVersion: v1
metadata:
  name: redis-deployment-svc
spec:
  ports:
    - port: 6379
      targetPort: 6379
      name: redis
      protocol: TCP
  selector:
    app: redis
    deploy: redis
  type: NodePort
```



## 二、物理编译安装

### 1、下载解压

```bash
[root@xiaowu ~]# wget https://download.redis.io/releases/redis-6.0.9.tar.gz

[root@tomcat redis]# tar -xf redis-6.0.9.tar.gz
```



### 2、安装依赖

> Centos-release-scl软件集的使用

```bash
	作用：CentOS7 gcc版本为4.8.5，Red Hat为了软件的稳定和版本支持，yum上版本也是4.8.5,所以无法使用yum的方式进行gcc的软件升级，所以使用scl。

	scl：(Software Collections)软件集，是为了给RHEL/CentOS用户提供一种以方便，安全地安装、使用应用程序和运行时环境的多个版本方式，同时避免吧系统搞乱。
```

```bash
[root@xiaowu ~/redis-6.0.9]# yum -y install centos-release-scl

[root@xiaowu ~/redis-6.0.9]# yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils 

[root@xiaowu ~/redis-6.0.9]# scl enable devtoolset-9 bash
```



### 3、编译编译安装

```bash
[root@xiaowu ~/redis-6.0.9]# make -j		#-j 多核编译
[root@xiaowu ~/redis-6.0.9]# make PREFIX=/usr/local/redis install
```



### 4、添加环境变量

```bash
[root@xiaowu /]# vim /etc/profile.d/redis.sh
export PATH=/usr/local/redis/bin:$PATH
```



### 5、修改配置文件

#### 1.添加配置文件

```bash
[root@xiaowu ~]# mkdir /usr/local/redis/conf/
[root@xiaowu ~]# cp /root/redis-6.0.9/redis.conf /usr/local/redis/conf/
```



#### 2.修改配置文件

```bash
[root@xiaowu ~]# cd /usr/local/redis/conf
[root@xiaowu /usr/local/redis/conf]# cp redis.conf redis.conf.bak
```

**修改配置文件**

```bash
[root@xiaowu /usr/local/redis/conf]# vim redis.conf
daemonize yes			#以守护进程的方式运行
bind 127.0.0.1 172.16.1.130    #添加内网监听地址
requirepass 123			#设置redis的登录密码
```



### 6、加入systemd管理

```bash
[root@xiaowu /usr/local/redis/conf]# vim /usr/lib/systemd/system/redis.service

[Unit]
Description=Redis
After=network.target

[Service]
Type=forking
PIDFile=/var/run/redis_6379.pid
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/conf/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target


# 重载
[root@xiaowu /usr/local/redis/conf]# systemctl daemon-reload
```



### 7、启动并加入开机自启

```bash
[root@xiaowu /usr/local/redis/conf]# systemctl start redis.service 
[root@xiaowu /usr/local/redis/conf]# systemctl enable redis.service 
```



### 8、验证设置的ip及密码

方式一：

```bash
[root@xiaowu ~]# redis-cli -h 172.16.1.130 -a 123
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
172.16.1.130:6379> 
```



方式二：保护密码更安全

```bash
[root@xiaowu ~]# redis-cli -h 172.16.1.130
172.16.1.130:6379> AUTH 123
OK
```



方式三：本机登录

```bash
[root@xiaowu ~]# redis-cli
127.0.0.1:6379> AUTH 123
OK
```

