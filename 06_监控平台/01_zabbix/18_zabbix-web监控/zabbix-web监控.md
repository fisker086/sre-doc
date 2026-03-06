# web网站可用性检测

## 一、使用curl命令，命令行模拟登陆zabbix

### 1、获取资源

#### 1）url

![1](1.png)

```bash
ajax：异步传输 前端技术
	登陆 --》 页面没有跳转。注册账号提示账号已存在
zabbix：
	登陆 --》页面跳转

url：http://10.0.0.71/zabbix/index.php
```



#### 2）提交的数据

![2](2.png)

```bash
name=Admin&password=123&enter=Sign+in
```



### 2、获取cookie

```bash
[root@web01 /opt]# curl -X GET -c cookie -b cookie http://10.0.0.71/zabbix/index.php


[root@web01 /opt]# ll
total 4
-rw-r--r-- 1 root root 214 Apr  4 17:41 cookie
```





### 3、请求

```bash
[root@web01 /opt]# curl -X POST -L -c cookie -b cookie -d "name=Admin&password=zabbix&enter=Sign+in" http://10.0.0.71/zabbix/index.php

[root@web01 /opt]# curl -X GET -L -c cookie -b cookie "http://10.0.0.71/zabbix/hosts.php?ddreset=1"

```



## 二、zabbix web检测

### 1、配置

![3](3.png)

![4](4.png)

![5](5.png)

![6](6.png)

![7](7.png)

![8](8.png)

![9](9.png)

![10](10.png)

![10](10.png)

**查看数据**

![11](11.png)





### 2、创建触发器

#### 1)页面状态码监控

![12](12.png)

![13](13.png)

![14](14.png)

![15](15.png)

**添加**

![16](16.png)

**测试：修改页面文件权限**







#### 2）页面加载速度

![17](17.png)

![18](18.png)

![19](19.png)

![20](20.png)

**测试 ab压测**