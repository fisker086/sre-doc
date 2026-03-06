# k8s安装APISIX-Ingress-Controller

## 一、官方介绍

> https://apisix.apache.org/zh/blog/2023/10/18/ingress-apisix/

## 二、helm安装

>https://apisix.apache.org/zh/docs/ingress-controller/install/

### 1、配置源和获取模板

```bash
helm repo add apisix https://charts.apiseven.com
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 导出apisix values.yaml模板
helm show values apisix/apisix > apisix-values.yaml
helm show values apisix/apisix-ingress-controller > apisix-ingress-controller-values.yaml

# 官方默认安装
helm install apisix \
  --namespace ingress-apisix \
  --create-namespace \
  --set apisix.deployment.role=traditional \
  --set apisix.deployment.role_traditional.config_provider=yaml \
  --set etcd.enabled=false \
  --set ingress-controller.enabled=true \
  --set ingress-controller.config.provider.type=apisix-standalone \
  --set ingress-controller.apisix.adminService.namespace=ingress-apisix \
  --set ingress-controller.gatewayProxy.createDefault=true \
  apisix/apisix
```

### 2、修改values.yaml

```yaml
timezone: "Asia/Shanghai"
image:
  repository: apache/apisix
  pullPolicy: IfNotPresent
  tag: 3.13.0-ubuntu
apisix:
  deployment:
    replicas: 2
    role: "traditional"
    role_traditional:
      # enum: etcd, yaml
      config_provider: "yaml"
  ssl:
    enabled: true
    enableHTTP3: true
    sslProtocols: "TLSv1.1 TLSv1.2 TLSv1.3"
  admin:
    # -- Enable Admin API
    enabled: true
    credentials:
      # -- Apache APISIX admin API admin role credentials
      admin: xxxxxxxxxxxxxxxxxxxxxxxxx
      # -- Apache APISIX admin API viewer role credentials
      viewer: xxxxxxxxxxxxxxxxxxxxxxxxxx
etcd:
  enabled: false
service:
  http:
    enabled: true
    servicePort: 80
    nodePort: 80
    containerPort: 9080
    additionalContainerPorts: []
  tls:
    servicePort: 443
    nodePort: 443
  stream:
    enabled: false
    tcp: []
    udp: []

ingress-controller:
  enabled: true
  deployment:
    image:
      repository: apache/apisix-ingress-controller
      pullPolicy: IfNotPresent
      tag: "2.0.0-rc3"
  config:
    provider:
      type: "apisix-standalone"
  apisix:
    adminService:
      namespace: ingress-apisix
  gatewayProxy:
    createDefault: true
    provider:
      type: ControlPlane
      controlPlane:
        endpoints: [ ]
        auth:
          type: AdminKey
          adminKey:
            value: "xxxxxxxxxxxxxxxxxxxxxxxxxx"
```

### 3、指定values.yaml方式安装

```bash
helm install apisix \
  --namespace ingress-apisix \
  --create-namespace \
  -f values.yaml \
  apisix/apisix
```

>更新

```bash
helm upgrade apisix \
  --namespace ingress-apisix \
  -f values.yaml \
  apisix/apisix
```

