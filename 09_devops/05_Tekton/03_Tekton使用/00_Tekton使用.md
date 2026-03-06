# Tekton使用

## 一、运行多任务流水线

### 1、声明式运行

```bash
kubectl apply -f goodby-task.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: goodbye
spec:
  steps:
    - name: goodbye
      image: ubuntu
      imagePullPolicy: IfNotPresent
      script: |
        #!/bin/bash
        echo "Goodbye goodby"
```

```bash
kubectl apply -f pipeline1-hello-goodby.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-hello-goodbye
spec:
  tasks:
    - name: one
      taskRef:
        name: hello
    - name: two
      taskRef:
        name: goodbye
```

```bash
kubectl apply -f pipelinerun0-goodbye.yaml
```

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: pipelinerun-goodbye-0
spec:
  pipelineRef:
    name: pipeline-hello-goodbye
```

### 2、Tkn命令行运行

```bash
tkn pipelines start hello-goodbye 
tkn pipelineruns list 
tkn pipelinerun logs hello-goodbye-run-4b27q -f
```

## 二、使用Secret拉取代码

### 1、创建Secret

```bash
kubectl apply -f git-secret.yaml
```

```yaml
apiVersion: v1
data:
  password: bXVrZTY2NjY=
  username: cm9vdA==
kind: Secret
metadata:
  name: gitlab-secert
type: Opaque
```

### 2、创建task

```bash
kubectl apply -f git-clone-task.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone
spec:
  description: Clone the code repository to the workspace. 
  steps:
    - name: git-clones
      image: bitnami/git:latest
      imagePullPolicy: IfNotPresent
      env: 
        - name: git_username
          valueFrom: 
            secretKeyRef: 
              name: gitlab-secert
              key: username 
        - name: git_password
          valueFrom: 
            secretKeyRef: 
              name: gitlab-secert
              key: password       
      script: |
        echo ${git_username}${git_password}
        echo http://${git_username}:${git_password}@10.0.7.30/golang/go.git
        git clone -b main -v http://${git_username}:${git_password}@10.0.7.30/golang/go.git /workspace
        ls -la
        pwd 
    - name: list-files
      image: ubuntu
      imagePullPolicy: IfNotPresent
      command:
        - /bin/sh
      args: ['-c', 'ls -la /workspace && pwd']

```

### 3、创建Pipeline

```bash
kubectl apply -f pipeline2-git-clone.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: git-clone
spec:
  tasks:
    - name: git-clone
      taskRef:
        name: git-clone
```

## 三、taskSpec、taskRef、pipelineRef、pipelineSpec

### 1、介绍

>- taskSpec: 在流水线（Pipeline）中定义了任务的实际执行逻辑和行为。一个taskSpec可以包含多个步骤，每个步骤定义了一个特定的操作或任务
>- taskRef（任务引用）是指向已定义的Task的引用。它允许在流水线（Pipeline）中引用和复用现有的任务定义
>- pipelineSpec：pipelineSpec定义了一个流水线模板，它可以在PipelineRun资源类型下通过pipelineRef直接定义流水线
>- pipelineRef（流水线引用）：pipelineRef是指向已定义的流水线的引用。它允许在其他流水线或任务中引用和复用现有的流水线定义。

### 2、pipelineRef与taskSpec结合使用

#### 1.调用流程

>PipelineRun -> pipelineRef -> pipeline->taskSpec->(task1/task2/task3)）

![image-20250514160042574](image-20250514160042574.png)

#### 2.运行pipeline

```bash
kubectl apply -f pipeline3-taskSpec.yaml
tkn pipelines start security-scans
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: security-scans
spec:
  tasks:
    - name: scorecards
      taskSpec:
        steps:
          - image: alpine
            name: step-1
            script: |
              echo "Generating scorecard report ..."
    - name: codeql
      taskSpec:
        steps:
          - image: alpine
            name: step-1
            script: |
              echo "Generating codeql report ..."
```

### 3、pipelineRef与taskRef结合使用(推荐)

#### 1.调用流程

>PipelineRun->pipelineRef->Pipeline->TaskRef->Task

![image-20250514160142435](image-20250514160142435.png)

#### 2.运行pipeline

```bash
kubectl apply -f pipelinerun1-git-clone.yaml
```

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: pipelinerun-git-clone-1
spec:
  pipelineRef:
    name: git-clone
```

