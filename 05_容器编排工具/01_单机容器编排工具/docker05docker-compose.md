# docker-compose

## 一、介绍

```bash
	Compose 定位是 「定义和运⾏多个 Docker 容器的应⽤（Defining and running multi-container Docker applications）」，其前身是开源项⽬ Fig。
	通过第⼀部分中的介绍，我们知道使⽤⼀个 Dockerfile 模板⽂件，可以让⽤户很⽅便的定义⼀个单独的应⽤容 器。然⽽，在⽇常⼯作中，经常会碰到需要多个容器相互配合来完成某项任务的情况。例如要实现⼀个 Web 项 ⽬，除了 Web 服务容器本身，往往还需要再加上后端的数据库服务容器，甚⾄还包括负载均衡容器等。 Compose 恰好满⾜了这样的需求。它允许⽤户通过⼀个单独的 docker-compose.yml 模板⽂件（YAML 格式） 来定义⼀组相关联的应⽤容器为⼀个项⽬。（project）
```

## 二、Compose 中有两个重要的概念：

```bash
服务 ( service )：⼀个应⽤的容器，实际上可以包括若⼲运⾏相同镜像的容器实例。 
项⽬ ( project )：由⼀组关联的应⽤容器组成的⼀个完整业务单元，在 docker-compose.yml ⽂件中定义。 
	Compose 的默认管理对象是项⽬，通过⼦命令对项⽬中的⼀组容器进⾏便捷地⽣命周期管理。 Compose 项⽬由 11_开发学习 编写，实现上调⽤了 Docker 服务提供的 API 来对容器进⾏管理。因此，只要所操作的平 台⽀持 Docker API，就可以在其上利⽤ Compose 来进⾏编排管理。
```

## 三、安装与卸载

### 1、安装

#### 1.二进制安装

 linux 在 Linux 上的也安装⼗分简单，从 官⽅ GitHub Release 处直接下载编译好的⼆进制⽂件即可。例如，在 Linux 64 位系统上直接下载对应的⼆进制包。

```bash
[root@docker ~]# curl -L https://github.com/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

##上面这个比较慢
[root@docker ~]# curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

[root@docker ~]# chmod +x /usr/local/bin/docker-compose
```

或者上传安装

```
[root@docker ~]# rz -E docker-compose-Linux-x86_64

[root@docker ~]# chmod +x docker-compose-Linux-x86_64

[root@docker ~]# mv docker-compose-Linux-x86_64  /usr/local/bin/docker-compose
```

#### 2.yum安装

```bash
yum install docker-compose -y
```



#### 3.验证安装

```
[root@docker ~]# docker-compose --version
docker-compose version 1.25.5, build 8a1c60f6
```



#### 4.卸载 

如果是⼆进制包⽅式安装的，删除⼆进制⽂件即可。

```
[root@docker ~]# rm /usr/local/bin/docker-compose
```



## 四、docker compose使用

⾸先介绍⼏个术语。 

### 1、相关概念

服务 ( service )：⼀个应⽤容器，实际上可以运⾏多个相同镜像的实例。 

项⽬ ( project )：由⼀组关联的应⽤容器组成的⼀个完整业务单元。∂⼀个项⽬可以由多个服务（容器）关联 ⽽成， Compose ⾯向项⽬进⾏管理。

### 2、场景

最常⻅的项⽬是 web ⽹站，该项⽬应该包含 web 应⽤和缓存。 Django应⽤ mysql服务 redis服务 elasticsearch服务

### 3、docker-compose模板

```
version: "3.0"
services:
  mysql:
    image: mysql:5.7
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 123
      MYSQL_DATABASE: wordpress
    networks:
      - wps

  php:
    hostname: php
    build:
      context: ./php
      dockerfile: Dockerfile
    ports:
      - "9000:9000"
    networks:
      - wps
    volumes:
      - /mnt/www/:/usr/share/nginx/html 

  nginx:
    hostname: nginx
    build:
      context: ./nginx
      dockerfile: Dockerfile
    ports:
      - "80:80"
    networks:
      - wps
    volumes:
      - /mnt/www/:/usr/share/nginx/html 
    depends_on:
      - php

networks:
  wps:
```

 4.通过docker-compose运⾏⼀组容器

```
[root@docker docker-compose]# docker-compose up        ## 在前台运行
[root@docker docker-compose]# docker-compose up -d     ## 启动守护进程
```



### 4、docker-compose 模板文件

模板⽂件是使⽤ Compose 的核⼼，涉及到的指令关键字也⽐较多。但⼤家不⽤担⼼，这⾥⾯⼤部分指令跟 docker run 相关参数的含义都是类似的。 默认的模板⽂件名称为 docker-compose.yml ，格式为 YAML 格式。

