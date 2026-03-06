# container-manager控制器

## 一、介绍

```bash
	k8s内拥有许多的控制器类型，用来控制pod的状态、行为、副本数量等等，控制器通过pod的标签来控制pod。
	Pod通过控制器实现应用的运维，如伸缩、升级等，控制器决定了创建pod资源的方式和类型，在集群上管理和运行容器的对象通过label-selector 相关联。
```



## 二、控制器常见类型

```bash
1、Deployment:一般用来部署长期运行的、无状态的应用
		特点：集群之中，随机部署
		
2、DaemonSet：每一个节点上部署一个Pod，删除节点自动删除对应的POD（zabbix-agent）
		特点：每一台上有且只有一台
		
3、StatudfluSet: 部署有状态应用
		特点：有启动顺序
```



## 三、Deployment

```bash
	在Deployment对象中描述所需的状态，然后Deployment控制器将实际状态以受控的速率更改为所需的状态。
	部署无状态应用
```



### 1、创建Depoyment资源

#### 1.编写yaml文件

```yaml
root@k8s-master-01:~/k8s-study.d# kubectl create -f test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  # 名称
  name: deployment
spec:
  # 创建副本数
  replicas: 1
  # 定义Deployment控制器如何找到要管理Pod，与 template 的 label（标签）对应。
  selector:
    matchLabels:
      release: stable
  template:
    metadata:
      name: test-tag
      # nginx 使用 label（标签）标记 Pod
      labels:
        release: stable
    # 表示 Pod 运行一个名字为 nginx 的容器
    spec:
      containers:
        - name: nginx
          image: nginx
```



#### 2、执行创建命令

```bash
root@k8s-master-01:~/k8s-study.d# kubectl apply -f test.yaml
deployment.apps/deployment created

root@k8s-master-01:~/k8s-study.d# kubectl get deployments.apps
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
deployment   1/1     1            1           34s
```



#### 3、查看部署状态

```bash
[root@k8s-master-01 ~]# kubectl get pod
NAME                          READY   STATUS    RESTARTS   AGE
deployment-5849786498-4dnsb   1/1     Running   1          17h
```



#### 4、查看部署详情

```bash
root@k8s-master-01:~/k8s-study.d# kubectl describe deployments.apps deployment
Name:                   deployment
Namespace:              default
CreationTimestamp:      Mon, 08 Aug 2022 15:04:39 +0800
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
                        kubernetes.io/change-cause: kubectl apply --filename=test.yaml --record=true
Selector:               release=stable
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  release=stable
  Containers:
   nginx:
    Image:        nginx
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   deployment-744d5688bb (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  19m   deployment-controller  Scaled up replica set deployment-744d5688bb to 1
```



#### 6、弹性扩容的几种方式

##### 1.修改配置清单

```bash
root@k8s-master-01:~/k8s-study.d# kubectl edit deployments.apps deployment
...
spec:
  progressDeadlineSeconds: 600
  replicas: 3
...

#查看
root@k8s-master-01:~/k8s-study.d# kubectl get pod -w
NAME                          READY   STATUS              RESTARTS   AGE
deployment-744d5688bb-vzdbg   1/1     Running             0          23s
deployment-744d5688bb-wnvqt   1/1     Running             0          49m
deployment-744d5688bb-nbskz   1/1     Running             0          38s
^Croot@k8s-master-01:~/k8s-study.d# kubectl get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
deployment-744d5688bb-nbskz   1/1     Running   0          56s   10.99.98.130   k8s-node-02   <none>           <none>
deployment-744d5688bb-vzdbg   1/1     Running   0          56s   10.111.44.12   k8s-node-01   <none>           <none>
deployment-744d5688bb-wnvqt   1/1     Running   0          49m   10.111.44.11   k8s-node-01   <none>           <none>
```



##### 2.scale

```bash
root@k8s-master-01:~/k8s-study.d# kubectl scale deployment/deployment --replicas=1
deployment.apps/deployment scaled
```





##### 3.打标签