### 4、使用pipelineSpec与taskSpec结合使用

#### 1.调用流程

>PipelineRun->PipelineSpec->TaskSpec

![image-20250514160626445](image-20250514160626445.png)

#### 2.运行pipeline

```bash
kubectl apply -f  pipelinerun2-all.yaml
```

```yaml
apiVersion: tekton.dev/v1 # or tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: pipelinerun-all-1
spec:
  pipelineSpec:
    tasks:
      - name: echo-hello
        taskSpec:
          steps:
            - name: echo
              image: ubuntu
              script: |
                #!/usr/bin/env bash
                echo "step 1"                
      - name: echo-bye
        taskSpec:
          steps:
            - name: echo
              image: ubuntu
              script: |
                #!/usr/bin/env bash
                echo "step 2"
```

### 5、PipelineSpec与TaskRef结合使用

#### 1.调用流程

>Pipelinerun->PipelineSpec->TaskRef->Task

![image-20250514160947045](image-20250514160947045.png)

#### 2.运行Pipeline

```bash
kubectl apply -f pipelinerun3-pipeline-taskref.yaml
```

```yaml
apiVersion: tekton.dev/v1 # or tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: pipelinerun-pipeline-specref-1
spec:
  pipelineSpec:
    tasks:
      - name: hello
        taskRef:
          name: hello
      - name: goodbye
        taskRef:
          name: goodbye
```

## 四、语法工具学习

### 1、When

>When字段用于指定条件，以控制任务或步骤是否应该执行

![image-20250514162236281](image-20250514162236281.png)

```bash
kubectl apply -f pipeline4-when.yaml
tkn pipelines start pipeline-when
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-when
spec:
  tasks:
    - name: hello
      taskRef:
        name: hello
      when:
        - input: "foos"
          operator: "in"
          values: ["foos", "bar"]
    - name: goodbye
      taskRef:
        name: goodbye
```

### 2、Timeout

>Timeout字段用于设置任务或步骤的超时时间

![image-20250514162558181](image-20250514162558181.png)

```bash
kubectl apply -f sleep-task.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: sleep
spec:
  steps:
    - name: sleep
      image: alpine
      script: |
        #!/bin/sh
        sleep 5
```

```bash
kubectl apply -f pipeline5-timeout.yaml
tkn pipelines start pipeline-timeout
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-timeout
spec:
  tasks:
    - name: sleep
      timeout: 2s
      taskRef:
        name: sleep
    - name: goodbye
      taskRef:
        name: goodbye
```

### 3、Retries

>Retries字段用于指定任务或步骤在失败或错误的情况下的重试次数

![image-20250514162739475](image-20250514162739475.png)

```bash
kubectl apply -f pipeline6-retries.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-retries
spec:
  tasks:
    - name: sleep
      timeout: 2s
      retries: 3
      taskRef:
        name: sleep
    - name: goodbye
      taskRef:
        name: goodbye
```

### 4、RunAfter

>RunAfter字段用于定义任务或步骤之间的依赖关系

![image-20250514162909012](image-20250514162909012.png)

```bash
kubectl apply -f pipeline7-runAfter.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-runafter
spec:
  tasks:
    - name: sleep
      taskRef:
        name: sleep
    - name: goodbye
      taskRef:
        name: goodbye
      runAfter:
        - sleep
```

### 5、Finally

>用于定义必须在流水线的最后执行的清理操作或收尾工作。不管前面所有的步骤是对还是错最后都会执行。

![image-20250514163039322](image-20250514163039322.png)

```bash
kubectl apply -f pipeline8-finally.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-finally
spec:
  tasks:
    - name: sleep
      taskRef:
        name: sleep
      timeout: 1s
    - name: goodbye
      taskRef:
        name: goodbye
      runAfter:
        - sleep
  finally:
    - name: cleanup
      taskRef:
        name: hello
```

### 6、Parameters

#### 1.Parameters在task中使用

>`Parameters` 是用于向 **Tasks** 或 **Pipelines** 传递数据的机制，类似于函数的参数。通过 `Parameters`，你可以实现任务模板的重用和流程的灵活控制。

