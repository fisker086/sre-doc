# 用dockefile搭建wordpress

## 一、创建专用网桥

> 创建网桥并制定网段

```bash
[root@oldboy ~]# docker network create --driver bridge --subnet=172.18.0.0/16 --gateway=172.18.0.1 wordpress
2713a5e58333f4dc7f793487d6b98b6a88c970af534aa47b5c807601d2d264db
[root@oldboy ~]# docker network ls
NETWORK ID     NAME        DRIVER    SCOPE
b497bef69509   bridge      bridge    local
13ae75f26408   host        host      local
de7cb7a83412   none        null      local
2713a5e58333   wordpress   bridge    local
```



## 二、创建工作目录

```bash
[root@oldboy ~]# mkdir wp_Dockerfile
[root@oldboy ~]# cd wp_Dockerfile/
[root@oldboy ~/wp_Dockerfile]# mkdir mysql
[root@oldboy ~/wp_Dockerfile]# mkdir nginx
[root@oldboy ~/wp_Dockerfile]# mkdir php
```



## 三、上传、解压wordpress

```bash
[root@oldboy /opt]# rz -E
rz waiting to receive.
```

**解压**

```bash
[root@oldboy ~/opt]# tar xf wordpress-5.8.tar.gz
[root@oldboy ~/opt]# mv wordpress /
```



## 四、nginx的Dockerfile

### 1、写Dockerfile

```bash
[root@oldboy ~/wp_Dockerfile/nginx]# vim Dockerfile
FROM nginx
RUN groupadd www -g 666 && \
    useradd www -u 666 -g 666 -s /sbin/nologin -M
ADD linux.wp.com.conf /etc/nginx/conf.d/
ADD nginx.conf /etc/nginx/
RUN mkdir /code/wordpress/ -p
RUN rm -rf /etc/nginx/conf.d/default.conf
EXPOSE 80
WORKDIR /root
CMD ["nginx","-g","daemon off;"]
```



### 2、创建修改配置文件

#### 1）**nginx.conf**

```bash
[root@oldboy ~/wp_Dockerfile/nginx]# vim nginx.conf
user  www;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    server_tokens off;
    sendfile        on;

    keepalive_timeout  65;


    include /etc/nginx/conf.d/*.conf;

    client_max_body_size 200m;
    tcp_nopush on;
    gzip on;
    gzip_disable "MSIE [1-6]\.";
    gzip_http_version 1.1;
    gzip_comp_level 4;
    gzip_buffers 16 8k;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascriptimage/jpeg image/png;

}


```



#### 2）**linux.wp.com.conf**

```bash
[root@oldboy ~/wp_Dockerfile/nginx]# vim linux.wp.com.conf
server {
    listen 80;

    location / {
        root /code/wordpress;
        index index.php;
    }

    location ~* \.php$ {
        fastcgi_pass php:9000;
        fastcgi_param SCRIPT_FILENAME /code/wordpress/$fastcgi_script_name;
        include fastcgi_params;
    }
}


#创建镜像
docker build -t nginx:v1 .
```







## 五、php的Dockerfile

### 1、写Dockerfile

```bash
[root@oldboy ~/wp_Dockerfile/php]# cat Dockerfile
FROM php:7.4-fpm
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# Mysqli 扩展 自带 直接安装即可(当前数据库使用的mysqli查询的)
RUN docker-php-ext-install mysqli
RUN groupadd www -g 666 && \
    useradd www -u 666 -g 666 -s /sbin/nologin -M
ADD www.conf /usr/local/etc/php-fpm.d/
ADD php.ini /usr/local/etc/
ADD php-fpm.conf /usr/local/etc/
EXPOSE 9000
CMD ["php-fpm","-F","-y","/usr/local/etc/php-fpm.d/www.conf"]
```

```bash
FROM php:7.4-fpm
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# Mysqli 扩展 自带 直接安装即可(当前数据库使用的mysqli查询的)
RUN docker-php-ext-install mysqli
RUN groupadd www -g 666 && \
    useradd www -u 666 -g 666 -s /sbin/nologin -M
EXPOSE 9000
CMD ["php-fpm","-F","-y","/usr/local/etc/php-fpm.d/www.conf"]
```

```bash
docker run -d  -e TZ="Asia/Shanghai" --name php --network=wordpress --ip 172.20.0.2 -v /etc/phpconf/www.conf:/usr/local/etc/php-fpm.d/www.conf -v /etc/phpconf/php.ini:/usr/local/etc/php.ini -v /wordpress/wordpress:/wordpress/wordpress -p 9000:9000 php:v1
```





**php扩展**

