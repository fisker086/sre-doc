## 一、keepalived 脑裂

```bash
由于某些原因，导致两台keepalived高可用服务器在指定时间内，无法检测到对方的心跳，各自取得资源及服务的所有权，而此时的两台高可用服务器又都还活着。
```

### 1.脑裂的故障

```bash
1.网线松动，网络故障
2.服务器硬件故障
3.服务器之间开启了防火墙
```

### 2.脑裂模拟

#### 1）开启防火墙

```bash
[root@lb01 ~]# systemctl start firewalld.service 
[root@lb01 ~]# ip addr | grep 10.0.0.3
    inet 10.0.0.3/32 scope global eth0
    
[root@lb02 ~]# systemctl start firewalld
[root@lb02 ~]# ip addr | grep 10.0.0.3
    inet 10.0.0.3/32 scope global eth0
```

#### 2）访问网站

```bash
#因为开启了firewalld防火墙，默认拒绝所有连接，要开启80端口
[root@lb01 ~]# firewall-cmd --add-service=http
success
[root@lb02 ~]# firewall-cmd --add-service=http
success

[root@lb01 ~]# firewall-cmd --add-service=https
success
[root@lb02 ~]# firewall-cmd --add-service=https
success

#访问页面没有任何问题
```

#### 3）关闭防火墙

```bash
[root@lb02 ~]# systemctl stop firewalld.service 
[root@lb02 ~]# ip addr | grep 10.0.0.3

[root@lb01 ~]# systemctl stop firewalld.service
[root@lb01 ~]# ip addr | grep 10.0.0.3
    inet 10.0.0.3/32 scope global eth0
```



### 3.脑裂解决的办法

```bash
#如果发生闹裂，则随机kill掉一台即可
#在备上编写检测脚本, 测试如果能ping通主并且备节点还有VIP的话则认为产生了脑裂
[root@lb02 ~]# cat check_split_brain.sh
#!/bin/sh
vip=10.0.0.3
lb01_ip=10.0.0.4
while true;do
    ping -c 2 $lb01_ip &>/dev/null
    if [ $? -eq 0 -a `ip add|grep "$vip"|wc -l` -eq 1 ];then
        echo "ha is split brain.warning."
    else
        echo "ha is ok"
    fi
sleep 5
done


[root@lb02 ~]# vim check_keepalive.sh 
#!/bin/sh
vip=10.0.0.3
lb01_ip=172.16.1.4
while true;do
    ssh $lb01_ip 'ip addr | grep 10.0.0.3' &>/dev/null
    if [ $? -eq 0 -a `ip add|grep "$vip"|wc -l` -eq 1 ];then
        echo "ha is split brain.warning."
    else
        echo "ha is ok"
    fi
sleep 3
done
```



## 二、高可用keepalived与nginx

```bash
Nginx默认监听在所有的IP地址上，VIP会飘到一台节点上，相当于那台nginx多了VIP这么一个网卡，所以可以访问到nginx所在机器

但是.....如果nginx宕机，会导致用户请求失败，但是keepalived没有挂掉不会进行切换，所以需要编写一个脚本检测Nginx的存活状态，如果不存活则kill掉keepalived
```

### 1.nginx故障切换脚本

```bash
[root@lb01 ~]# vim check_web.sh
#!/bin/sh
nginxpid=$(ps -ef | grep [n]ginx | wc -l)

#1.判断Nginx是否存活,如果不存活则尝试启动Nginx
if [ $nginxpid -eq 0 ];then
    systemctl start nginx &>/dev/null
    sleep 3
    #2.等待3秒后再次获取一次Nginx状态
    nginxpid=$(ps -ef | grep [n]ginx | wc -l) 
    #3.再次进行判断, 如Nginx还不存活则停止Keepalived,让地址进行漂移,并退出脚本  
    if [ $nginxpid -eq 0 ];then
        systemctl stop keepalived
    fi
fi

#给脚本增加执行权限
[root@lb01 ~]# chmod +x /root/check_web.sh
```

### 2.使用keepalived配置文件调用nginx切换脚本

#### 1）配置抢占式时

```bash
#只需要在备节点配置
[root@lb02 ~]# vim /etc/keepalived/keepalived.conf 
global_defs {
    router_id lb02
}

#每5秒执行一次脚本，脚本执行内容不能超过5秒，否则会中断再次重新执行脚本
vrrp_script check_web {
    script "/root/check_web.sh"
    interval 5
}

vrrp_instance VI_1 {
    state BACKUP
    nopreempt
    interface eth0
    virtual_router_id 50
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.0.0.3
    }
    #调用计划的脚本
    track_script {
        check_web
    }
}
```

