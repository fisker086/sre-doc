# Apisix和cert-manager自动签发https证书

## 一、安装 Cert Manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager --namespace cert-manager --set prometheus.enabled=false --set crds.enabled=true --create-namespace

helm delete cert-manager -n cert-manager



# 离线安装
https://charts.jetstack.io/charts/cert-manager-v1.15.3.tgz

helm install cert-manager ./cert-manager-v1.15.3.tgz --namespace cert-manager --set prometheus.enabled=false --set crds.enabled=true
```

## 二、安装cert-manager-alidns-webhook

>[AliDNS-Webhook](https://github.com/pragkent/alidns-webhook)已经使用的api过于老旧，所以报错。且长时间没人维护，故使用[cert-manager-alidns-webhook](https://github.com/DEVmachine-fr/cert-manager-alidns-webhook)

### 1、安装工具

```bash
helm repo add cert-manager-alidns-webhook https://devmachine-fr.github.io/cert-manager-alidns-webhook
helm repo update
helm install alidns-webhook cert-manager-alidns-webhook/alidns-webhook --namespace=cert-manager --set groupName=dudewu.top


helm delete alidns-webhook -n cert-manager


# 离线安装
https://github.com/DEVmachine-fr/cert-manager-alidns-webhook/releases
```

### 2、创建阿里云 AccessKey ID 和 AccessKey Secret 的 secret

```bash
kubectl create secret generic alidns-secrets -n apisix --from-literal="access-token=xxxxxxxxxx" --from-literal="secret-key=xxxxxxxxxxxxxx"
```

### 3、RBAC权限添加

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cert-manager-alidns-solver
rules:
- apiGroups: ["dudewu.top"]
  resources: ["alidns-solver"]
  verbs: ["create", "update", "delete", "get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cert-manager-alidns-solver-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cert-manager-alidns-solver
subjects:
- kind: ServiceAccount
  name: cert-manager
  namespace: apisix
```

### 4、编写ClusterIssuer资源清单

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
  namespace: apisix
spec:
  acme:
    email: xxxxxxxxxx@163.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # 正式申请证书的API
    # server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - dns01:
        webhook:
            config:
              accessTokenSecretRef:
                key: access-token
                name: alidns-secrets
              regionId: cn-beijing
              secretKeySecretRef:
                key: secret-key
                name: alidns-secrets
            groupName: dudewu.top # groupName must match the one configured on webhook deployment (see Helm chart's values) !
            solverName: alidns-solver
```

## 三、使用

### 1、创建证书

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  # 自定义个名字
  name: test
  # 此处写你要使用证书的命名空间
  namespace: apisix
spec:
  # 要签发证书的域名，替换成你自己的
  dnsNames:
    - "test.dudewu.top"
  issuerRef:
    kind: ClusterIssuer
    # 引用 ClusterIssuer，名字和 clusterIssuer.yaml 中保持一致
    name: letsencrypt
  # 最终签发出来的证书会保存在这个 Secret 里面
  secretName: test
  # 证书有效期，默认时间90天，可申请最短时间是1小时
  duration: 2160h
  # 距离证书到期多久续订证书，如果未设置，则默认为已颁发证书生存期的1/3。最小可接受值为5分钟。
  renewBefore: 360h
```

### 2、apisix引用证书

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixTls
metadata:
  name: httpbin1-tls
#  annotations:
#    # 指定关联的 ClusterIssuer
#    cert-manager.io/cluster-issuer: letsencrypt-issuer
spec:
  hosts:
    - "test.dudewu.top"
  secret:
    name: test-tls
    namespace: default
```

### 3、提取证书

```bash
kubectl get secrets -n default test.dudewu.top -o yaml | grep tls.key | awk -F [:\ ] '{print $5}' | base64 -d
kubectl get secrets -n default test.dudewu.top -o yaml | grep tls.crt | awk -F [:\ ] '{print $5}' | base64 -d
```

