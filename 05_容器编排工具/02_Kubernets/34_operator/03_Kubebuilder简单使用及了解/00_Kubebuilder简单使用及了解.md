# Kubebuilder简单使用及了解

## 一、环境安装

### 1、go环境准备

#### 1.下载并解压

```bash
wget https://golang.google.cn/dl/go1.24.3.linux-amd64.tar.gz
tar xf go1.24.3.linux-amd64.tar.gz
mv go /usr/local/
```

#### 2.设置环境变量

```bash
cat >> /etc/profile << 'EOF'

export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin

EOF

source /etc/profile

go version
```

#### 3.设置go代理

```bash
export GOPROXY=https://goproxy.cn,direct
go env -w GOPROXY=https://goproxy.cn,direct
```

### 2、安装 Kubebuilder

```bash
wget -c https://github.com/kubernetes-sigs/kubebuilder/releases/download/v4.2.0/kubebuilder_darwin_amd64
mv kubebuilder_darwin_amd64 /usr/local/bin/kubebuilder
chmod +x /usr/local/bin/kubebuilder

kubebuilder version
Version: main.version{KubeBuilderVersion:"4.2.0", KubernetesVendor:"1.31.0", GitCommit:"c7cde5172dc8271267dbf2899e65ef6f9d30f91e", BuildDate:"2024-08-17T09:41:45Z", GoOs:"darwin", GoArch:"amd64"}
```

## 二、初次使用

### 1、项目初始化

#### 1.创建项目

```bash
mkdir -p /src/application-operator
cd /src/application-operator
go mod init application-operator
```

#### 2.初始化项目

```bash
kubebuilder init [flags]

kubebuilder init --domain=xiaowu.com --owner xiaowu
```

