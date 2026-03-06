# Rocketmq容器化

## 一、镜像制作

### 1、Dockerfile内容

```bash
FROM centos:7

RUN yum install -y java-1.8.0-openjdk-devel.x86_64 unzip gettext nmap-ncat openssl, which gnupg, telnet \
 && yum clean all -y

# FROM openjdk:8-jdk
# RUN apt-get update && apt-get install -y --no-install-recommends \
#		bash libapr1 unzip telnet wget gnupg ca-certificates \
#	&& rm -rf /var/lib/apt/lists/*

ARG user=rocketmq
ARG group=rocketmq
ARG uid=3000
ARG gid=3000

# RocketMQ is run with user `rocketmq`, uid = 3000
# If you bind mount a volume from the host or a data container,
# ensure you use the same uid
RUN groupadd -g ${gid} ${group} \
    && useradd -u ${uid} -g ${gid} -m -s /bin/bash ${user}

ARG version=4.9.3

# Rocketmq version
ENV ROCKETMQ_VERSION ${version}

# Rocketmq home
ENV ROCKETMQ_HOME  /home/rocketmq/rocketmq-${ROCKETMQ_VERSION}
RUN mkdir ${ROCKETMQ_HOME}

WORKDIR  ${ROCKETMQ_HOME}

RUN set -eux; \
    curl https://archive.apache.org/dist/rocketmq/${ROCKETMQ_VERSION}/rocketmq-all-${ROCKETMQ_VERSION}-bin-release.zip -o rocketmq.zip; \
    unzip rocketmq.zip ; \
	mv rocketmq-${ROCKETMQ_VERSION}/* . ; \
	rmdir rocketmq-${ROCKETMQ_VERSION}  ; \
	rm -rf rocketmq.zip

# add scripts
COPY scripts/ ${ROCKETMQ_HOME}/bin/

# add customized scripts for namesrv
# add customized scripts for broker
RUN mkdir -p /home/rocketmq/logs \
    && mkdir -p /home/rocketmq/store \
    && chown -R ${uid}:${gid} /home/rocketmq \
    && chown -R ${uid}:${gid} ${ROCKETMQ_HOME} \
 && mv -bf ${ROCKETMQ_HOME}/bin/runserver-customize.sh ${ROCKETMQ_HOME}/bin/runserver.sh \
 && chmod a+x ${ROCKETMQ_HOME}/bin/runserver.sh \
 && chmod a+x ${ROCKETMQ_HOME}/bin/mqnamesrv \
 && mkdir -p /etc/rocketmq \
 && mv -bf ${ROCKETMQ_HOME}/bin/runbroker-customize.sh ${ROCKETMQ_HOME}/bin/runbroker.sh \
 && chmod a+x ${ROCKETMQ_HOME}/bin/runbroker.sh \
 && chmod a+x ${ROCKETMQ_HOME}/bin/mqbroker \
 && ln -s ${ROCKETMQ_HOME}/bin/mqadmin /usr/local/bin/mqadmin  \
    && ln -s ${ROCKETMQ_HOME}/bin/runbroker /usr/local/bin/runbroker \
    && ln -s ${ROCKETMQ_HOME}/bin/mqnamesrv /usr/local/bin/mqnamesrv \
    && ln -s ${ROCKETMQ_HOME}/bin/mqbroker /usr/local/bin/mqbroker \
    && ln -s ${ROCKETMQ_HOME}/bin/runbroker.sh /usr/local/bin/runbroker.sh \
    && ln -s ${ROCKETMQ_HOME}/bin/runserver.sh /usr/local/bin/runserver.sh \
    && ln -s ${ROCKETMQ_HOME}/bin/runbroker.sh /usr/local/bin/runbroker-customize.sh \
    && ln -s ${ROCKETMQ_HOME}/bin/runserver.sh /usr/local/bin/runserver-customize.sh
# expose namesrv port
# expose broker ports
EXPOSE 9876 10909 10911 10912

# export Java options
# Add ${JAVA_HOME}/lib/ext as java.ext.dirs
RUN export JAVA_OPT=" -Duser.home=/home/rocketmq" \
 && sed -i 's/${JAVA_HOME}\/jre\/lib\/ext/${JAVA_HOME}\/jre\/lib\/ext:${JAVA_HOME}\/lib\/ext/' ${ROCKETMQ_HOME}/bin/tools.sh

USER ${user}

WORKDIR ${ROCKETMQ_HOME}/bin
```

### 2、相关文件

#### 1.runbroker-customize.sh