```
version: "3.0"
services:
  mysql:
    image: mysql:5.7
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 123
      MYSQL_DATABASE: wordpress
    networks:
      - wps
```

注意每个服务都必须通过 image 指令指定镜像或 build 指令（需要 Dockerfile）等来⾃动构建⽣成镜像。 如果使⽤ build 指令，在 Dockerfile 中设置的选项(例如： CMD , EXPOSE , VOLUME , ENV 等) 将会⾃动被获 取，⽆需在 docker-compose.yml 中重复设置。 下⾯分别介绍各个指令的⽤法。

#### 1.build指定Dockerfile所在文件夹名称

指定 Dockerfile 所在⽂件夹的路径（可以是绝对路径，或者相对 docker-compose.yml ⽂件的路径）。 Compose 将会利⽤它⾃动构建这个镜像，然后使⽤这个镜像。

```
version: "3.0"
services:
  nginx：
    build: ./nginx
```

#### 2.context指定dockerfile文件名

 指令指定 Dockerfile 所在⽂件夹的路径。

使⽤ dockerfile 指令指定 Dockerfile ⽂件名。 

    version: "3.0"
    services:
      nginx：
        build:
          context: ./nginx
          dockerfile: Dockerfile
#### 3.command容器启动命令

 覆盖容器启动后默认执⾏的命令。

```
command: echo "hello world" 
```

####  4.container_name指定容器名称

 指定容器名称。默认将会使⽤ 项⽬名称_服务名称_序号 这样的格式。 注意: 指定容器名称后，该服务将⽆法进⾏扩展（scale），因为 Docker 不允许多个容器具有相同的名称。 

```
container_name: docker-nginx-container
```

#### 5.depends_on容器启动依赖容器

 解决容器的依赖、启动先后的问题。以下例⼦中会先启动 php 再启动 nginx 注意： nginx 服务不会等待 php 「完全启动」之后才启动。 

```
version: "3.0"
services:
  php:
    hostname: php
    build:
      context: ./php
      dockerfile: Dockerfile

  nginx:
    hostname: nginx
    build:
      context: ./nginx
      dockerfile: Dockerfile
    depends_on:
      - php
```

#### 5.env_file环境变量文件

从⽂件中获取环境变量，可以为单独的⽂件路径或列表。 如果通过 docker-compose -f FILE ⽅式来指定 Compose 模板⽂件，则 env_file 中变量的路径会基于模板 ⽂件路径。 如果有变量名称与 environment 指令冲突，则按照惯例，以后者为准。 

```yaml
env_file: .env 
env_file: 
  - ./common.env 
  - ./apps/nginx.env 
  - /opt/secrets.env
```

环境变量⽂件中每⼀⾏必须符合格式，⽀持 # 开头的注释⾏

```
MYSQL_ROOT_PASSWORD=123
```

#### 6.environment 设置环境变量

设置环境变量。

你可以使⽤数组或字典两种格式。 只给定名称的变量会⾃动获取运⾏ Compose 主机上对应变量的值，可以⽤来防⽌泄露不必要的数据。

    environment:
      MYSQL_ROOT_PASSWORD: 123
      MYSQL_DATABASE: wordpress
      
    environment:
      - MYSQL_ROOT_PASSWORD=123
      - MYSQL_DATABASE=wordpress
#### 7.healthcheck健康检查

通过命令检查容器是否健康运⾏。

```
healthcheck:
 test: ["CMD", "curl", "-f", "http://localhost"]
 interval: 1m30s
 timeout: 10s
 retries: 3
```

#### 8.image指定镜像名称

 指定为镜像名称或镜像 ID。如果镜像在本地不存在， Compose 将会尝试拉取这个镜像。

```
version: "3.0"
services:
  mysql:
    image: mysql:5.7
```

#### 9.networks指定网络

配置容器连接的⽹络。

```
version: "3.0"
services:
  mysql:
    image: mysql:5.7
    networks:
      - wps

networks:
  wps:
```

#### 10.ports指定端口

 暴露端⼝信息。 使⽤宿主端⼝：容器端⼝ (HOST:CONTAINER) 格式，或者仅仅指定容器的端⼝（宿主将会随机选择端⼝）都可 以。

```
version: "3.0"
services:
  mysql:
    image: mysql:5.7
    ports:
      - "3306:3306"
      
      
      
      
- "127.0.0.1:3306:3306"
- "3306"
```

#### 11.sysctls容器内核参数配置

 配置容器内核参数。

