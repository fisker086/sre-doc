# gerrit升级及插件安装

## 一、3.2升级3.5

### 1、升级前确认下前提

#### 1.jdk需要11版本

```bash
#清除原有jdk环境
apt-get remove --purge jdk* -y

#openjdk升级为11
apt-get install openjdk-11-jdk -y

root@gerrit:~# java -version
openjdk version "11.0.14.1" 2022-02-08
OpenJDK Runtime Environment (build 11.0.14.1+1-Ubuntu-0ubuntu1.18.04)
OpenJDK 64-Bit Server VM (build 11.0.14.1+1-Ubuntu-0ubuntu1.18.04, mixed mode, sharing)
```

#### 2.确认本次版本升级不需要reindex

### 2、下载安装包

```bash
cd /home/gerrit2/wars
wget https://gerrit-releases.storage.googleapis.com/gerrit-3.5.1.war
```



### 3、升级

#### 1.先停止gerrit

```bash
 systemctl stop gerrit.service 
 
 #或者
 bash /home/gerrit2/gerrit_site/bin/gerrit.sh stop
```

#### 2.修改配置文件

```bash
vim /home/gerrit2/gerrit_site/etc/gerrit.config
...
javaHome = /usr/lib/jvm/java-11-openjdk-amd64/
...
```

#### 3.升级

```bash
java -jar gerrit-3.5.1.war init -d /home/gerrit2/gerrit_site

#如果需要reindex
java -jar gerrit-3.5.1.war reindex -d /home/gerrit2/gerrit_site
```

#### 4.重新授权gerrit目录

```bash
 chown -R gerrit2.gerrit2 gerrit_site
 
 
  chown -R gerrit2.adm /home/ggerrit_site
```

#### 5.启动

```bash
 systemctl start gerrit.service 
 
 #或者
 bash /home/gerrit2/gerrit_site/bin/gerrit.sh start
```

## 二、安装常用插件

### 1、下载需要的插件

```bash
cd /home/gerrit2/gerrit_site/plugins
wget https://gerrit-ci.gerritforge.com/view/Plugins-stable-3.5/job/plugin-rename-project-bazel-master-stable-3.5/lastSuccessfulBuild/artifact/bazel-bin/plugins/rename-project/rename-project.jar

#重启gerrit即可


wget https://gerrit-ci.gerritforge.com/view/Plugins-stable-3.5/job/plugin-rename-project-bazel-master-stable-3.5/lastSuccessfulBuild/artifact/bazel-bin/plugins/rename-project/rename-project.jar
```





## 三、遇到的问题

### 1、reindex

#### 1.日志内容

```bash
cdcfcfbe-2044-3b1f-b5d4-da77ad542329

173816e5-2b9a-37c3-8a2e-48639d4f1153
```





#### 2.解决

·