#### 2）配置非抢占式时

```bash
#配置非抢占式时，两边都要配置脚本
[root@lb02 ~]# scp check_web.sh 172.16.1.4:/root

#主节点也要配置
[root@lb01 ~]# cat /etc/keepalived/keepalived.conf 
global_defs {
    router_id lb01
}

vrrp_script check_web {
    script "/root/check_web.sh"
    interval 5
}

vrrp_instance VI_1 {
    state BACKUP
    nopreempt
    interface eth0
    virtual_router_id 50
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.0.0.3
    }
    track_script {
        check_web
    }
}
```

### 3.测试

```bash
1.将VIP所在机器nginx的配置文件修改错误
2.停止nginx
3.查看VIP是否切换
```



# Nginx常见问题

## 一、nginx多server优先级

```bash
在开始处理一个http请求时，nginx会取出header头中的Host变量，与nginx.conf中的每个server_name进行匹配，以此决定到底由哪一个server来处理这个请求，但nginx如何配置多个相同的server_name，会导致server_name出现优先级访问冲突。
```

### 1.准备多个配置文件

```bash
[root@web01 conf.d]#  cat server1.conf 
server {
    listen 80;
    server_name localhost test1.com;

    location / {
        root /code/test1;
        index index.html;
    }
}

[root@web01 conf.d]# cat server2.conf 
server {
    listen 80;
    server_name localhost test2.com;

    location / {
        root /code/test2;
        index index.html;
    }
}

[root@web01 conf.d]# cat server3.conf 
server {
    listen 80;
    server_name localhost test3.com;

    location / {
        root /code/test3;
        index index.html;
    }
}
```

### 2.创建站点文件

```bash
[root@web01 conf.d]# mkdir /code/test{1..3}
[root@web01 conf.d]# chown -R www.www /code/test*

[root@web01 conf.d]# echo server1 > /code/test1/index.html
[root@web01 conf.d]# echo server2 > /code/test2/index.html
[root@web01 conf.d]# echo server3 > /code/test3/index.html
```

### 3.访问测试

```bash
1.重启，访问IP
[root@web01 conf.d]# systemctl restart nginx

#根据ip访问
	1）用户第一次访问，读取server1.conf配置返回结果
    [root@web01 ~]# curl 10.0.0.7
    test1

    2）此时将server1.conf修改为server4.conf重启nginx
    [root@web01 conf.d]# mv server1.conf server4.conf
    [root@web01 conf.d]# systemctl restart nginx

    3）再次访问时，读取server2.conf配置返回结果
    [root@web01 conf.d]# curl 10.0.0.7
    test2

2.配置hosts，访问域名
10.0.0.7 test1.com test2.com test3.com
```

### 4.多server优先级总结

```bash
1.首先选择所有的字符串完全匹配的server_name。（完全匹配  www.mumusir.com）
2.选择通配符在前面的server_name，如 *.mumusir.com
3.选择通配符在后面的server_name，如 www.mumusir.*
4.最后选择使用正则表达式匹配的server_name，如：~^www\.(.*)\.com$
5.如果全部都没有匹配到，那么将选择在listen配置项后加入[default_server]的server块
6.如果没写，那么就找到匹配listen端口的第一个Server块的配置文件
```

### 5.多server优先级总结验证

#### 1）配置完全匹配的配置文件

```bash
[root@web01 conf.d]# vim server.conf
server {
    listen 80;
    server_name www.test.com;

    location / {
        root /code/test;
        index index.html;
    }
}
[root@web01 conf.d]# echo "完全匹配" > /code/test/index.html
```

#### 2）配置通配符在前面的配置文件

```bash
[root@web01 conf.d]# vim server1.conf
server {
    listen 80;
    server_name *.test.com;

    location / {
        root /code/test1;
        index index.html;
    }
}
[root@web01 conf.d]# echo "通配符在前面" > /code/test1/index.html
```

#### 3）配置通配符在后面的配置文件

```bash
[root@web01 conf.d]# vim server2.conf
server {
    listen 80;
    server_name www.test.*;

    location / {
        root /code/test2;
        index index.html;
    }
}
[root@web01 conf.d]# echo "通配符在后面" > /code/test2/index.html
```

#### 4）正则表达式的配置文件