| 参数                      | 作用                                                         | 示例                                                         |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `--domain`                | 设置 API 组的默认域名部分（用于 Group 名字）                 | `--domain example.com` 则生成的 Group 可能是 `cache.example.com` |
| `--repo`                  | 指定 Go module 名称（即 go.mod 中的模块路径）                | `--repo github.com/my-org/my-operator`                       |
| `--project-version`       | 指定项目结构版本，常用为 `3` 或 `2`（推荐使用 3）            | `--project-version 3`                                        |
| `--component-config`      | 启用 [ComponentConfig](https://book.kubebuilder.io/component-config-tutorial/tutorial.html)，用于将控制器配置参数以 Kubernetes 资源的方式管理（推荐开启） | `--component-config=true`                                    |
| `--plugins`               | 指定使用的插件（影响项目结构、功能支持）                     | `--plugins go.kubebuilder.io/v3`                             |
| `--skip-go-version-check` | 跳过 Go 版本检查（不建议一般使用）                           | `--skip-go-version-check=true`                               |
| `--make=false`            | 初始化项目时不运行 `make` 命令（适用于只想生成模板）         | `--make=false`                                               |
| `--owner` | 设置代码文件顶部的版权归属人信息，写入到 `LICENSE` 文件以及自动生成的 Go 源文件中 | `--owner "Your Company Name"` |

#### 3.创建api

```bash
kubebuilder create api --group apps --version v1 --kind Application # 设定的kind的首字母必须大写
Create Resource [y/n]
y
Create Controller [y/n]
y
......
# --kind Application，指定你要创建的resource type的名字，注意首字母必须大写
```

>**`--group apps`**：定义 API 组为 `apps`，通常与 Kubernetes 中已有的 `apps` 组保持一致。
>
>**`--version v1`**：定义 API 版本为 `v1`，表示资源版本为 `v1`。
>
>**`--kind MySQL`**：定义新自定义资源的种类（Kind）为 `MySQL`。
>
>**`--resource`**：生成与自定义资源 (CRD) 相关的代码。
>
>**`--controller`**：生成控制器相关的代码，控制器将用于管理 MySQL 实例的生命周期。

### 2、编写CRD

>将 api/v1/application_types.go 中修改以下部分（注意，//后的注释也有用，会生成字段的description信息）

```go
/*
Copyright 2024 xiaowu.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// ApplicationSpec defines the desired state of Application
// 定义CRD应该有的字段
type ApplicationSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of Application. Edit application_types.go to remove/update
    // Product 该应用所属的产品
	Product string `json:"product,omitempty"`
}

// ApplicationStatus defines the observed state of Application
type ApplicationStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

// Application is the Schema for the applications API
type Application struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ApplicationSpec   `json:"spec,omitempty"`
	Status ApplicationStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// ApplicationList contains a list of Application
type ApplicationList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []Application `json:"items"`
}

func init() {
	SchemeBuilder.Register(&Application{}, &ApplicationList{})
}

```

### 3、编写controller

>kubebuilder 已经帮我们实现了 Operator 所需的大部分逻辑，我们只需要在Reconcile调谐函数 中实现业务逻辑就行了
>该函数在资源发生变化时被调用，负责协调当前的集群状态与期望状态

```go
/*
Copyright 2024 xiaowu.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controller

import (
	"context"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	appsv1 "application-operator/api/v1"
)

// ApplicationReconciler reconciles a Application object
type ApplicationReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=apps.xiaowu.com,resources=applications,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps.xiaowu.com,resources=applications/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps.xiaowu.com,resources=applications/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the Application object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.18.4/pkg/reconcile
func (r *ApplicationReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	// 第一个参数是我们定制的日志输出
	// 日志输出是controller-runtime在我们定制的输出之后添加一段json输出，以帮助调试和跟踪

	// 我们可以在这段json输出中添加key与value，例如
	// xxx是key，对应的value是req.Namespace
	// yyy是key，对应的value是req.name
	// TODO(user): your logic here
	logger.Info("我们收到的ctx值: ", "xxx", ctx)
	logger.Info("我们收到的req值: ", "xxx", req.Namespace, "yyy", req.Name)

	return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *ApplicationReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&appsv1.Application{}).
		Complete(r)
}
```

### 4、CRD测试

#### 1.部署crd

```bash
make install
```

#### 2.报错问题解决

##### 1）报错信息

```bash
[root@k8s-master-01 /src/application-operator]# make install
test -s /src/application-operator/bin/controller-gen && /src/application-
operator/bin/controller-gen --version | grep -q v0.10.0 || \
GOBIN=/src/application-operator/bin go install sigs.k8s.io/controller-
tools/cmd/controller-gen@v0.10.0
/src/application-operator/bin/controller-gen rbac:roleName=manager-role crd
webhook paths="./..." output:crd:artifacts:config=config/crd/bases
test -s /src/application-operator/bin/kustomize || { curl -Ss
"https://raw.githubusercontent.com/kubernetes-
sigs/kustomize/master/hack/install_kustomize.sh" | bash -s -- 3.8.7
/src/application-operator/bin; }
curl: (7) Failed connect to raw.githubusercontent.com:443; 拒绝连接
make: *** [/src/application-operator/bin/kustomize] 错误 7
```

##### 2）原因

>原因是因为网络问题导致无法下载kustomize 到项目目录/src/application-operator/bin/下，
>kustomize工具的作用是：通过kustomization 文件定制kubernetes 对象

##### 3）解决办法

>我们可以手动下载kustomize工具（请与kubebuild版本对齐）

>https://github.com/kubernetes-sigs/kustomize

```bash
wget https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.4.3/kustomize_v5.4.3_linux_amd64.tar.gz

tar xf kustomize_v5.4.3_linux_amd64.tar.gz -C /src/application-operator/bin/ #解压开就是一个二进制命令

# 然后重新make install即可
```

#### 3.验证

```bash
kubectl get crd
```

### 5、启动controller

```bash
make run
```

```bash
[root@k8s-master-01 /src/application-operator]# make run
/src/application-operator/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
/src/application-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go run ./cmd/main.go
2024-08-05T23:17:52+08:00 INFO setup starting manager
2024-08-05T23:17:52+08:00 INFO starting server {"name": "health probe","addr": "[::]:8081"}
2024-08-05T23:17:52+08:00 INFO Starting EventSource {"controller":"application", "controllerGroup": "apps.xiaowu.com", "controllerKind":"Application", "source": "kind source: *v1.Application"}
2024-08-05T23:17:52+08:00 INFO Starting Controller {"controller":"application", "controllerGroup": "apps.xiaowu.com", "controllerKind":"Application"}
2024-08-05T23:17:53+08:00 INFO Starting workers {"controller":"application", "controllerGroup": "apps.xiaowu.com", "controllerKind":"Application", "worker count": 1}
```

### 6、基于CRD来部署CR来验证controller的执行

#### 1.编辑Application资源清单

>vim config/samples/apps_v1_application.yaml改为如下

```yaml
apiVersion: apps.xiaowu.com/v1
kind: Application
metadata:
  labels:
    app.kubernetes.io/name: application-operator
    app.kubernetes.io/managed-by: kustomize
  name: application-sample
spec:
  # TODO(user): Add fields here
  # Add fields here
  product: xiaowuTest
```

#### 2.部署（发生变更则会触发controller的调谐函数执行）

```bash
[root@test application-operator]# kubectl apply -f config/samples/apps_v1_application.yaml
application.apps.qwbyx.com/application-sample created
```

#### 3.查询

```bash
[root@test ~]# kubectl get application
NAME               AGE
application-sample 3s
```

#### 4.查看controller日志

```bash
2024-08-05T23:34:19+08:00 INFO 我们的应用发生变更 {"controller":"application", "controllerGroup": "apps.xiaowu.com", "controllerKind":"Application", "Application": {"name":"application-sample","namespace":"default"}, "namespace": "default", "name": "application-sample", "reconcileID": "f673aea9-0e70-4227-a3ff-a6ec6db18a70", "xxx": "default","yyy": "application-sample"}
```

### 7、以容器形式部署controller

>docker环境准备

#### 1.修改dockerfile

>/src/application-operator/Dockerfile

```dockerfile
# Build the manager binary
#FROM golang:1.22 AS builder
# 修改镜像地址方便下载
FROM registry.cn-hangzhou.aliyuncs.com/xiaowu-k8s-test/golang:1.22 AS builder
ARG TARGETOS
ARG TARGETARCH

WORKDIR /workspace
# Copy the Go Modules manifests
COPY go.mod go.mod
COPY go.sum go.sum
# cache deps before building and copying source so that we don't need to re-download as much
# and so that source changes don't invalidate our downloaded layer
# 设置阿里云go代理
ENV GOPROXY=https://mirrors.aliyun.com/goproxy/,direct
RUN go mod download

# Copy the go source
COPY cmd/main.go cmd/main.go
COPY api/ api/
COPY internal/controller/ internal/controller/

# Build
# the GOARCH has not a default value to allow the binary be built according to the host where the command
# was called. For example, if we call make docker-build in a local env which has the Apple Silicon M1 SO
# the docker BUILDPLATFORM arg will be linux/arm64 when for Apple x86 it will be linux/amd64. Therefore,
# by leaving it empty we can ensure that the container and binary shipped on it will have the same platform.
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} go build -a -o manager cmd/main.go

# Use distroless as minimal base image to package the manager binary
# Refer to https://github.com/GoogleContainerTools/distroless for more details
#FROM gcr.io/distroless/static:nonroot
# 修改镜像地址方便下载
FROM registry.cn-shanghai.aliyuncs.com/xiaowu-k8s-test/static:nonroot
WORKDIR /
COPY --from=builder /workspace/manager .
USER 65532:65532

ENTRYPOINT ["/manager"]

```

#### 2.镜像构建

```bash
make docker-build IMG=application-operator:v0.0.1
```

#### 3.推送

```bash
docker login --username=xiaowu registry.cn-shanghai.aliyuncs.com
docker tag application-operator:v0.0.1 registry.cn-shanghai.aliyuncs.com/xiaowu-k8s-test/application-operator:v0.0.1
docker push registry.cn-shanghai.aliyuncs.com/xiaowu-k8s-test/application-operator:v0.0.1
```

#### 4.部署

```bash
make deploy IMG=registry.cn-shanghai.aliyuncs.com/xiaowu-k8s-test/application-operator:v0.0.1
```

#### 5.查询

>默认在application-operator-system名称空间下

```bash
kubectl -n application-operator-system get deployments.apps
kubectl -n application-operator-system get pods
```

## 三、资源清理

### 1、卸载controller

```bash
make undeploy
```

### 2、卸载CRD

```bash
make uninstall
```

## 四、目录结构相接

```bash
kubebuilder create api --group apps --version v1 --kind MySQL
```

### 1、总览

```bash
├── Dockerfile                     # 用于构建 Operator 的容器镜像
├── Makefile                       # 定义了常用的构建、测试、部署命令，简化操作流程
├── PROJECT                        # Kubebuilder 项目元数据文件，记录项目配置和版本信息
├── README.md                      # 项目介绍文件，记录项目的目标、安装步骤、功能等信息
├── api
│   └── v1
│       ├── groupversion_info.go   # 定义了 API 版本信息和资源组信息
│       ├── mysql_types.go         # 自定义资源 (CRD) 的结构定义，包含 CRD 的字段和序列化逻辑
│       └── zz_generated.deepcopy.go  # 自动生成的代码，用于深度拷贝自定义资源对象
├── bin
│   ├── controller-gen             # Kubebuilder 用于生成控制器的工具，提供 CRD、RBAC 等生成功能的二进制文件
│   └── controller-gen-v0.16.1     # 特定版本的 controller-gen 工具
├── cmd
│   └── main.go                    # Operator 入口点，初始化 Manager 并启动控制器
├── config
│   ├── crd
│   │   ├── kustomization.yaml     # CRD 的 kustomize 配置，用于定制 CRD 的生成
│   │   └── kustomizeconfig.yaml   # 自定义资源定义 (CRD) 的额外配置文件
│   ├── default
│   │   ├── kustomization.yaml     # 默认的 kustomize 配置文件，用于管理 Operator 的部署
│   │   ├── manager_metrics_patch.yaml  # 配置 Manager 的指标导出补丁
│   │   └── metrics_service.yaml   # 用于暴露 Operator 监控指标的服务配置
│   ├── manager
│   │   ├── kustomization.yaml     # 用于部署 Manager 的 kustomize 配置
│   │   └── manager.yaml           # Manager 的 Kubernetes 部署清单
│   ├── network-policy
│   │   ├── allow-metrics-traffic.yaml  # 网络策略，允许访问指标服务
│   │   └── kustomization.yaml     # 网络策略的 kustomize 配置
│   ├── prometheus
│   │   ├── kustomization.yaml     # Prometheus 监控的 kustomize 配置
│   │   └── monitor.yaml           # Prometheus 对 Operator 进行监控的规则
│   ├── rbac
│   │   ├── kustomization.yaml     # 用于生成 RBAC 配置的 kustomize 配置
│   │   ├── leader_election_role.yaml   # Leader 选举的角色权限配置
│   │   ├── leader_election_role_binding.yaml  # Leader 选举的角色绑定
│   │   ├── metrics_auth_role.yaml # 监控指标授权的 RBAC 配置
│   │   ├── metrics_auth_role_binding.yaml # 监控指标授权的角色绑定
│   │   ├── metrics_reader_role.yaml  # 用于读取指标的角色
│   │   ├── mysql_editor_role.yaml    # MySQL 资源编辑者角色
│   │   ├── mysql_viewer_role.yaml    # MySQL 资源查看者角色
│   │   ├── role.yaml                 # 默认的 Operator 角色
│   │   ├── role_binding.yaml         # 角色绑定，将角色分配给 ServiceAccount
│   │   └── service_account.yaml      # 定义 Operator 的 ServiceAccount
│   └── samples
│       ├── apps_v1_mysql.yaml       # 示例自定义资源，定义 MySQL 资源的 YAML 文件
│       └── kustomization.yaml       # 样例资源的 kustomize 配置
├── go.mod                           # Go 模块文件，记录依赖关系
├── go.sum                           # Go 依赖的版本锁定文件
├── hack
│   └── boilerplate.go.txt           # 代码文件的版权声明模板
├── internal
│   └── controller
│       ├── mysql_controller.go      # MySQL 控制器的核心逻辑，处理 CR 的状态同步和管理
│       ├── mysql_controller_test.go # 控制器的单元测试
│       └── suite_test.go            # 控制器的测试套件配置
└── test
    ├── e2e
    │   ├── e2e_suite_test.go        # 端到端测试的套件配置
    │   └── e2e_test.go              # 端到端测试的逻辑
    └── utils
        └── utils.go                 # 测试过程中使用的工具函数
```

### 2、目录和文件详细说明

>**`Dockerfile`**：用于构建 Operator 的容器镜像。部署到 Kubernetes 集群之前，Operator 会被打包为 Docker 镜像。
>
>**`Makefile`**：包含常见的构建命令，如生成 CRD、安装 CRD、编译控制器代码、运行测试、打包 Operator 等。
>
>**`api/v1/`**：该目录包含自定义资源的定义文件。
>
>- **`mysql_types.go`**：定义了 MySQL CRD 的 API 结构体，包括 spec 和 status 字段。
>- **`zz_generated.deepcopy.go`**：通过代码生成工具自动生成的代码，用于深度拷贝自定义资源对象。
>
>**`controllers/`**：这个目录存放控制器逻辑。
>
>- **`mysql_controller.go`**：实现了核心的控制器逻辑，监听 MySQL CR 的变化并执行相应的操作，如创建、删除、更新 MySQL 实例。
>
>**`config/`**：存放所有与 Kubernetes 相关的配置文件，如 CRD、RBAC、部署和监控配置。
>
>- **`crd/`**：定义 CRD 的生成与管理。
>- **`rbac/`**：定义 Operator 所需的 RBAC 权限，包括 ServiceAccount 和角色绑定。
>- **`samples/`**：提供了示例自定义资源文件，用于创建 MySQL 实例。
>
>**`cmd/`**：存放 `main.go` 文件，作为 Operator 的入口点，负责启动控制器并与 Kubernetes API 交互。
>
>**`test/`**：存放测试代码，分为端到端测试（e2e）和辅助工具文件。

### 3、API、Controller 和配置文件的作用

>api/ 目录：该目录用于定义自定义资源 (CRD) 的 API，包括资源的结构和字段。开发者可以在这里定义 Go 结构体，Kubebuilder 会根据这些结构体自动生成 CRD。
>
>controllers/ 目录：该目录用于编写控制器逻辑。控制器负责监听自定义资源的状态变化，并根据需要采取相应的行动。Kubebuilder 会生成控制器的基础代码，开发者只需填充业务逻辑即可。
>
>config/ 目录：包含了与 Operator 相关的配置文件，包括：CRD 定义：在 config/crd 目录下生成的 CRD 清单文件。RBAC 配置：在 config/rbac 目录下定义了 Operator 所需的 RBAC 权限。 Manager 配置：在 config/manager 目录下定义了控制器管理器的部署配置。
