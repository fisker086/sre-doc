# gitlab基础使用

[TOC]



## 一、组件介绍

```bash
1、nginx: 静态web服务器在这里插入代码片

2、gitlab-shell: 用于处理Git命令和修改authorized keys列表
3、gitlab-workhorse: 轻量级的反向代理服务器，可以处理一些大的HTTP请求（磁盘上的 CSS、JS 文件、文件上传下载等），处理 Git Push/Pull 请求，处理到Rails 的连接会反向代理给后端的unicorn（修改由 Rails 发送的响应或发送给 Rails 的请求，管理 Rails 的长期 WebSocket 连接等）。

4、logrotate：日志文件管理工具

5、postgresql：repository 中的数据（元数据，issue，合并请求 merge request 等 , 可以登录 Web 的用户

6、redis：缓存每个客户端的sessions和后台队列，负责分发任务。Redis需求的存储空间很小，大约每个用户25KB

7、sidekiq：用于在后台执行队列任务（异步执行）

8、unicorn：Gitlab 自身的 Web 服务器(Ruby Web Server)，包含了 Gitlab 主进程，负责处理快速/一般任务，与 Redis 一起工作，配置参考：CPU核心数 + 1 = unicorn workers数量。
    1. 通过检查存储在 Redis 中的用户会话来检查权限
    2. 为 Sidekiq 制作任务
    3. 从仓库（warehouse）取东西或在那里移动东西

9、gitlab-shell：用于 SSH 交互，而不是 HTTP。gitlab-shell 通过 Redis 与 Sidekiq 进行通信，并直接或通过 TCP 间接访问 Unicorn。用于处理Git命令和修改authorized keys列表

10、Gitaly：后台服务，专门负责访问磁盘以高效处理 gitlab-shell 和 gitlab-workhorse 的git 操作，并缓存耗时操作。所有的 git 操作都通过 Gitaly 处理，并向 GitLab web 应用程序提供一个 API，以从 git（例如 title, branches, tags, other meta data）获取属性，并获取 blob（例如 diffs，commits，files）

11、Sidekiq：后台核心服务，可以从redis队列中提取作业并对其进行处理。后台作业允许GitLab通过将工作移至后台来提供更快的请求/响应周期。Sidekiq任务需要来自Redis

12、prometheus:提供监控
```



## 二、常用命令

### 1、服务控制命令

#### 1）启动/停止/重启所有 gitlab 组件：

```bash
gitlab-ctl start/stop/restart
```



#### 2）启动指定模块组件：

```bash
 gitlab-ctl start redis/postgresql/gitlab-workhorse/logrotate/nginx/sidekiq/unicorn
```



#### 3）停止指定模块组件：

```bash
gitlab-ctl stop 模块名
```



#### 4）查看服务状态

```bash
gitlab-ctl status
```



#### 5）生成配置并启动服务

```bash
 gitlab-ctl reconfigure
```



### 2、其他常用管理命令

#### 1）查看版本

```bash
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION 
```



#### 2）检查gitlab

```bash
gitlab-rake gitlab:check SANITIZE=true --trace
```



#### 3）实时查看日志

```bash
gitlab-ctl tail
```



#### 4）关系数据库升级

```bash
gitlab-rake db:migrate
```



#### 5）清理redis缓存

```bash
gitlab-rake cache:clear
```



#### 6）升级gitlab-ce版本

```bash
# 关闭gitlab服务
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
gitlab-ctl stop nginx
# 备份gitlab
gitlab-rake gitlab:backup:create
# 升级rpm包
rpm -Uvh gitlab-ce-xxx.rpm
# 启动并查看gitlab版本信息
gitlab-ctl reconfigure
gitlab-ctl restart
head -1 /opt/gitlab/version-manifest.txt

# 常见报错
Error executing action `run` on resource 'ruby_block[directory resource: /var/opt/gitlab/git-data/repositories]'
# 解决方法:
sudo chmod 2770 /var/opt/gitlab/git-data/repositories
```



#### 7）升级postgreSQL最新版本

```bash
gitlab-ctl pg-upgrade
```



### 3、日志

#### 1）实时查看所有日志

```bash
gitlab-ctl tail
```



#### 2）实时查看某个组件日志

```bash
gitlab-ctl tail [组件名称]
```



### 4、常用配置文件目录