##### 1）task准备

```bash
kubectl apply -f parameters-git-clone-task.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: parameters-git-clone-task
spec:
  params:
    - name: giturl
      type: string
    - name: branch
      type: string
  description: Clone the code repository to the workspace. 
  steps:
    - name: git-clone
      image: bitnami/git:latest
      imagePullPolicy: IfNotPresent
      env: 
        - name: git_username
          valueFrom: 
            secretKeyRef: 
              name: gitlab-secert
              key: username 
        - name: git_password
          valueFrom: 
            secretKeyRef: 
              name: gitlab-secert
              key: password       
      script: |
        echo ${git_username} ${git_password}
        git clone -b $(params.branch) -v http://${git_username}:${git_password}@$(params.giturl)
        ls -la
        pwd 
    - name: list-file
      image: ubuntu
      imagePullPolicy: IfNotPresent
      command:
        - /bin/sh
      args: ['-c', 'ls -la && pwd']
```

##### 2）taskrun创建

![image-20250514163530410](image-20250514163530410.png)

##### 3）指定参数运行

```bash
tkn task start parameters-git-clone-task -p giturl=10.0.7.30/golang/go.git -p branch=main
```

#### 2.Parameters在Pipeline中的使用

##### 1）传参数步骤

>pipeline声明参数->传给pipeline中的task->传给task

![image-20250514164001030](image-20250514164001030.png)

##### 2）运行部署

```bash
kubectl apply -f pipeline9-param.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-params
spec:
  params:
    - name: giturl
      type: string
    - name: branch
      type: string
  tasks:
    - name: clone
      taskRef:
        name: parameters-git-clone-task
      params:
        - name: giturl
          value: "$(params.giturl)"
        - name: branch
          value: "$(params.branch)"
    - name: goodbye
      taskRef:
        name: goodbye
      runAfter:
        - clone

```

```bash
tkn pipeline start pipeline-params
```

#### 3.Parameters在Pipelinerun中的使用

##### 1）传参数步骤

>Pipelinerun声明参数->Pipeline声明参数->传给Pipeline中的Task->传给Task

![image-20250514171135524](image-20250514171135524.png)

##### 2）运行流水线

```bash
kubectl apply -f  pipelinerun4-params.yaml
注意：Pipelinerun每次运行的流水线名字不可一样
```

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: pipelinerun-params-04
spec:
  pipelineRef:
    name: pipeline-params
  params:
    - name: giturl
      value: 10.0.7.30/golang/go.git
    - name: branch
      value: main
```

### 7、Workspace

#### 1.介绍

>Workspace用于在任务执行期间用于存储和共享数据。每个任务可以定义一个或多个Workspace，允许在不同任务之间传递输入和输出数据。Workspace通常用于将源代码、构建产物、配置文件等传递给任务，并从任务中获取生成的输出。

#### 2.Workspace支持的存储卷

>VolumeClaimTemplate：支持跨task共享数据。动态分配独立的工作空间目录
>
>Secret：只读的workspace
>
>EmptyDir：一个空的临时目录（emptyDir="""），只能在一个task中共享数据
>
>ConfigMap：主要用途是保存配置数据，这些数据以键值对的形式存储。

![image-20250514171904662](image-20250514171904662.png)

##### 1)VolumeClaimTemplate

###### ①介绍

>Tekton 的工作空间（Workspace）用于在任务（Task）之间共享数据和文件。通过定义工作空间，任务可以读取、写入和共享数据，从而实现数据的持久性和共享性
>
>在 Tekton 中，`$(workspace.<name>.path)` 是一个用于获取工作空间中指定名称的路径的变量表达式。路径为绝对路径。`<name>`为声明的工作空间名称

###### ②Workspace动态预配VolumeClaimTemplate实战

>需要先定义SC，在Pipelinerun定义VTC调用SC

![image-20250514172616889](image-20250514172616889.png)

```bash
kubectl apply -f vct-git-clone-task.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: vct-git-clone-task
spec:
  params:
    - name: giturl
      type: string
    - name: branch
      type: string
  workspaces:
    - name: code
  description: Clone the code repository to the workspace. 
  steps:
    - name: git-clone
      image: bitnami/git:latest
      imagePullPolicy: IfNotPresent
      env: 
        - name: git_username
          valueFrom: 
            secretKeyRef: 
              name: gitlab-secert
              key: username 
        - name: git_password
          valueFrom: 
            secretKeyRef: 
              name: gitlab-secert
              key: password       
      script: |
        echo ${git_username} ${git_password}
        git clone -b $(params.branch) -v http://${git_username}:${git_password}@$(params.giturl) $(workspaces.code.path)/test
        ls -la
        pwd 
        echo $(workspaces.code.path)
        cd $(workspaces.code.path)/test
        ls -la
        pwd
    - name: list-file
      image: ubuntu
      imagePullPolicy: IfNotPresent
      command:
        - /bin/sh
      args: ['-c', 'ls -la $(workspaces.code.path)/test']