```bash
#!/bin/bash

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#===========================================================================================
# Java Environment Setting
#===========================================================================================
error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

find_java_home()
{
    case "`uname`" in
        Darwin)
            JAVA_HOME=$(/usr/libexec/java_home)
        ;;
        *)
            JAVA_HOME=$(dirname $(dirname $(readlink -f $(which javac))))
        ;;
    esac
}

find_java_home

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
calculate_heap_sizes()
{
    case "`uname`" in
        Linux)
            system_memory_in_mb=`free -m| sed -n '2p' | awk '{print $2}'`
            system_cpu_cores=`egrep -c 'processor([[:space:]]+):.*' /proc/cpuinfo`
        ;;
        FreeBSD)
            system_memory_in_bytes=`sysctl hw.physmem | awk '{print $2}'`
            system_memory_in_mb=`expr $system_memory_in_bytes / 1024 / 1024`
            system_cpu_cores=`sysctl hw.ncpu | awk '{print $2}'`
        ;;
        SunOS)
            system_memory_in_mb=`prtconf | awk '/Memory size:/ {print $3}'`
            system_cpu_cores=`psrinfo | wc -l`
        ;;
        Darwin)
            system_memory_in_bytes=`sysctl hw.memsize | awk '{print $2}'`
            system_memory_in_mb=`expr $system_memory_in_bytes / 1024 / 1024`
            system_cpu_cores=`sysctl hw.ncpu | awk '{print $2}'`
        ;;
        *)
            # assume reasonable defaults for e.g. a modern desktop or
            # cheap server
            system_memory_in_mb="2048"
            system_cpu_cores="2"
        ;;
    esac

    # some systems like the raspberry pi don't report cores, use at least 1
    if [ "$system_cpu_cores" -lt "1" ]
    then
        system_cpu_cores="1"
    fi

    # set max heap size based on the following
    # max(min(1/2 ram, 1024MB), min(1/4 ram, 8GB))
    # calculate 1/2 ram and cap to 1024MB
    # calculate 1/4 ram and cap to 8192MB
    # pick the max
    half_system_memory_in_mb=`expr $system_memory_in_mb / 2`
    quarter_system_memory_in_mb=`expr $half_system_memory_in_mb / 2`
    if [ "$half_system_memory_in_mb" -gt "1024" ]
    then
        half_system_memory_in_mb="1024"
    fi
    if [ "$quarter_system_memory_in_mb" -gt "8192" ]
    then
        quarter_system_memory_in_mb="8192"
    fi
    if [ "$half_system_memory_in_mb" -gt "$quarter_system_memory_in_mb" ]
    then
        max_heap_size_in_mb="$half_system_memory_in_mb"
    else
        max_heap_size_in_mb="$quarter_system_memory_in_mb"
    fi
    MAX_HEAP_SIZE="${max_heap_size_in_mb}M"

    # Young gen: min(max_sensible_per_modern_cpu_core * num_cores, 1/4 * heap size)
    max_sensible_yg_per_core_in_mb="100"
    max_sensible_yg_in_mb=`expr $max_sensible_yg_per_core_in_mb "*" $system_cpu_cores`

    desired_yg_in_mb=`expr $max_heap_size_in_mb / 4`

    if [ "$desired_yg_in_mb" -gt "$max_sensible_yg_in_mb" ]
    then
        HEAP_NEWSIZE="${max_sensible_yg_in_mb}M"
    else
        HEAP_NEWSIZE="${desired_yg_in_mb}M"
    fi
}

calculate_heap_sizes

# Dynamically calculate parameters, for reference.
Xms=$MAX_HEAP_SIZE
Xmx=$MAX_HEAP_SIZE
Xmn=$HEAP_NEWSIZE
MaxDirectMemorySize=$MAX_HEAP_SIZE
# Set for `JAVA_OPT`.
JAVA_OPT="${JAVA_OPT} -server -Xms${Xms} -Xmx${Xmx} -Xmn${Xmn}"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0 -XX:SurvivorRatio=8"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:/dev/shm/mq_gc_%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -XX:+PrintAdaptiveSizePolicy"
JAVA_OPT="${JAVA_OPT} -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT} -XX:+AlwaysPreTouch"
JAVA_OPT="${JAVA_OPT} -XX:MaxDirectMemorySize=${MaxDirectMemorySize}"
JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages -XX:-UseBiasedLocking"
JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"
echo "======================================"
echo $JAVA_OPT

numactl --interleave=all pwd > /dev/null 2>&1
if [ $? -eq 0 ]
then
	if [ -z "$RMQ_NUMA_NODE" ] ; then
		numactl --interleave=all $JAVA ${JAVA_OPT} $@
	else
		numactl --cpunodebind=$RMQ_NUMA_NODE --membind=$RMQ_NUMA_NODE $JAVA ${JAVA_OPT} $@
	fi
else
	$JAVA ${JAVA_OPT} $@
fi
```

