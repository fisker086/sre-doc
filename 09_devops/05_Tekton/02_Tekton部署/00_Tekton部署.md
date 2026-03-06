# Tekton部署

## 一、安装 Tekton 管道

### 1、安装

```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

### 2、查看

```bash
kubectl get pods --namespace tekton-pipelines --watch
```

## 二、安装 Tekton Dashboard

### 1、安装

```bash
curl -sL https://raw.githubusercontent.com/tektoncd/dashboard/main/scripts/release-installer | bash -s --install latest --read-write
```

### 2、检查运行情况

```bash
kubectl get pods --namespace tekton-pipelines --watch
```

## 三、安装 Tekton cli

```bash
curl -LO https://github.com/tektoncd/cli/releases/download/v0.34.0/tkn_0.34.0_Linux_x86_64.tar.gz
sudo tar xvzf tkn_0.34.0_Linux_x86_64.tar.gz -C /usr/local/bin/ tkn
```

## 四、运行第一个task示例

### 1、准备task和taskrun

#### 1.hello-task.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: echo-1
      image: alpine
      script: |
        #!/bin/sh
        echo "Hello alpine"
    - name: echo-2
      image: alpine
      script: |
        #!/bin/sh
        echo "Hello alpine:3.16"
```

#### 2.hello-taskrun.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: hello-task-run-2
spec:
  taskRef:
    name: hello
```

### 2、运行

```bash
kubectl apply -f hello-taskrun.yaml -f hello-task.yaml
```

### 3、查看Taskrun和日志

```bash
kubectl get taskrun
kubectl logs --selector=tekton.dev/taskRun=hello-run-lxz78
```

### 4、Tkn命令行运行查看

```bash
tkn task start hello 
tkn taskrun list
tkn taskrun logs -f hello-run-lxz78
```