```
sysctls:
 net.core.somaxconn: 1024
 net.ipv4.tcp_syncookies: 0
 
sysctls:
  - net.core.somaxconn=1024
  - net.ipv4.tcp_syncookies=0
```

###### ulimits

 指定容器的 ulimits 限制值。 例如，指定最⼤进程数为 65535，指定⽂件句柄数为 20000（软限制，应⽤可以随时修改，不能超过硬限制） 和 40000（系统硬限制，只能 root ⽤户提⾼）。

```
 ulimits:
 nproc: 65535
 nofile:
 soft: 20000
 hard: 40000
```

#### 12.volumes数据持久化

 数据卷所挂载路径设置。可以设置为宿主机路径( HOST:CONTAINER )或者数据卷名称( VOLUME:CONTAINER )，并且 可以设置访问模式 （ HOST:CONTAINER:ro ）。

 该指令中路径⽀持相对路径。

```
version: "3.0"
services:
  php:
    hostname: php
    build:
      context: ./php
      dockerfile: Dockerfile
      - /mnt/www/:/usr/share/nginx/html
```

###  5、docker-compose 常⽤命令

#### 1.命令对象与格式 

对于 Compose 来说，⼤部分命令的对象既可以是项⽬本身，也可以指定为项⽬中的服务或者容器。如果没有特别 的说明，命令对象将是项⽬，这意味着项⽬中所有的服务都会受到命令影响。 

执⾏ docker-compose [COMMAND] --help 或者 docker-compose help [COMMAND] 可以查看具体某个命令的 使⽤格式。

```
docker-compose [-f=...] [options] [COMMAND] [ARGS...]
```

#### 2.命令选项 

```bash
-f, --file FILE 指定使⽤的 Compose 模板⽂件，默认为 docker-compose.yml ，可以多次指定。 

-p, --project-name NAME 指定项⽬名称，默认将使⽤所在⽬录名称作为项⽬名。 

--x-networking 使⽤ Docker 的可拔插⽹络后端特性 

--x-network-driver DRIVER 指定⽹络后端的驱动，默认为 bridge 

--verbose 输出更多调试信息。 

-v, --version 打印版本并退出。
```

#### 3.子命令使⽤说明 

##### 1）up

up 格式为   up [options] [SERVICE...] 。

 该命令⼗分强⼤，它将尝试⾃动完成包括构建镜像，（重新）创建服务，启动服务，并关联服务相关容器的⼀ 系列操作。 链接的服务都将会被⾃动启动，除⾮已经处于运⾏状态。 可以说，⼤部分时候都可以直接通过该命令来启动⼀个项⽬。 默认情况， docker-compose up 启动的容器都在前台，控制台将会同时打印所有容器的输出信息，可以很 ⽅便进⾏调试。 当通过 Ctrl-C 停⽌命令时，所有容器将会停⽌。 

如果使⽤ docker-compose up -d ，将会在后台启动并运⾏所有的容器。⼀般推荐⽣产环境下使⽤该选项。 默认情况，如果服务容器已经存在， docker-compose up 将会尝试停⽌容器，然后重新创建（保持使⽤ volumes-from 挂载的卷），以保证新启动的服务匹配 docker-compose.yml ⽂件的最新内容 



##### 2）down

down 此命令将会停⽌ up 命令所启动的容器，并移除⽹络 exec 进⼊指定的容器。 ps 格式为 docker-compose ps [options] [SERVICE...] 。 列出项⽬中⽬前的所有容器。 选项： -q 只打印容器的 ID 信息。 1 docker-compose [-f=...] [options] [COMMAND] [ARGS...] 



##### 3）restart

restart 格式为 docker-compose restart [options] [SERVICE...] 。 重启项⽬中的服务。 选项： -t, --timeout TIMEOUT 指定重启前停⽌容器的超时（默认为 10 秒）。 



##### 4）rm

rm 格式为 docker-compose rm [options] [SERVICE...] 。 删除所有（停⽌状态的）服务容器。推荐先执⾏ docker-compose stop 命令来停⽌容器。 选项： -f, --force 强制直接删除，包括⾮停⽌状态的容器。⼀般尽量不要使⽤该选项。 -v 删除容器所挂载的数据卷。 start 格式为 docker-compose start [SERVICE...] 。 启动已经存在的服务容器。 



##### 5）stop

stop 格式为 docker-compose stop [options] [SERVICE...] 。 停⽌已经处于运⾏状态的容器，但不删除它。通过 docker-compose start 可以再次启动这些容器。 选项： -t, --timeout TIMEOUT 停⽌容器时候的超时（默认为 10 秒）。



