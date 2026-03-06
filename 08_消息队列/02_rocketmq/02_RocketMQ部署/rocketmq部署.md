# 部署rocketmq

## 一、环境准备

> 官网：https://rocketmq.apache.org/

```bash
#rocketmq依赖环境：
1、JDK1.8+
2、Maven 3.2.X+
3、Git

PS:rocketmq默认jdk的位置为/usr/local/jdk，若位置不在此处，需要手动修改配置文件
```

### 1、安装JDK1.8和git

```bash
apt-get install openjdk-8-jdk git -y
```

### 2、检查JDK和git是否安装成功

```bash
java -version
git --version
```

### 3、安装Maven

> https://maven.apache.org/download.cgi

#### 1.下载并解压Maven二进制包

```bash
cd /opt
wget https://dlcdn.apache.org/maven/maven-3/3.8.5/binaries/apache-maven-3.8.5-bin.tar.gz
tar xf apache-maven-3.8.5-bin.tar.gz
```

#### 2.移动到指定目录

```bash
mv apache-maven-3.8.5 /usr/local/
```

#### 3.设置环境变量

```bash
vim /etc/profile.d/maven.sh
export M2_HOME=/usr/local/apache-maven-3.8.5
export PATH=${M2_HOME}/bin:$PATH

source /etc/profile
```

#### 4.测试

```bash
mvn -v
```



## 二、部署rocketmq

### 1、创建目录

```bash
mkdir /rocketmq
cd /rocketmq
```

### 2、下载rocketmq

```bash

wget https://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.9.3/rocketmq-all-4.9.3-bin-release.zip

#官方下载链接
wget https://archive.apache.org/dist/rocketmq/4.9.3/rocketmq-all-4.9.3-bin-release.zip
```

### 3、解压包

```bash
unzip rocketmq-all-4.9.3-bin-release.zip
#制作软连接
ln -s rocketmq-4.9.3 rocketmq
```



### 3、根据实际，修改jvm参数

#### 1.修改runserver参数

```bash
cd /rocketmq/rocketmq/bin
cp runserver.sh runserver.sh.bak
vim runserver.sh
... ...
#可以在此处修改jdk位置
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr
#[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}
... ...
#在此处修改内存堆栈大小
      #JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
      JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn1g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
... ...
```



#### 2.修改runbroker参数

```bash
cd /rocketmq/rocketmq/bin
cp runbroker.sh runbroker.sh.bak
vim runbroker.sh
... ...
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
#[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"
... ...


... ...
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn1g"
#JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g"
... ...
```

## 三、修改配置文件，配置集群

> 本次实验配置1m-0s集群,可以根据自己需要扩展集群

### 1、namesrv.properties文件设置

```bash
cd /rocketmq/rocketmq/conf
vim namesrv.properties

# RocketMQ 主目录 
rocketmqHome=/rocketmq/rocketmq

# kv 配置文件路径，包含顺序消息主题的配置信息
kvConfigPath=/root/namesrv/kvConfig.json

# 环境
productEnvName=center

# 是否开启集群测试
clusterTest=false

# 是否支持顺序消息
orderMessageEnable=false

# 服务端监听端口
listenPort=9876

# Netty 业务线程池线程个数
serverWorkerThreads=8

# Netty public 任务线程池线程个数，Netty 网络设计，根据业务类型会创建不同的线程池，比如处理发送消息、消息消费、心跳检测等。如果该业务类型(RequestCode)未注册线程池，则由 public 线程池执行
serverCallbackExecutorThreads=0

# IO 线程池线程个数，主要是 NameServer、Broker 端解析请求、返回响应的线程个数，这类线程池主要是处理网络请求的，解析请求包，然后转发到各个业务线程池完成具体的业务操作，然后将结果再返回调用方
serverSelectorThreads=3

# send oneway 消息请求并发度
serverOnewaySemaphoreValue=256

# 异步消息发送最大并发度
serverAsyncSemaphoreValue=64

# 网络连接最大空闲时间，单位秒，如果连接空闲时间超过该参数设置的值，连接将被关闭
serverChannelMaxIdleTimeSeconds=120

# 网络 socket 发送缓存区大小，单位 B，即默认为 64KB
serverSocketSndBufSize=65535

# 网络 socket 接收缓存区大小，单位 B，即默认为 64KB
serverSocketRcvBufSize=65535

# ByteBuffer 是否开启缓存，建议开启
serverPooledByteBufAllocatorEnable=true

# 是否启用 Epoll IO 模型
useEpollNativeSelector=true
```



