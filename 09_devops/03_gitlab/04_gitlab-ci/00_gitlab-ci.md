# gitlab-ci

## 一、概述

>gitlab-runner 是 gitlab 提供的一种执行 CICD pipline 的组件。它有多种执行器，每一个执行器都提供一种实现 pipline 的方式，例如：shell 执行器是使用 shell 指令实现，docker 执行器是使用 docker api 实现。其中，最有难度的一种是 kubernetes 执行器。这种执行器会使用 k8s api 来实现 CICD pipline。

## 二、执行流程

>runner 的 k8s 执行器是这样执行 pipline 的：
>
>>- 首先，runner 会通过 RBAC 认证获取到调用 k8s 集群 API 的权限。
>>
>>- runner 会监听 gitlab，当有合适的 job 时，runner 会自动抓取任务执行。请注意，一个流水线中可以有很多个 stage，这些 stage 是串行执行的，而一个 stage 中又可以有多个并行的 job，runner 抓取的任务是以 job 为单位，而不是 stage，更不是 pipline。
>>
>>- 随后，runner 会调用 k8s API，创建一个用于执行该 job 的 pod。通常来说，runner 创建的所有 pod 有一个通用模板，我们需要在 runner 的 config.toml 配置文件中配置这个模板。但 pod 中具体使用什么镜像、在 pod 中执行什么命令，这些都是在后续的 .gitlab-ci.yml 文件中配置，并且随着 job 的不同而不同。
>>- 在完成了 job 内的工作后，runner 会将这个临时 pod 删除。
>>- 将主节点 kubeconfig 内容添加到 secret 中。这个文件的内容是 kubectl 访问 k8s 集群的准入 Token，只有在指定了该 Token 后，才能使用 kubectl 指令来对集群内的各种资源进行增删改查。由于 runner 在 CICD 过程中需要对 k8s 集群进行操作，因此，每一个 runner 中都必须具备 Token以供 gitrunner 的 k8s 执行器使用。
>>-  使用 secrete 将这个 Token 以卷挂载的方式加入到 runner 创建的 pod 中。 温馨提示：请务必保证 config 文件中包含的证书与密钥没有过期

## 三、安装

### 1、基于docker

#### 1.docker-compose准备

```yaml
root@lb-kubeasz:/docker-service/01_gitlab_runner# cat docker-compose.yaml 
services:
  gitlab-runner:
    image: gitlab/gitlab-runner:v18.3.0
    container_name: gitlab-runner
    restart: always
    privileged: true   # 必须，加上才能用 dind
    volumes:
#      - /var/run/docker.sock:/var/run/docker.sock
      - ./runner:/etc/gitlab-runner
    environment:
      TZ: Asia/Shanghai
    networks:
      - gitlab_runner_net

  docker-dind:
    image: docker:28.3.3-dind
    container_name: docker-dind
    restart: always
    privileged: true
    environment:
      DOCKER_TLS_CERTDIR: ""   # 关闭 TLS，runner 里好用
      TZ: Asia/Shanghai 
    volumes:
      - ./dind:/var/lib/docker
    networks:
      - gitlab_runner_net

networks:
  gitlab_runner_net:
    driver: bridge
```

#### 2.runner注册

```bash
docker-compose run --rm gitlab-runner register \
  --non-interactive \
  --url "https://gitlab-devops-k8s-local.gmbaifa.online/" \
  --token "RUNNER_TOKEN" \
  --executor "docker" \
  --docker-image alpine:latest \
  --description "docker-runner"

```

#### 3.启动

```bash
docker-compose up -d
```

### 2、基于k8s

#### 1.helm包获取

```bash
helm repo add gitlab https://charts.gitlab.io
helm search repo -l gitlab/gitlab-runner

helm fetch gitlab/gitlab-runner --version=0.80.1
```

#### 2.修改内部参数

Values.yaml

```yaml
#gitlab地址
gitlabUrl: http://gitlab-service.git-repository/
...
#gitlab-runner注册用到的tocken
runnerToken: "xxxxxxxxxxxx"
...
sessionServer:
...
  serviceType: ClusterIP
#角色设置
rbac:
...
  create: true
...
  clusterWideAccess: true
#prometheus metrics数据暴露
metrics:
  enabled: true

```

#### 3.部署

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: 10-gitlab-runner-helm
  namespace: argocd
  labels:
    gmbaifa-host: s8030
    gmbaifa-type: level-2
    install-type: helm
    app: gitlab
spec:
  project: cluster-init
  source:
    repoURL: 'git@codeup.aliyun.com:618e50b14d2b371c479a84be/devops/argocd-app.git'
    path: 10_gitlab-runner/01_helm
    targetRevision: cluster-init
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: gitlab-runner
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