```

```bash
kubectl apply -f pipeline10-vct.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-vct
spec:
  params:
    - name: giturl
      type: string
    - name: branch
      type: string
  workspaces:
    - name: code
  tasks:
    - name: clone
      taskRef:
        name: vct-git-clone-task
      params:
        - name: giturl
          value: "$(params.giturl)"
        - name: branch
          value: "$(params.branch)"
      workspaces:
        - name: code
          workspace: code
    - name: ls-files
      taskRef:
        name: ls-files
      runAfter:
        - clone
      workspaces:
        - name: code
          workspace: code
```

```bash
kubectl apply -f pipelinerun5-vct.yaml
```

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: pipelinerun-vct-2
spec:
  pipelineRef:
    name: pipeline-vct
  params:
    - name: giturl
      value: 10.0.7.30/golang/go.git
    - name: branch
      value: main
  workspaces:
    - name: code
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-client
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 12M
```

### 8、Workspace Secret实战

#### 1.运行Task

```bash
kubectl apply -f ../task/build-dockerfile-bush.yaml
```

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: image-build-and-push
spec:
  workspaces:
    - name: code
    - name: aliyun-image-registry-secert
      mountPath: /kaniko/.docker
  steps:
    - name: image-build-and-push
      image: gcr.io/kaniko-project/executor:latest
      imagePullPolicy: IfNotPresent
      securityContext:
        runAsUser: 0
      env:
        - name: DOCKER_CONFIG
          value: /kaniko/.docker
      command:
        - /kaniko/executor
      args:
        - --dockerfile=Dockerfile
        - --context=$(workspaces.code.path)/test
        - --destination=registry.cn-hangzhou.aliyuncs.com/tool-bucket/xiaowu:test
        - --cache=true
        - --cache-repo=registry.cn-shenzhen.aliyuncs.com/tool-bucket/xiaowu
        - --cache-copy-layers
```

#### 2.运行Pipeline

```bash
kubectl apply -f ../pipeline/pipeline11-secret.yaml 
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-secret
spec:
  params:
    - name: giturl
      type: string
    - name: branch
      type: string
  workspaces:
    - name: code
    - name: aliyun-image-registry-secert
  tasks:
    - name: clone
      taskRef:
        name: vct-git-clone-task
      params:
        - name: giturl
          value: "$(params.giturl)"
        - name: branch
          value: "$(params.branch)"
      workspaces:
        - name: code
          workspace: code
    - name: image-build-and-push
      taskRef:
        name: image-build-and-push
      workspaces:
        - name: aliyun-image-registry-secert
          workspace: aliyun-image-registry-secert
        - name: code
          workspace: code
      runAfter:
        - clone
```

#### 3.运行Pipelinerun

```bash
kubectl apply -f pipelinerun6-secret.yaml
```

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: pipelinerun-secret-19
spec:
  pipelineRef:
    name: pipeline-secret
  params:
    - name: giturl
      value: 10.0.7.30/golang/go.git
    - name: branch
      value: main
  workspaces:
    - name: code
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-client
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 12M
    - name: aliyun-image-registry-secert
      secret:
        secretName: aliyun-image-registry-secert
```

### 9、Volume

#### 1.介绍

>Volume可以用于在Tekton Task的执行过程中持久化存储数据。这对于需要在多个Pipeline Task之间共享数据或者保留任务执行状态的场景非常有用。通过在多个Pipeline Task之间共享Volume，可以实现数据共享。