### 2、broker.properties文件设置

```bash
cd /rocketmq/rocketmq/conf
vim broker.properties

# 集群名，不同broker节点集群名是一样的
brokerClusterName=brokercluster

# broker名字，不同broker节点brokerName是唯一的
brokerName=broker-a

# 0表示Master，大于0表示 Slave
brokerId=0

#nameServer地址，多个用英文分号;分割
namesrvAddr=127.0.0.1:9876

# 删除文件时间点，默认凌晨 4点
deleteWhen=04

# 文件保留时间，默认 48 小时
fileReservedTime=48

# 当前节点角色，ASYNC_MASTER=异步复制Master模式，SYNC_MASTER=同步双写Master模式，复制节点设置为SLAVE
brokerRole=ASYNC_MASTER

# 刷盘模式，ASYNC_FLUSH=异步刷盘，SYNC_FLUSH=同步刷盘
flushDiskType=ASYNC_FLUSH

# 数据存储路径(同一节点上master与slave需在不同目录)
storePathRootDir=/rocketmq/rocketmq/store

# commitLog 存储路径(同一节点上master与slave需在不同目录)
storePathCommitLog=/rocketmq/rocketmq/store/commitlog

# 消费队列存储路径存储路径(同一节点上master与slave需在不同目录)
storePathConsumeQueue=/rocketmq/rocketmq/store/consumequeue

# 消息索引存储路径(同一节点上master与slave需在不同目录)
storePathIndex=/rocketmq/rocketmq/store/index

# checkpoint 文件存储路径(同一节点上master与slave需在不同目录)
storeCheckpoint=/rocketmq/rocketmq/store/checkpoint

# abort 文件存储路径(同一节点上master与slave需在不同目录)
abortFile=/rocketmq/rocketmq/store/abort

# 是否允许 Broker 自动创建Topic，建议关闭
autoCreateTopicEnable=false

# 是否允许 Broker 自动创建订阅组，建议关闭
autoCreateSubscriptionGroup=false

# 设置brokerIP，如果不设置当服务器有很多网卡，默认会读取第一个网卡的IP地址（如安装docker后），会导致客户端无法连接
brokerIP1=172.16.1.12

# broker对外服务的监听端口，默认10911（同一节点启动多个服务，需要修改默认端口）
listenPort=10911
```

### 3、根据配置创建环境

```bash
mkdir /rocketmq/rocketmq/store
mkdir /rocketmq/rocketmq/store/commitlog
mkdir /rocketmq/rocketmq/store/consumequeue
mkdir /rocketmq/rocketmq/store/index


checkpoint abort不需要创建
```

## 四、启动集群

```bash
nohup /rocketmq/rocketmq/bin/mqnamesrv -c /rocketmq/rocketmq/conf/namesrv.properties >/dev/null 2>& 1 &

nohup /rocketmq/rocketmq/bin/mqbroker -c /rocketmq/rocketmq/conf/broker.properties >/dev/null 2>& 1 &
```

## 五、安装控制台

### 1、下载源码

```bash
cd /rocketmq
git clone https://github.com/apache/rocketmq-externals

#更新下载链接
wget https://github.com/apache/rocketmq-externals/archive/refs/tags/rocketmq-console-1.0.0.tar.gz

tar xf rocketmq-console-1.0.0.tar.gz
```



### 5.2、修改配置文件

```bash
vim cd /rocketmq/rocketmq-externals-rocketmq-console-1.0.0/rocketmq-console/src/main/resources

vim application.properties

# 服务端口号
server.port=8081
# NameServer服务地址，多台服务器用英文分号分割
rocketmq.config.namesrvAddr=127.0.0.1:9876
```



### 5.3、编译

```bash
cd /rocketmq/rocketmq-externals-rocketmq-console-1.0.0/rocketmq-console#
mvn clean package -Dmaven.test.skip=true
```



### 5,4、运行

```bash
cd /rocketmq/rocketmq-externals-rocketmq-console-1.0.0/rocketmq-console/target
nohup java -jar rocketmq-console-ng-1.0.0.jar >/dev/null 2>& 1 &
```



### 5.7、打开浏览器访问

```bash
http://172.16.1.12:8081
```