#### 2.runserver-customize.sh

```bash
#!/bin/bash

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#===========================================================================================
# Java Environment Setting
#===========================================================================================
error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

find_java_home()
{
    case "`uname`" in
        Darwin)
            JAVA_HOME=$(/usr/libexec/java_home)
        ;;
        *)
            JAVA_HOME=$(dirname $(dirname $(readlink -f $(which javac))))
        ;;
    esac
}

find_java_home

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
calculate_heap_sizes()
{
    case "`uname`" in
        Linux)
            system_memory_in_mb=`free -m| sed -n '2p' | awk '{print $2}'`
            system_cpu_cores=`egrep -c 'processor([[:space:]]+):.*' /proc/cpuinfo`
        ;;
        FreeBSD)
            system_memory_in_bytes=`sysctl hw.physmem | awk '{print $2}'`
            system_memory_in_mb=`expr $system_memory_in_bytes / 1024 / 1024`
            system_cpu_cores=`sysctl hw.ncpu | awk '{print $2}'`
        ;;
        SunOS)
            system_memory_in_mb=`prtconf | awk '/Memory size:/ {print $3}'`
            system_cpu_cores=`psrinfo | wc -l`
        ;;
        Darwin)
            system_memory_in_bytes=`sysctl hw.memsize | awk '{print $2}'`
            system_memory_in_mb=`expr $system_memory_in_bytes / 1024 / 1024`
            system_cpu_cores=`sysctl hw.ncpu | awk '{print $2}'`
        ;;
        *)
            # assume reasonable defaults for e.g. a modern desktop or
            # cheap server
            system_memory_in_mb="2048"
            system_cpu_cores="2"
        ;;
    esac

    # some systems like the raspberry pi don't report cores, use at least 1
    if [ "$system_cpu_cores" -lt "1" ]
    then
        system_cpu_cores="1"
    fi

    # set max heap size based on the following
    # max(min(1/2 ram, 1024MB), min(1/4 ram, 8GB))
    # calculate 1/2 ram and cap to 1024MB
    # calculate 1/4 ram and cap to 8192MB
    # pick the max
    half_system_memory_in_mb=`expr $system_memory_in_mb / 2`
    quarter_system_memory_in_mb=`expr $half_system_memory_in_mb / 2`
    if [ "$half_system_memory_in_mb" -gt "1024" ]
    then
        half_system_memory_in_mb="1024"
    fi
    if [ "$quarter_system_memory_in_mb" -gt "8192" ]
    then
        quarter_system_memory_in_mb="8192"
    fi
    if [ "$half_system_memory_in_mb" -gt "$quarter_system_memory_in_mb" ]
    then
        max_heap_size_in_mb="$half_system_memory_in_mb"
    else
        max_heap_size_in_mb="$quarter_system_memory_in_mb"
    fi
    MAX_HEAP_SIZE="${max_heap_size_in_mb}M"

    # Young gen: min(max_sensible_per_modern_cpu_core * num_cores, 1/4 * heap size)
    max_sensible_yg_per_core_in_mb="100"
    max_sensible_yg_in_mb=`expr $max_sensible_yg_per_core_in_mb "*" $system_cpu_cores`

    desired_yg_in_mb=`expr $max_heap_size_in_mb / 4`

    if [ "$desired_yg_in_mb" -gt "$max_sensible_yg_in_mb" ]
    then
        HEAP_NEWSIZE="${max_sensible_yg_in_mb}M"
    else
        HEAP_NEWSIZE="${desired_yg_in_mb}M"
    fi
}

calculate_heap_sizes

# Dynamically calculate parameters, for reference.
Xms=$MAX_HEAP_SIZE
Xmx=$MAX_HEAP_SIZE
Xmn=$HEAP_NEWSIZE
# Set for `JAVA_OPT`.
JAVA_OPT="${JAVA_OPT} -server -Xms${Xms} -Xmx${Xmx} -Xmn${Xmn}"
JAVA_OPT="${JAVA_OPT} -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8  -XX:-UseParNewGC"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:/dev/shm/rmq_srv_gc.log -XX:+PrintGCDetails"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT}  -XX:-UseLargePages"
JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

$JAVA ${JAVA_OPT} $@
```

## 二、docker-compose

### 1、common-server

```bash
version: '3.8'

services:

  base-service:
    restart: always
    networks:
      - rocket-network

  volume-server:
    extends: base-service
    ulimits:
      nproc: 65535
      nofile:
        soft: 65535
        hard: 120000
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
```