#### 2.实战

##### 1）部署Task

```bash
kubectl apply -f volume-build-dockerfile-bush.yaml
```

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: volume-image-build-and-push
spec:
  workspaces:
    - name: code
    - name: aliyun-image-registry-secert
      mountPath: /kaniko/.docker
  steps:
    - name: image-build-and-push
      image: gcr.io/kaniko-project/executor:latest
      imagePullPolicy: IfNotPresent
      securityContext:
        runAsUser: 0
      env:
        - name: DOCKER_CONFIG
          value: /kaniko/.docker
      command:
        - /kaniko/executor
      args:
        - --dockerfile=Dockerfile
        - --context=$(workspaces.code.path)/test
        - --destination=registry.cn-hangzhou.aliyuncs.com/tool-bucket/xiaowu:node
        - --cache=true
        - --cache-repo=registry.cn-hangzhou.aliyuncs.com/tool-bucket/xiaowu
        - --cache-copy-layers
      volumeMounts:
        - name: npm-cache
          mountPath: /usr/local/node
  volumes:
    - name: npm-cache
      persistentVolumeClaim:
        claimName: tekton-npm-cache-pvc
```

```bash
kubectl apply -f ls-npm-cache-task.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: ls-npm-cache
spec:
  steps:
    - name: goodbye
      image: ubuntu
      imagePullPolicy: IfNotPresent
      script: |
        #!/bin/bash
        ls -la /usr/local/node/node_modules
      volumeMounts:
        - name: npm-cache
          mountPath: /usr/local/node
  volumes:
    - name: npm-cache
      persistentVolumeClaim:
        claimName: tekton-npm-cache-pvc
```

##### 2）部署Pipeline

```bash
kubectl apply -f pipeline12-volume.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-volume
spec:
  params:
    - name: giturl
      type: string
    - name: branch
      type: string
  workspaces:
    - name: code
    - name: aliyun-image-registry-secert
  tasks:
    - name: clone
      taskRef:
        name: vct-git-clone-task
      params:
        - name: giturl
          value: "$(params.giturl)"
        - name: branch
          value: "$(params.branch)"
      workspaces:
        - name: code
          workspace: code
    - name: volume-image-build-and-push
      taskRef:
        name: volume-image-build-and-push
      workspaces:
        - name: aliyun-image-registry-secert
          workspace: aliyun-image-registry-secert
        - name: code
          workspace: code
      runAfter:
        - clone
    - name: ls-npm-cache
      taskRef:
        name: ls-npm-cache
      runAfter:
        - volume-image-build-and-push
```

##### 3）部署Pipelinerun

```bash
kubectl apply -f pipelinerun7-volume.yaml
```

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: pipelinerun-volume-2
spec:
  pipelineRef:
    name: pipeline-volume
  params:
    - name: giturl
      value: 10.0.7.30/node/vue.git
    - name: branch
      value: main
  workspaces:
    - name: code
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-client
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 12M
    - name: aliyun-image-registry-secert
      secret:
        secretName: aliyun-image-registry-secert
```

### 10、Results

#### 1.介绍

>"Results" 可以被看作是任务执行的输出，它们可以被其他任务使用或共享。通过在任务中定义结果，可以将数据从一个任务传递到另一个任务，实现数据共享和流水线中的协作
>
>在 Tekton 中，可以使用 "Results" 字段来声明任务的输出结果，并在流水线的其他任务中引用这些结果。这样，后续任务就可以访问和使用之前任务产生的数据

#### 2.实战

```bash
kubectl apply -f task/results-datetime.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: results-datetime
spec:
  results:
    - name: datetime
  steps:
    - name: define-datetime
      image: alpine:3.16
      imagePullPolicy: IfNotPresent
      script: |
        #!/bin/sh
        datetime=`date +%Y%m%d-%H%M%S`
        echo -n ${datetime} | tee $(results.datetime.path)        
    - name: echo-datetime
      image: alpine:3.16
      imagePullPolicy: IfNotPresent
      script: |
        #!/bin/sh
        echo $(results.datetime.path)
        ls -la $(results.datetime.path)
        cat $(results.datetime.path)
```

