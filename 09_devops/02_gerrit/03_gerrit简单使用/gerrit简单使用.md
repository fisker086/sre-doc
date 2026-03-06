# gerrit简单使用

## 一、Gerrit 引用说明
|引用|说明|
| ---------- | ---------- |
|refs/heads/*|对应git的refs/heads/* 意味这可以直接改变指针引用跳过所有的code reviewer过程，用于网页操作以及后台直接git命令操作|
|refs/tags/*|tag权限|
|refs/meta/config|ACL的存储位置|
|refs/meta/dashboards/*|dashboards相关，jenkins|
|refs/notes/review|基本不需要|
|refs/for/<branch ref>|push必须的操作|
|refs/publish/*|= refs/for/*|
|refs/drafts/*|push后不是所有人可见，通过publish转换为可见，用于自己调试|
|refs/changes/*|通过push commit到refs/for/<branch-name>，意味着自动创建一个refs/changes/<last two digits of change number>/<change number>/<patch set number>的引用。在submit后将push到refs/heads/*的namespace中|
|refs/for/refs/heads/*|一般只需要开通push和push merged commit即可|

## 二、Gerrit ACL

|权限|说明|refs/head/*|refs/for/refs/head/*|
| ---- | ---- | ---- | ---- |
|Owner|仓库拥有者，可以更改仓内所有权限|Leader|Leader|
|Read|下载代码的权限，以及仓库可见性|Developer|
|Abandon|废弃提交（change）|Approver|Approver|
|Create Reference|创建引用（分支、标签）|Leader|Leader|
|Forge Author|push时不检查作者是否一致|Approver|Approver|
|Forge Committer|push时不检查提交者是否一致|Approver|Approver|
|Forge Server|push含有由gerrit生成的merge提交时不检查上述信息|Leader|Leader|
|Push|提交权限，拥有Create Reference的人可以创建分支，勾选Force option可以删除分支|Leader|Leader|
|Push Merge Commits|是否可以push带有merge的commit|Leader|Approver|
|Push Annotated Tag|提交带描述的标签|Leader|Approver|
|Push Signed Tag|提交签名的标签|Leader|Approver|
|Rebase|在页面上进行rebase操作|submitter和owner默认就有|
|Remove Reviewer|移除审核人员，一般不太用|Approver|
|Label Verified|审核权限|Developer|
|Label Code-Review|审核权限|Approver|
|Submit|入库权限|Approver|

## 三、Gerrit用户管理

### 1. 创建密码存储文件

    sudo htpasswd -c ~/gerrit_site/etc/passwords

### 2. 增加新用户

    sudo htpasswd -b ~/gerrit_site/etc/passwords username password

### 3. 删除指定用户

    sudo htpasswd -D ~/gerrit_site/etc/passwords username

## 四、Gerrit基本操作

### 1. 测试ssh密钥是否生效

    ssh -p 29418 username@172.16.0.34

### 2. <del>通过命令创建仓库</del>（尽可能在网页上创建）

    ssh -p 29418 username@gerrit-ip-address gerrit create-project --name project_name

### 3. <del>强制推送git仓库到Gerrit服务器</del>（使用Import插件后，尽量不使用这种方式）

    git push ssh://username@localhost:29418/demo-project *:*

### 4. 强制删除远端标签

    git push origin --delete tag 删除远端标签

### 5. repo替换规则

    git config --global url."ssh://mrobot@172.16.0.34".insteadOf "ssh://username@172.16.0.34"



## 五、同步远端Git仓库

### 1. 在git仓库中克隆镜像

    git clone --mirror https://github.xxx.git

### 2. 若无法在页面上显示项目

命令行为远端项目增加父工程，从而继承可见权限

    ssh -p 29418 gerrit2@localhost gerrit set-project-parent --parent Share-Projects x86/external/rgbd_launch


## 六、重命名仓库

$ ssh -p @SSH_PORT@ @SSH_HOST@ @PLUGIN@ project-1 project-2

## 七、下载代码

$ repo init -u ssh://jenkins@172.16.0.34:29418/project/manifests.git -b release -m mty_catkin_ws.xml --no-repo-verify

## 八、有用的链接

[谷歌官方Gerrit插件CI服务器](https://gerrit-ci.gerritforge.com/)  
[谷歌官方Gerrit服务器war包下载地址](https://gerrit-ci.gerritforge.com/gerrit-2.10.war)  
[谷歌官方Gerrit主站](https://www.gerritcodereview.com/)