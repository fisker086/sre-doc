# 常见yaml文件详解

## 一、yaml与json

> 在线yaml与json编辑器：http://www.bejson.com/validators/yaml_editor

### 1、json

#### 1.格式

```json
{ "人员名单'":
  { "张三": { "年龄": 18, "职业": "Linux运维工程师", "爱好": [ "看书", "学习", "加班" ] }, #
    "李四": { "年龄": 20, "职业": "Java开发工程师", "爱好": [ "开源技术", "微服务", "分布式存储"] } } }
```

#### 2.特点

```bash
json 不能注释
json 可读性较差
json 语法很严格
比较适用于API 返回值，也可用于配置文件,例如：jenkins
```

### 2、yaml

#### 1.格式

```yaml
人员名单:
  张三:
    年龄: 18 #
    职业: Linux运维工程师
    爱好:
      - 看书
      - 学习
      - 加班
  李四:
    年龄: 20
    职业: Java开发工程师 # 这是职业
    爱好:
      - 开源技术
      - 微服务
      - 分布式存储
```

#### 2.特点

```bash
大小写敏感
使用缩进表示层级关系
缩进时不允许使用Tal键，只允许使用空格
缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
使用”#” 表示注释，从这个字符一直到行尾，都会被解析器忽略
比json更适用于配置文件
yaml格式：
```

