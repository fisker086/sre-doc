# Grafana查看系统

## 1、介绍

![image-20250610165846168](image-20250610165846168.png)

## 2、Istio Dashboard

### 1.Mesh Dashboard：查看应用（服务）数据

>网格数据总览
>
>服务视图
>
>工作负载视图

### 2.Performance Dashboard：查看 Istio 自身（各组件）数据

>Istio 系统总览
>
>各组件负载情况

## 3、实战

>192.168.6.101:3000

```bash
--set values.grafana.enabled=true
```

### 1.mesh

![image-20250610170229885](image-20250610170229885.png)

![image-20250610170342088](image-20250610170342088.png)

### 2.service

![image-20250610170436729](image-20250610170436729.png)

![image-20250610170445155](image-20250610170445155.png)

### 3.workload

![image-20250610170622096](image-20250610170622096.png)

![image-20250610170636625](image-20250610170636625.png)

### 4.Performance

![image-20250610170713605](image-20250610170713605.png)

![image-20250610171113549](image-20250610171113549.png)