##### 6）top

top 查看各个服务容器内运⾏的进程。 



##### 7）unpause

unpause 格式为 docker-compose unpause [SERVICE...] 。 恢复处于暂停状态中的服务。



## wordpress项目

### 构建docker-compose.yaml

```
[root@docker ~]# mkdir -p /root/docker/docker-compose

[root@docker ~]# cd /root/docker/docker-compose

[root@docker docker-compose]# pwd
/root/docker/docker-compose

[root@docker docker-compose]# ls
docker-compose.yml  nginx  php

[root@docker docker-compose]# cat docker-compose.yml 
version: "3.0"
services:
  mysql:
    image: mysql:5.7
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 123
      MYSQL_DATABASE: wordpress
    networks:
      - wps

  php:
    hostname: php
    build:
      context: ./php
      dockerfile: Dockerfile
    ports:
      - "9000:9000"
    networks:
      - wps
    volumes:
      - /mnt/www/:/usr/share/nginx/html 

  nginx:
    hostname: nginx
    build:
      context: ./nginx
      dockerfile: Dockerfile
    ports:
      - "80:80"
    networks:
      - wps
    volumes:
      - /mnt/www/:/usr/share/nginx/html 
    depends_on:
      - php

networks:
  wps:
```

### 构建php镜像Dockerfile

```
[root@docker docker-compose]# mkdir php

[root@docker docker-compose]# cd php
[root@docker php]# ls
Dockerfile  php.repo  php.tar  www.conf

[root@docker php]# cat Dockerfile 
FROM centos@sha256:0f4ec88e21daf75124b8a9e5ca03c37a5e937e0e108a255d890492430789b60e

MAINTAINER name:小王 date:2021.07.29 

RUN rm -rf /etc/yum.repos.d/*

RUN curl -o /etc/yum.repos.d/CentOS-Base.repo https://repo.huaweicloud.com/repository/conf/CentOS-7-reg.repo

RUN curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

RUN yum clean all && yum makecache

ADD php.tar /opt/

RUN yum -y localinstall /opt/*.rpm

ADD www.conf /etc/php-fpm.d/

RUN groupadd www -g 1000 && \
   useradd www -u 1000 -g 1000

VOLUME /usr/share/nginx/html/

EXPOSE 9000 

CMD ["php-fpm","-F"]
```



### 构建nginx镜像Dockerfile

```
[root@docker docker-compose]# mkdir nginx

[root@docker docker-compose]# cd nginx

[root@docker nginx]# ls
default.conf  Dockerfile  nginx.conf  nginx.repo

[root@docker php]# ls
Dockerfile  php.repo  php.tar  www.conf

[root@docker nginx]# cat Dockerfile 
FROM centos@sha256:0f4ec88e21daf75124b8a9e5ca03c37a5e937e0e108a255d890492430789b60e

MAINTAINER name:小王 date:2021.07.29

ADD nginx.repo /etc/yum.repos.d/

RUN yum install nginx -y

RUN groupadd www -g 1000 && \
   useradd www -u 1000 -g 1000

RUN rm -rf /etc/nginx/conf.d/default.conf

ADD default.conf /etc/nginx/conf.d/

ADD nginx.conf  /etc/nginx/

EXPOSE 80 

VOLUME /usr/share/nginx/html

CMD ["nginx","-g","daemon off;"]
```



### 挂载wordpress

```
[root@docker mnt]# cd wordpress/

[root@docker wordpress]# cp -r ./* ../www/

[root@docker wordpress]# chown -R www.www /mnt/www/


```

### 启动docker-compose.yaml

```
[root@docker docker-compose]# docker-compose up -d 
[root@docker docker-compose]# docker-compose ps
         Name                      Command             State                 Ports              
------------------------------------------------------------------------------------------------
docker-compose_mysql_1   docker-entrypoint.sh mysqld   Up      0.0.0.0:3306->3306/tcp, 33060/tcp
docker-compose_nginx_1   nginx -g daemon off;          Up      0.0.0.0:80->80/tcp               
docker-compose_php_1     php-fpm -F                    Up      0.0.0.0:9000->9000/tcp           
[root@docker docker-compose]# docker-compose ps
         Name                      Command             State                 Ports              
------------------------------------------------------------------------------------------------
docker-compose_mysql_1   docker-entrypoint.sh mysqld   Up      0.0.0.0:3306->3306/tcp, 33060/tcp
docker-compose_nginx_1   nginx -g daemon off;          Up      0.0.0.0:80->80/tcp               
docker-compose_php_1     php-fpm -F                    Up      0.0.0.0:9000->9000/tcp  
```

