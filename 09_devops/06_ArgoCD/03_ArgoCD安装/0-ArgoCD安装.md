# ArgoCD安装

## 一、准备K8s环境

>略

## 二、获取版本

>去Github上查看最新版本
>
>写这篇文章的时候，更新到v2.7.5

```bash
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
echo $VERSION
```



## 三、安装

### 1、创建名称空间

```bash
kubectl create namespace gitops
```

### 2、部署Argocd集群

#### 1.非高可用部署

```bash
kubectl apply -n gitops -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### 2.高可用部署

```bash
kubectl apply -n gitops -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

```bash
kubectl apply -n gitops -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.7.5/manifests/ha/install.yaml
```

**ps：镜像拉不了，想法用代理拉取。容器起不来更改TZ时区为集群内时区**

#### 3.遇到的问题解决

##### 1）argocd-repo-server服务起不来

**日志**

```bash
kubectl logs -f -n argocd argocd-repo-server-9548f48c8-fw4l2
time="2023-06-21T13:49:09+08:00" level=info msg="ArgoCD Repository Server is starting" built="2023-06-16T14:35:08Z" commit=a2430af1c356b283e5e3fc5bde1f5e2b5199f258 port=8081 version=v2.7.5+a2430af.dirty
time="2023-06-21T13:49:09+08:00" level=info msg="Generating self-signed TLS certificate for this session"
time="2023-06-21T13:49:09+08:00" level=info msg="Initializing GnuPG keyring at /app/config/gpg/keys"
time="2023-06-21T13:49:09+08:00" level=info msg="gpg --no-permission-warning --logger-fd 1 --batch --gen-key /tmp/gpg-key-recipe3491814707" dir= execID=8752a
time="2023-06-21T13:49:15+08:00" level=error msg="`gpg --no-permission-warning --logger-fd 1 --batch --gen-key /tmp/gpg-key-recipe3491814707` failed exit status 2" execID=8752a
time="2023-06-21T13:49:15+08:00" level=info msg=Trace args="[gpg --no-permission-warning --logger-fd 1 --batch --gen-key /tmp/gpg-key-recipe3491814707]" dir= operation_name="exec gpg" time_ms=6024.097901
time="2023-06-21T13:49:15+08:00" level=fatal msg="`gpg --no-permission-warning --logger-fd 1 --batch --gen-key /tmp/gpg-key-recipe3491814707` failed exit status 2"
```



**修改部署文件**

```yaml
...
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: repo-server
    app.kubernetes.io/name: argocd-repo-server
    app.kubernetes.io/part-of: argocd
  name: argocd-repo-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: argocd-repo-server
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: argocd-repo-server
              topologyKey: topology.kubernetes.io/zone
            weight: 100
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app.kubernetes.io/name: argocd-repo-server
            topologyKey: kubernetes.io/hostname
      automountServiceAccountToken: false
      containers:
      - args:
        - /usr/local/bin/argocd-repo-server
        env:
        - name: ARGOCD_REDIS
          valueFrom:
            configMapKeyRef:
              key: redis.server
              name: argocd-cmd-params-cm
        - name: ARGOCD_RECONCILIATION_TIMEOUT
          valueFrom:
            configMapKeyRef:
              key: timeout.reconciliation
              name: argocd-cm
              optional: true
        - name: ARGOCD_REPO_SERVER_LOGFORMAT
          valueFrom:
            configMapKeyRef:
              key: reposerver.log.format
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_REPO_SERVER_LOGLEVEL
          valueFrom:
            configMapKeyRef:
              key: reposerver.log.level
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_REPO_SERVER_PARALLELISM_LIMIT
          valueFrom:
            configMapKeyRef:
              key: reposerver.parallelism.limit
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_REPO_SERVER_DISABLE_TLS
          valueFrom:
            configMapKeyRef:
              key: reposerver.disable.tls
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_TLS_MIN_VERSION
          valueFrom:
            configMapKeyRef:
              key: reposerver.tls.minversion
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_TLS_MAX_VERSION
          valueFrom:
            configMapKeyRef:
              key: reposerver.tls.maxversion
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_TLS_CIPHERS
          valueFrom:
            configMapKeyRef:
              key: reposerver.tls.ciphers
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_REPO_CACHE_EXPIRATION
          valueFrom:
            configMapKeyRef:
              key: reposerver.repo.cache.expiration
              name: argocd-cmd-params-cm
              optional: true
        - name: REDIS_SERVER
          valueFrom:
            configMapKeyRef:
              key: redis.server
              name: argocd-cmd-params-cm
              optional: true
        - name: REDIS_COMPRESSION
          valueFrom:
            configMapKeyRef:
              key: redis.compression
              name: argocd-cmd-params-cm
              optional: true
        - name: REDISDB
          valueFrom:
            configMapKeyRef:
              key: redis.db
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_DEFAULT_CACHE_EXPIRATION
          valueFrom:
            configMapKeyRef:
              key: reposerver.default.cache.expiration
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_REPO_SERVER_OTLP_ADDRESS
          valueFrom:
            configMapKeyRef:
              key: otlp.address
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_REPO_SERVER_MAX_COMBINED_DIRECTORY_MANIFESTS_SIZE
          valueFrom:
            configMapKeyRef:
              key: reposerver.max.combined.directory.manifests.size
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_REPO_SERVER_PLUGIN_TAR_EXCLUSIONS
          valueFrom:
            configMapKeyRef:
              key: reposerver.plugin.tar.exclusions
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_REPO_SERVER_ALLOW_OUT_OF_BOUNDS_SYMLINKS
          valueFrom:
            configMapKeyRef:
              key: reposerver.allow.oob.symlinks
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_REPO_SERVER_STREAMED_MANIFEST_MAX_TAR_SIZE
          valueFrom:
            configMapKeyRef:
              key: reposerver.streamed.manifest.max.tar.size
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_REPO_SERVER_STREAMED_MANIFEST_MAX_EXTRACTED_SIZE
          valueFrom:
            configMapKeyRef:
              key: reposerver.streamed.manifest.max.extracted.size
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_GIT_MODULES_ENABLED
          valueFrom:
            configMapKeyRef:
              key: reposerver.enable.git.submodule
              name: argocd-cmd-params-cm
              optional: true
        - name: HELM_CACHE_HOME
          value: /helm-working-dir
        - name: HELM_CONFIG_HOME
          value: /helm-working-dir
        - name: HELM_DATA_HOME
          value: /helm-working-dir
        image: quay.io/argoproj/argocd:v2.7.6
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz?full=true
            port: 8084
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
        name: argocd-repo-server
        ports:
        - containerPort: 8081
        - containerPort: 8084
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8084
          initialDelaySeconds: 5
          periodSeconds: 10
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          seccompProfile:
            type: Unconfined                            ##############修改这里
...
```



