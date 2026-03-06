# Operator部署

## 一、官方文档

https://github.com/nacos-group/nacos-k8s/blob/master/operator/README-CN.md

## 二、下载并安装Operator

```bash
wget https://codeload.github.com/nacos-group/nacos-k8s/zip/refs/heads/master
unzip nacos-k8s-master.zip
cd nacos-k8s-master/operator
helm install nacos-operator ./chart/nacos-operator -n nacos
```

## 三、集群版部署

### 1、资源清单准备

#### 1.nacos.yaml

```yaml
apiVersion: nacos.io/v1alpha1
kind: Nacos
metadata:
  name: nacos
spec:
  type: cluster
  image: nacos/nacos-server:v3.1.1
  replicas: 3
  env:
    - name: DOMAIN_NAME
      value: s8030.gmbaifa.online
    # 是否启用鉴权功能
    - name: NACOS_AUTH_ENABLE
      value: "true"
    # JWT Token 签名的密钥（Secret Key) openssl rand -base64 32 生成
    - name: NACOS_AUTH_TOKEN
      value: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    # 服务端之间通信的身份标识 Key（HTTP Header 名称）
    - name: NACOS_AUTH_IDENTITY_KEY
      value: "nacos"
    # 服务端之间通信的身份标识 Value（HTTP Header 值）
    - name: NACOS_AUTH_IDENTITY_VALUE
      value: "nacos"
  database:
    type: mysql
    mysqlHost: tidb-tidb.tidb-cluster
    mysqlDb: nacos
    mysqlUser: root
    mysqlPort: "4000" # tiDB服务，默认值3306
    mysqlPassword: "123456"
```

#### 2.ingres

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: nacos
spec:
  ingressClassName: apisix
  http:
  - name: nacos
    match:
      hosts:
      - nacos.gmbaifa.online
      paths:
      - /*
    backends:
    - serviceName: nacos-headless
      servicePort: 8080
    plugins:
      - name: redirect
        enable: false
        config:
          http_to_https: false
```

### 2、安装

```bash
sql初始化
https://raw.githubusercontent.com/alibaba/nacos/develop/distribution/conf/mysql-schema.sql
```



```bash
kubectl apply -f nacos.yaml -n nacos
```

### 3、报错解决

#### 1.解释器修改（临时解决）

```bash
kubectl logs -f -n nacos nacos-0 
sh: syntax error: unexpected "("

kubectl patch statefulset nacos -n nacos --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/command/0", "value": "bash"}]'
```

#### 2.Operator bug修复

##### 1）修改源码

```bash
vim operator/pkg/service/operator/Kind.go
const CONSOLE_PORT = 8080 # 打开8080端口
...
var initScrit = `array=(%s)
succ=0

for element in ${array[@]}
do
  while true
  do
    ping -c 1 $element > /dev/stdout  # 改这里  官方命令不支持最新镜像
...
				{
					Name:     "console",
					Port:     CONSOLE_PORT, # 打开8080端口
					Protocol: "TCP",
				},
...
	for i := 0; i < int(*nacos.Spec.Replicas); i++ {
		serivce = fmt.Sprintf("%v%v-%d.%v.%v.%v.%v:%v ", serivce, e.generateName(nacos), i, e.generateHeadlessSvcName(nacos), nacos.Namespace, "svc", domain, NACOS_PORT)
		serivceNoPort = fmt.Sprintf("%v%v-%d.%v.%v.%v.%v ", serivceNoPort, e.generateName(nacos), i, e.generateHeadlessSvcName(nacos), nacos.Namespace, "svc", domain)
	}
	serivce = serivce[0 : len(serivce)-1]
	serivceNoPort = serivceNoPort[0 : len(serivceNoPort)-1] # 增加这行，解决serivceNoPort 的尾随空格，当 ping 空元素时变成：ping  -c 1，-c 被当作主机名了！
	env := []v1.EnvVar{
		{
			Name:  "NACOS_SERVERS",
			Value: serivce,
		},
	}
	ss.Spec.Template.Spec.Containers[0].Env = append(ss.Spec.Template.Spec.Containers[0].Env, env...)
	// 先检查域名解析再启动
	ss.Spec.Template.Spec.Containers[0].Command = []string{"bash", "-c", fmt.Sprintf("%s&&bin/docker-startup.sh", fmt.Sprintf(initScrit, serivceNoPort))}  # 修改sh 为bash 永久解决问题1
	return ss

```

##### 2）构建修改镜像

```bash
export IMG=registry.cn-shanghai.aliyuncs.com/gmqsbf/nacos-operator:v1.0.1-baifa
make docker-build IMG=$IMG
make docker-push IMG=$IMG
```

##### 3）修改operator/chart/nacos-operator/values.yaml镜像地址

>更改镜像然后重新部署Operator