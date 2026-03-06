# Nacos物理单机安装

## 一、部署环境

```bash
    Nacos定义为一个IDC内部应用组件，并非面向公网环境的产品，建议在内部隔离网络环境中部署，强烈不建议部署在公共网络环境。
```

## 二、Nacos支持三种部署模式

- 单机模式 - 用于测试和单机试用。
- 集群模式 - 用于生产环境，确保高可用。
- 多集群模式 - 用于多数据中心场景。

## 三、物理部署

### 1、环境准备

```bash
     Nacos 依赖 Java 环境来运行。如果您是从代码开始构建并运行Nacos，还需要为此配置 Maven环境，请确保是在以下版本环境中安装使用:
     64 bit OS，支持 Linux/Unix/Mac/Windows，推荐选用 Linux/Unix/Mac。
     64 bit JDK 1.8+
     Maven 3.2.x+
     git
```

#### 1.安装JDK1.8和git

```bash
apt-get install openjdk-8-jdk git -y
```

#### 2.检查JDK和git是否安装成功

```bash
java -version
git --version
```

#### 3.安装Maven

> https://maven.apache.org/download.cgi

##### 1)下载并解压Maven二进制包

```bash
cd /opt
wget https://dlcdn.apache.org/maven/maven-3/3.8.5/binaries/apache-maven-3.8.5-bin.tar.gz
tar xf apache-maven-3.8.5-bin.tar.gz
```

##### 2）移动到指定目录

```bash
mv apache-maven-3.8.5 /usr/local/
```

##### 3）设置环境变量

```bash
vim /etc/profile.d/maven.sh
export M2_HOME=/usr/local/apache-maven-3.8.5
export PATH=${M2_HOME}/bin:$PATH

source /etc/profile
```

##### 4）测试

```bash
mvn -v
```

### 2、下载安装包并解压

```bash
cd /opt
wget https://github.com/alibaba/nacos/releases/download/2.0.4/nacos-server-2.0.4.tar.gz
tar xf nacos-server-2.0.4.tar.gz

#生产环境可以移动到项目目录

```

### 3、修改配置文件

> vim /opt/nacos/conf/application.properties

#### 1.端口冲突

```bash
#如果端口冲突修改端口
server.port=8848
```

#### 2.使用外置数据库

**使用外置数据前要先初始化数据库，既执行./conf/nacos-mysql.sql**

```bash
#*************** Config Module Related Configurations ***************#
### If use MySQL as datasource:
spring.datasource.platform=mysql

### Count of DB:
db.num=1

### Connect URL of DB:
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=nacos
db.password.0=nacos
```

### 4、启动

```bash
cd /opt/nacos/bin
bash startup.sh -m standalone
```

### 5、访问

```bash
http://IP:port/nacos

#默认账号密码
nacos/nacos
```

### 6、服务注册&发现和配置管理

#### 1.服务注册

```bash
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'
```

#### 2.服务发现

```bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=nacos.naming.serviceName'
```

#### 3.发布配置

```bash
curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=HelloWorld"
```

#### 4.获取配置

```bash
curl -X GET "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test"
```

### 7、关闭服务器

```bash
sh shutdown.sh
```

## 四、容器化部署

### 1、容器变量

| name                          | description                     | option                                 |
| ----------------------------- | ------------------------------- | -------------------------------------- |
| MODE                          | cluster模式/standalone模式      | cluster/standalone default **cluster** |
| NACOS_SERVERS                 | nacos cluster地址               | eg. ip1,ip2,ip3                        |
| PREFER_HOST_MODE              | 是否支持hostname                | hostname/ip default **ip**             |
| NACOS_SERVER_PORT             | nacos服务器端口                 | default **8848**                       |
| NACOS_SERVER_IP               | 多网卡下的自定义nacos服务器IP   |                                        |
| SPRING_DATASOURCE_PLATFORM    | standalone 支持 mysql           | mysql / empty default empty            |
| MYSQL_MASTER_SERVICE_HOST     | mysql 主节点host                |                                        |
| MYSQL_MASTER_SERVICE_PORT     | mysql 主节点端口                | default : **3306**                     |
| MYSQL_MASTER_SERVICE_DB_NAME  | mysql 主节点数据库              |                                        |
| MYSQL_MASTER_SERVICE_USER     | 数据库用户名                    |                                        |
| MYSQL_MASTER_SERVICE_PASSWORD | 数据库密码                      |                                        |
| MYSQL_SLAVE_SERVICE_HOST      | mysql从节点host                 |                                        |
| MYSQL_SLAVE_SERVICE_PORT      | mysql从节点端口                 | default :3306                          |
| MYSQL_DATABASE_NUM            | 数据库数量                      | default :2                             |
| JVM_XMS                       | -Xms                            | default :2g                            |
| JVM_XMX                       | -Xmx                            | default :2g                            |
| JVM_XMN                       | -Xmn                            | default :1g                            |
| JVM_MS                        | -XX:MetaspaceSize               | default :128m                          |
| JVM_MMS                       | -XX:MaxMetaspaceSize            | default :320m                          |
| NACOS_DEBUG                   | 开启远程调试                    | y/n default :n                         |
| TOMCAT_ACCESSLOG_ENABLED      | server.tomcat.accesslog.enabled | default :false                         |

### 1、docker-run

```bash
docker pull nacos/nacos-server:v2.0.4
docker run --name nacos-quick -e MODE=standalone -p 8849:8848 -d nacos/nacos-server:2.0.2
```

#### 1.挂载存储卷

```bash
docker run --name nacos -d -p 8848:8848 --privileged=true --restart=always -e JVM_XMS=256m -e JVM_XMX=256m -e MODE=standalone -e PREFER_HOST_MODE=hostname -v /home/nacos/logs:/home/nacos/logs -v /home/nacos/conf:/home/nacos/conf  nacos/nacos-server:v2.0.4
```

### 2、docker-compose

>https://nacos.io/zh-cn/docs/quick-start-docker.html

```bash
如果使用对自己独立的外置数据库，可以通过修改docker-compose去除数据库部分并修改配置文件即可
```



## 五、物理集群化

>https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html

可以采用nginx做轮询