```bash
kubectl apply -f task/echo-results-datetime.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: echo-results-datetime
spec:
  params:
    - name: datetime
  steps:
    - name: echo-results-datetime
      image: alpine
      script: |
        #!/bin/sh
        echo $(params.datetime)
```

```bash
kubectl apply -f pipeline/pipeline13-results.yaml
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-results
spec:
  tasks:
    - name: results-datetime
      taskRef:
        name: results-datetime
    - name: echo-results-datetime
      taskRef:
        name: echo-results-datetime
      params:
        - name: datetime
          value: "$(tasks.results-datetime.results.datetime)"
```

```bash
kubectl apply -f pipelinerun/pipelinerun8-results.yaml
```

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: pipelinerun-results-5
spec:
  pipelineRef:
    name: pipeline-results
```

## 五、配置Gitlab Webhook触发自动部署运行Pipeline

### 1、触发器

#### 1.介绍

>在Tekton中，触发器（Triggers）是一种用于在特定事件发生时触发流水线执行的机制。触发器可以根据各种事件源（EventSource）进行配置，如代码仓库的提交、合并等。当事件源触发时，触发器会将事件传递给Tekton流水线，从而启动流水线的执行

#### 2.核心概念

>触发器的核心组件是EventListeners、TriggerBindings和TriggerTemplates

![image-20250514175320354](image-20250514175320354.png)

>EventListeners事件监听器：用于监听事件源，它负责接收触发事件并将其传递给相应的TriggerBinding和TriggerTemplate

>TriggerBindings（触发器绑定）：TriggerBindings定义了事件数据和流水线参数之间的映射关系

>TriggerTemplates（触发器模板）：TriggerTemplates定义了触发器触发时要执行的Tekton流水线

#### 3.部署

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
```

#### 4.触发器相关资源操作步骤

>1、创建SA账户分配给Eventlistener资源使用
>
>2、创建Eventlistener
>
>3、创建Ingress访问eventlistener
>
>4、创建Secret，配置Gitlab Webhook
>
>5、创建Binding接收参数
>
>6、创建TriggerTemplate
>
>7、创建Pipeline
>
>8、创建Task

>注意: 多个项目需要在EventListener配置多个Trigger并指定Secret与Pipeline，并在该项目Gitlab配置Webhook，不同命名空间以上步骤需要重新执行一遍

#### 5.配置webhook并运行



sa.yaml

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-triggers-gitlab-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tekton-triggers-gitlab-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: tekton-triggers-gitlab-sa
  namespace: default
```

eventListener.yaml

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: event-listener
spec:
  serviceAccountName: tekton-triggers-gitlab-sa
  triggers:
  - name: go-gitlab-push-events-trigger
    interceptors:
    - ref:
        name: "gitlab"
      params:
      - name: "secretRef"
        value:
          secretName: go-gitlab-webhook-token
          secretKey: SecretToken
      - name: "eventTypes"
        value:
          - "Push Hook"
    bindings:
    - ref: common-tb
    template:
      ref: go-tt
```

ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: node-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: node-tekton.1853whitleyave.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: el-event-listener
            port:
             number: 8080
```

secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: go-gitlab-webhook-token
type: Opaque
stringData:
  SecretToken: "gofweqrlljsf8735431wr"
```

binding.yaml

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: common-tb
spec:
  params:
  - name: git-repo-url
    value: $(body.project.http_url)
  - name: build-branch
    value: $(body.ref)
```

triggertemplage.yaml

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: go-tt
  namespace: default
spec:
  params:
  - name: git-repo-url
  - name: build-branch
  resourcetemplates:
  - apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: node-trigger-run-
    spec:
      pipelineRef:
        name: pipeline-tt
      params:
        - name: giturl
          value: $(tt.params.git-repo-url)
        - name: branch
          value:  $(tt.params.build-branch) 
      workspaces:
        - name: code
          volumeClaimTemplate:
            spec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 12M
              storageClassName: nfs-client
        - name: aliyun-image-registry-secert
          secret:
            secretName: aliyun-image-registry-secert
```

pipeline.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-tt
spec:
  params:
    - name: giturl
      type: string
    - name: branch
      type: string
  tasks:
    - name: clone
      taskRef:
        name: test-trigger
      params:
        - name: giturl
          value: "$(params.giturl)"
        - name: branch
          value: "$(params.branch)"