```bash
root@k8s-master-01:~/k8s-study.d# kubectl patch deployments.apps deployment -p '{"spec":{"replicas":6}}'
deployment.apps/deployment patched

#查看
root@k8s-master-01:~/k8s-study.d# kubectl get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
deployment-744d5688bb-blps9   1/1     Running   0          24s     10.111.44.14   k8s-node-01   <none>           <none>
deployment-744d5688bb-dhrz8   1/1     Running   0          24s     10.99.98.131   k8s-node-02   <none>           <none>
deployment-744d5688bb-f5hrk   1/1     Running   0          24s     10.111.44.15   k8s-node-01   <none>           <none>
deployment-744d5688bb-nbskz   1/1     Running   0          2m48s   10.99.98.130   k8s-node-02   <none>           <none>
deployment-744d5688bb-s7nc7   1/1     Running   0          25s     10.111.44.13   k8s-node-01   <none>           <none>
deployment-744d5688bb-sr5rq   1/1     Running   0          24s     10.99.98.132   k8s-node-02   <none>           <none>
```



##### 4.修改配置清单

```bash
root@k8s-master-01:~/k8s-study.d# vim test.yaml
...
spec:
  replicas: 2
...
root@k8s-master-01:~/k8s-study.d# kubectl apply -f test.yaml
deployment.apps/deployment configured

#查看
root@k8s-master-01:~/k8s-study.d# kubectl get pod -w
NAME                          READY   STATUS    RESTARTS   AGE
deployment-744d5688bb-dhrz8   1/1     Running   0          69m
deployment-744d5688bb-s7nc7   1/1     Running   0          69m
```





#### 7、更新镜像的几种方式

##### 1.打标签

```bash
root@k8s-master-01:~/k8s-study.d# kubectl patch deployments.apps deployment -p '{"spec":{"template":{"spec":{"containers":[{"image":"nginx:1.18.0","name":"nginx"}]}}}}'
deployment.apps/deployment patched

#查看
root@k8s-master-01:~/k8s-study.d# kubectl describe pod -n default deployment-59dd5fdcd6-98l47 | grep Image
    Image:          nginx:1.18.0
    Image ID:       docker.io/library/nginx@sha256:e90ac5331fe095cea01b121a3627174b2e33e06e83720e9a934c7b8ccc9c55a0
```



##### 2.修改配置清单

```bash
root@k8s-master-01:~/k8s-study.d# vim test.yaml
...
    spec:
      containers:
        - name: nginx
          image: nginx:1.19.0

...

# 应用
root@k8s-master-01:~/k8s-study.d# kubectl apply -f test.yaml
deployment.apps/deployment configured

# 验证
root@k8s-master-01:~/k8s-study.d# kubectl describe pod -n default deployment-65d8689449-w9rhp |grep Image:
    Image:          nginx:1.19.0
```



##### 3.设置镜像

```bash
root@k8s-master-01:~/k8s-study.d# kubectl set image deployment/deployment nginx=nginx:1.17.0
deployment.apps/deployment image updated

root@k8s-master-01:~/k8s-study.d# kubectl describe pod -n default deployment-7684c98759-xt8lz |grep Image:
    Image:          nginx:1.17.0
```



#### 8、回滚

##### 1.回滚到上一版本

```bash
root@k8s-master-01:~/k8s-study.d# kubectl rollout undo deployment deployment
deployment.apps/deployment rolled back

root@k8s-master-01:~/k8s-study.d# kubectl describe pod -n default deployment-65d8689449-tr7wb |grep Image:
    Image:          nginx:1.19.0

```



##### 2.查看历史版本

```bash
root@k8s-master-01:~/k8s-study.d# kubectl rollout history deployment deployment
deployment.apps/deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
4         <none>
6         <none>
7         <none>
```



##### 3.回滚到指定历史版本

```bash
root@k8s-master-01:~/k8s-study.d# kubectl rollout undo deployment deployment --to-revision=1
deployment.apps/deployment rolled back

root@k8s-master-01:~/k8s-study.d# kubectl describe pod -n default deployment-744d5688bb-t26d4 |grep Image:
    Image:          nginx
```



## 四、Statefulset

>Statefulset为了解决有状态服务的集群部署、集群之间的数据同步问题(MySQL主从等)
>Statefulset所管理的Pod拥有唯一且固定的Pod名称
>Statefulset按照顺序对pod进行创建、启停、伸缩和回收
>Headless Services(无头服务，请求的解析直接解析到pod IP)

