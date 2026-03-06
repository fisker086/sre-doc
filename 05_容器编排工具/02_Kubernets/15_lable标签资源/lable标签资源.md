# Label标签

## 一、介绍

```bash
	Label是kubernetes系统中的一个重要概念。它的作用就是在资源上添加标识，用来对它们进行区分和选择。Label的特点:

	一个Label会以key/value键值对的形式附加到各种对象上，如Node、Pod、Service等等
	一个资源对象可以定义任意数量的Label，同一个Label也可以被添加到任意数量的资源对象上去.
	Label通常在资源对象定义时确定，当然也可以在对象创建后勃态添加或者删除
	可以通过Label实现资源的多维度分组，以便灵活、方便地进行资源分配、调度、配置、部署等管理工作。
	
	k8s当做标签是用来管理（识别一系列）容器，方便与管理和监控拥有同一标签的所有容器
```



## 二、常见标签

```bash
版本标签："release":"stable","release":"canary"
环境标签："environment":"dev","environment":"production"
架构标签："tier":"frontend","tier":"backend","tier":"middleware"
分区标签："partition":"customerA","partition":"customerB"
质量管控标签："track":"daily","track":"weekly"
```



## 三、使用

### 1、编写yaml

```bash
root@k8s-master-01:~/k8s-study.d# vim test1.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-tag
  labels:
    release: stable
spec:
  containers:
    - name: nginx
      image: nginx

# 应用
root@k8s-master-01:~/k8s-study.d# kubectl create -f test.yaml
pod/test-tag created
```



### 2、查看label

```bash
root@k8s-master-01:~/k8s-study.d# kubectl get pod  --show-labels
NAME       READY   STATUS    RESTARTS   AGE   LABELS
test-tag   1/1     Running   0          64s   release=stable
```



### 3、添加标签

```bash
# 添加
root@k8s-master-01:~/k8s-study.d# kubectl label pod test-tag app=tag
pod/test-tag labeled

# 查看
root@k8s-master-01:~/k8s-study.d# kubectl get pod  --show-labels
NAME       READY   STATUS    RESTARTS   AGE    LABELS
test-tag   1/1     Running   0          2m3s   app=tag,release=stable
```



### 4、删除标签

```bash
root@k8s-master-01:~/k8s-study.d# kubectl label pod test-tag app-
pod/test-tag unlabeled
root@k8s-master-01:~/k8s-study.d# kubectl get pod  --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
test-tag   1/1     Running   0          3m11s   release=stable
```
