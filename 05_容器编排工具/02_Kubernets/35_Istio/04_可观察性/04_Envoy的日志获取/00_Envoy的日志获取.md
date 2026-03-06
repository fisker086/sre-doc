# Envoy的日志获取

## 一、目标

>通过查看 Envoy 日志了解流量信息
>
>学会通过分析日志调试流量异常

## 二、实战

### 1、日志内容获取

>istio configmap修改

```bash
--set values.global.proxy.accessLogFile="/dev/stdout"
```

![image-20250610171743691](image-20250610171743691.png)

![image-20250610171827447](image-20250610171827447.png)

### 2、日志项分析

![image-20250610171935076](image-20250610171935076.png)

![image-20250610171945273](image-20250610171945273.png)

### 3、Envoy 流量五元组

![image-20250610172121015](image-20250610172121015.png)

### 4、调试关键字段：RESPONSE_FLAGS

>UH：upstream cluster 中没有健康的 host，503
>UF：upstream 连接失败，503
>UO：upstream overflow（熔断）
>NR：没有路由配置，404
>URX：请求被拒绝因为限流或最大连接次数
>
>...

## 三、Envoy 日志配置项

![image-20250610172538987](image-20250610172538987.png)