>k8s中的yaml文件以及其他场景的yaml文件，大部分都是以下类型：
> 上下级关系
> 列表
> 键值对(也称为maps，即key:value 格式的键值对数据

## 二、有助编写的工具方式

### 1、查看api接口版本

```bash
root@k8s-master1:~# kubectl api-versions
admissionregistration.k8s.io/v1
apiextensions.k8s.io/v1
apiregistration.k8s.io/v1
apps/v1
authentication.k8s.io/v1
authorization.k8s.io/v1
autoscaling/v1
autoscaling/v2
autoscaling/v2beta2
batch/v1
certificates.k8s.io/v1
coordination.k8s.io/v1
discovery.k8s.io/v1
events.k8s.io/v1
flowcontrol.apiserver.k8s.io/v1beta1
flowcontrol.apiserver.k8s.io/v1beta2
networking.k8s.io/v1
node.k8s.io/v1
policy/v1
rbac.authorization.k8s.io/v1
scheduling.k8s.io/v1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
velero.io/v1
```

### 2、查看api接口使用详细信息

```bash
root@k8s-master1:~# kubectl api-resources
资源类型                           简称         接口版本                                是否依赖名称空间   yaml文件里面kind的值          
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
events                            ev           v1                                     true         Event
limitranges                       limits       v1                                     true         LimitRange
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv           v1                                     false        PersistentVolume
pods                              po           v1                                     true         Pod
podtemplates                                   v1                                     true         PodTemplate
replicationcontrollers            rc           v1                                     true         ReplicationController
resourcequotas                    quota        v1                                     true         ResourceQuota
secrets                                        v1                                     true         Secret
serviceaccounts                   sa           v1                                     true         ServiceAccount
services                          svc          v1                                     true         Service
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io/v1              false        APIService
controllerrevisions                            apps/v1                                true         ControllerRevision
daemonsets                        ds           apps/v1                                true         DaemonSet
deployments                       deploy       apps/v1                                true         Deployment
replicasets                       rs           apps/v1                                true         ReplicaSet
statefulsets                      sts          apps/v1                                true         StatefulSet
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io/v1                true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io/v1                false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io/v1                false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io/v1                false        SubjectAccessReview
horizontalpodautoscalers          hpa          autoscaling/v2                         true         HorizontalPodAutoscaler
cronjobs                          cj           batch/v1                               true         CronJob
jobs                                           batch/v1                               true         Job
certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
leases                                         coordination.k8s.io/v1                 true         Lease
endpointslices                                 discovery.k8s.io/v1                    true         EndpointSlice
events                            ev           events.k8s.io/v1                       true         Event
flowschemas                                    flowcontrol.apiserver.k8s.io/v1beta2   false        FlowSchema
prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io/v1beta2   false        PriorityLevelConfiguration
ingressclasses                                 networking.k8s.io/v1                   false        IngressClass
ingresses                         ing          networking.k8s.io/v1                   true         Ingress
networkpolicies                   netpol       networking.k8s.io/v1                   true         NetworkPolicy
runtimeclasses                                 node.k8s.io/v1                         false        RuntimeClass
poddisruptionbudgets              pdb          policy/v1                              true         PodDisruptionBudget
clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io/v1           true         RoleBinding
roles                                          rbac.authorization.k8s.io/v1           true         Role
priorityclasses                   pc           scheduling.k8s.io/v1                   false        PriorityClass
csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
csinodes                                       storage.k8s.io/v1                      false        CSINode
csistoragecapacities                           storage.k8s.io/v1                      true         CSIStorageCapacity
storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
backups                                        velero.io/v1                           true         Backup
backupstoragelocations            bsl          velero.io/v1                           true         BackupStorageLocation
deletebackuprequests                           velero.io/v1                           true         DeleteBackupRequest
downloadrequests                               velero.io/v1                           true         DownloadRequest
podvolumebackups                               velero.io/v1                           true         PodVolumeBackup
podvolumerestores                              velero.io/v1                           true         PodVolumeRestore
resticrepositories                             velero.io/v1                           true         ResticRepository
restores                                       velero.io/v1                           true         Restore
schedules                                      velero.io/v1                           true         Schedule
serverstatusrequests              ssr          velero.io/v1                           true         ServerStatusRequest
volumesnapshotlocations                        velero.io/v1                           true         VolumeSnapshotLocation
```

### 3、编写帮助

```bash
root@k8s-master1:~# kubectl explain Deployment
KIND:     Deployment
VERSION:  apps/v1

DESCRIPTION:
     Deployment enables declarative updates for Pods and ReplicaSets.

FIELDS:
   apiVersion   <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata     <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec <Object>
     Specification of the desired behavior of the Deployment.

   status       <Object>
     Most recently observed status of the Deployment.

root@k8s-master1:~# kubectl explain Deployment.spec
KIND:     Deployment
VERSION:  apps/v1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior of the Deployment.

     DeploymentSpec is the specification of the desired behavior of the
     Deployment.

FIELDS:
   minReadySeconds      <integer>
     Minimum number of seconds for which a newly created pod should be ready
     without any of its container crashing, for it to be considered available.
     Defaults to 0 (pod will be considered available as soon as it is ready)

   paused       <boolean>
     Indicates that the deployment is paused.

   progressDeadlineSeconds      <integer>
     The maximum time in seconds for a deployment to make progress before it is
     considered to be failed. The deployment controller will continue to process
     failed deployments and a condition with a ProgressDeadlineExceeded reason
     will be surfaced in the deployment status. Note that progress will not be
     estimated during the time a deployment is paused. Defaults to 600s.

   replicas     <integer>
     Number of desired pods. This is a pointer to distinguish between explicit
     zero and not specified. Defaults to 1.

   revisionHistoryLimit <integer>
     The number of old ReplicaSets to retain to allow rollback. This is a
     pointer to distinguish between explicit zero and not specified. Defaults to
     10.

   selector     <Object> -required-
     Label selector for pods. Existing ReplicaSets whose pods are selected by
     this will be the ones affected by this deployment. It must match the pod
     template's labels.

   strategy     <Object>
     The deployment strategy to use to replace existing pods with new ones.

   template     <Object> -required-
     Template describes the pods that will be created
```

### 4、官方文档搜索

> https://kubernetes.io/docs/home/

## 三、常见资源清单解释

### 1、pod

```bash
apiVersion: v1 # 必选，指定api接口资源版本
kind: Pod    # 必选，定义资源接口类型/角色。pod为容器资源
metadata:    # 必选，定义资源的元数据信息
  name: nginx    # 必选，定义资源名称，在同一个namespace中必须是唯一的
  namespace: web-testing # 可选，不指定默认为default，指定资源所在的命名空间
  labels:    # 可选，定义资源标签
    - app: nginx
  annotations:    # 可选，注释列表
    - app: nginx
spec:    # 必选，用于定义容器的详细信息
  containers:    # 必选，容器列表
  - name: nginx    # 必选，符合RFC 1035规范的容器名称
    image: nginx:v1 # 必选，容器所用的镜像的地址
    imagePullPolicy: Always    # 可选，镜像拉取策略
    workingDir: /usr/share/nginx/html    # 可选，容器的工作目录
    volumeMounts:    # 可选，存储卷配置
    - name: webroot # 存储卷名称
      mountPath: /usr/share/nginx/html # 挂载目录
      readOnly: true    # 只读
    ports:    # 可选，容器需要暴露的端口号列表
    - name: http    # 端口名称
      containerPort: 80    # 端口号
      protocol: TCP    # 端口协议，默认TCP
    env:    # 可选，环境变量配置
    - name: TZ    # 变量名
      value: Asia/Shanghai #变量
    - name: LANG
      value: en_US.utf8
    resources:    # 可选，资源限制和资源请求限制
      limits:    # 最大限制设置
        cpu: 1000m
        memory: 1024MiB
      requests:    # 启动所需的资源
        cpu: 100m
        memory: 512MiB
    readinessProbe: # 可选，容器状态检查
      httpGet:    # 检测方式
        path: /    # 检查路径
        port: 80    # 监控端口
      timeoutSeconds: 2    # 超时时间 
      initialDelaySeconds: 60    # 初始化时间
    livenessProbe:    # 可选，监控状态检查
      exec:    # 检测方式
        command: 
        - cat
        - /health
      httpGet:    # 检测方式
        path: /_health
        port: 8080
        httpHeaders:
        - name: end-user
          value: jason
      tcpSocket:    # 检测方式
        port: 80
      initialDelaySeconds: 60    # 初始化时间
      timeoutSeconds: 2    # 超时时间
      periodSeconds: 5    # 检测间隔
      successThreshold: 2 # 检查成功为2次表示就绪
      failureThreshold: 1 # 检测失败1次表示未就绪
    securityContext:    # 可选，限制容器不可信的行为
      provoleged: false
  restartPolicy: Always    # 可选，默认为Always
  nodeSelector:    # 可选，指定Node节点
    region: subnet7
  imagePullSecrets:    # 可选，拉取镜像使用的secret
  - name: default-dockercfg-86258
  hostNetwork: false    # 可选，是否为主机模式，如是，会占用主机端口
  volumes:    # 共享存储卷列表
  - name: webroot # 名称，与上述对应
    emptyDir: {}    # 共享卷类型，空
    hostPath:        # 共享卷类型，本机目录
      path: /etc/hosts
    secret:    # 共享卷类型，secret模式，一般用于密码
      secretName: default-token-tf2jp # 名称
      defaultMode: 420 # 权限
      configMap:    # 一般用于配置文件
      name: nginx-conf
      defaultMode: 420
```

### 2、Nginx

```yaml
kind: Deployment  #类型，是deployment控制器，kubectl explain  Deployment
apiVersion: extensions/v1beta1  #API版本，# kubectl explain  Deployment.apiVersion
metadata: #pod的元数据信息，kubectl explain  Deployment.metadata
  labels: #自定义pod的标签，# kubectl explain  Deployment.metadata.labels
    app: nginx-deployment-label #标签名称为app值为nginx-deployment-label，后面会用到此标签 
  name: nginx-deployment #pod的名称
  namespace: nginx #pod的namespace，默认是defaule
spec: #定义deployment中容器的详细信息，kubectl explain  Deployment.spec
  replicas: 1 #创建出的pod的副本数，即多少个pod，默认值为1
  selector: #定义标签选择器
    matchLabels: #定义匹配的标签，必须要设置
      app: nginx-selector #匹配的目标标签，
  template: #定义模板，必须定义，模板是起到描述要创建的pod的作用
    metadata: #定义模板元数据
      labels: #定义模板label，Deployment.spec.template.metadata.labels
        app: nginx-selector #定义标签，等于Deployment.spec.selector.matchLabels
    spec: #定义pod信息
      containers: #定义pod中容器列表，可以多个至少一个，pod不能动态增减容器
      - name: nginx-container #容器名称
        image: harbor.trevorwu.com/nginx/nginx-web1:v1 #镜像地址
        #command: ["/apps/tomcat/bin/run_tomcat.sh"] #容器启动执行的命令或脚本
        #imagePullPolicy: IfNotPresent
        imagePullPolicy: Always #拉取镜像策略
        ports: #定义容器端口列表
        - containerPort: 80 #定义一个端口
          protocol: TCP #端口协议
          name: http #端口名称
        - containerPort: 443 #定义一个端口
          protocol: TCP #端口协议
          name: https #端口名称
        env: #配置环境变量
        - name: "password" #变量名称。必须要用引号引起来
          value: "123456" #当前变量的值
        - name: "age" #另一个变量名称
          value: "18" #另一个变量的值
        resources: #对资源的请求设置和限制设置
          limits: #资源限制设置，上限
            cpu: 500m  #cpu的限制，单位为core数，可以写0.5或者500m等CPU压缩值
            memory: 2Gi #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
          requests: #资源请求的设置
            cpu: 200m #cpu请求数，容器启动的初始可用数量,可以写0.5或者500m等CPU压缩值
            memory: 512Mi #内存请求大小，容器启动的初始可用数量，用于调度pod时候使用
---
kind: Service #类型为service
apiVersion: v1 #service API版本， service.apiVersion
metadata: #定义service元数据，service.metadata
  labels: #自定义标签，service.metadata.labels
    app: nginx #定义service标签的内容
  name: nginx-spec #定义service的名称，此名称会被DNS解析
  namespace: nginx #该service隶属于的namespaces名称，即把service创建到哪个namespace里面
spec: #定义service的详细信息，service.spec
  type: NodePort #service的类型，定义服务的访问方式，默认为ClusterIP， service.spec.type
  ports: #定义访问端口， service.spec.ports
  - name: http #定义一个端口名称
    port: 80 #service 80端口
    protocol: TCP #协议类型
    targetPort: 80 #目标pod的端口
    nodePort: 30001 #node节点暴露的端口
  - name: https #SSL 端口
    port: 443 #service 443端口
    protocol: TCP #端口协议
    targetPort: 443 #目标pod端口
    nodePort: 30043 #node节点暴露的SSL端口
  selector: #service的标签选择器，定义要访问的目标pod
    app: nginx #将流量路到选择的pod上，须等于Deployment.spec.selector.matchLabels
```

### 3、Tomcat

```bash
kind: Deployment   # 必选，定义资源接口类型/角色。pod为容器资源
#apiVersion: extensions/v1beta1 # API接口版本
apiVersion: apps/v1 # API接口版本
metadata: # 必选，定义资源的元数据信息
  labels: # 可选，定义资源标签
    app: tomcat-app1-deployment-label #标签名称为app值为tomcat-app1-deployment-label，后面会用到此标签 
  name: tomcat-app1-deployment # 必选，定义资源名称，在同一个namespace中必须是唯一的
  namespace: tomcat # 可选，不指定默认为default，指定资源所在的命名空间
spec: #定义deployment中容器的详细信息，kubectl explain  Deployment.spec
  replicas: 1  #创建出的pod的副本数，即多少个pod，默认值为1
  selector: #定义标签选择器
    matchLabels: #定义匹配的标签，必须要设置
      app: tomcat-app1-selector #匹配的目标标签，
  template: #定义模板，必须定义，模板是起到描述要创建的pod的作用
    metadata: #定义模板元数据
      labels: #定义模板label，Deployment.spec.template.metadata.labels
        app: tomcat-app1-selector #定义标签，等于Deployment.spec.selector.matchLabels
    spec: #定义pod信息
      containers: #定义pod中容器列表，可以多个至少一个，pod不能动态增减容器
      - name: tomcat-app1-container #容器名称
        image: tomcat:7.0.94-alpine #镜像地址
        #command: ["/apps/tomcat/bin/run_tomcat.sh"] #容器启动执行的命令或脚本
        #imagePullPolicy: IfNotPresent #如果本地有，优先使用本地
        imagePullPolicy: Always #拉取镜像策略，创建重新拉取镜像
        ports: #定义容器端口列表
        - containerPort: 8080 #定义一个容器端口
          protocol: TCP #端口协议
          name: http #端口名称
        env: #配置环境变量
        - name: "password" #变量名称。必须要用引号引起来
          value: "123456" #当前变量的值
        - name: "age" #另一个变量名称
          value: "18" #另一个变量的值
        resources: #对资源的请求设置和限制设置
          limits: #资源限制设置，上限
            cpu: 1  #cpu的限制，单位为core数，可以写0.5或者500m等CPU压缩值
            memory: "512Mi" #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
          requests: #资源请求的设置
            cpu: 500m #cpu请求数，容器启动的初始可用数量,可以写0.5或者500m等CPU压缩值
            memory: "512Mi" #内存请求大小，容器启动的初始可用数量，用于调度pod时候使用
---
kind: Service #类型为service
apiVersion: v1 #service API版本， service.apiVersion
metadata: #定义service元数据，service.metadata
  labels: #自定义标签，service.metadata.labels
    app: tomcat-app1-service-label #定义service标签的内容
  name: tomcat-app1-service #定义service的名称，此名称会被DNS解析
  namespace: tomcat #该service隶属于的namespaces名称，即把service创建到哪个namespace里面
spec: #定义service的详细信息，service.spec
  #type: NodePort #service的类型，定义服务的访问方式，默认为ClusterIP， service.spec.type
  ports: #定义访问端口， service.spec.ports
  - name: http #定义一个端口名称
    port: 80 #service 80端口
    protocol: TCP #协议类型
    targetPort: 8080 #目标pod端口
    #nodePort: 40003 # 宿主机端口
  selector: #service的标签选择器，定义要访问的目标pod
    app: tomcat-app1-selector #将流量路到选择的pod上，须等于Deployment.spec.selector.matchLabels
```

### 4、常见服务资源使用

```bash
nginx #静态服务器 
        2C/2G
        1C/1G
java #动态服务 2C/2G
              2c/4G
php  2C/2G

go/python  1C/2G 1C/1G 
job/cronjob 0.3/512Mi
elastisearch  4C/12G 
mysql          4C/8G
```