```

task.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: test-trigger
spec:
  params:
    - name: giturl
    - name: branch
  steps:
    - name: list-file
      image: ubuntu
      imagePullPolicy: IfNotPresent
      script:
        echo $(params.giturl) $(params.branch)
```



![image-20250514180241250](image-20250514180241250.png)



## 六、实战：使用Tekton部署Node项目到K8S集群

### 1、task准备

01-git-clone.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone
spec:
  description: Clone the code repository to the workspace.
  params:
    - name: giturl
    - name: branch
  workspaces:
    - name: code
  steps:
    - name: git-clone
      image: bitnami/git:latest
      imagePullPolicy: IfNotPresent
      env:
        - name: git_username
          valueFrom:
            secretKeyRef:
              name: gitlab-secert
              key: username
        - name: git_password
          valueFrom:
            secretKeyRef:
              name: gitlab-secert
              key: password
      script: |
        url=$(params.giturl)
        new_url=$(echo $url | awk -F"http://" '{print $2}')
        branch=$(params.branch)
        build_branch=$(echo "$branch" | awk -F'/' '{print $NF}')
        echo ${build_branch} ${new_url} ${git_username}:${git_password}
        git clone -b ${build_branch} -v http://${git_username}:${git_password}@${new_url} $(workspaces.code.path)/node
    - name: list-file
      image: alpine
      imagePullPolicy: IfNotPresent
      command:
        - /bin/sh
      args: ['-c', 'ls -la $(workspaces.code.path)/node']

```

02-generate-image-tag.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: generate-build-tag
spec:
  params:
    - name: branch
  results:
    - name: datetime
  steps:
    - name: generate-datetime
      image: ubuntu
      imagePullPolicy: IfNotPresent
      script: |
        datetime=`date +%Y%m%d-%H%M%S`
        branch=`echo $(params.branch) | awk -F'/' '{print $NF}'`
        buildId=node-${datetime}-${branch}
        echo -n ${buildId} | tee $(results.datetime.path)
```

03-mage-build-and-push.yaml

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: image-build-and-push
spec:
  workspaces:
    - name: code
    - name: aliyun-image-registry-secert
      mountPath: /kaniko/.docker
  params:
    - name: image-tag
  steps:
    - name: image-build-and-push
      image: gcr.io/kaniko-project/executor:latest
      imagePullPolicy: IfNotPresent
      securityContext:
        runAsUser: 0
      env:
        - name: DOCKER_CONFIG
          value: /kaniko/.docker
      command:
        - /kaniko/executor
      args:
        - --dockerfile=Dockerfile
        - --context=$(workspaces.code.path)/node
        - --destination=registry.cn-hangzhou.aliyuncs.com/tool-bucket/public:$(params.image-tag)
        - --cache=true
        - --cache-repo=registry.cn-hangzhou.aliyuncs.com/tool-bucket/public
        - --cache-copy-layers
      volumeMounts:
        - name: npm-cache
          mountPath: /usr/local/node
  volumes:
    - name: npm-cache
      persistentVolumeClaim:
        claimName: tekton-npm-cache-pvc

```

04-kubectl-apply.yaml

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: kubectl-apply
spec:
  workspaces:
    - name: code
    - name: kubeconfig-secret 
      mountPath: /root/.kube
  params:
    - name: image-tag
  steps:
    - name: kubectl-apply
      image: registry.cn-hangzhou.aliyuncs.com/tool-bucket/tool:kubectl
      imagePullPolicy: IfNotPresent
      script: |
        tag=$(params.image-tag)
        cd $(workspaces.code.path)/node 
        sed -i "s#image: .*#image: registry.cn-hangzhou.aliyuncs.com/tool-bucket/public:${tag}#" deploy.yaml
        kubectl apply -f deploy.yaml 
```

05-finally.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: cleanup-workspace-and-alert
  namespace: default
