# zabbix自定义模板

## 一、新建模板

### 1、复制配置文件到客户端

```bash
[root@zabbix /etc/zabbix/zabbix_agentd.d]# scp user_def.conf 10.0.0.7:`pwd`

[root@web01 ~]# systemctl restart zabbix-agent.service 
```



### 2、新建模板

![1](1.png)

![2](2.png)

## 二、复制监控项

![3](3.png)

![4](4.png)

![5](5.png)

![6](6.png)



## 三、复制的监控项添加应用集

![7](7.png)

![8](8.png)

![9](9.png)

## 四、添加触发器

![10](10.png)

![11](11.png)

![12](12.png)

![13](13.png)

## 五、添加图形

![14](14.png)

![15](15.png)

![16](16.png)

![17](17.png)

## 六、使用创建好的模板

![18](18.png)

![19](19.png)

![20](20.png)



## 七、检测数据

![21](21.png)



## 八、导出、导入模板

![22](22.png)

![23](23.png)

![24](24.png)

ps：模板导入使用的前提是客户端有相关key的配置文件，且客户端服务重启