```ba
主配置文件: /etc/gitlab/gitlab.rb
文档根目录: /opt/gitlab
默认存储库位置: /var/opt/gitlab/git-data/repositories
Nginx配置文件: /var/opt/gitlab/nginx/conf/gitlab-http.conf
Postgresql数据目录: /var/opt/gitlab/postgresql/data
```



### 5、gitlab管理员密码重置

#### 1.查看默认root密码

```bash
root@lb-kubeasz:~# docker exec -it gitlab bash
root@gitlab:/# cat /etc/gitlab/initial_root_password 
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: xxxxxxxxxxxxxxxxxxxxx

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
```

#### 2.修改密码

```bash
[root@test bin]# gitlab-rails console production
-------------------------------------------------------------------------------------
 GitLab:       11.10.4 (62c464651d2)
 GitLab Shell: 9.0.0
 PostgreSQL:   9.6.11
-------------------------------------------------------------------------------------
Loading production environment (Rails 5.0.7.2)
irb(main):001:0> user = User.where(id:1).first
=> #<User id:1 @root>
irb(main):002:0> user.password = 'qwer1234'
=> "qwer1234"
irb(main):003:0> user.password_confirmation = 'qwer1234'
=> "qwer1234"
irb(main):004:0> user.save
Enqueued ActionMailer::DeliveryJob (Job ID: 4752a4a4-4e85-4e8b-9f27-72788abfe97c) to Sidekiq(mailers) with arguments: "DeviseMailer", "password_change", "deliver_now", #<GlobalID:0x00007f519e7501d8 @uri=#<URI::GID gid://gitlab/User/1>>
=> true
irb(main):005:0> exit
```



### 6、使用smtp来发送邮件通知

```bash
vim /etc/gitlab/gitlab.rb

 gitlab_rails['smtp_address'] = "smtp.yourdomain.com"
 gitlab_rails['smtp_port'] = 25
 gitlab_rails['smtp_user_name'] = "xxx"
 gitlab_rails['smtp_password'] = "xxx"
 gitlab_rails['smtp_domain'] = "smtp.yourdomain.com" 
 gitlab_rails['smtp_authentication'] = 'plain'
 gitlab_rails['smtp_enable_starttls_auto'] = true
```



### 7、配置gitlab访问模式为https

```bash
# 创建ssl证书存放目录
    mkdir -p /etc/gitlab/ssl
    chmod 0700 /etc/gitlab/ssl

# 上传证书，修改证书访问权限
    chmod 600 /etc/gitlab/ssl/gitlab.xxx.com.crt

# 修改住配置，支持ssl访问
    vim /etc/gitlab/gitlab.rb
    external_url "[https://gitlab.bjwf125.com]            (https://gitlab.bjwf125.com)"
    nginx['redirect_http_to_https'] = true
    nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.xxx.com.crt"
    nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.xxx.com.key"

# 重启
    gitlab-ctl reconfigure

# 开启防火墙

　　firewall-cmd --zone=public --add-port=443/tcp --permanent
　　firewal-cmd reload
```



### 8、Gitlab备份

```bash
gitlab备份的默认目录是/var/opt/gitlab/backups
```



#### 1）备份命令

```bash
gitlab-rake gitlab:backup:create
#该命令会在备份目录（默认：/var/opt/gitlab/backups/）下创建一个tar压缩包xxxxxxxx_gitlab_backup.tar，其中开头的xxxxxx是备份创建的时间戳，这个压缩包包括GitLab整个的完整部分。
```



#### 2）自动备份

```bash
#通过任务计划crontab 实现自动备份
0 2 * * * /usr/bin/gitlab-rake gitlab:backup:create 
# 每天两点备份gitlab数据
```



#### 3）备份配置文件

```bash
vim /etc/gitlab/gitlab.rb
gitlab_rails['manage_backup_path'] = true
gitlab_rails['backup_path'] = "/data/gitlab/backups"    #指定gitlab备份目录
gitlab_rails['backup_archive_permissions'] = 0644       #生成的备份文件权限
gitlab_rails['backup_keep_time'] = 7776000              #备份保留天数为3个月（即90天，这里是7776000秒）

#重载生效
gitlab-ctl reconfigure
```



### 9、备份恢复

```bash
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
gitlab-rake gitbab:backup:restore BACKUP=xxxxxx(恢复文件)
gitlab-ctl start 启动gitlab
```