```yaml
#apiVersion: extensions/v1beta1
apiVersion: apps/v1
kind: StatefulSet 
metadata:
  name: myserver-myapp
  namespace: myserver
spec:
  replicas: 3
  serviceName: "myserver-myapp-service"
  selector:
    matchLabels:
      app: myserver-myapp-frontend
  template:
    metadata:
      labels:
        app: myserver-myapp-frontend
    spec:
      containers:
      - name: myserver-myapp-frontend
        image: nginx:1.20.2-alpine 
        ports:
          - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: myserver-myapp-service
  namespace: myserver
spec:
  clusterIP: None
  ports:
  - name: http
    port: 80
  selector:
    app: myserver-myapp-frontend 
```



## 五、Daemonset

>https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/

```bash
在每一台节点上部署一个Pod，一般用来监控、收集日志。
```

### 1、创建DaemonSet

#### 1.zabbix-client

```yaml
root@k8s-master-01:~/k8s-study.d# kubectl apply -f test_Daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: zabbix-agent
spec:
  selector:
    matchLabels:
      app: zabbix-agent
  template:
    metadata:
      labels:
        app: zabbix-agent
    spec:
      containers:
        - name: zabbix-agent
          image: zabbix/zabbix-agent
```



```bash
root@k8s-master-01:~/k8s-study.d# kubectl apply -f test_Daemonset.yaml
daemonset.apps/zabbix-agent created
```

#### 2.fluentd

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # this toleration is to have the daemonset runnable on master nodes
      # remove it if your masters can't run pods
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

#### 3.prometheus

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring 
  labels:
    k8s-app: node-exporter
spec:
  selector:
    matchLabels:
        k8s-app: node-exporter
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
      containers:
      - image: prom/node-exporter:v1.3.1 
        imagePullPolicy: IfNotPresent
        name: prometheus-node-exporter
        ports:
        - containerPort: 9100
          hostPort: 9100
          protocol: TCP
          name: metrics
        volumeMounts:
        - mountPath: /host/proc
          name: proc
        - mountPath: /host/sys
          name: sys
        - mountPath: /host
          name: rootfs
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
      hostNetwork: true
      hostPID: true
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: "true"
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: monitoring 
spec:
  type: NodePort
  ports:
  - name: http
    port: 9100
    nodePort: 39100
    protocol: TCP
  selector:
    k8s-app: node-exporter
```

#### 4.nginx

> 作为业务访问入口，减少跨主机资源消耗

```yaml
---
#apiVersion: extensions/v1beta1
apiVersion: apps/v1
kind: DaemonSet 
metadata:
  name: myserver-myapp
  namespace: myserver
spec:
  selector:
    matchLabels:
      app: myserver-myapp-frontend
  template:
    metadata:
      labels:
        app: myserver-myapp-frontend
    spec:
      tolerations:
      # this toleration is to have the daemonset runnable on master nodes
      # remove it if your masters can't run pods
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      # 使用宿主机网络
      hostNetwork: true
      hostPID: true
      containers:
      - name: myserver-myapp-frontend
        image: nginx:1.20.2-alpine 
        ports:
          # 直接映射到宿主机的80端口
          - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: myserver-myapp-frontend
  namespace: myserver
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30018
    protocol: TCP
  type: NodePort
  selector:
    app: myserver-myapp-frontend

```



### 2、更新

#### 1.修改主机清单

```bash
root@k8s-master-01:~/k8s-study.d# kubectl edit daemonsets.apps zabbix-agent
daemonset.apps/zabbix-agent edited
```



#### 2.打标签的方式

```bash
root@k8s-master-01:~/k8s-study.d# kubectl patch daemonsets.apps zabbix-agent  -p '{"spec":{"template":{"spec":{"containers":[{"image":"zabbix/zabbix-agent:centos-5.2.4", "name":"zabbix-agent"}]}}}}'
daemonset.apps/zabbix-agent patched
```



#### 3.设置镜像

```bash
root@k8s-master-01:~/k8s-study.d# kubectl set image daemonset/zabbix-agent zabbix-agent=zabbix/zabbix-agent:centos-5.2.3
daemonset.apps/zabbix-agent image updated
```



### 3、回滚

#### 1.回滚到上一版本

```bash
root@k8s-master-01:~/k8s-study.d# kubectl rollout undo daemonset zabbix-agent 
daemonset.apps/zabbix-agent rolled back
```



#### 2.回滚到指定版本

```bash
root@k8s-master-01:~/k8s-study.d# kubectl rollout undo daemonset zabbix-agent --to-revision=1
daemonset.apps/zabbix-agent rolled back
```

