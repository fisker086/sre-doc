# promethus安装

## 一、安装

### 1、下载软件包

```bash
[root@promethus ~]# mkdir /prometheus
[root@promethus /opt]# cd /prometheus/
[root@promethus /prometheus]# wget https://github.com/prometheus/prometheus/releases/download/v2.25.0/prometheus-2.25.0.linux-amd64.tar.gz
```



### 2、解压

```bash
[root@promethus /prometheus]# tar xf prometheus-2.25.0.linux-amd64.tar.gz 
[root@promethus /prometheus]# cd prometheus-2.25.0.linux-amd64/
[root@promethus /prometheus/prometheus-2.25.0.linux-amd64]# mv ./* ../
[root@promethus /prometheus]# rm -rf prometheus-2.25.0.linux-amd64*
```



### 3、创建用户并授权

```bash
[root@promethus /prometheus]# useradd -s /sbin/nologin prometheus -M
[root@promethus /prometheus]# chown -R prometheus.prometheus /prometheus
```



### 4、添加环境变量

```bash
[root@promethus /prometheus]# vim /etc/profile.d/prometheus.sh
export PATH=/prometheus:$PATH
```



### 5、配置systemd管理

```bash
[root@promethus ~]# vim /usr/lib/systemd/system/prometheus.service

[Unit]
Description=prometheus server daemon

[Service]
ExecStart=/prometheus/prometheus --config.file=/prometheus/prometheus.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target

# 重载启动列表
[root@promethus ~]# systemctl daemon-reload
```



### 6、启动

```bash
[root@promethus ~]# systemctl start prometheus.service

# 开机自启
[root@promethus ~]# systemctl enable prometheus.service 
```



### 7、访问

```bash
http://192.168.15.120:9090/
```



## 二、安装后文件说明

### 1、目录下文件说明

```bash
[root@promethus /prometheus]# ls 
console_libraries	 	--->控制台函数库
consoles				--->控制台
data					--->数据存放目录

LICENSE					--->许可证
NOTICE					--->通知
prometheus				--->启动脚本
prometheus.yml			--->主配置文件
promtool				--->系统工具
```



### 2、主配置文件说明

```bash
[root@promethus /prometheus]# cat prometheus.yml

global:					--->全局变量
  scrape_interval:     15s # 抓取时间间隔，每隔15秒去抓取一次
  evaluation_interval: 15s # 监控数据评估间隔

scrape_configs:
  - job_name: 'prometheus'           --->定义job名字

    static_configs:
    - targets: ['localhost:9090','web01:9100','10.0.0.8:9100']    --->定义监控节点
```

