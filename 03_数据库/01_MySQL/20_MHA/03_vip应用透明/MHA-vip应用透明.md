# MHA vip应用透明

## 一、说明

```bash
1）通过keepalived的方式，管理虚拟IP的漂移
2）通过MHA自带脚本方式，管理虚拟IP的漂移
	只能同机房使用，无法跨机房跨网络使用
```



## 二、配置读取脚本（db03）

```bash
#添加一下参数
[root@db03 ~]# vim /etc/mha/app1.cnf
master_ip_failover_script=/usr/local/bin/master_ip_failover
注意：/usr/local/bin/master_ip_failover，必须事先准备好
```



## 三、修改脚本内容

```bash
[root@db03 ~]# vim /usr/local/bin/master_ip_failover
#!/usr/bin/env perl
 
use strict;
use warnings FATAL => 'all';
 
use Getopt::Long;
 
my (
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);
 
my $vip = '10.0.0.55/24';
my $key = '1';
my $ssh_start_vip = "/sbin/ifconfig eth0:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig eth0:$key down";
 
GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);
 
exit &main();
 
sub main {
 
    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
 
    if ( $command eq "stop" || $command eq "stopssh" ) {
 
        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {
 
        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}
 
sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
     return 0  unless  ($ssh_user);
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}
sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip 
            --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
```



## 四、检查脚本，添加权限

```bash
注意：
[root@db03 ~]# dos2unix /usr/local/bin/master_ip_failover 
dos2unix: converting file /usr/local/bin/master_ip_failover to Unix format ...
[root@db03 ~]# chmod +x /usr/local/bin/master_ip_failover 
```



## 五、主库上（db01），手工生成第一个vip地址

```bash
手工在主库上绑定vip，注意一定要和配置文件中的ethN一致，我的是eth0:1(1是key指定的值)
[root@db01 ~]# ifconfig eth0:1 10.0.0.55/24
```



## 六、重启mha

```jsx
masterha_stop --conf=/etc/mha/app1.cnf

nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &
```



## 七、检测状态

```bash
[root@db03 ~]# masterha_check_status --conf=/etc/mha/app1.cnf
app1 (pid:7041) is running(0:PING_OK), master:10.0.0.51
```



**注意**

```bash
keepalive,需要candidata_master=1和check_reol_delay=0进行配合。防止vip和主库选择不在一个节点
```

