#  binlog server数据补偿（db03）

## 一、参数：

```bash
binlogserver配置：
找一台额外的机器，必须要有5.6以上的版本，支持gtid并开启，我们直接用的第二个slave（db03）
vim /etc/mha/app1.cnf 
[binlog1]
no_master=1					#不参与选主
hostname=10.0.0.53
master_binlog_dir=/data/mysql/binlog
```



## 二、创建必要目录

```bash
[root@db03 ~]# mkdir -p /data/mysql/binlog
[root@db03 ~]# chown -R mysql.mysql /data/*
修改完成后，将主库binlog拉过来（从000001开始拉，之后的binlog会自动按顺序过来）
```



## 三、拉取主库binlog日志

```bash
[root@db03 ~]# cd /data/mysql/binlog/     	-----》必须进入到自己创建好的目录

[root@db03 /data/mysql/binlog]# mysqlbinlog  -R --host=10.0.0.51 --user=mha --password=mha --raw  --stop-never mysql-bin.000001 &


注意：
拉取日志的起点,需要按照目前从库的已经获取到的二进制日志点为起点
需要在主库查询位置点，不必从第一个开始
```



## 四、重启MHA

```bash
[root@db03 ~]# masterha_stop --conf=/etc/mha/app1.cnf

[root@db03 ~]# nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &

```



## 五、故障处理

```undefined
主库宕机，binlogserver 自动停掉，manager 也会自动停止。
处理思路：
1、重新获取新主库的binlog到binlogserver中
2、重新配置文件binlog server信息
3、最后再启动MHA
```