```bash
FROM php:7.2.4-fpm
MAINTAINER xiexinyang <983600849@qq.com>

#docker中php扩展安装方式
#1、PHP源码文件目录自带扩展 docker-php-ext-install直接安装
#2、pecl扩展 因为一些扩展不包含在PHP源码文件中，PHP 的扩展库仓库中存在。用 pecl install 安装扩展，再用 docker-php-ext-enable 命令 启用扩展
#3、其他扩展 一些既不在 PHP 源码包，也不再 PECL 扩展仓库中的扩展，可以通过下载扩展程序源码，编译安装的方式安装

# 扩展版本号定义 

#redis 扩展
ENV PHPREDIS_VERSION 4.0.0
#msgpack扩展
ENV MSGPACK_VERSION 2.0.3
#memcached扩展
ENV MEMCACHED_VERSION 3.1.3
#mongodb扩展
ENV MONGODB_VERSION 1.5.3
#xhprof扩展 https://github.com/longxinH/xhprof/releases(pecl 不支持php7 使用这里的)
ENV XHPROF_VERSION 2.0.5
#swoole安装 如果以后用到的话，不用再安装了，4.0之后性能更好
ENV SWOOLE_VERSION 4.0.3
#swoole依赖hiredis
ENV HIREDIS_VERSION 0.13.3
# 设置时间
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo 'Asia/Shanghai' > /etc/timezone

# 扩展依赖
RUN apt-get update \
    && apt-get install -y \
        curl \
        wget \
        git \
        zip \
        libz-dev \
        libssl-dev \
        libnghttp2-dev \
        libpcre3-dev \
        libmemcached-dev \
        zlib1g-dev \
    && apt-get clean \
    && apt-get autoremove

# Composer安装
RUN curl -sS https://getcomposer.org/installer | php \
    && mv composer.phar /usr/local/bin/composer \
    && composer self-update --clean-backups

# Mysqli 扩展 自带 直接安装即可(当前数据库使用的mysqli查询的)
RUN docker-php-ext-install mysqli
# PDO 扩展 自带 直接安装即可
RUN docker-php-ext-install pdo_mysql
# Bcmath 扩展 自带 直接安装即可
RUN docker-php-ext-install bcmath
# Redis 扩展下载 pecl本地安装 开启扩展
RUN wget http://pecl.php.net/get/redis-${PHPREDIS_VERSION}.tgz -O /tmp/redis.tgz \
    && pecl install /tmp/redis.tgz \
    && rm -rf /tmp/redis.tgz \
    && docker-php-ext-enable redis

# msgpack 扩展下载 pecl本地安装 开启扩展(延迟队列使用减少源数据占用空间)
RUN wget http://pecl.php.net/get/msgpack-${MSGPACK_VERSION}.tgz -O /tmp/msgpack.tgz \
    && pecl install /tmp/msgpack.tgz \
    && rm -rf /tmp/msgpack.tgz \
    && docker-php-ext-enable msgpack

# memcached 扩展下载 pecl本地安装 开启扩展 前面已经通过 apt-get安装了libmemcached-dev依赖
RUN wget http://pecl.php.net/get/memcached-${MEMCACHED_VERSION}.tgz -O /tmp/memcached.tgz \
    && pecl install /tmp/memcached.tgz \
    && rm -rf /tmp/memcached.tgz \
    && docker-php-ext-enable memcached

# mongodb 扩展下载 pecl本地安装 开启扩展 前面已经通过 
RUN wget http://pecl.php.net/get/mongodb-${MONGODB_VERSION}.tgz -O /tmp/mongodb.tgz \
    && pecl install /tmp/mongodb.tgz \
    && rm -rf /tmp/mongodb.tgz \
    && docker-php-ext-enable mongodb


# xhprof github上下载支持php7的扩展 安装 开启扩展
RUN wget https://github.com/longxinH/xhprof/archive/v${XHPROF_VERSION}.tar.gz -O /tmp/xhprof.tar.gz \
    && mkdir -p /tmp/xhprof \
    && tar -xf /tmp/xhprof.tar.gz -C /tmp/xhprof --strip-components=1 \
    && rm /tmp/xhprof.tar.gz \
    && ( \
        cd /tmp/xhprof/extension \
        && phpize \
        && ./configure  \
        && make -j$(nproc) \
        && make install \
    ) \
    && rm -r /tmp/xhprof \
    && docker-php-ext-enable xhprof


# Hiredis依赖安装
RUN wget https://github.com/redis/hiredis/archive/v${HIREDIS_VERSION}.tar.gz -O /tmp/hiredis.tar.gz \
  && mkdir -p /tmp/hiredis \
    && tar -xf /tmp/hiredis.tar.gz -C /tmp/hiredis --strip-components=1 \
    && rm /tmp/hiredis.tar.gz \
    && ( \
        cd /tmp/hiredis \
        && make -j$(nproc) \
        && make install \
        && ldconfig \
    ) \
    && rm -r /tmp/hiredis

# Swoole 扩展安装 开启扩展
RUN wget https://github.com/swoole/swoole-src/archive/v${SWOOLE_VERSION}.tar.gz -O /tmp/swoole.tar.gz \
    && mkdir -p /tmp/swoole \
    && tar -xf /tmp/swoole.tar.gz -C /tmp/swoole --strip-components=1 \
    && rm /tmp/swoole.tar.gz \
    && ( \
        cd /tmp/swoole \
        && phpize \
        && ./configure --enable-async-redis --enable-mysqlnd --enable-openssl --enable-http2 \
        && make -j$(nproc) \
        && make install \
    ) \
    && rm -r /tmp/swoole \
    && docker-php-ext-enable swoole

```





