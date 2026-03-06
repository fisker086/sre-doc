# prometheus配置文件详解

## 一、基础配置

>prometheus.yml为主配置文件，该文件大致分为了global全局配置、alerting告警配置、rules_file、scrape_configs被监控端配置。下面是一个基础配置文件说明

```bash
# 全局配置
global:
  scrape_interval:     15s # 数据收集频率
  evaluation_interval: 15s # 多久评估一次规则
  scrape_timeout: 10s  # 收集数据的超时时间

#####Alertmanager配置模块

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
      - targets:
        - ['127.0.0.1:9093'] #配置告警信息接收端口

# ###规则文件，支持通配符
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
  # - "rules/*.rules"
  # - "*.rules"


# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'    # 被监控资源组的名称

    # metrics_path defaults to '/metrics' #该行可不写，获取数据URI，默认为/metrics
    # scheme defaults to 'http'.          # #默认http方式采集

    static_configs:           ##节点地址与获取Metrics数据端口，多个地址用逗号分隔，也可以写多行
    - targets: ['localhost:9090']
   #- targets: ['localhost:9090','192.168.1.100:9100']
     #- 192.168.1.101:9100
     #- 192.168.1.102:9100
     #- 192.168.1.103:9100
```



> 告警规则可以去这里看看：https://awesome-prometheus-alerts.grep.to/



## 二、标签配置

> Prometheus通过标签可以实现查询过滤，并且还支持重新标签实现动态生成标签、过滤、删除无用标签等灵活配置。在采集数据之前可以使用relabel_configs进行重新标记，存储数据之前可以使用metric_relabel_configs重新标记。两种重新打标签的方式都支持以下动作：

```bash
replace：默认动作，将匹配到的标签内容做替换 
keep：通过正则匹配，仅保留正则匹配到的标签 
drop：通过正则匹配，删除正则匹配到的标签 
labeldrop：删除指定标签，比如一些默认标签并不需要，可以用该动作删除
labelkeep：仅保留指定标签 
```

**配置文件说明**

```bash
global:  #全局配置，这里的配置项可以单独配置在某个job中
  scrape_interval: 15s  #采集数据间隔，默认15秒
  evaluation_interval: 15s  #告警规则监测频率，比如已经设置了内存使用大于70%就告警的规则，这里就会每15秒执行一次告警规则
  scrape_timeout: 10s   #采集超时时间

scrape_configs:
  - job_name: 'prometheus-server'  #定义一个监控组名称
    metrics_path defaults to '/metrics'  #获取数据URI默认为/metrics
    scheme defaults to 'http'  #默认http方式采集
    static_configs:
    - targets: ['localhost:9090','192.168.1.100:9100']  #节点地址与获取Metrics数据端口，多个地址用逗号分隔，也可以写多行。

  - job_name: 'web_node'  #定义另一个监控组
    metrics_path defaults to '/metrics'  #获取数据URI默认为/metrics
    scheme defaults to 'http'  #默认http方式采集
    static_configs:
    - targets: ['10.160.2.107:9100','192.168.1.100:9100']  #组内多个被监控主机
      labels:  #自定义标签，通过标签可以进行查询过滤
        server: nginx  #将上面2个主机打上server标签，值为nginx

  - job_name: 'mysql_node'
    static_configs:
    - targets: ['10.160.2.110:9100','192.168.1.111:9100']
    metric_relable_configs:   #声明要重命名标签
    - action: replace  #指定动作，replace代表替换标签，也是默认动作
      source_labels: ['job']  #指定需要被action所操作的原标签
      regex: (.*)  #原标签里的匹配条件，符合条件的原标签才会被匹配，支持正则
      replacement: $1  #原标签需要被替换的部分，$1代表regex正则的第一个分组
      target_label: idc  #将$1内容赋值给idc标签

    - action: drop  #正则删除标签示例
      regex: "192.168.100.*"  #正则匹配标签值
      source_labels: ["__address__"]  #需要进行正则匹配的原标签

    - action: labeldrop  #直接删除标签示例
      regex: "job"  #直接写标签名即可
```



## 三、检查工具

> 在启动Prometheus之前可以使用protool工具对配置文件进行检查

```bash
protool check config prometheus.yml
```



## 四、启动命令详解

### 1、启动命令

```bash
prometheus --config.file="/usr/local/prometheus-2.16.0.linux-amd64/prometheus.yml" --web.listen-address="0.0.0.0:9090" --storage.tsdb.path="/data/prometheus" --storage.tsdb.retention.time=15d --web.enable-lifecycle &
```



### 2、常用参数详解

```bash
--config.file="/usr/local/prometheus/prometheus.yml"  #指定配置文件路径

--web.listen-address="0.0.0.0:9090"  #指定服务端口

--storage.tsdb.path="/data/prometheus"  #指定数据存储路径

--storage.tsdb.retention.time=15d  #数据保留时间

--collector.systemd #开启systemd的服务状态监控，开启后在WEB上可以看到多出相关监控项

--collector.systemd.unit-whitelist=(sshd|nginx).service  #对systemd具体要监控的服务名

--web.enable-lifecycle  #开启热加载配置
```

