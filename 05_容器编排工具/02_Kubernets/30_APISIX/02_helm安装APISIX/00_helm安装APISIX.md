# helm安装APISIX

>https://artifacthub.io/packages/helm/apisix/apisix

## 一、values.yaml准备

```yaml
apisix:
  ssl:
    enabled: true
  enableServerTokens: false
timezone: "Asia/Shanghai"

service:
  type: NodePort
  http:
    nodePort: 80
  tls:
    nodePort: 443

dashboard:
  enabled: true
  service:
    type: NodePort
    port: 80
  config:
    authentication:
      users:
        - username: admin
          password: admin
  replicaCount: 1

admin:
  credentials:
    admin: <访问凭证>
    viewer: <访问凭证>

ingress-controller:
  clusterDomain: "集群域名"
  enabled: true
  replicaCount: 1
  gateway:
    type: NodePort
  config:
    log_level: debug
    apisix:
      adminAPIVersion: "v3"
      serviceNamespace: <部署的k8s名称空间>
      adminKey: "<访问凭证>"

etcd:
  clusterDomain: "集群域名"
  enable: true
  replicaCount: 1
  persistence:
    size: 6Gi
    storageClass: nfs-storage-cw

rbac:
  create: true

global:
  storageClass: "nfs-storage-cw"

```

## 二、部署

### 1、配源

```bash
helm repo add apisix https://charts.apiseven.com
```

### 2、安装

```bash
helm install apisix apisix/apisix --version 2.8.1 --create-namespace --namespace apisix -f values.yaml
```

### 3、更新

```yaml
helm upgrade apisix apisix/apisix --version 2.8.1 --create-namespace --namespace apisix -f values.yaml
```

### 4、删除

```bash
helm delete -n apisix apisix
```