### 2、docker-compose

```bash
version: '3.8'

networks:
  rocket-network:

services:
  rocket-nameserver:
    image: rocketmq:4.9.3
    extends:
      file: ./common-service.yml
      service: volume-server
    ports:
      - 9876:9876
    volumes:
      - ./middleware.d/rocketmq-server/nameserver/conf.d/nameserver.properties:/etc/rocketmq/nameserver.propertie
    environment:
      JAVA_OPT_EXT: "-Duser.home=/home/rocketmq -Xms512M -Xmx512M -Xmn128m"
    command: ["sh","mqnamesrv","-c","/etc/rocketmq/nameserver.propertie"]

  rocket-broker:
    image: rocketmq:4.9.3
    extends:
      file: ./common-service.yml
      service: volume-server
    ports:
      - 10909:10909
      - 10911:10911
    volumes:
      - ./middleware.d/rocketmq-server/broker/store:/home/rocketmq/store
      - ./middleware.d/rocketmq-server/broker/conf.d/broker.properties:/etc/rocketmq/broker.properties
    environment:
        JAVA_OPT_EXT: "-Duser.home=/home/rocketmq -Xms512M -Xmx512M -Xmn128m"
    command: ["sh","mqbroker","-c","/etc/rocketmq/broker.properties","-n","rocket-nameserver:9876"]
    depends_on:
      - rocket-nameserver

  rocket-console:
    image: styletang/rocketmq-console-ng
    extends:
      file: ./common-service.yml
      service: volume-server
    ports:
      - 8081:8080
    environment:
        JAVA_OPTS: "-Drocketmq.namesrv.addr=rocket-nameserver:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false"
    depends_on:
      - rocket-nameserver

  event-server:
    extends:
      file: ./common-service.yml
      service: common-springcloud-server
    ports:
      - "8091:8091"
    volumes:
      - ./event-server/jar/event.jar:/usr/src/skyboot/mrobot-server.jar:ro
      - ./event-server/logs:/home/mrobot/logs
    environment:
      - EVENT_SERVER_PROT=8091
      - MARIADB_HOST=mariadb-server
      - MARIADB_HOST_PORT=3306
      - MARIADB_USER=root
      - MARIADB_USER_PASSWD=root1qaz@WSX
```

### 3、相关配置文件

#### 1.broker.properties

```bash
# 集群名，不同broker节点集群名是一样的
brokerClusterName=brokercluster

# broker名字，不同broker节点brokerName是唯一的
brokerName=broker-a

# 设置brokerIP，如果不设置当服务器有很多网卡，默认会读取第一个网卡的IP地址（如安装docker后），会导致客户端无法连接
brokerIP1=172.16.1.13

# 0表示Master，大于0表示 Slave
brokerId=0

# 删除文件时间点，默认凌晨 4点
deleteWhen=04

# 文件保留时间，默认 48 小时
fileReservedTime=48

#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824

#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000

# 当前节点角色，ASYNC_MASTER=异步复制Master模式，SYNC_MASTER=同步双写Master模式，复制节点设置为SLAVE
brokerRole=ASYNC_MASTER

# 刷盘模式，ASYNC_FLUSH=异步刷盘，SYNC_FLUSH=同步刷盘
flushDiskType=ASYNC_FLUSH

# 数据存储路径(同一节点上master与slave需在不同目录)
storePathRootDir=/home/rocketmq/store

# commitLog 存储路径(同一节点上master与slave需在不同目录)
storePathCommitLog=/home/rocketmq/store/commitlog

# 消费队列存储路径存储路径(同一节点上master与slave需在不同目录)
storePathConsumeQueue=/home/rocketmq/store/consumequeue

# 消息索引存储路径(同一节点上master与slave需在不同目录)
storePathIndex=/home/rocketmq/store/index

# checkpoint 文件存储路径(同一节点上master与slave需在不同目录)
storeCheckpoint=/home/rocketmq/store/checkpoint

# abort 文件存储路径(同一节点上master与slave需在不同目录)
abortFile=/home/rocketmq/store/abort

# 是否允许 Broker 自动创建Topic，建议关闭
autoCreateTopicEnable=false

# 是否允许 Broker 自动创建订阅组，建议关闭
autoCreateSubscriptionGroup=false

# broker对外服务的监听端口，默认10911（同一节点启动多个服务，需要修改默认端口）
listenPort=10911
```

#### 2.nameserver.properties

```bash
# RocketMQ 主目录
rocketmqHome=/home/rocketmq/rocketmq-4.9.3

# kv 配置文件路径，包含顺序消息主题的配置信息
kvConfigPath=/home/rocketmq/kvConfig.json

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