### 4、时区问题

```bash
env:
  - name: TZ
    value: "Asia/Shanghai"
```



### 3、配置ingress

>Argo CD 会运行一个 gRPC 服务（由 CLI 使用）和 HTTP/HTTPS 服务（由 UI 使用），这两种协议都由 `argocd-server` 服务在以下端口进行暴露：
>
>- 443 - gRPC/HTTPS
>- 80 - HTTP（重定向到 HTTPS）

>我们可以通过配置 Ingress 的方式来对外暴露服务，其他 Ingress 控制器的配置可以参考官方文档 https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/ 进行配置。

>Argo CD 在同一端口 (443) 上提供多个协议 (gRPC/HTTPS)，所以当我们为 argocd 服务定义单个 nginx ingress 对象和规则的时候有点麻烦，因为 nginx.ingress.kubernetes.io/backend -protocol 这个 annotation 只能接受一个后端协议（例如 HTTP、HTTPS、GRPC、GRPCS）。

>为了使用单个 ingress 规则和主机名来暴露 Argo CD APIServer，必须使用 nginx.ingress.kubernetes.io/ssl-passthrough 这个 annotation 来传递 TLS 连接并校验 Argo CD APIServer 上的 TLS。

#### 1.方案一

```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: https
```

>上述规则在 Argo CD APIServer 上校验 TLS，该服务器检测到正在使用的协议，并做出适当的响应。请注意，nginx.ingress.kubernetes.io/ssl-passthrough 注解要求将 --enable-ssl-passthrough 标志添加到 nginx-ingress-controller 的命令行参数中。

#### 2.方案二

>由于 `ingress-nginx` 的每个 Ingress 对象仅支持一个协议，因此另一种方法是定义两个 Ingress 对象。一个用于 HTTP/HTTPS，另一个用于 gRPC。

```yaml
# 如下所示为 HTTP/HTTPS 的 Ingress 对象：
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-http-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: http
      host: argocd.k8s.local
  tls:
    - hosts:
        - argocd.k8s.local
      secretName: argocd-secret # do not change, this is provided by Argo CD
---
# gRPC 协议对应的 Ingress 对象如下所示：
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-grpc-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: https
      host: grpc.argocd.k8s.local
  tls:
    - hosts:
        - grpc.argocd.k8s.local
      secretName: argocd-secret # do not change, this is provided by Argo CD
```

```bash
kubectl apply -f ingress.yaml -n gitops
```

>然后我们需要在禁用 TLS 的情况下运行 APIServer。编辑 argocd-server 这个 Deployment 以将 --insecure 标志添加到 argocd-server 命令，或者简单地在 argocd-cmd-params-cm ConfigMap 中设置 server.insecure: "true" 即可。
>
>创建完成后，我们就可以通过 argocd.k8s.local 来访问 Argo CD 服务了，不过需要注意我们这里配置的证书是自签名的，所以在第一次访问的时候会提示不安全，强制跳转即可。

>如果redis连接超时，把k8s中argocd所有的网络策略删除掉看看

#### 3.直接服务暴漏

```bash
kubectl patch svc -n argocd argocd-server -p '{"spec": {"type": "NodePort"}}'
kubectl get svc -n argocd argocd-server -o jsonpath='{.spec.ports[0].nodePort}'
```

### 4、获取初始密码

> 默认情况下 `admin` 帐号的初始密码是自动生成的，会以明文的形式存储在 Argo CD 安装的命名空间中名为 `argocd-initial-admin-secret` 的 Secret 对象下的 `password` 字段下，我们可以用下面的命令来获取：

```bash
kubectl -n gitops get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### 5、argocd cli安装

```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.7.6/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

### ArgoCD自动部署成功
<font color="info">k8s集群名称</font>: {{.app.spec.destination.server}}
<font color="info">环境名称</font>: {{.app.spec.destination.namespace}}
<font color="info">服务名称</font>: {{.app.metadata.name}}
<font color="info">更新镜像</font>: {{.app.status.summary.images}}
<font color="info">app同步状态</font>: {{.app.status.operationState.phase}}
<font color="info">app服务状态</font>: {{.app.status.health.status}}
<font color="info">时间</font>: {{.app.status.operationState.startedAt}}
<font color="info">URL</font>: [点击跳转ArgoCD]({{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true)

