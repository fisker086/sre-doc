
> 运维、云原生、开发与 AI 学习实践笔记目录（自动同步整理）。

## 目录导航（按编号顺序）

1. `01_linux基础`
2. `02_分布式架构`
3. `03_数据库`
4. `04_容器`
5. `05_容器编排工具`
6. `06_监控平台`
7. `07_日志收集平台`
8. `08_消息队列`
9. `09_devops`
10. `10_云原生`
11. `11_开发学习`
12. `12_网络安全`
13. `13_winserver`
14. `14_IT类`
15. `15_存储`
16. `16_AI`

## Repository Tree

<details open>
<summary>点击折叠/展开完整目录结构</summary>

```text
├── 01_linux基础
│   ├── 01_第一部分_linux的由来及发展
│   │   ├── 01_运维介绍
│   │   │   └── 一章一节_运维介绍.md
│   │   ├── 02_概念引入
│   │   │   └── 一章二节概念引入_.md
│   │   ├── 03_计算机分类及服务器分类
│   │   │   └── 一章三节_计算机分类及服务器分类.md
│   │   ├── 04_计算机和服务器的硬件组成
│   │   │   └── 一章四节_计算机和服务器的硬件组成.md
│   │   └── 05_操作系统的发展史
│   │       └── 一章五节操作系统发展史.md
│   ├── 02_第二部分_linux安装及shell入门
│   │   ├── 06_虚拟机介绍及linux操作系统安装
│   │   │   └── 二章一节虚拟机介绍及linux操作系统安装.md
│   │   ├── 07_shell介绍安装优化及centos系统优化
│   │   │   └── 二章二节shell介绍安装优化及centos系统优化.md
│   │   ├── 08_操作系统启动流程
│   │   │   └── 二章三节_操作系统的启动流程.md
│   │   ├── 09_shell入门
│   │   └── 10_bash特性解释器及使用
│   │       └── 二章五节_bash解释器特性及使用.md
│   └── 03_第三部分_文件管理
│       ├── 11_文件和目录操作入门
│       │   └── 三章一节文件和目录操作入门.md
│       ├── 12_vim编辑器
│       │   └── 三章二节_vim编辑器.md
│       ├── 13_文本处理三剑客入门
│       │   └── 三章三节文本处理三剑客入门.md
│       └── 14_文件查找和上传与下载
│           └── 文件查找及文件上传下载.md
├── 02_分布式架构
│   ├── 01_rsync
│   │   └── 0-Rsync.md
│   ├── 02_nfs
│   │   ├── 0-nfs服务概念.md
│   │   └── 1-NFS服务.md
│   ├── 03_sersync
│   │   └── 0-sersync服务.md
│   ├── 04_sshd
│   │   └── 0-SSH服务.md
│   ├── 05_http协议
│   │   └── 0-HTTP协议.md
│   ├── 06_nginx
│   │   ├── 01_nginx基础
│   │   │   ├── 01_nginx-web入门.md
│   │   │   └── 02_nginx配置文件模块扩展.md
│   │   ├── 02_nginx进阶
│   │   │   ├── 01_lnmp架构.md
│   │   │   ├── 02_架构拆分.md
│   │   │   └── 03_负载均衡与会话保持.md
│   │   ├── 03_代理负载
│   │   │   └── 01_代理负载.md
│   │   ├── 04_四层负载均衡
│   │   │   └── 4-4层负载均衡.md
│   │   ├── 05_动静分离&rewrite
│   │   │   └── 5.rewrite.md
│   │   ├── 06_https
│   │   │   └── HTTPS.md
│   │   └── 07_keepalive
│   │       ├── 7keepalived.md
│   │       └── 8keepalived与nginx问题.md
│   └── 07_防火墙
│       └── 11_防火墙笔记.md
├── 03_数据库
│   ├── 00_数据库介绍
│   │   └── 1.数据库介绍.md
│   ├── 01_MySQL
│   │   ├── 01_MySQL安装前置准备
│   │   │   └── 00_MySQL安装前置准备.md
│   │   ├── 02_MySQL安装
│   │   │   └── 00_MySQL8.0安装.md
│   │   ├── 03_MySQL体系结构管理
│   │   │   └── 00_MySQL体系结构管理.md
│   │   ├── 04_MySQL基础管理
│   │   │   └── 00_MySQL基础管理.md
│   │   ├── 05_MySQLl初始化及多实例
│   │   │   └── 00_MySQL初始化及多实例.md
│   │   ├── 06_mysql-SQL基础
│   │   │   ├── 00_SQL基础介绍.md
│   │   │   └── 01_SQL应用.md
│   │   ├── 07_MySQL-高级开发
│   │   │   └── 00_MySQL-高级开发.md
│   │   ├── 08_元数据
│   │   │   └── 元数据.md
│   │   ├── 09_索引
│   │   │   ├── 01_索引概念
│   │   │   │   └── 01_索引概念.md
│   │   │   ├── 02_索引的管理及执行计划.md
│   │   │   └── 03_索引自由化
│   │   │       └── 03_索引的自由化.md
│   │   ├── 10_MySQL存储引擎
│   │   │   └── 01_介绍
│   │   │       └── 01_存储引擎介绍.md
│   │   ├── 11_事务
│   │   │   ├── 01_事务
│   │   │   │   └── 01_事务介绍.md
│   │   │   ├── 02_隔离级别和锁机制
│   │   │   │   └── 02_隔离级别和锁机制.md
│   │   │   └── 03_存储引擎核心参数
│   │   │       └── 存储引擎核心参数.md
│   │   ├── 12_日志管理
│   │   │   ├── 01_常规日志和错误日志
│   │   │   │   └── 00_常规日志和错误日志.md
│   │   │   ├── 02_bin_log二进制日志
│   │   │   │   └── bin_log二进制日志.md
│   │   │   └── 03_slowlog慢日志
│   │   │       └── slowlog.md
│   │   ├── 13_数据库备份恢复
│   │   │   ├── 01_逻辑备份
│   │   │   │   └── 00_数据库逻辑备份恢复.md
│   │   │   └── 02_物理备份
│   │   │       └── 00_物理备份.md
│   │   ├── 14_主从复制基础（precision号）
│   │   │   ├── 01_位置号搭建
│   │   │   │   └── 主从复制.md
│   │   │   ├── 02_主从复制的原理
│   │   │   │   └── 主从的原理及监控.md
│   │   │   ├── 03_主从复制故障的分析及处理
│   │   │   │   └── 主从复制故障的分析及处理.md
│   │   │   └── 04_主从复制延时的分析及处理
│   │   │       └── 主从复制的分析及处理.md
│   │   ├── 15_延时从库
│   │   │   └── 延时从库.md
│   │   ├── 16_过滤复制
│   │   │   └── 过滤复制.md
│   │   ├── 17_半同步复制
│   │   │   └── 半同步复制.md
│   │   ├── 18_GTID复制
│   │   │   └── GTID复制.md
│   │   ├── 19_主从复制架构演变
│   │   │   └── 主从架构演变.md
│   │   ├── 20_MHA
│   │   │   ├── 01_MHA的介绍和搭建
│   │   │   │   └── MHA的介绍和基础搭建.md
│   │   │   ├── 02_MHA高可用工作原理
│   │   │   │   └── MHA高可用工作原理.md
│   │   │   ├── 03_vip应用透明
│   │   │   │   └── MHA-vip应用透明.md
│   │   │   ├── 04_binlogserver数据补偿
│   │   │   │   └── binlog_server数据补偿.md
│   │   │   ├── 05_高可用邮箱提醒
│   │   │   │   └── 高可用邮箱提醒.md
│   │   │   └── 06_高可用故障修复
│   │   │       └── 高可用故障修复.md
│   │   ├── 21_atlas读写分离
│   │   │   ├── 01_atlas安装部署
│   │   │   │   └── atlas介绍及安装.md
│   │   │   └── 02_atlas管理操作
│   │   │       └── atlas管理操作.md
│   │   ├── 22_ProxySQL读写分离
│   │   │   └── 00_ProxySQL读写分离.md
│   │   └── 23_mysql优化
│   │       ├── mysql优化.md
│   │       └── 数据库优化2.md
│   ├── 02_mysql8.0
│   │   └── 8.0-博客.md
│   ├── 03_redis
│   │   ├── 01_缓存数据库简述
│   │   │   └── 缓存数据库简述.md
│   │   ├── 02_redis简介
│   │   │   └── redis简介.md
│   │   ├── 03_redis安装
│   │   │   └── Redis安装.md
│   │   ├── 04_redis字符串数据结构
│   │   │   └── redis字符串数据结构.md
│   │   ├── 05_redis哈希数据结构
│   │   │   └── redis哈希数据结构.md
│   │   ├── 06_数据结构之列表
│   │   │   └── 数据结构之列表.md
│   │   ├── 07_数据结构之无序集合
│   │   │   └── 数据结构之无序集合.md
│   │   ├── 08_数据集合之有序集合
│   │   │   └── 数据集合之有序集合.md
│   │   ├── 09_redis基础管理命令
│   │   │   └── redis基础管理命令.md
│   │   ├── 10_redis数据安全-持久化
│   │   │   └── Redis数据安全-持久化.md
│   │   ├── 11_ACL安全策略
│   │   │   └── ACL安全策略.md
│   │   ├── 12_Redis的发布与订阅
│   │   │   └── Redis发布与订阅.md
│   │   ├── 13_Redis主从复制
│   │   │   └── Redis主从复制.md
│   │   ├── 14_哨兵
│   │   │   └── 哨兵.md
│   │   ├── 15_集群（待）
│   │   │   └── 集群.md
│   │   └── 16_优化
│   │       └── redis优化.md
│   ├── 04_Mongodb
│   │   ├── 01_介绍
│   │   │   └── 00_介绍.md
│   │   ├── 02_安装
│   │   │   └── 00_MongoDB安装.md
│   │   ├── 03_用户的基本管理
│   │   │   └── 00_用户的基本管理.md
│   │   ├── 04_MongoDB基本CRUD
│   │   │   └── 00_MongoDB基本CRUD.md
│   │   ├── 06_MongoDB分片集群-介绍
│   │   │   └── 00_MongoDB分片集群-介绍.md
│   │   ├── 07_MongoDB分片集群-搭建和管理
│   │   │   └── 00_MongoDB分片集群-搭建和管理.md
│   │   ├── 08_企业中分片集群设计
│   │   │   └── 00_企业中分片集群设计.md
│   │   ├── 09_高级集群设计
│   │   │   └── 00_高级集群设计.md
│   │   ├── 10_MongoDB备份恢复和迁移
│   │   │   └── 00_MongoDB备份恢复和迁移.md
│   │   ├── 11_MongoDB监控实战
│   │   │   └── 00_MongoDB监控实战.md
│   │   ├── 12_MongoDB索引管理
│   │   │   └── 00_MongoDB索引管理.md
│   │   ├── 13_MongoDB性能判断
│   │   │   └── 00_MongoDB性能判断.md
│   │   ├── 14_MongoDB的生产上线及版本升级
│   │   │   └── 00_MongoDB的生产上线及版本升级.md
│   │   ├── 15_异构平台在线迁移
│   │   │   └── 00_异构平台在线迁移.md
│   │   └── 16_MongoDB集群中的数据一致性和隔离性保证
│   │       └── 00_MongoDB集群中的数据一致性和隔离性保证.md
│   └── 05_TiDB
│       ├── 01_介绍
│       │   └── 00介绍.md
│       ├── 02_部署
│       │   └── 00_TiDB部署.md
│       ├── 03_基础管理
│       │   └── 00_管理.md
│       ├── 04_集群扩缩容及重建删除
│       │   └── 00集群扩缩容及重建删除.md
│       ├── 05_集群升级
│       │   └── 00_集群升级.md
│       ├── 06_监控管理
│       │   └── 00_监控管理.md
│       ├── 07_TiDB备份恢复实战
│       │   └── 00_备份恢复实战.md
│       ├── 08_迁移
│       │   └── 00_DM工具实现数据迁移.md
│       └── 09_数据同步
│           └── 00_数据同步.md
├── 04_容器
│   ├── 01_docker
│   │   ├── 01_docker介绍
│   │   │   └── docker介绍.md
│   │   ├── 02_docker的安装
│   │   │   └── 2-docker的安装.md
│   │   ├── 03_docker镜像命令
│   │   │   └── 3-docker镜像常用命令.md
│   │   ├── 04_docker容器命令
│   │   │   └── 4-docker容器命令.md
│   │   ├── 05_docker网络
│   │   │   └── docker网络.md
│   │   ├── 06_dockerfile
│   │   │   └── dockerfile.md
│   │   └── 07_用Docker搭建lump部署wp
│   │       └── 用dockerfile搭建wp.md
│   └── 02_containerd
│       └── 轻量级容器管理工具Containerd.md
├── 05_容器编排工具
│   ├── 01_单机容器编排工具
│   │   └── docker05docker-compose.md
│   └── 02_Kubernets
│       ├── 00_云原生介绍
│       │   └── 0-云原生介绍.md
│       ├── 01_k8s简介
│       │   └── k8s简介.md
│       ├── 02_k8s弃用docker
│       │   └── 2k8s弃用docker.md
│       ├── 03_centos二进制安装k8s1.18
│       │   └── k8s二进制安装.md
│       ├── 04_ubuntu用kubeasz安装k8s1.27
│       │   └── ubuntu用kubeasz安装k8s1.27.md
│       ├── 05_ubuntu用kubeadm安装k8s1.24.3
│       │   └── ubuntu用kubeadm安装k8s1.24.3.md
│       ├── 06_sealos部署k8s
│       ├── 07_kubectl常用命令
│       │   └── kubectl常用命令详解.md
│       ├── 08_etcd管理及备份
│       │   └── 0-etcd管理及备份.md
│       ├── 09_velero和Minio实现k8s备份迁移
│       │   └── 0-velero和Minio实现k8s备份迁移.md
│       ├── 10_k8s集群升级和扩容
│       │   └── 0-k8s集群升级和kubeasz扩容k8s集群.md
│       ├── 11_nginx&tomcat.yaml文件详解
│       │   └── 0-nginx&Tomcat.yaml文件详解.md
│       ├── 12_资源对象简介
│       │   └── 0-资源对象简介.md
│       ├── 13_应用容器与Pod资源
│       │   └── 1-应用容器与Pod资源.md
│       ├── 14_namespace命名空间资源
│       │   └── 命名空间资源.md
│       ├── 15_lable标签资源
│       │   └── lable标签资源.md
│       ├── 16_job与cronjob
│       │   └── 0-job与cronjob.md
│       ├── 17_container-manager控制器资源
│       │   └── container-manager控制器资源.md
│       ├── 18_service资源
│       │   └── service资源.md
│       ├── 19_存储卷-数据存储
│       │   └── 0-存储卷.md
│       ├── 20_存储卷-配置中心
│       │   └── 0-存储卷-配置中心.md
│       ├── 21_服务探针
│       │   └── 0-服务探针.md
│       ├── 22_K8S监控组件metrics-server
│       │   └── 0-K8S监控组件-metrics-server.md
│       ├── 23_HPA自动伸缩
│       │   └── 0-HPA自动伸缩.md
│       ├── 24_实战案例
│       │   ├── 1-Zookeeper集群运行
│       │   │   └── 0-Zookeeper集群运行.md
│       │   └── 2-redis集群搭建
│       │       └── 0-redis集群搭建.md
│       ├── 25_资源限制
│       │   └── 0-资源限制.md
│       ├── 26_亲和与反亲和，污点与容忍、驱逐
│       │   └── 0-亲和与反亲和、污点与容忍、驱逐.md
│       ├── 27_k8s准入机制
│       │   └── k8s准入机制.md
│       ├── 28_Kubernetes-Ingress-https自动化
│       │   └── 0-Kubernetes-Ingress-https自动化.md
│       ├── 29_Helm-云原生包管理方案
│       │   └── 0-Helm和kubeapps.md
│       ├── 30_APISIX
│       │   ├── 01_APISIX介绍
│       │   │   └── 00_APISIX介绍.md
│       │   ├── 02_helm安装APISIX
│       │   │   └── 00_helm安装APISIX.md
│       │   ├── 03_Apisix_Ingress_Controller使用指南
│       │   │   └── 00_Apisix_Ingress_Controller使用指南.md
│       │   └── 04_Apisix和cert-manager自动签发https证书
│       │       └── 00_Apisix和cert-manager自动签发https证书.md
│       ├── 31_K8s-Intel_iGPU调度
│       │   └── 00_K8s-Intel_iGPU调度.md
│       ├── 32_K8s-NVDIA_GPU调度
│       │   └── 00_K8s-NVDIA_GPU调度.md
│       ├── 33_k8s运行windows系统
│       │   └── 00_k8s运行windows系统.md
│       ├── 34_operator
│       │   ├── 01_介绍及引入
│       │   │   └── 00_介绍及引入.md
│       │   ├── 02_Kubebuilder介绍
│       │   │   └── 00_Kubebuilder介绍.md
│       │   ├── 03_Kubebuilder简单使用及了解
│       │   │   └── 00_Kubebuilder简单使用及了解.md
│       │   └── 04_MySQLOperator
│       │       └── 00_MySQLOperator.md
│       ├── 35_Istio
│       │   ├── 01_介绍
│       │   │   └── 00_介绍.md
│       │   ├── 02_安装
│       │   │   └── 00_安装.md
│       │   ├── 03_流量控制
│       │   │   ├── 01_动态路由：虚拟服务和目标规则
│       │   │   │   └── 00_动态路由：虚拟服务和目标规则.md
│       │   │   ├── 02_网关
│       │   │   │   └── 00_网关.md
│       │   │   ├── 03_服务入口
│       │   │   │   └── 00_服务入口.md
│       │   │   ├── 04_流量转移：灰度发布
│       │   │   │   └── 00_流量转移：灰度发布.md
│       │   │   ├── 05_Ingress
│       │   │   │   └── 00_Ingress.md
│       │   │   ├── 06_Egress网关实现访问外部服务
│       │   │   │   └── 00_Egress网关实现访问外部服务.md
│       │   │   ├── 07_超时重试
│       │   │   │   └── 00_超时重试.md
│       │   │   ├── 08_熔断
│       │   │   │   └── 00_熔断.md
│       │   │   ├── 09_故障注入
│       │   │   │   └── 00_故障注入.md
│       │   │   └── 10_流量镜像
│       │   │       └── 00_流量镜像.md
│       │   ├── 04_可观察性
│       │   │   ├── 01_Kiali观测微服务应用
│       │   │   │   └── 00_Kiali观测微服务应用.md
│       │   │   ├── 02_Prometheus收集指标
│       │   │   │   └── 00_Prometheus收集指标.md
│       │   │   ├── 03_Grafana查看系统
│       │   │   │   └── 00_Grafana查看系统.md
│       │   │   ├── 04_Envoy的日志获取
│       │   │   │   └── 00_Envoy的日志获取.md
│       │   │   └── 05_分布式追踪
│       │   │       └── 00_分布式追踪.md
│       │   ├── 05_安全
│       │   │   ├── 01_TLS安全网关
│       │   │   │   └── 00_TLS安全网关.md
│       │   │   ├── 02_双向TLS
│       │   │   │   └── 00_双向TLS.md
│       │   │   └── 03_JWT身份认证与授权
│       │   │       └── 00_JWT身份认证与授权.md
│       │   └── 06_实战
│       │       ├── 01_实战项目准备和gitops实现
│       │       │   └── 00_实战项目准备和gitops实现.md
│       │       ├── 02_自动化灰度发布
│       │       │   └── 00_自动化灰度发布.md
│       │       ├── 03_提升系统的弹性能力
│       │       │   └── 00_提升系统的弹性能力.md
│       │       ├── 04_配置安全策略
│       │       │   └── 00_配置安全策略.md
│       │       ├── 05_使用现有Prometheus
│       │       │   └── 00_使用现有Prometheus.md
│       │       ├── 06_ELK日志收集
│       │       │   └── 00_ELK日志收集.md
│       │       ├── 07_分布式链路监控
│       │       │   └── 00_分布式链路监控.md
│       │       └── 08_调试网格的工具和方法
│       │           └── 00_调试网格的工具和方法.md
│       └── 36_Kustomize%20入门与实战详解
├── 06_监控平台
│   ├── 01_zabbix
│   │   ├── 01_zabbix安装部署
│   │   │   └── zabbix安装.md
│   │   ├── 02_监控简介及zabbix介绍
│   │   │   └── 监控介绍及zabbix介绍.md
│   │   ├── 03_其他机器添加客户端
│   │   │   └── 其他机器客户端安装及监控.md
│   │   ├── 04_自定义监控项
│   │   │   └── 自定义监控项.md
│   │   ├── 05_自定义触发器
│   │   │   └── 自定义触发器.md
│   │   ├── 06_邮件报警
│   │   │   └── 邮件报警.md
│   │   ├── 07_微信报警
│   │   │   └── 微信报警.md
│   │   ├── 08_报警升级
│   │   │   └── 报警升级.md
│   │   ├── 09_自定义图形
│   │   │   └── 自定义图形.md
│   │   ├── 11_grafana创建自定义图形
│   │   │   └── grafana创建自定义图形.md
│   │   ├── 12_自定义模板
│   │   │   ├── 01_tcp的十一种状态
│   │   │   │   └── tcp的十一种状态.md
│   │   │   ├── 02_批量创建自定义监控项
│   │   │   │   └── 批量创建自定义监控项.md
│   │   │   └── 03_zabbix自定义模板
│   │   │       └── zabbix自定义模板.md
│   │   ├── 13_使用模板监控nginx服务
│   │   │   └── 使用模板监控nginx服务.md
│   │   ├── 14_使用模板监控php服务
│   │   │   └── 使用模板监控php.md
│   │   ├── 15_使用模板监控redis
│   │   │   ├── 01_搭建discuz论坛使用redis加速
│   │   │   │   └── 搭建discuz论坛使用redis加速.md
│   │   │   └── 02_使用模板监控redis服务
│   │   │       └── 使用模板监控redis服务.md
│   │   ├── 16_监控的维度
│   │   │   └── 监控的维度.md
│   │   ├── 17_PV-UV-IP_matomo监控
│   │   │   └── pv-uv-ip监控.md
│   │   ├── 18_zabbix-web监控
│   │   │   └── zabbix-web监控.md
│   │   ├── 19_使用percona插件监控mysql
│   │   │   └── 使用percona插件监控mysql.md
│   │   ├── 20_snmp监控linux系统
│   │   │   └── snmp监控linux.md
│   │   ├── 21_zabbix自动发现
│   │   │   └── zabbix自动发现.md
│   │   ├── 22_zabbix自动注册
│   │   │   └── zabbix自动注册.md
│   │   └── 23_zabbix-agent主动模式和被动模式
│   │       └── zabbix-agent主动模式和被动模式.md
│   └── 02_prometheus监控
│       ├── 01_promethus简介
│       │   └── promethus介绍.md
│       ├── 02_promethus安装
│       │   └── promethus安装.md
│       ├── 03_prometheus监控远程linux主机
│       │   └── 监控远程linux主机.md
│       ├── 04_prometheus监控远程mysql服务
│       │   └── prometheus监控远程mysql服务.md
│       ├── 05_prometheus集成grafana
│       │   └── prometheus集成grafana.md
│       ├── 06_k8s部署prometheus
│       │   └── 0-k8s部署prometheus集群.md
│       ├── 07_promQL基本使用
│       │   └── promQL基本使用.md
│       ├── 08_监控携带metrics接口服务
│       │   └── 监控携带metrics接口服务.md
│       ├── 09_监控非携带metrics接口服务-nginx
│       │   └── 监控非携带metrics服务.md
│       ├── 10_Prometheus服务自动发现机制
│       │   └── 0-prometheus服务发现机制.md
│       ├── 11_告警简介
│       │   └── 告警简介.md
│       ├── 12_告警之prometheus配置详解
│       │   └── prometheus配置文件详解.md
│       ├── 13_告警之altermanager配置详解
│       │   └── altermanager配置详解.md
│       └── 14_配置企业微信告警
│           └── 配置企业微信告警.md
├── 07_日志收集平台
│   └── 01_ELK日志收集
│       ├── 01_简介
│       │   └── ELK简介.md
│       ├── 02_ELK物理机安装
│       │   └── ELK物理机安装.md
│       ├── 03_企业架构及beats介绍
│       │   └── 企业架构及beats介绍.md
│       ├── 04_企业ELK搭建
│       │   └── 0-企业ELK搭建.md
│       ├── 05_ELK配置
│       │   └── 2ELK配置.md
│       └── 06_ELK使用文档
│           └── 0-ELK使用文档.md
├── 08_消息队列
│   ├── 01_pulsar
│   │   ├── 01_了解pulsar
│   │   │   └── 1-了解pulsar.md
│   │   └── 02_k8s安装pulsar
│   │       └── 0-k8s安装pulsar.md
│   └── 02_rocketmq
│       ├── 01_RocketMQ介绍及基本概念
│       │   └── 1RocketMQ及基本概念.md
│       ├── 02_RocketMQ部署
│       │   └── rocketmq部署.md
│       ├── 03_RocketMQ高可用
│       │   └── RocketMQ高可用.md
│       └── 04_RocketMQ容器化
│           └── Rocketmq容器化.md
├── 09_devops
│   ├── 00_介绍
│   │   ├── 01_devops简介
│   │   │   └── Devops简介.md
│   │   ├── 02_持续集成简介
│   │   │   └── 持续集成简介.md
│   │   └── 03_K8s-CICD流程介绍
│   │       └── 0-K8s-CICD流程介绍.md
│   ├── 01_git
│   │   ├── 01_git介绍
│   │   │   └── Git简介.md
│   │   ├── 02_git安装
│   │   │   └── git安装.md
│   │   └── 03_git基础
│   │       └── git基础使用.md
│   ├── 02_gerrit
│   │   ├── 01_gerrit介绍
│   │   │   └── gerrit介绍.md
│   │   ├── 02_gerrit安装
│   │   │   └── Gerrit安装.md
│   │   ├── 03_gerrit简单使用
│   │   │   └── gerrit简单使用.md
│   │   ├── 04_gerrit备份
│   │   │   └── gerrit备份.md
│   │   ├── 05_gerrit升级及插件安装
│   │   │   └── gerrit升级及插件安装.md
│   │   ├── 06_gerrit3.5.1变化文档
│   │   │   └── gerrit3.5.1变化文档.md
│   │   ├── 07_rsa算法不支持深究
│   │   │   └── 密码学：常见的信息加密算法.md
│   │   ├── 08_gerrit+ldap验证
│   │   │   └── gerrit+ldap验证.md
│   │   └── 09_仓库设置介绍
│   │       └── 00_仓库设置介绍.md
│   ├── 03_gitlab
│   │   ├── 01_Gitlab安装
│   │   │   └── Gitlab安装.md
│   │   ├── 02_gitlab常用命令
│   │   │   └── gitlab常用命令.md
│   │   ├── 03_gitlab页面操作
│   │   │   └── gitlab页面操作.md
│   │   └── 04_gitlab-ci
│   │       └── 00_gitlab-ci.md
│   ├── 04_jenkins
│   │   ├── 01_jenkins安装
│   │   │   └── jenkins安装.md
│   │   ├── 02_用户管理
│   │   │   └── 用户管理.md
│   │   ├── 03_凭证管理
│   │   │   └── 凭证管理.md
│   │   ├── 04_自由风格构建项目
│   │   │   └── 自由风格项目.md
│   │   ├── 05_jenkins参数化构建了解
│   │   │   └── jenkins参数化构建了解.md
│   │   ├── 06_jenkins配置邮件
│   │   │   └── jenkins配置邮件通知.md
│   │   ├── 07_jenkins容器化安装
│   │   │   └── 00_jenkins容器化安装.md
│   │   ├── 08_jenkins基础入门
│   │   │   └── 00_jenkins基础入门.md
│   │   ├── 09_jenkins快速进阶
│   │   │   └── 00_jenkins快速进阶.md
│   │   ├── 10_jenkins接入其他工具
│   │   │   └── 00_jenkins接入其他工具.md
│   │   └── 11_jenkins实战
│   │       └── 00_jenkins实战.md
│   ├── 05_Tekton
│   │   ├── 01_Tekton介绍
│   │   │   └── 00_Tekton介绍.md
│   │   ├── 02_Tekton部署
│   │   │   └── 00_Tekton部署.md
│   │   └── 03_Tekton使用
│   │       └── 00_Tekton使用.md
│   ├── 06_ArgoCD
│   │   ├── 01_Gitops介绍
│   │   │   └── 0-GitOps介绍.md
│   │   ├── 02_ArgoCD介绍
│   │   │   └── 0-ArgoCD介绍.md
│   │   ├── 03_ArgoCD安装
│   │   │   └── 0-ArgoCD安装.md
│   │   ├── 04_ArgoCD使用
│   │   │   └── 0-Argocd使用.md
│   │   └── 05_ArgoCD钉钉通知
│   │       └── ArgoCD钉钉通知.md
│   ├── 07_devops实现
│   │   ├── 01_commit提交规范
│   │   │   └── 00_commit提交规范.md
│   │   └── 02_gerrit代码同步到gitlab
│   │       └── 00_gerrit代码同步到gitlab.md
│   ├── 08_Harbor
│   │   └── 01_Harbor安装
│   │       └── 00_Harbor安装.md
│   └── 09_AIOps
│       ├── 01_基础
│       │   ├── 01_介绍
│       │   │   └── 00_介绍.md
│       │   └── 02_IaC和Terraform
│       │       └── 00_IaC和Terraform.md
│       ├── 02_AIOps%E5%85%A5%E9%97%A8
│       │   ├── 04_%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%BE%AE%E8%B0%83%EF%BC%88Fine-tuning%EF%BC%89%E5%AE%9E%E6%88%98%E8%AF%A6%E8%A7%A3
│       │   └── 05_检索增强生成（RAG、GraphRAG）
│       ├── 02_AIOps入门
│       │   ├── 01_Prompt%20Engineering%20入门与实战：提升大模型输出质量的关键
│       │   ├── 02_大模型接入、记忆机制与%20JSON%20Mode%20实战详解
│       │   └── 03_函数调用实战：实现大模型与程序的动态交互
│       │       └── 00_函数调用实战：实现大模型与程序的动态交互.md
│       ├── 03_Agent入门
│       │   ├── 01_Agent的四种设计模式
│       │   │   └── 00_Agent的四种设计模式.md
│       │   ├── 02_Agent开发实战：基于LangGraph的RAG增强型Agent开发实践：循环反思机制与翻译智能体实现
│       │   │   └── 00_Agent开发实战：基于LangGraph的RAG增强型Agent开发实践：循环反思机制与翻译智能体实现.md
│       │   ├── 03_Agent开发实战：LangGraph实战：构建可调试、可循环、自适应的运维智能体
│       │   │   └── 00_Agent开发实战：LangGraph实战：构建可调试、可循环、自适应的运维智能体.md
│       │   └── 04_Agent开发实战：自适应RAG与网络搜索融合的智能运维知识库Agent构建
│       │       └── 00_Agent开发实战：自适应RAG与网络搜索融合的智能运维知识库Agent构建.md
│       ├── 04_Client-go入门及实战
│       │   ├── 01_Kubernetes%20客户端开发入门：Client-go%20核心原理与集群内外认证实践
│       │   ├── 02_Client-go%20四大客户端深度解析：从%20REST%20Client%20到%20Dynamic%20Client%20的演进与实践
│       │   ├── 03_Kubernetes%20客户端事件监听最佳实践：从%20Watch%20到%20Informer%20+%20工作队列的演进
│       │   ├── 04_Kubernetes%20Informer%20机制深度解析：Reflector、Indexer%20与%20Workqueue%20协同架构
│       │   └── 05_使用%20Dynamic%20Client%20实现%20Kubernetes%20自定义资源（CRD）的%20GET%20操作
│       ├── 05_Client-go_AIOps实战
│       │   ├── 01_使用%20Cobra%20构建%20Kubernetes%20智能%20CLI%20工具：从零开发%20ChatK8S%20与%20K8SGPT%20故障诊断器
│       │   ├── 02_Kubernetes%20Copilot：用自然语言交互式管理%20K8S%20资源
│       │   └── 03_基于%20Kubernetes%20事件与日志的%20ChatGPT%20故障诊断工具开发实战
│       ├── 06_KubernetesOperator入门
│       │   ├── 01_KubernetesOperator入门与实战：从概念到框架深度解析
│       │   │   └── 00_KubernetesOperator入门与实战：从概念到框架深度解析.md
│       │   ├── 02_使用%20KubeBuilder开发ApplicationOperator：一键编排Deployment、Service与Ingress
│       │   ├── 03_KubeBuilder项目结构解析与定时弹性伸缩Operator实战指南
│       │   │   └── 00_KubeBuilder项目结构解析与定时弹性伸缩Operator实战指南.md
│       │   ├── 04_使用OperatorSDK将HelmChart快速转换为KubernetesOperator
│       │   │   └── 00_使用OperatorSDK将HelmChart快速转换为KubernetesOperator.md
│       │   ├── 05_OperatorLifecycleManager（OLM）：KubernetesOperator的标准化分发与生命周期管理工具
│       │   │   └── 00_OperatorLifecycleManager（OLM）：KubernetesOperator的标准化分发与生命周期管理工具.md
│       │   └── 06_KubernetesOperator开发三大最佳实践与端到端测试实战指南
│       │       └── 00_KubernetesOperator开发三大最佳实践与端到端测试实战指南.md
│       ├── 07_OperatorAIOps实战
│       │   ├── 01_基于KubernetesOperator的GPU竞价实例资源池自动化运维实战
│       │   │   └── 00_基于KubernetesOperator的GPU竞价实例资源池自动化运维实战.md
│       │   ├── 02_基于Operator+Ollama+Kong实现大模型私有化高可用推理部署
│       │   │   └── 00_基于Operator+Ollama+Kong实现大模型私有化高可用推理部署.md
│       │   ├── 03_基于大模型的日志流智能监测Operator：LogPilot实战解析
│       │   │   └── 00_基于大模型的日志流智能监测Operator：LogPilot实战解析.md
│       │   └── 04_基于RAGFlow的运维专家知识库故障排查实战
│       │       └── 00_基于RAGFlow的运维专家知识库故障排查实战.md
│       ├── 08_训练流量预测模型实现自动扩容
│       │   ├── 01_AIOps实战：基于时序预测的Kubernetes自动扩缩容系统
│       │   │   └── 00_AIOps实战：基于时序预测的Kubernetes自动扩缩容系统.md
│       │   └── 02_基于预测模型的Kubernetes自定义Operator弹性扩缩容实践
│       │       └── 00_基于预测模型的Kubernetes自定义Operator弹性扩缩容实践.md
│       ├── 09_基于多Agent协同的Kubernetes故障自动修复
│       │   ├── 01_多Agent协同实现K8S故障自动修复：原理、模式与实战
│       │   │   └── 00_多Agent协同实现K8S故障自动修复：原理、模式与实战.md
│       │   └── 02_多智能体协同自动修复Kubernetes故障实战指南
│       │       └── 00_多智能体协同自动修复Kubernetes故障实战指南.md
│       ├── 10_OpenTelemetry
│       │   ├── 01_OpenTelemetry%20可观测性入门：从历史演进到统一标准
│       │   ├── 02_OpenTelemetry%20数据采集核心机制详解：Trace、Metrics%20与组件架构
│       │   ├── 03_深入解析OpenTelemetry数据流与集成方式
│       │   │   └── 00_深入解析OpenTelemetry数据流与集成方式.md
│       │   ├── 04_Loki日志采集实战：轻量级可观测性日志方案全解析
│       │   │   └── 00_Loki日志采集实战：轻量级可观测性日志方案全解析.md
│       │   ├── 05_Prometheus指标采集实战：从K8s系统监控到业务指标集成
│       │   │   └── 00_Prometheus指标采集实战：从K8s系统监控到业务指标集成.md
│       │   ├── 06_Python微服务中集成OpenTelemetry实现可观测性三合一（Tracing%20+%20Metrics%20+%20Logs）
│       │   ├── 07_打造日志、指标与分布式追踪三合一的可观测性查询面板
│       │   │   └── 00_打造日志、指标与分布式追踪三合一的可观测性查询面板.md
│       │   └── 08_零代码集成%20OpenTelemetry：原理、实践与关键限制
│       └── 11_eBPF
│           ├── 01_eBPF%20入门：让%20Linux%20内核真正“可编程”的革命性技术
│           ├── 02_eBPF%20程序从编写到内核执行的完整技术链路解析
│           ├── 03_eBPF%20Map%20全解析：内核与用户态高效数据共享的核心机制
│           ├── 04_XDP：Linux内核级高速网络包处理框架详解
│           │   └── 00_XDP：Linux内核级高速网络包处理框架详解.md
│           ├── 05_eBPF入门指南：核心组件与用户空间工具全景解析
│           │   └── 00_eBPF入门指南：核心组件与用户空间工具全景解析.md
│           ├── 06_EBPF零侵入可观测性开发实战：从BCC入门到HTTP延迟监控
│           │   └── 00_EBPF零侵入可观测性开发实战：从BCC入门到HTTP延迟监控.md
│           ├── 07_Bella：基于eBPF的零侵入式可观测性工具详解
│           │   └── 00_Bella：基于eBPF的零侵入式可观测性工具详解.md
│           ├── 08_Cilium+Hubble：基于eBPF的零侵入云原生可观测性实践
│           └── 09_基于eBPF与Falco实现Kubernetes实时安全威胁检测
│               └── 00_基于eBPF与Falco实现Kubernetes实时安全威胁检测.md
├── 10_云原生
│   ├── 00_云原生
│   │   ├── 01_云原生介绍
│   │   │   └── 00_云原生介绍.md
│   │   ├── 02_debian安装k8s集群
│   │   │   └── 00_debian安装k8s集群.md
│   │   ├── 03_k8s安装ceph集群
│   │   │   └── 01_介绍及安装
│   │   │       └── 00_介绍及安装.md
│   │   └── 04_k8s安装APISIX-Ingress-Controller
│   │       └── 00_k8s安装APISIX-Ingress-Controller.md
│   └── 01_微服务
│       ├── 01_微服务介绍
│       │   └── 00_微服务介绍.md
│       ├── 02_Nacos
│       │   ├── 01_Nacos介绍
│       │   │   └── 1-Nacos介绍.md
│       │   ├── 02_Nacos安装
│       │   │   └── Nacos单机安装.md
│       │   ├── 03_Nacos单节点容器化使用外置mysql部署
│       │   │   └── Nacos单节点容器化使用外置mysql部署.md
│       │   └── 05_Operator部署
│       │       └── 00_Operator部署.md
│       └── 03_链路追踪
│           ├── 01_分布式链路追踪介绍
│           │   └── 00_分布式链路追踪介绍.md
│           ├── 02_zipkin介绍
│           │   └── zipkin介绍.md
│           └── 03_skywalking介绍
│               └── 00_skywalking介绍.md
├── 11_开发学习
│   ├── 01_python
│   │   ├── 01_python介绍
│   │   │   └── python介绍.md
│   │   ├── 02_python语法
│   │   │   ├── 01_注释以及变量和常量
│   │   │   │   └── 1注释以及变量和常量.md
│   │   │   ├── 02_基本的数据类型
│   │   │   │   └── 基本的数据类型.md
│   │   │   ├── 03_Python内存管理
│   │   │   │   └── Python内存管理.md
│   │   │   ├── 04_用户交互和格式化输出
│   │   │   │   └── 用户交互和格式化输出.md
│   │   │   ├── 05_基本运算符
│   │   │   │   └── 基本运算符.md
│   │   │   └── 06_流程控制
│   │   │       └── 流程控制.md
│   │   ├── 03_基本数据类型和内置方法
│   │   │   └── 基本的数据类型及其内置方法.md
│   │   ├── 04_可变与不可变类型和总结
│   │   │   └── 可变与不可变类型和总结.md
│   │   ├── 05_文件处理
│   │   │   └── 0文件处理.md
│   │   ├── 06_函数
│   │   │   ├── 01_函数的基本使用
│   │   │   │   └── 1函数的基本使用.md
│   │   │   ├── 02_函数的参数
│   │   │   │   └── 函数的参数.md
│   │   │   ├── 03_名称空间与作用域
│   │   │   │   └── 名称空间与作用域.md
│   │   │   ├── 04_函数对象与闭包
│   │   │   │   └── 函数对象与闭包.md
│   │   │   ├── 05_装饰器
│   │   │   │   └── 1-装饰器.md
│   │   │   ├── 06_迭代器与生成器
│   │   │   │   └── 0-迭代器与生成器.md
│   │   │   ├── 07_函数递归
│   │   │   │   └── 0-函数递归.md
│   │   │   └── 08_面向过程与函数式
│   │   │       └── 0面向过程与函数式.md
│   │   ├── 07_模块与包
│   │   │   ├── 01_模块
│   │   │   │   └── 0-模块.md
│   │   │   ├── 02_包
│   │   │   │   └── 1-包.md
│   │   │   ├── 03_软件开发目录的规范
│   │   │   │   └── 1-软件开发目录的规范.md
│   │   │   ├── 04_时间模块
│   │   │   │   └── 0-时间模块.md
│   │   │   ├── 05_random随机模块
│   │   │   │   └── 0-random随机模块.md
│   │   │   ├── 06_os模块
│   │   │   │   └── os模块.md
│   │   │   ├── 07_sys模块
│   │   │   │   └── 0-sys模块.md
│   │   │   ├── 08_shutil模块
│   │   │   │   └── 0-shutil模块.md
│   │   │   ├── 09_json和pickie模块
│   │   │   │   └── json和pickie模块.md
│   │   │   ├── 10_shelve模块
│   │   │   │   └── shelve模块.md
│   │   │   ├── 11_xml模块
│   │   │   │   └── xml模块.md
│   │   │   ├── 12_configparser模块
│   │   │   │   └── configparser模块.md
│   │   │   ├── 13_hashlib模块
│   │   │   │   └── hashlib模块.md
│   │   │   ├── 14_subprocess模块
│   │   │   │   └── 0-subprocess模块.md
│   │   │   ├── 15_logging模块
│   │   │   │   └── 0-logging模块.md
│   │   │   └── 16_re模块
│   │   │       └── 0-re模块.md
│   │   ├── 08_面向对象编程
│   │   │   ├── 01_面相对象编程介绍
│   │   │   │   └── 00_面相对象编程介绍.md
│   │   │   ├── 02_封装
│   │   │   │   └── 00_封装.md
│   │   │   ├── 03_继承与派生
│   │   │   │   └── 00_继承与派生.md
│   │   │   ├── 04_多态、多态性、鸭子类型
│   │   │   │   └── 00_多态、多态性、鸭子类型.md
│   │   │   ├── 05_绑定方法和非绑定方法
│   │   │   │   └── 00_绑定方法和非绑定方法.md
│   │   │   ├── 06_反射、内置方法
│   │   │   │   └── 00_反射和内置方法.md
│   │   │   └── 07_元类
│   │   │       └── 00_元类.md
│   │   ├── 09_异常处理
│   │   │   └── 00_异常处理.md
│   │   ├── 10_网络编程
│   │   │   └── 00_网络编程.md
│   │   ├── 11_并发编程
│   │   │   └── 00_并发编程.md
│   │   └── 12_Mysql数据库
│   │       └── 00_Mysql数据库.md
│   ├── 02_Java
│   │   ├── 01_Java介绍
│   │   │   └── 00_Java介绍.md
│   │   └── 02_Java语言基础
│   │       ├── 01_Java入门
│   │       │   └── 00_Java入门.md
│   │       ├── 02_变量与运算符
│   │       │   └── 00_变量与运算符.md
│   │       ├── 03_流程控制语句
│   │       │   └── 00_流程控制语句.md
│   │       ├── 04_IDEA的安装与使用
│   │       │   └── 00_IDEA的安装与使用.md
│   │       └── 05_数组
│   │           └── 00_数组.md
│   ├── 03_Golang
│   │   ├── 01_介绍
│   │   │   └── 00_介绍.md
│   │   └── 02_入门
│   │       └── 01_变量
│   │           └── 00_变量.md
│   └── 04_前端
│       ├── 01_HTML标签
│       │   └── 00_HTML标签.md
│       └── 02_CSS
│           └── 00_CSS.md
├── 12_网络安全
│   ├── 00_声明.md
│   ├── 01_基础入门
│   │   ├── 01_基础概念了解
│   │   │   └── 00_基础概念了解.md
│   │   ├── 02_文件下载和反弹命令
│   │   │   └── 00_文件下载和反弹.md
│   │   ├── 03_网站搭建和http数据包及代理
│   │   │   └── 00_网站搭建和http数据包及代理.md
│   │   ├── 04_抓包和封包
│   │   │   └── 00_抓包和封包.md
│   │   ├── 05_常见的加密方式_初识
│   │   │   └── 00_常见的加密方式_初识.md
│   │   ├── 06_常见的信息加密算法_非对称加密深入
│   │   │   └── 00_密码学：常见的信息加密算法_非对称加密深入.md
│   │   └── 07_web网站会遇到的安全问题
│   │       └── 00_web建站安全问题.md
│   └── 02_信息打点
│       ├── 01_web架构-域名-开发语言-中间件-数据库-系统-源码获取
│       │   └── 00_web架构-域名-开发语言-中间件-数据库-系统-源码获取.md
│       └── 02_网站源码资产泄露
│           └── 网站源码资产泄露.md
├── 13_winserver
│   ├── 01_winserver安装
│   │   └── 00_winserver安装.md
│   ├── 02_apache环境部署
│   │   └── 00_apache环境部署.md
│   ├── 03_mysql环境搭建
│   │   └── 00_mysql环境搭建.md
│   ├── 04_php环境搭建
│   │   └── 00_php环境搭建.md
│   ├── 05_wordpress搭建
│   │   └── wordpress环境搭建.md
│   └── 06_winserver裸装django
│       └── 00_winserver裸装运行django.md
├── 14_IT类
│   └── 02_svn+ldap+ifSVNadmin
│       └── svn+ldap+ifSVNadmin部署.md
├── 15_存储
│   └── 01_Ceph
│       ├── 01_ceph介绍
│       │   └── 1-cpeh介绍.md
│       ├── 02_ceph介绍2
│       │   └── 00_ceph介绍.md
│       ├── 03_ceph部署
│       │   └── 0-ceph部署.md
│       ├── 04_块存储-RBD使用
│       │   └── 00_块存储-RBD使用.md
│       ├── 05_Ceph-FS文件存储
│       │   └── 00_Ceph-FS文件存储.md
│       ├── 06_Ceph常用命令
│       │   └── 00_Ceph常用命令.md
│       ├── 07_存储池、PG与CRUSH
│       ├── 08_PG的状态
│       │   └── 00_PG的状态.md
│       ├── 09_数据读写流程
│       │   └── 00_数据读写流程.md
│       ├── 10_ceph存储池操作
│       │   └── 00_ceph存储池操作.md
│       ├── 11_CephX认证机制
│       │   └── 00_CephX认证机制.md
│       ├── 12_Ceph对象存储网关RadosGW
│       │   └── 00_Ceph对象存储网关RadosGW.md
│       ├── 13_Ceph集群维护
│       │   └── 00_ceph集群维护.md
│       ├── 14_Ceph-crush进阶
│       │   └── 00_Ceph-crush进阶.md
│       └── 15_ceph使用案例
│           └── 00_Ceph使用案例.md
├── 16_AI
│   ├── 00_AI基础
│   │   └── 01_大模型入门：从人工智能到深度学习的核心概念解析
│   │       └── 00_大模型入门：从人工智能到深度学习的核心概念解析.md
│   ├── 01_Stable_Diffusion
│   │   ├── 01_安装以及配置中文
│   │   │   └── 00_安装以及配置中文.md
│   │   ├── 02_模型安装
│   │   │   └── 00_模型安装.md
│   │   └── 03_绘世-forge-aki安装
│   │       └── 00_绘世-Forge-aki安装.md
│   ├── 02_HAMI
│   │   ├── 01_HAMI介绍
│   │   │   └── 00_HAMI介绍.md
│   │   └── 02_HAMI安装
│   │       └── 00_HAMI安装.md
│   └── 03_claude-code使用
│       └── 01_基础操作和使用
│           ├── 00_claude-code基础操作和使用.md
│           ├── ARCHITECTURE.md
│           ├── CLAUDE.md
│           ├── DEVELOPMENT_PLAN.md
│           └── PROGRESS.md
```

</details>
