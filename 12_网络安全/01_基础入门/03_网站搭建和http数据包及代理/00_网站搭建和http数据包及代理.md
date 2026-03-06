# 网站搭建和http数据包及代理

## 一、网站基础了解

### 1、域名

https://blog.csdn.net/m0_46881353/article/details/121328457

### 2、DNS

>DNS（Domain Name System）即域名系统，是一种将域名转换为对应IP地址的服务，是完成互联网中名称解析的基础设施之一。它采用c/s的结构，客户机是用户用于查找一个名字对应的地址，而服务器通常用于为别人提供查询服务。这种系统的主要优点在于，用户访问网站时不需要记住复杂的IP地址，只需要记住简单的域名即可。

### 3、http

https://dudewu.top/website/#sort=./02_%E5%88%86%E5%B8%83%E5%BC%8F%E6%9E%B6%E6%9E%84&doc=06_nginx/06_https/HTTPS.md

### 4、https/CA

https://dudewu.top/website/#sort=./02_%E5%88%86%E5%B8%83%E5%BC%8F%E6%9E%B6%E6%9E%84&doc=05_http%E5%8D%8F%E8%AE%AE/0-HTTP%E5%8D%8F%E8%AE%AE.md

### 5、CDN

>CDN（Content Delivery Network）即内容分发网络，是一种利用节点服务器和高速缓存技术，将源站的内容发布到全球各地的边缘节点上，并通过智能路由算法，使用户就近获取所需内容的网络加速服务。这种技术可以显著提高用户访问网站的响应速度，并降低网站的负载压力。CDN服务并不具备DNS解析功能，而是基于DNS智能解析功能，由DNS根据用户所在地、所用线路进行智能分配最合适的CDN服务节点，然后把缓存在该服务节点的静态缓存内容返回给用户。

## 二、web应用开发组成角色

### 1、开发语言

>asp,php,aspx,jsp,java,python,ruby,go,html,javascript 等

### 2、程序源码

>根据开发语言分类；应用类型分类；开源 CMS 分类；开发框架分类等

### 3、中间件容器

>IIS,Apache,Nginx,Tomcat,Weblogic,Jboos,glasshfish 等

### 4、数据库

>Access,Mysql,Mssql,Oracle,db2,Sybase,Redis,MongoDB 等

### 5、服务器操作系统

>Windows 系列，Linux 系列，Mac 系列等

### 6、第三方软件

>phpmyadmin,vs-ftpd,VNC,ELK,Openssh 等

## 三、web应用安全漏洞分类

>SQL 注入，文件安全，RCE 执行，XSS 跨站，CSRF/SSRF/CRLF，反序列化，逻辑越权，未授权访问，XXE/XML，弱口令安全等

## 四、web请求返回工作过程了解

>https://www.jianshu.com/p/558455228c43
>
>https://www.cnblogs.com/cherrycui/p/10815465.html

## 五、JDK切换工具

>电脑安装了jdk21 但是Burp Suite貌似不支持，所以需要安装低版本jdk

>https://github.com/ystyle/jvms

>https://blog.csdn.net/m0_64583630/article/details/132018641

## 六、Burpsuite汉化版详细安装教程及中文安装包

>https://blog.csdn.net/qq_34780861/article/details/128259166