### 2、修改配置文件

#### 1）**www.conf**

```bash
[root@oldboy ~/wp_Dockerfile/php]# vim www.conf
....
user = www
group = www
listen = 9000
;listen.allowed_clients = 127.0.0.1
request_terminate_timeout = 0
....
```



#### 2）**php.ini**

```bash
[root@oldboy ~/wp_Dockerfile/php]# vim php.ini
...
upload_max_filesize = 200M
post_max_size = 200M
...




#创建镜像
	docker build -t php:v1 .
```



## 六、mysql的Dockerfile

### 1、写 Dockerfile

```bash
[root@oldboy ~/wp_Dockerfile/mysql]# cat Dockerfile
FROM mysql:5.7

#设置免密登录
ENV MYSQL_ALLOW_EMPTY_PASSWORD yes

#添加所需文件
ADD setup.sh /mysql/setup.sh
ADD schema.sql /mysql/schema.sql
ADD privileges.sql /mysql/privileges.sql

#设置容器启动时执行的命令
CMD ["sh", "/mysql/setup.sh"]

```

### 2、写脚本及sql文件

#### 1）setup.sh

```bash
[root@oldboy ~/wp_Dockerfile/mysql]# cat setup.sh
#!/bin/bash
set -e

#查看mysql服务的状态，方便调试，这条语句可以删除
echo `service mysql status`

echo '1.启动mysql....'
#启动mysql
service mysql start
sleep 3
echo `service mysql status`

echo '2.开始导入数据....'
#导入数据
mysql < /mysql/schema.sql
echo '3.导入数据完毕....'

sleep 3
echo `service mysql status`

#重新设置mysql密码
echo '4.开始修改密码....'
mysql < /mysql/privileges.sql
echo '5.修改密码完毕....'

#sleep 3
echo `service mysql status`
echo `mysql容器启动完毕,且数据导入成功`

tail -f /dev/null

```

**这里是先导入数据，然后才是设置用户和权限，是因为mysql容器一开始为免密登录，Dockerfile中有如下设置：`ENV MYSQL_ALLOW_EMPTY_PASSWORD yes`,此时执行导入数据命令不需要登录验证操作，如果是先执行权限操作，那么导入数据则需要登录验证，整个过程就麻烦了许多。**



#### 2）schema.sql

```bash
[root@oldboy ~/wp_Dockerfile/mysql]# cat schema.sql
create database wordpress;
```



#### 3）privileges.sql

```bash
[root@oldboy ~/wp_Dockerfile/mysql]# cat privileges.sql
use mysql;

create user wp@'%' identified by '123';
grant all on wordpress.* to wp@'%';
SET PASSWORD=PASSWORD('123456');
flush privileges;



#创建镜像
	docker build -t mysql:v1 .
```



## 七、创建启动容器

```bash
docker run -d -e TZ="Asia/Shanghai" --name mysql --network=wordpress --ip 172.18.0.2 wp_mysql:v1

docker run -d  -e TZ="Asia/Shanghai" --name php --network=wordpress --ip 172.18.0.4 -v /wordpress:/code/wordpress/ php:v1

docker run -d -e TZ="Asia/Shanghai" --name nginx --network=wordpress --ip 172.18.0.3 -v /wordpress:/code/wordpress/ -p 81:80 nginx:v1

```

**其他**

```bash
docker run -d -e TZ="Asia/Shanghai" --name wp_nginx --network=wordpress --ip 172.18.0.3 -v /www/wwwroot/blog.sholdboyedu.com/wordpress:/www/wwwroot/blog.sholdboyedu.com/wordpress -v /www/wwwroot/blog.sholdboyedu.com/docker/nginx.conf:/etc/nginx/nginx.conf -v /www/wwwroot/blog.sholdboyedu.com/docker/conf.d:/etc/nginx/conf.d -v /www/wwwroot/blog.sholdboyedu.com/docker/ssl_key:/etc/nginx/ssl_key/blog.sholdboyedu.com -p 6680:80 -p 6443:443 nginx:v1

docker run -d -e TZ="Asia/Shanghai" -e MARIADB_ROOT_PASSWORD=root1qaz@WSX --name mariadb_test -p3306:3306 bitnami/mariadb:10.2.21
docker run -d -e TZ="Asia/Shanghai" -e MARIADB_ROOT_PASSWORD=root1qaz@WSX -v /home/cloud/db_test/mariadb:/bitnami/mariadb --name mariadb_test -p3306:3306 bitnami/mariadb:10.2.21
```



## 八、上传wordpress

```bash
上传
rz wordpress*.tar.gz

解压
tar -xvf wordpress*.tar.gz

mv wordpress /root/wp_Dockerfile/

创建用户
	groupadd www -g 666 && \
    useradd www -u 666 -g 666 -s /sbin/nologin -M
    
授权
	chown -R www.www /root/wp_Dockerfile/wordpress
```



 