# Gerrit安装-物理机安装

## 一、环境准备

### 1、安装jdk和git

```bash
apt-get install openjdk-17-jdk git -y
```

### 2、创建用户

```bash
groupadd gerrit -g 666
useradd gerrit -u 666 -g 666 -s /sbin/nologin -M
```

## 二、安装

### 1、准备安装目录和war包存放位置

```bash
mkdir -p /home/gerrit/gerrit_site
mkdir -p /home/gerrit/wars
```

### 2、下载安装包

>https://www.gerritcodereview.com/

```bash
cd /home/gerrit/wars
wget https://gerrit-releases.storage.googleapis.com/gerrit-3.10.1.war
```

### 3、设置全局环境变量

```bash
vim /etc/profile.d/gerrit.sh
export GERRIT_SITE=/home/gerrit/gerrit_site

##生效
source /etc/profile
```

### 4、安装

```bash
java -jar gerrit-3.10.1.war init -d $GERRIT_SITE


#其他都默认，plugin插件都选y

##一路回车，然后再改配置
```

### 5、修改配置

```bash
cp $GERRIT_SITE/etc/gerrit.config $GERRIT_SITE/etc/gerrit.config.bak
vim $GERRIT_SITE/etc/gerrit.config

[gerrit]
        basePath = git
        canonicalWebUrl = http://172.16.3.50:8080/
        serverId = b4ceb906-c94a-4049-9d2f-28b2441e2521
[auth]
        type = HTTP
[sendemail]
        enable = true
        smtpServer = smtp.qq.com
        smtpServerPort = 465
        smtpEncryption = SSL
        sslVerify = true
        smtpUser = 1426115933@qq.com
        from = 1426115933@qq.com
        smtpPass = (企业邮箱密码或者授权码)
[container]
        javaOptions = "-Dflogger.backend_factory=com.google.common.flogger.backend.log4j.Log4jBackendFactory#getInstance"
        javaOptions = "-Dflogger.logging_context=com.google.gerrit.server.logging.LoggingContext#getInstance"
        user = gerrit
        javaHome = /usr/lib/jvm/java-8-openjdk-amd64/jre
        heapLimit = 4g
[sshd]
        listenAddress = *:29418
[httpd]
        listenUrl = proxy-http://*:8081/
[cache]
        directory = cache
[gitweb]
        cgi = /usr/lib/cgi-bin/gitweb.cgi
        type = gitweb
[index]
        type = lucene
[receive]
        enableSignedPush = false
[capability]
        accessDatabase = group Administrators
[plugins]
        allowRemoteAdmin = true
```

### 6、gerrit目录授权

```bash
chown -R gerrit:gerrit /home/gerrit
```

### 7、安装gitweb

```bash
apt-get install gitweb -y
```

### 8、新增管理员账号

```bash
touch /home/gerrit/gerrit_site/etc/passwords
htpasswd -c /home/gerrit/gerrit_site/etc/gerrit.password gerrit    # 创建第一个用户admin，同时会生成一个gerrit.password文件
htpasswd -m /home/gerrit/gerrit_site/etc/gerrit.password user1     # 在gerrit.password增加用户用 -m
htpasswd -b /home/gerrit/gerrit_site/etc/gerrit.password gerrit  gerrit
```

### 9、方式一：安装apache2

#### 1.安装

```bash
apt-get install apache2 -y
```

#### 2.配置

##### 1)httpd.conf

```bash
vim /etc/apache2/httpd.conf
<VirtualHost *>
    ServerName localhost

    ProxyRequests Off
    ProxyVia Off
    ProxyPreserveHost On

    <Proxy *>
          Order deny,allow
          Allow from all
    </Proxy>

    <Location "/login/">
          AuthType Basic
          AuthName "Gerrit Code Review"
          AuthBasicProvider file
          Require valid-user
          AuthUserFile /home/gerrit/gerrit_site/etc/gerrit.password
    </Location>

    AllowEncodedSlashes On
    ProxyPass / http://127.0.0.1:8081/ nocanon
</VirtualHost>
```

