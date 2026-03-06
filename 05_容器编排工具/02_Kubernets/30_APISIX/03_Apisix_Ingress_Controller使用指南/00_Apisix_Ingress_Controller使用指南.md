# Apisix_Ingress_Controller使用指南

## 一、ingress.class示例

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: apisix-dashboard
  annotations:
    kubernetes.io/ingress.class: apisix
spec:
  rules:
  - host: apisix-dashboard.example.com
    http:
      paths:
      - backend:
          serviceName: apisix-dashboard
          servicePort: 80
        path: /*
        pathType: Prefix
```

>首先确定apifix-ingress-controller配置文件中ingress_class的值， 默认为apisix

>注意如果要匹配跟下面的所有路径，需要将path配置为`\*`, 也可以配置`pathType: Prefix`会创建`\`、`\*`两个路径其它的用法完全符合ingress的默认配置，annotation可配置参数参考[官方文档](https://apisix.apache.org/zh/docs/ingress-controller/concepts/annotations/)

## 二、crd基础示例

### 1、httpbin服务部署

```bash
kubectl run httpbin --image kennethreitz/httpbin --port 80
kubectl expose pod httpbin --port 80
```

### 2、ApisixRoute基本用法

#### 1.部署

>可以使用两种路径类型prefix、exact,默认为exact，如果prefix需要，只需附加一个，例如，/id/匹配前缀为 的所有路径/id/。

>01_httpbin-route.yaml

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
    - name: rule1
      match:
        hosts:
          - httpbin.example.com
        paths:
          - /*
      backends:
        - serviceName: httpbin
          servicePort: 80
```

#### 2.测试

```bash
curl http://httpbin.example.com/headers -v
* Host httpbin.example.com:80 was resolved.
* IPv6: (none)
* IPv4: 192.168.6.101
*   Trying 192.168.6.101:80...
* Connected to httpbin.example.com (192.168.6.101) port 80
> GET /headers HTTP/1.1
> Host: httpbin.example.com
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: application/json
< Content-Length: 160
< Connection: keep-alive
< Date: Wed, 21 Aug 2024 09:49:22 GMT
< Access-Control-Allow-Origin: *
< Access-Control-Allow-Credentials: true
< Server: APISIX
<
{
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.example.com",
    "User-Agent": "curl/8.5.0",
    "X-Forwarded-Host": "httpbin.example.com"
  }
}
* Connection #0 to host httpbin.example.com left intact

```

### 3、ApisixTls基本用法

>ApisixTls主要用来维护https证书
>
>需要先创建tls secret（可以看下一篇文章，用Cert Manager签发自动续签证书）

```bash
kubectl -n default create secret tls example-tls -tls-cert --key=tls.key --cert=tls.crt
```

>使用证书

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixTls
metadata:
  name: httpbin-tls
#  annotations:
#    # 指定关联的 ClusterIssuer
#    cert-manager.io/cluster-issuer: letsencrypt-issuer
spec:
  hosts:
    - "test.dudewu.top"
  secret:
    name: example-tls
    namespace: default
```

### 4、ApisixUpstream基本用法

>ApisixUpstream 需要配置成和Kubernetes Service相同的名字，通过添加负载均衡、健康检查、重试、超时参数等使 Kubernetes Service 更加丰富。

>如果不使用ApisixUpstream，则loadbalancer的类型为roundrobin
>
>*其中loadbalancer支持如下四种类型*
>
>Round robin: *轮询，配置为`type: roundrobin`
>
>Least loaded:最小链接,配置为`type: least_conn`
>
>Peak EWMA:维护每个副本往返时间的移动平均值，按未完成请求的数量加权，并将**流量**分配到成本函数最小的副本。配置为`type: ewma`
>
>chash:一致性哈希 配置为`type: chash`

```bash
apiVersion: apisix.apache.org/v2
kind: ApisixUpstream
metadata:
  name: httpbin
spec:
  loadbalancer:
    type: ewma
---
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpbin
spec:
  http:
    - name: rule1
      match:
        hosts:
          - httpbin.dudewu.top
        paths:
          - /*
      backends:
        - serviceName: httpbin
          servicePort: 8080
```

## 三、ApisixUpstream高级用法

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixUpstream
metadata:
  name: httpbin
  namespace: httpbin
spec:
  # 重试次数
  retries: 222
  # 对应 "type" 字段
  loadbalancer:
    type: ewma
  # 对应 "timeout" 配置
  timeout:
    # 连接超时
    connect: 6s
    # 发送超时
    send: 6s
    # 接收超时
    read: 6s
  # 对应 "scheme" 字段
  scheme: http
  # 对应 "pass_host" 字段
  passHost: pass
  # 对应 "checks" 配置，健康检查配置（可选）
  healthCheck:
    active:
      type: http
      # 健康检查的 HTTP 路径
      httpPath: /
      host: httpbin
      port: 8080
      # 健康检查的超时时间
      timeout: 1
      # 健康检查的并发数量
      concurrency: 10
      healthy:
        # 健康状态检查的间隔时间
        interval: 1s
        # 判定健康所需的连续成功次数
        successes: 2
        # 判定健康的 HTTP 状态码
        httpCodes:
          - 200
          - 302
      unhealthy:
        # 不健康状态检查的间隔时间
        interval: 1s
        # 判定不健康所需的 HTTP 失败次数
        httpFailures: 5
        # 判定不健康的 TCP 失败次数
        tcpFailures: 2
        # 判定不健康的超时次数
        timeouts: 3
        # 判定不健康的 HTTP 状态码
        httpCodes:
          - 429
          - 404
          - 500
          - 501
          - 502
          - 503
          - 504
          - 505
    passive:
      type: http
      healthy:
        # 被动健康检查的成功次数
        successes: 5
        # 被动健康检查的 HTTP 状态码
        httpCodes:
          - 200
          - 201
          - 202
          - 203
          - 204
          - 205
          - 206
          - 207
          - 208
          - 226
          - 300
          - 301
          - 302
          - 303
          - 304
          - 305
          - 306
          - 307
          - 308
      unhealthy:
        # 被动不健康检查的 HTTP 失败次数
        httpFailures: 2
        # 被动不健康检查的 TCP 失败次数
        tcpFailures: 2
        # 被动不健康检查的超时次数
        timeouts: 7
        # 被动不健康检查的 HTTP 状态码
        httpCodes:
          - 429
          - 500
          - 503
```

## 四、ApisixRoute使用进阶

### 1、ApisixRoute配置多个域名和路径

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: rule1
    match:
      hosts:
      - local.example.com
      - local01.example.com
      paths:
      - /*
      - /api
    backends:
       - serviceName: httpbin
         servicePort: 80

```

### 2、路径`/api`特殊配制，路径`/`无特殊配制

>apisix中会创建两个route，分别是httpserver-route 和 default-route

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /api*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    plugins:
    - name: proxy-rewrite 
      enable: true
      config: 
        regex_uri: ["^/api(/|$)(.*)", "/$2"]
  - name: default-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
```

### 3、ApisixRoute添加插件

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpbin-route
spec:
  http:
    - name: httpbin
      match:
        hosts:
        - local.httpbin.org
        paths:
          - /*
      backends:
      - serviceName: foo
        servicePort: 80
      plugins:
        - name: gzip
          enable: true
```

### 4、HTTP强跳HTTPS

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    plugins:
    - name: redirect
      enable: true
      config: 
        http_to_https: true
```

### 5、域名跳转

>local.example.com跳转到local01.example.com

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    plugins:
    - name: redirect
      enable: true
      config: 
        uri: "https://local01.example.com$request_uri"
```

### 6、rewrite路径跳转

#### 1./api/header 转/header

##### 1）方法一： 使用redirect插件，页面会发生302跳转

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    plugins:
    - name: redirect
      enable: true
      config: 
        regex_uri: ["^/api(/|$)(.*)", "/$2"] 
```

##### 2）方法二： 使用proxy-rewrite 插件，页面不会302跳转

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    plugins:
    - name: proxy-rewrite 
      enable: true
      config: 
        regex_uri: ["^/api(/|$)(.*)", "/$2"] 
```

#### 2./header转/api/header

>使用proxy-rewrite 插件，页面不会302跳转

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    plugins:
    - name: proxy-rewrite 
      enable: true
      config: 
        uri: /api/$uri
```

### 7、基于用户名和密码的认证

>basix-auth插件需要与 Consumer 一起使用才能实现该功能。 开启认证的apisixroute会自动匹配用户名

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  name: httpserver-basicauth
spec:
  authParameter:
    basicAuth:
      value:
        username: admin
        password: admin
---
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    authentication:
      enable: true 
      type: basicAuth
```

>测试：

```bash
curl -i -uadmin:admin https://local.example.com/
```

### 8、基于apikey的认证

>key-auth插件需要与Consumer一起使用才能实现该功能。 开启认证的apisixroute会自动匹配key

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  name: httpserver-basicauth
spec:
  authParameter:
    keyAuth:
      value:
        key: admin
---
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    authentication:
      enable: true 
      type: keyAuth
```

>测试:

```bash
curl -H 'apikey:admin' https://local.example.com/ 
```

### 9、限制站点的用户名和密码

>有没有办法用户名和密码只针对一个站点生效, 创建两个consumer，分别是admin01和宫秀德，只允许admin可以进行认证。

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  name: httpserver-basicauth-admin
spec:
  authParameter:
    basicAuth:
      value:
        username: admin
        password: admin
---
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  name: httpserver-basicauth-admin01
spec:
  authParameter:
    basicAuth:
      value:
        username: admin01
        password: admin01
---
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    plugins:
    - name: consumer-restriction
      enable: true
      config:
        whitelist: 
        - ingress_apisix_httpserver_basicauth_admin 
    authentication:
      enable: true 
      type: basicAuth
```

>httpserver-basicauth-admin需要添加ingress-controller添加的默认前缀ingress_apisix，否则会报403，所以全名为ingress_apisix_httpserver_basicauth_admin

测试admin01访问

```bash
> curl -i -uadmin01:admin01 https://local.example.com/
HTTP/2 403
date: Tue, 30 Aug 2022 08:00:04 GMT
content-type: text/plain; charset=utf-8
server: APISIX/2.15.0

{"message":"The consumer_name is forbidden."}
```

测试admin访问

```bash
curl -i -uadmin:admin https://local.example.com/
HTTP/2 200
content-type: text/html; charset=utf-8
content-length: 615
date: Tue, 30 Aug 2022 07:55:55 GMT
last-modified: Wed, 25 May 2022 10:01:40 GMT
etag: "628dfe84-267"
accept-ranges: bytes
server: APISIX/2.15.0
.........
```

### 10、基于IP的白名单配置

>ip-restriction插件实现该功能, 只允许`10.10.50.207` `10.0.0.0/16` 访问站点

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    plugins:
    - name: ip-restriction
      enable: true
      config:
        whitelist: 
        - 10.10.50.207
        - 10.0.0.0/16
```

>白名单中地址测试正常打开页面
>
>非白名单地址测试

```bash
> curl  https://local.example.com/
{"message":"Your IP address is not allowed"}
```

### 11、基于IP的黑名单限制

>ip-restriction插件实现该功能, 不允许`10.10.50.207` 访问站点

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: httpserver-route
    match:
      hosts:
      - local.example.com
      paths:
      - /*
    backends:
    - serviceName: httpbin
      servicePort: 8000
    plugins:
    - name: ip-restriction
      enable: true
      config:
        blacklist: 
        - 10.10.50.207
```

测试

```bash
> curl  https://local.example.com/
{"message":"Your IP address is not allowed"}
```

## 五、灰度发布

### 1、准备环境

#### 1.stable版本

```yaml
# vim s1-stable.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-stable-service
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 80
    name: http-port
  selector:
    app: myapp
    version: stable
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/public-registry-fzh/myapp:v1
        imagePullPolicy: IfNotPresent
        name: myapp-stable
        ports:
        - name: http-port
          containerPort: 80
        env:
        - name: APP_ENV
          value: stable
```

#### 2.canary版本

```yaml
# vim s2-canary.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-canary-service
  namespace: canary
spec:
  ports:
  - port: 80
    targetPort: 80
    name: http-port
  selector:
    app: myapp
    version: canary
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
  namespace: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/public-registry-fzh/myapp:v2
        imagePullPolicy: IfNotPresent
        name: myapp-canary
        ports:
        - name: http-port
          containerPort: 80
        env:
        - name: APP_ENV
          value: canary
```

### 2、基于weight分流

```yaml
# vim s3-apisixroute-weight.yaml 
---
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: myapp-canary-apisixroute
  namespace: canary 
spec:
  http:
  - name: myapp-canary-rule
    match:
      hosts:
      - myapp.example.com
      paths:
      - /
    backends:
    - serviceName: myapp-stable-service
      servicePort: 80
      weight: 10
    - serviceName: myapp-canary-service
      servicePort: 80
      weight: 1
```

创建服务和路由等

```bash
➜  kubectl apply -f ./
service/myapp-stable-service unchanged
deployment.apps/myapp-stable unchanged
service/myapp-canary-service unchanged
deployment.apps/myapp-canary unchanged
apisixroute.apisix.apache.org/myapp-canary-apisixroute unchanged
➜  kubectl get pods -n canary
NAME                           READY   STATUS    RESTARTS   AGE
myapp-canary-9ff4c55f9-ndjrs   1/1     Running   0          15s
myapp-stable-5fdf6bd75-sq8df   1/1     Running   0          75s
➜  kubectl get apisixroute -n canary
NAME                       HOSTS                   URIS   AGE
myapp-canary-apisixroute   [myapp.example.com]   [/]    97s
```

>测试weight灰度
>
>Stable和 Canary 的比例约为10:1

```bash
  ~ for i in `seq 30`;do curl https://myapp.example.com;done
myapp:v1
myapp:v1
.......
myapp:v2
myapp:v1
myapp:v1
myapp:v2
myapp:v1
myapp:v1
myapp:v1
.......
myapp:v1
myapp:v1
myapp:v2
myapp:v1
myapp:v1
myapp:v2
```

### 3、基于优先级分流

```yaml
# vim s4-priority.yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: myapp-canary-apisixroute2
  namespace: canary
spec:
  http:
  - name: myapp-stable-rule2
    priority: 1
    match:
      hosts:
      - myapp.example.com
      paths:
      - /
    backends:
    - serviceName: myapp-stable-service
      servicePort: 80
  - name: myapp-canary-rule2
    priority: 2
    match:
      hosts:
      - myapp.example.com
      paths:
      - /
    backends:
    - serviceName: myapp-canary-service
      servicePort: 80
```

测试，流量会优先打入优先级高的pod

```bash
➜  ~ for i in `seq 30`;do curl https://myapp.example.com;done
myapp:v2
myapp:v2
........
myapp:v2
myapp:v2
myapp:v2
myapp:v2
```

### 4、基于header分流

```yaml
# vim  canary-header.yaml 
---
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: myapp-canary-apisixroute3
  namespace: canary 
spec:
  http:
  - name: myapp-stable-rule3
    priority: 1
    match:
      hosts:
      - myapp.example.com
      paths:
      - /
    backends:
    - serviceName: myapp-stable-service
      servicePort: 80
  - name: myapp-canary-rule3
    priority: 2
    match:
      hosts:
      - myapp.example.com
      paths:
      - /
      exprs:
      - subject:
          scope: Header
          name: canary
        op: RegexMatch
        value: ".*myapp.*"
    backends:
    - serviceName: myapp-canary-service
      servicePort: 80
```

测试

```bash
➜  ~ curl https://myapp.example.com
myapp:v1
➜  ~ curl https://myapp.example.com
myapp:v1
➜  ~ curl https://myapp.example.com
myapp:v1
➜  ~ curl https://myapp.example.com -X GET -H "canary: 124myapp"
myapp:v2
➜  ~ curl https://myapp.example.com -X GET -H "canary: myapp"
myapp:v2
➜  ~ curl https://myapp.example.com -X GET -H "canary: xiamyapp"
myapp:v2
➜  ~ curl https://myapp.example.com -X GET -H "stable: xiamyapp"
myapp:v1
➜  ~ curl https://myapp.example.com -X GET -H "stable: myapp"
myapp:v1
```

### 5、基于参数的分流

```yaml
# cat vars.yaml 
---
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: myapp-canary-apisixroute3
  namespace: canary 
spec:
  http:
  - name: myapp-stable-rule3
    priority: 1
    match:
      hosts:
      - myapp.example.com
      paths:
      - /
    backends:
    - serviceName: myapp-stable-service
      servicePort: 80
  - name: myapp-canary-rule3
    priority: 2
    match:
      hosts:
      - myapp.example.com
      paths:
      - /
      exprs:
      - subject:
          scope: Query
          name: id
        op: In
        set:
        - "12"
        - "23"
        - "45"
        - "67"
    backends:
    - serviceName: myapp-canary-service
      servicePort: 80
```

测试

```bash
➜  ~ curl https://myapp.example.com
myapp:v1
➜  ~ curl https://myapp.example.com
myapp:v1
➜  ~ curl https://myapp.example.com
myapp:v1
➜  ~ curl https://myapp.example.com
myapp:v1
➜  ~ curl https://myapp.example.com/\?id\=12
myapp:v2
➜  ~ curl https://myapp.example.com/\?id\=23
myapp:v2
➜  ~ curl https://myapp.example.com/\?id\=45
myapp:v2
➜  ~ curl https://myapp.example.com/\?id\=67
myapp:v2
➜  ~ curl https://myapp.example.com/\?id\=89
myapp:v1
➜  ~ curl https://myapp.example.com/\?id\=143
myapp:v1
```

### 6、基于cookie分流

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: myapp-canary-apisixroute
  namespace: default 
spec:
  http:
  - name: myapp-stable-rule3
    priority: 1
    match:
      hosts:
      - myapp.example.com
      paths:
      - /
    backends:
    - serviceName: myapp-stable-service
      servicePort: 80
  - name: myapp-canary-rule3
    priority: 2
    match:
      hosts:
      - myapp.example.com
      paths:
      - /
      exprs:
      - subject:
          scope: Cookie
          name: canary_v5
        op: Equal
        value: "always"
    backends:
    - serviceName: myapp-canary-service
      servicePort: 80
```

测试

```bash
➜  ~ curl --cookie "canary_v5=always" "http://myapp.example.com"
myapp:v2

➜  ~ curl  "http://myapp.example.com"
myapp:v1
```