```bash
[root@web01 conf.d]# vim server3.conf 
server {
    listen 80;
    server_name ~^www\.(.*)\.com$;

    location / {
        root /code/test3;
        index index.html;
    }
}
[root@web01 conf.d]# echo "正则表达式" > /code/test3/index.html
```

#### 5）配置default_server的配置文件

```bash
[root@web01 conf.d]# vim server4.conf 
server {
    listen 80 default_server;
    server_name localhost;

    location / {
        root /code/test4;
        index index.html;
    }
}
[root@web01 conf.d]# echo "default_server" > /code/test4/index.html
```

#### 6）放在第一个的配置文件

```bash
[root@web01 conf.d]# vim a.conf 
server {
    listen 80;
    server_name localhost;

    location / {
        root /code/test5;
        index index.html;
    }
}
[root@web01 conf.d]# echo "第一个" > /code/test5/index.html
```

#### 7）重启nginx测试

```bash
#配置hosts
10.0.0.7 www.test.com
```



## 二、nginx禁止IP访问

```bash
当用户通过访问IP或者未知域名访问你得网站的时候，你希望禁止显示任何有效内容，可以给他返回500，目前国内很多机房都要求网站关闭空主机头，防止未备案的域名指向过来造成麻烦
```

### 1.禁止IP访问直接返回错误

```bash
[root@web01 conf.d]# vim a.conf 
server {
    listen 80 default_server;
    server_name localhost;
    return 500;
}
```

### 2.引流的方式，跳转到其他网站

```bash
[root@web01 conf.d]# vim a.conf 
server {
    listen 80 default_server;
    server_name localhost;
    return 302 http://www.baidu.com;
}
```

### 3.返回指定的内容

```bash
[root@web01 conf.d]# vim a.conf 
server {
    listen 80 default_server;
    server_name localhost;
    default_type text/plain;
    return 200 "请使用域名访问正规网站！！！";
}
```

### 4.跳转到指定文件

```bash
[root@web01 ~]# vim /etc/nginx/conf.d/a.conf 
server {
    listen 80 default_server;
    server_name localhost;
    root /code;
    rewrite (.*) /1.jpg;
}
```



## 三、nginx的include

```bash
一台服务器配置多个网站，如果配置都写在nginx.conf主配置文件中，会导致nginx.conf主配置文件变得非常庞大而且可读性非常的差。那么后期的维护就变得麻烦。 

假设现在希望快速的关闭一个站点，该怎么办？ 
	1.如果是写在nginx.conf中，则需要手动注释，比较麻烦 
	2.如果是include的方式，那么仅需修改配置文件的扩展名，即可完成注释 
	Include包含的作用是为了简化主配置文件，便于人类可读。
	
inlcude /etc/nginx/online/*.conf 	#线上使用的配置

/etc/nginx/offline 					#保留配置，不启用（下次使用在移动到online中）
```



## 四、nginx的root与alias

```bash
root与alias路径匹配主要区别在于nginx如何解释location后面的uri，这会使两者分别以不同的方式将请求映射到服务器文件上，alias是一个目录别名的定义，root则是最上层目录的定义。

root的处理结果是：root路径＋location路径
alias的处理结果是：使用alias定义的路径
```

### 1.root和alias的配置

```bash
[root@lb01 conf.d]# cat image.conf 
server {
    listen 80;
    server_name image.com;

    location /picture {
        root /code;
    }
}
#使用root时，用户访问http://image.com/picture/1.jpg时，实际上Nginx会找到/code/picture/1.jpg文件

[root@lb01 conf.d]# cat image.conf 
server {
    listen 80;
    server_name image.com;

    location /picture {
        alias /code;
    }
}
#使用alias时，用户访问http://image.com/picture/1.jpg时，实际上Nginx会找到/code/1.jpg文件
```

### 2.线上配置

```bash
server {
    listen 80;
    server_name image.com;

    location / {
        root /code;
    }

    location ~* ^.*\.(png|jpg|gif)$ {
        alias /code/images/;
    }
}
```



## 五、Nginx调整上传文件大小

```bash
在nginx使用上传文件的过程中，通常需要设置文件大小限制，避免出现413 Request Entity Too Large
```

### 1.nginx上传文件大小限制配置语法

```bash
Syntax:  client_max_body_size size;
Default: client_max_body_size 1m;
Context: http, server, location
```

### 2.nginx长传文件大小限制配置示例

```bash
#也可以放入http层，全局生效
server {
    listen 80;
    server_name _;
    client_max_body_size 200m;
}
```













