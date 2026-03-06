# HPA自动伸缩

> ​    在生产环境中，总会有一些意想不到的事情发生，比如公司网站流量突然升高，此时之前创建的Pod已不足以撑住所有的访问，而运维人员也不可能24小时守着业务服务，这时就可以通过配置HPA，实现负载过高的情况下自动扩容Pod副本数以分摊高并发的流量，当流量恢复正常后，HPA会自动缩减Pod的数量。HPA是根据CPU的使用率、内存使用率自动扩展Pod数量的，所以要使用HPA就必须定义Requests参数。

## 一、Pod伸缩介绍

>- 根据当前pod的负载，动态调整 pod副本数量，业务高峰期自动扩容pod的副本数以尽快响应pod的请求。
>- 在业务低峰期对pod进行缩容，实现降本增效的目的。
>- 公有云支持node级别的弹性伸缩。

## 二、动态伸缩控制器类型

### 1、水平pod自动缩放器(HPA)：

> 基于pod 资源利用率横向调整pod副本数量。

### 2、垂直pod自动缩放器(VPA)：

> 基于pod资源利用率，调整对单个pod的最大资源限制，不能与HPA同时使用。

### 3、集群伸缩(Cluster Autoscaler,CA)

> 基于集群中node 资源分配情况，动态伸缩node节点，从而保证有CPU和内存资源用于创建pod。

## 三、创建HPA



```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: 10m
              memory: 50Mi
            requests:
              cpu: 10m
              memory: 50Mi
---
kind: Service
apiVersion: v1
metadata:
  name: svc
  namespace: default
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: autoscaling/v1 
kind: HorizontalPodAutoscaler
metadata:
  name: docs
  namespace: default
spec:
  # HPA的最小pod数量和最大pod数量
  maxReplicas: 10
  minReplicas: 1
  # HPA的伸缩对象描述，HPA会动态修改该对象的pod数量
  scaleTargetRef:  #定义水平伸缩的目标对象
    kind: Deployment #目标对象类型为deployment
    name: nginx #deployment 的具体名称
    apiVersion: apps/v1
  # 监控的指标数组，支持多种类型的指标共存
  metrics:
    - type: Resource
      #  核心指标，包含cpu和内存两种（被弹性伸缩的pod对象中容器的requests和limits中定义的指标。）
      resource:
        name: cpu
        # CPU 阈值
        # 计算公式：所有目标pod的metric的使用率（百分比）的平均值，
        # 例如limit.cpu=1000m，实际使用500m，则utilization=50%
        # 例如deployment.replica=3, limit.cpu=1000m，则pod1实际使用cpu=500m, pod2=300m, pod=600m
        ## 则averageUtilization=(500/1000+300/1000+600/1000)/3 = (500 + 300 + 600)/(3*1000)）
        targetAverageUtilization: 50 #CPU使用率
        
      resource: #定义资源
        name: memory #资源名称为memory
        targetAverageValue: 1024Mi #memory使用率
```

