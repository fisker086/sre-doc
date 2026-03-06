# 安装MySQL8.0

## 一、确认支持列表：Supported Platforms

>https://www.mysql.com/support/supportedplatforms/database.html
>https://www.mysql.com/support/eol-notice.html

## 二、确定版本

### 1、软件命名规则

>MySQL 8.0中的命名方案使用的发行版名称由三个数字和一个可选的后缀组成（例如， **mysql-
>8.0.1-dmr**）。版本名称中的数字解释如下：
>- 第一个数字（**8**）是主版本号。
>- 第二个数字（**0**）是次要版本号。总而言之，主要和次要数字构成发行版本号。序列号描述了稳定
>的功能集。
>- 第三个数字（**1**）是发行系列中的版本号。对于每个新的错误修正版本，此值均递增。在大多数情
>况下，系列中的最新版本是最佳选择。
>版本名称也可以包含一个后缀，以指示版本的稳定性。在一系列发行中，发布会通过一组后缀来指示稳定
>性水平如何提高。可能的后缀是：
>- **dmr**指示开发里程碑版本（DMR）。MySQL开发使用里程碑模型，其中每个里程碑都引入了一小部
>分经过全面测试的功能。从一个里程碑到下一个里程碑，基于尝试这些正常发布的社区成员提供的反馈，功能
>界面可能会更改，甚至功能可能会被删除。里程碑版本中的功能可能被视为具有预生产质量。
>- **rc**表示发布候选（RC）。通过了MySQL的所有内部测试后，发布候选版本被认为是稳定的。RC版
>本中可能仍会引入新功能，但是重点将转移到修复错误上，以稳定本系列中较早引入的功能。
>- 没有后缀表示具有一般可用性（GA）或正式版。GA版本稳定，已成功通过了较早的发行阶段，并且被认
>为是可靠的，没有严重的错误并且适合在生产系统中使用。

### 2、确定版本

>备安装MySQL时，请确定要使用哪个版本和发行格式（二进制或源码）。
>首先，决定要安装开发版本还是通用版本（GA）。开发版本具有最新功能，但不建议用于生产环境。
>GA版本（也称为生产版本或稳定版本）是供生产使用的。

## 三、获取 MySQL软件

>https://downloads.mysql.com/archives/community/

**校验**

```bash
md5sum mysql-xxx.tar.gz
aaab65abbec64d5e907dcd41b8699945 mysql-xxx.tar.gz
```

## 四、安装

### 1、创建用户

```bash
useradd mysql
```

### 2、上传软件

```bash
cd /opt
tar xf mysql-8.0.24-linux-glibc2.12-x86_64.tar.xz
ln -s /opt/mysql-8.0.24-linux-glibc2.12-x86_64 /usr/local/mysql
```

### 3、设置环境变量

```bash
vim /etc/profile
export PATH=/usr/local/mysql/bin:$PATH

source /etc/profile

# 验证
mysql -V
mysql Ver 8.0.24 for Linux on x86_64 (MySQL Community Server - GPL)
```

### 4、创建数据目录并授权

```bash
mkdir -p /data/3306/data
chown -R mysql.mysql /data
```

### 5、配置文件准备

```bash
rm -rf /etc/my.cnf
vim /etc/my.cnf

[mysqld]
user=mysql
basedir=/usr/local/mysql
datadir=/data/3306/data
socket=/tmp/mysql.sock
server_id=51
[mysql]
socket=/tmp/mysql.sock
```

### 6、卸载无用软件

```bash
yum remove -y mariadb-libs
```

### 7、下载异步IO接口

```bash
yum install -y libaio-devel
```



### 8、初始化数据库

```bash
mysqld --initialize-insecure

# 或者
mysqld --initialize
```

>--initialize ： 初始化时，会自动创建超级管理员（root@'localhost'）,生成随机密码，12位，4种密码复杂度。这个密码，需要在第一次登陆时修改掉才可以正常管理数据。

### 9、复制启动脚本

```bash
[root@web02 /service]# cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
```



### 10、添加systemd管理

```bash
vim /usr/lib/systemd/system/mysqld.server
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=https://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target
[Install]
WantedBy=multi-user.target
[Service]
User=mysql
Group=mysql
ExecStart=/etc/init.d/mysqld --defaults-file=/etc/my.cnf
LimitNOFILE = 5000


systemctl daemon-reload
```