##### 2)ports.conf

```bash
vim ports.conf
...
Listen 8080
...
```

##### 3)修改主配置文件生效

```bash
vim apache2.conf
...
Include ports.conf
Include httpd.conf
...
```

#### 3.开启SSL、Proxy、Rewrite等模块

```bash
cd /etc/apache2/mods-enabled/
sudo ln -s ../mods-available/proxy.load
sudo ln -s ../mods-available/proxy.conf
sudo ln -s ../mods-available/proxy_http.load
sudo ln -s ../mods-available/proxy_balancer.conf
sudo ln -s ../mods-available/proxy_balancer.load
sudo ln -s ../mods-available/rewrite.load
sudo ln -s ../mods-available/ssl.conf
sudo ln -s ../mods-available/ssl.load
sudo ln -s ../mods-available/slotmem_shm.load
sudo ln -s ../mods-available/socache_shmcb.load
```

### 10、方式二：安装Nginx反向代理

#### 1.安装

```bash
apt-get install nginx
```

#### 2.配置

>/etc/nginx/conf.d/gerrit.conf

```yaml
server {
    listen 8080;
    server_name localhost;
    allow all;
    deny all;

    auth_basic "Welcome to Gerrit Code Review Site!";
    auth_basic_user_file /home/gerrit/gerrit_site/etc/gerrit.password;
    location / {
        proxy_pass  http://127.0.0.1:8081;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host;
    }
}
```

### 11、gerrit加入system管理

>vim /lib/systemd/system/gerrit.service

```bash
[Unit]
Description=Gerrit Web System.
After=network.target

[Service]
Type=forking
User=gerrit
EnvironmentFile=/home/gerrit/gerrit_site/etc/gerrit.config
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=gerrit
ExecStart=/home/gerrit/gerrit_site/bin/gerrit.sh start
ExecStop=/home/gerrit/gerrit_site/bin/gerrit.sh stop
PIDFile=/home/gerrit/gerrit_site/logs/gerrit.pid

[Install]
WantedBy=multi-user.target
```

### 11、启动并开机自启apache

```bash
systemctl start apache2.service
systemctl enable apache2.service
```

### 12、启动并开机自启gerrit

```bash
systemctl start gerrit.service
systemctl enable gerrit.service
```

### 13、访问

>http://172.16.3.50:8080/

# k8s-statefulset部署

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: gerrit
spec:
  serviceName: gerrit
  replicas: 1
  selector:
    matchLabels:
      app: gerrit
  template:
    metadata:
      labels:
        app: gerrit
    spec:
      securityContext:
        runAsUser: 1000       # Gerrit 用户 UID
        runAsGroup: 1000      # Gerrit 用户 GID
        fsGroup: 1000
      containers:
        - name: gerrit
          image: gerritcodereview/gerrit:3.12.1
          ports:
            - containerPort: 29418
            - containerPort: 8080
          env:
            - name: CANONICAL_WEB_URL
              value: "http://gerrit-devops-k8s-local.gmbaifa.online"
            - name: TZ
              value: Asia/Shanghai
          volumeMounts:
            - name: etc
              mountPath: /var/gerrit/etc
            - name: git
              mountPath: /var/gerrit/git
            - name: db
              mountPath: /var/gerrit/db
            - name: index
              mountPath: /var/gerrit/index
            - name: cache
              mountPath: /var/gerrit/cache
  volumeClaimTemplates:
    - metadata:
        name: etc
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 10Gi
        storageClassName: rook-cephfs
    - metadata:
        name: git
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 100Gi
        storageClassName: rook-cephfs
    - metadata:
        name: db
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 100Gi
        storageClassName: rook-cephfs
    - metadata:
        name: index
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 100Gi
        storageClassName: rook-cephfs
    - metadata:
        name: cache
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 100Gi
        storageClassName: rook-cephfs
```





