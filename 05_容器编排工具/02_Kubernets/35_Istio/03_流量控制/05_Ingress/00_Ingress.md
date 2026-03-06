# Ingress

## 一、基本概念

>服务的访问入口，接收外部请求并转发到后端服务
>
>Istio 的 Ingress gateway 和 Kubernetes Ingress 的区别
>
>>Kubernetes: 针对L7协议（资源受限），可定义路由规则
>>
>>Istio: 针对 L4-6 协议，只定义接入点，复用 Virtual Service 的 L7 路由定义

## 二、目标

>为 httpbin 服务配置 Ingress 网关
>
>理解 Istio 实现自己的 Ingress 的意义
>
>复习 Gateway 的配置方法
>
>复习 Virtual Service 的配置方法

## 三、实操

```bash
kubectl apply -f samples/httpbin/httpbin.yaml
```

>vs.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin

```

>gateway.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin

```