spec:
  params:
    - name: task1Status
    - name: task2Status
    - name: task3Status
    - name: task4Status
  steps:
    - name: task-alert
      image: curlimages/curl
      imagePullPolicy: IfNotPresent
      script: |
        if [ $(params.task1Status) == "Failed" ];then
          export status="Failed | reason: task1 git clone execute failed,please check"
          echo "$status"
        elif [ $(params.task2Status) == "Failed" ];then
          export status="Failed | reason: task2 generate-build-tag execute failed,please check"
          echo "$status"
        elif [ $(params.task3Status) == "Failed" ];then
          export status="Failed | reason: task3 build push execute failed,please check"
          echo "$status"
        elif [ $(params.task4Status) == "Failed" ];then
          export status="Failed | reason: task4 kubectl apply execute failed,please check"
          echo "$status"
        else
          status="success"
          echo "$status"
        fi
```

### 2、触发器和Pipeline准备

eventListener.yaml

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: event-listener
spec:
  serviceAccountName: tekton-triggers-gitlab-sa
  triggers:
  - name: go-gitlab-push-events-trigger
    interceptors:
    - ref:
        name: "gitlab"
      params:
      - name: "secretRef"
        value:
          secretName: go-gitlab-webhook-token
          secretKey: SecretToken
      - name: "eventTypes"
        value:
          - "Push Hook"
    bindings:
    - ref: common-tb
    template:
      ref: go-tt
  - name: node-gitlab-push-events-trigger
    interceptors:
    - ref:
        name: "gitlab"
      params:
      - name: "secretRef"
        value:
          secretName: node-gitlab-webhook-token
          secretKey: SecretToken
      - name: "eventTypes"
        value:
          - "Push Hook"
    bindings:
    - ref: common-tb
    template:
      ref: node-tt
```

Secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: node-gitlab-webhook-token
type: Opaque
stringData:
  SecretToken: "fweqrlljsf8735431wr"
```

TriggerTemplate.yaml

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: node-tt
  namespace: default
spec:
  params:
  - name: git-repo-url
  - name: build-branch
  resourcetemplates:
  - apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: node-trigger-run-
    spec:
      pipelineRef:
        name: pipeline-node-project
      params:
        - name: giturl
          value: $(tt.params.git-repo-url)
        - name: branch
          value:  $(tt.params.build-branch) 
      workspaces:
        - name: code
          volumeClaimTemplate:
            spec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 12M
              storageClassName: nfs-client
        - name: aliyun-image-registry-secert
          secret:
            secretName: aliyun-image-registry-secert
        - name: kubeconfig-secret 
          secret:
            secretName: kubeconfig-secret 
```

Pipeline.yaml

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-node-project
spec:
  params:
    - name: giturl
    - name: branch
  workspaces:
    - name: code
    - name: aliyun-image-registry-secert
    - name: kubeconfig-secret
  tasks:
    - name: git-clone
      taskRef:
        name: git-clone
      params:
        - name: giturl
          value: "$(params.giturl)"
        - name: branch
          value: "$(params.branch)"
      workspaces:
        - name: code
          workspace: code
    - name: generate-build-tag
      taskRef:
        name: generate-build-tag
      params:
        - name: branch
          value: "$(params.branch)"
      runAfter:
        - git-clone
    - name: image-build-and-push
      taskRef:
        name: image-build-and-push
      params:
        - name: image-tag
          value: "$(tasks.generate-build-tag.results.datetime)"
      workspaces:
        - name: aliyun-image-registry-secert
          workspace: aliyun-image-registry-secert
        - name: code
          workspace: code
      runAfter:
        - generate-build-tag
    - name: kubectl-apply
      taskRef:
        name: kubectl-apply
      params:
        - name: image-tag
          value: "$(tasks.generate-build-tag.results.datetime)"
      workspaces:
        - name: kubeconfig-secret 
          workspace: kubeconfig-secret 
        - name: code
          workspace: code
      runAfter:
        - image-build-and-push
  finally:
    - name: cleanup-workspace-and-alert
      taskRef:
        name: cleanup-workspace-and-alert
      params:
        - name: task1Status
          value: "$(tasks.git-clone.status)"
        - name: task2Status
          value: "$(tasks.generate-build-tag.status)"
        - name: task3Status
          value: "$(tasks.image-build-and-push.status)"
        - name: task4Status
          value: "$(tasks.kubectl-apply.status)"
```

### 3、部署

```bash
kubectl apply -f project/task/
kubectl apply -f project/
```
