# MySQLOperator

## 一、初始化项目

>在已经初始化的项目中（你已经运行过 `kubebuilder init`），你可以定义新的 API 和控制器逻辑，而不直接应用到集群。首先，我们生成 API 和控制器的代码：

```bash
kubebuilder create api --group apps --version v1 --kind MySQL --resource --controller
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

### 1、生成文件内容

>`api/v1/mysql_types.go`：此文件将包含自定义资源的定义，包括 `Spec` 和 `Status` 的结构。
>
>`controllers/mysql_controller.go`：此文件将包含初始的控制器代码，用于后续管理 MySQL 资源的生命周期。
>
>`config/crd/`：此目录将包含生成的 Kubernetes CRD 定义清单文件。

## 二、CRD的定义与生成

### 1、流程描述

>**生成 API 和控制器文件**：使用 `kubebuilder create api` 命令生成自定义资源相关的文件，但不应用。
>
>**编辑 `mysql_types.go`**：定义 MySQL 资源的 `Spec` 和 `Status` 字段。
>
>**生成代码**：使用 `make generate` 生成深度拷贝函数等辅助代码。
>
>**生成 CRD 清单文件**：使用 `make manifests` 生成 Kubernetes CRD YAML 文件，但不将其应用到集群。
>
>**手动查看或修改生成的文件**：在 `config/crd/bases/` 中找到生成的 CRD 文件，进一步检查或修改。

>我们将编写自定义资源 `MySQL` 的 `Spec` 和 `Status` 字段，并使用 Kubebuilder 的标注 (annotations) 自动生成深度拷贝函数、CRD 定义等。

### 2、CRD编辑

>修改 `mysql_types.go`: `mysql_types.go` 文件是定义自定义资源类型的主要文件。在这里，我们需要定义 MySQL 的 `Spec`（期望状态）和 `Status`（当前状态）。
>
>**文件路径：`api/v1/mysql_types.go`**

```go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// 1、定制CRD必须要有的字段：CR文件中引用的字段是json后的字段
// 子结构体：用于定制存储
type StorageConfig struct {
	StorageClassName string `json:"storageClassName"`
	Size             string `json:"size"`
}

// 子结构体：用于资源限制
// 资源请求定义
type ResourceRequests struct {
	CPU    string `json:"cpu"`
	Memory string `json:"memory"`
}

// 资源限制定义
type ResourceLimits struct {
	CPU    string `json:"cpu"`
	Memory string `json:"memory"`
}

// 资源要求定义
type ResourceRequirements struct {
	Requests ResourceRequests `json:"requests"`
	Limits   ResourceLimits   `json:"limits"`
}

type MysqlClusterSpec struct {
	//MasterConfig  string `json:"masterConfig"`
	//SlaveConfig   string `json:"slaveConfig"`
	Image         string               `json:"image"`
	Replicas      int32                `json:"replicas"`
	MasterService string               `json:"masterService"`
	SlaveService  string               `json:"slaveService"`
	Storage       StorageConfig        `json:"storage"`
	Resources     ResourceRequirements `json:"resources"`
}

// 2、定制status信息
type MysqlClusterStatus struct {
	Master string   `json:"master"`
	Slaves []string `json:"slaves"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

type MysqlCluster struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   MysqlClusterSpec   `json:"spec,omitempty"`
	Status MysqlClusterStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

type MysqlClusterList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []MysqlCluster `json:"items"`
}

func init() {
	SchemeBuilder.Register(&MysqlCluster{}, &MysqlClusterList{})
}

```

#### 1.代码解释

>**`MySQLSpec`**：定义期望的 MySQL 实例的配置信息（如用户名、密码、副本数、备份计划等）。
>
>**`MySQLStatus`**：定义 MySQL 实例的运行状态，包括副本数、最后备份时间和状态条件。
>
>**`+kubebuilder:object:root=true`**：告诉 Kubebuilder 这是自定义资源的根对象,
>
>>是 Kubebuilder 中的一条标记（marker），用于生成 Kubernetes 自定义资源定义（CRD）相关代码。它的作用是标记某个结构体为 **CRD 对象的根对象**（Root Object)
>>
>>表明该结构体是一个 **顶级 Kubernetes 对象**（比如 `Deployment`、`Service` 也是顶级对象），可以通过 `kubectl` 等工具直接操作。
>>
>>Kubebuilder 会根据这个标记生成 CRD YAML 文件中的主体结构。
>>
>>如果不加该标记，Kubebuilder 不会将其识别为 CRD 根对象，不会为其生成 `Kind`、`APIVersion` 等字段。
>>
>>简单理解：我要定义一个完整的 Kubernetes 对象，而不是它里面的某个字段或子结构。
>>
>>如果不加：
>>
>>>如果你不加这个注解，Kubebuilder 就不会生成该结构体对应的 `Kind`、`apiVersion`、`metadata` 等顶级字段的 CRD 定义。
>>>
>>>生成的 YAML 文件就不会包含你这个资源。
>>>
>>>你也不能用 `kubectl apply` 或 `kubectl get` 来操作它，因为它不是一个完整的 CRD 类型。
>
>**`+kubebuilder:subresource:status`**：让 Kubernetes 自动创建一个子资源来管理 `status` 字段。
>
>>是 Kubebuilder 中的另一个非常重要的标记，用来声明你的 CRD **支持 `status` 子资源**，用于控制器更新状态信息。
>>
>>简单理解：我的 CRD 对象需要支持单独的 `.status` 字段更新。
>>
>>>**防止竞态冲突**
>>> `.spec` 是用户配置，`.status` 是系统写的。分开处理可以防止两个不同来源同时更新导致的冲突。
>>>
>>>**遵守声明式设计**
>>> 用户只负责声明 `.spec`，控制器只负责填充 `.status`（比如当前状态、进度、运行信息等）。
>>>
>>>**让 controller 更安全**
>>> 如果 controller 修改整个对象（包括用户的配置），容易因为版本冲突报错。只更新 `.status` 是安全的操作。

### 3、生成深度拷贝函数

#### 1.`make generate` 的作用：

>生成深度拷贝函数 `zz_generated.deepcopy.go`，确保在 Kubernetes 控制器中可以安全地拷贝自定义资源对象。
>
>确保自定义资源的代码与 Kubernetes API 兼容。

```bash
make generate
```

>你将在 `api/v1/` 目录下看到生成的 `zz_generated.deepcopy.go` 文件。

### 4、**生成 Kubernetes CRD 清单文件**

>为了将定义的 CRD 注册到 Kubernetes，我们需要生成相应的 CRD 定义 YAML 文件。此时，**生成代码但不应用**。你可以使用 `make manifests` 命令生成清单文件。

#### 1.`make manifests` 的作用

>该命令会基于 `mysql_types.go` 中的定义生成相应的 Kubernetes CRD 定义文件。
>
>文件会被生成到 `config/crd/bases/` 目录下。

```bash
make manifests
```

#### 2.**生成的文件**

>`config/crd/bases/apps.example.com_mysqls.yaml`：这个 YAML 文件包含了 MySQL 自定义资源的定义，你可以查看文件，里面会包含以下内容：
>
>`spec` 和 `status` 字段的定义。
>
>API 组名、版本信息和其他元数据。

## 三、Controller开发

### 1、流程综述

>**生成控制器文件**：使用 `kubebuilder create api --controller` 生成控制器文件。
>
>**编写控制器逻辑**：在 `mysql_controller.go` 中编写 `Reconcile` 函数，处理自定义资源的状态变化。
>
>**生成代码**：运行 `make generate` 生成辅助代码。
>
>**手动查看和修改文件**：生成的文件位于 `config/` 目录下，可手动查看或修改生成的 Kubernetes 清单。

>当你运行 `kubebuilder create api --controller` 时，已经生成了一个初始的控制器文件。接下来，我们将编写控制器的业务逻辑。

### 2、路径

>internal/controller

### 3、mysqlcluster_controller.go文件准备

```go
package controller

import (
	"context"                         // 上下文库，用于设置超时、取消等机制
	"github.com/go-logr/logr"         // 日志记录库
	"k8s.io/apimachinery/pkg/runtime" // Kubernetes API 类型通用机制支持，如 Scheme

	"sigs.k8s.io/controller-runtime/pkg/client" // controller-runtime 客户端，用于操作 Kubernetes 资源
	"sigs.k8s.io/controller-runtime/pkg/log"    // controller-runtime 日志工具

	databasev1 "github.com/xiaowu/api/v1" // 引入自定义资源 MysqlCluster 的定义
	v1 "k8s.io/api/core/v1"               // 引入核心 API 资源，如 Pod、Service、ConfigMap 等
	ctrl "sigs.k8s.io/controller-runtime" // controller-runtime 的核心模块，用于构建控制器
)

// 常量定义区域
const (
	MySQLPassword           = "password"              // 默认的 MySQL 密码（仅示例，生产不建议硬编码）
	MysqlClusterKind        = "MysqlCluster"          // 自定义资源 Kind 名称
	MysqlClusterAPIVersion  = "apps.xiaowu.com/v1"    // 自定义资源 API 版本
)

// MysqlClusterReconciler 是一个结构体，实现了 Reconciler 接口，用于调谐 MysqlCluster 对象
type MysqlClusterReconciler struct {
	client.Client                  // 嵌入 Kubernetes 客户端接口，简化资源操作
	Log                logr.Logger // 日志记录器
	Scheme             *runtime.Scheme // 用于资源注册与转换
	MasterGTIDSnapshot string          // 当前主库的 GTID 快照，用于后续同步校验
	SnapGoIsEnabled    bool            // 协程启动标识位，防止重复启动
}

/*
RBAC 权限定义，kubebuilder 注解用于生成对应的 ClusterRole YAML
用于控制器运行时访问各种 Kubernetes 资源
*/
// 访问自定义资源 MysqlCluster 的权限
// +kubebuilder:rbac:groups=apps.xiaowu.com,resources=mysqlclusters,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps.xiaowu.com,resources=mysqlclusters/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps.xiaowu.com,resources=mysqlclusters/finalizers,verbs=update

// 访问核心 Kubernetes 资源（Pod、Service、ConfigMap 等）所需权限
// +kubebuilder:rbac:groups="",resources=pods;services;configmaps,verbs=get;list;watch;create;update;delete
// +kubebuilder:rbac:groups="",resources=pods/exec,verbs=create;get;list;watch
// +kubebuilder:rbac:groups="",resources=endpoints,verbs=get;list;watch
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch;create;update;delete
// +kubebuilder:rbac:groups="",resources=namespaces,verbs=get;list;watch
// +kubebuilder:rbac:groups="",resources=events,verbs=create;get;list;watch
// +kubebuilder:rbac:groups="",resources=persistentvolumeclaims,verbs=get;list;watch;create;update;delete
// +kubebuilder:rbac:groups="",resources=persistentvolumes,verbs=get;list;watch;create;update;delete

// Reconcile 是核心调谐逻辑，每当资源状态变更时都会触发调用
func (r *MysqlClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := log.FromContext(ctx)
	log.Info("调谐函数触发执行", "req", req)

	// 1. 获取当前请求对应的 MysqlCluster 对象
	var cluster databasev1.MysqlCluster
	if err := r.Get(ctx, req.NamespacedName, &cluster); err != nil {
		// 如果资源不存在，忽略错误（如被删除）
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// 2. 检查集群是否已经初始化（通过 Annotation 判断）
	if _, ok := cluster.Annotations["initialized"]; !ok {
		// 尚未初始化，调用初始化逻辑
		if err := r.init(ctx, &cluster); err != nil {
			log.Info("初始化集群失败")
			return ctrl.Result{}, err
		} else {
			log.Info("初始化集群成功")
		}

		// 初始化成功后设置 Annotation 标识，避免重复初始化
		if cluster.Annotations == nil {
			cluster.Annotations = make(map[string]string)
		}
		cluster.Annotations["initialized"] = "true"
		// 更新资源以保存 Annotation
		if err := r.Update(ctx, &cluster); err != nil {
			return ctrl.Result{}, err
		}
	} else {
		// 已初始化，则开始执行调谐逻辑
		// 1. 副本数调谐（如根据 spec 进行 Pod 管理）
		result, err := r.reconcileReplicas(ctx, cluster)
		if err != nil {
			return result, err
		}
		// 2. 主从状态检测与调谐
		result, err = r.reconcileMasterSlave(ctx, cluster)
		if err != nil {
			return result, err
		}
	}

	// 3. 启动协程记录主库 GTID 状态（仅启动一次）
	if !r.SnapGoIsEnabled {
		r.startAndUpdateGTIDSnapshot(ctx, cluster) // 启动定时任务，记录 GTID
		r.SnapGoIsEnabled = true
	}

	// 4. 返回调谐结果（不重排队）
	return ctrl.Result{}, nil
}

// SetupWithManager 会在程序启动时被调用，注册控制器到 Manager 并指定它要管理哪些资源
func (r *MysqlClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&databasev1.MysqlCluster{}). // 管理 MysqlCluster 对象
		Owns(&v1.Pod{}).                 // 管理其创建的 Pod（可以触发 reconcile）
		Complete(r)                      // 注册完成
}

```

### 4、initialization.go文件准备

```go
package controller

import (
	"context" // 用于处理上下文，提供超时、取消等操作
	"fmt"     // 格式化I/O函数，如字符串格式化和打印
	"sigs.k8s.io/controller-runtime/pkg/log"

	databasev1 "github.com/xiaowu/api/v1" // 导入自定义的 MySQLCluster API 资源定义
)

// 初始化函数：当集群首次创建，尚未完成初始化时执行
func (r *MysqlClusterReconciler) init(ctx context.Context, cluster *databasev1.MysqlCluster) error {
	// 获取上下文中的日志对象，便于输出调试信息
	log := log.FromContext(ctx)

	// 第1步：创建主库和从库的 Service 资源
	if _, err := r.getOrCreateService(ctx, cluster.Spec.MasterService, "master", cluster.Namespace, *cluster); err != nil {
		log.Info("创建主库 Service 失败")
		return fmt.Errorf("failed to create master service: %v", err)
	}
	if _, err := r.getOrCreateService(ctx, cluster.Spec.SlaveService, "slave", cluster.Namespace, *cluster); err != nil {
		log.Info("创建从库 Service 失败")
		return fmt.Errorf("failed to create slave service: %v", err)
	}

	// 第2步：读取副本数量
	replicas := cluster.Spec.Replicas
	if replicas < 1 {
		// 副本数量必须大于0，否则直接报错
		return fmt.Errorf("invalid replica count: %d", replicas)
	}

	// 第3步：为每个副本创建对应的 ConfigMap 配置
	for i := int32(1); i <= replicas; i++ {
		configMapName := fmt.Sprintf("mysql-config-%02d", i) // 命名规范：mysql-config-01, mysql-config-02, ...
		serverID := int(i)                                   // 每个副本的 server-id 用于 MySQL 主从复制

		// 创建 ConfigMap，其中包含 MySQL 配置文件（如 my.cnf）
		if err := r.createConfigMap(ctx, configMapName, serverID, cluster.Namespace, cluster); err != nil {
			return fmt.Errorf("failed to create configmap %s: %v", configMapName, err)
		}
	}

	// 第4步：为每个副本创建对应的 PVC（持久化存储卷）
	storageClassName := cluster.Spec.Storage.StorageClassName
	storageSize := cluster.Spec.Storage.Size
	for i := int32(1); i <= replicas; i++ {
		pvcName := fmt.Sprintf("mysql-%02d", i) // PVC 名称与 Pod 一致
		// 创建 PVC，指定 StorageClass 和容量
		if err := r.createPVC(ctx, pvcName, storageClassName, cluster.Namespace, storageSize, cluster); err != nil {
			return err
		}
	}

	// 第5步：为每个副本创建对应的 Pod 实例
	for i := int32(1); i <= replicas; i++ {
		podName := fmt.Sprintf("mysql-%02d", i)
		pvcName := fmt.Sprintf("mysql-%02d", i) // 与 Pod 同名的 PVC
		log.Info("准备创建 Pod", "podName", podName)

		// 使用对应的 ConfigMap 和 PVC 创建 Pod
		configMapName := fmt.Sprintf("mysql-config-%02d", i)
		if err := r.createPod(ctx, podName, cluster.Spec.Image, configMapName, pvcName, cluster.Namespace, cluster); err != nil {
			return fmt.Errorf("failed to create pod %s: %v", podName, err)
		}
	}

	// 第6步：设置主从复制关系
	masterPodName := "mysql-01" // 默认第一个 Pod 为主库
	slavePodNames := []string{}
	for i := int32(2); i <= replicas; i++ {
		slavePodNames = append(slavePodNames, fmt.Sprintf("mysql-%02d", i))
	}
	log.Info("设置主从复制关系", "masterPodName", masterPodName, "slavePodNames", slavePodNames)

	// 执行主从复制配置，例如设置 binlog、server-id、change master 等命令
	if err := r.setupMasterSlaveReplication(ctx, masterPodName, slavePodNames, *cluster); err != nil {
		log.Info("配置主从复制失败", "err", err)
		return fmt.Errorf("failed to setup master-slave: %v", err)
	}

	return nil
}

```

### 5、utlis.go

```go
package controller

import (
	"context"                            // 用于处理上下文，提供超时、取消等操作
	"fmt"                                // 格式化I/O函数，如字符串格式化和打印
	"k8s.io/apimachinery/pkg/api/errors" // 提供与 Kubernetes API 相关的错误处理函数
	"k8s.io/apimachinery/pkg/api/resource"
	"k8s.io/apimachinery/pkg/util/intstr"
	"k8s.io/client-go/kubernetes"
	"os"
	"strings"
	"time"

	//"k8s.io/client-go/tools/clientcmd" // 本地连接：通过config.yml方式链接k8s
	"k8s.io/client-go/rest" // 集群内部连接：通过serviceaccess方式链接k8s
	"k8s.io/client-go/tools/remotecommand"
	"sigs.k8s.io/controller-runtime/pkg/client" // 提供与 Kubernetes API 交互的客户端
	"sigs.k8s.io/controller-runtime/pkg/log"

	databasev1 "github.com/xiaowu/api/v1" // 导入自定义的 MySQLCluster API 资源定义
	v1 "k8s.io/api/core/v1"               // 核心 Kubernetes API 对象，例如 Pod 和 Service
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// 创建configmap：
func (r *MysqlClusterReconciler) createConfigMap(ctx context.Context, name string, serverID int, namespace string, cluster *databasev1.MysqlCluster) error {
	// 检查 ConfigMap 是否已经存在
	existingConfigMap := &v1.ConfigMap{}
	err := r.Get(ctx, client.ObjectKey{Namespace: namespace, Name: name}, existingConfigMap)
	if err == nil {
		// 如果没有报错，说明 ConfigMap 已经存在，直接返回，不需要重复创建
		r.Log.Info("ConfigMap already exists", "ConfigMap.Name", name)
		return nil
	}

	// 定义 OwnerReference，表示这个 ConfigMap 是 MysqlCluster 的子资源
	ownerRef := metav1.OwnerReference{
		APIVersion: MysqlClusterAPIVersion,                 // MysqlCluster 的 API 版本，使用常量以保持统一
		Kind:       MysqlClusterKind,                       // MysqlCluster 的资源类型（Kind），使用常量以保持统一
		Name:       cluster.Name,                           // 所属 MysqlCluster 的名称
		UID:        cluster.UID,                            // 所属 MysqlCluster 的唯一标识
		Controller: func(b bool) *bool { return &b }(true), // 标记这是一个控制器的 OwnerReference，这样当控制器资源删除时，该 ConfigMap 会被级联删除
	}

	// 定义 ConfigMap 中的 my.cnf 配置内容（MySQL 的配置文件）
	configMapData := fmt.Sprintf(`[mysqld]
server-id=%d
binlog_format=row
log-bin=mysql-bin
skip-name-resolve
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
relay_log_purge=0
# other configurations`, serverID) // 使用 serverID 替换模板中的变量

	// 创建 ConfigMap 对象
	configMap := &v1.ConfigMap{
		ObjectMeta: metav1.ObjectMeta{
			Name:      name,      // ConfigMap 的名称
			Namespace: namespace, // ConfigMap 所在的命名空间
			OwnerReferences: []metav1.OwnerReference{
				ownerRef, // 设置上面定义的所有者引用
			},
		},
		Data: map[string]string{
			"my.cnf": configMapData, // 设置 key 为 my.cnf 的数据内容，value 为 MySQL 配置文本
		},
	}

	// 调用 Kubernetes API 创建 ConfigMap
	if err := r.Create(ctx, configMap); err != nil {
		// 如果创建失败，记录错误日志并返回错误
		r.Log.Error(err, "Failed to create ConfigMap", "ConfigMap.Name", name)
		return err
	}

	// 创建成功，记录日志
	r.Log.Info("ConfigMap created successfully", "ConfigMap.Name", name)
	return nil
}

// 创建 PersistentVolumeClaim（PVC）：
func (r *MysqlClusterReconciler) createPVC(ctx context.Context, name, storageClassName, namespace string, storageSize string, cluster *databasev1.MysqlCluster) error {
	// 首先检查 PVC 是否已经存在，避免重复创建
	existingPVC := &v1.PersistentVolumeClaim{}
	err := r.Get(ctx, client.ObjectKey{Namespace: namespace, Name: name}, existingPVC)
	if err == nil {
		// 如果没有报错，说明 PVC 已经存在，记录日志并返回
		r.Log.Info("PVC already exists", "PVC.Name", name)
		return nil
	}

	// 定义 PVC 的拥有者（OwnerReference），用于垃圾回收（随着 MysqlCluster 删除而自动删除 PVC）
	ownerRef := metav1.OwnerReference{
		APIVersion: "database.kubebuilder.io/v1",           // MysqlCluster 对象的 API 版本，确保与 CRD 一致
		Kind:       "MysqlCluster",                         // MysqlCluster 资源的 Kind
		Name:       cluster.Name,                           // 当前 MysqlCluster 实例的名称
		UID:        cluster.UID,                            // 当前 MysqlCluster 实例的唯一标识
		Controller: func(b bool) *bool { return &b }(true), // 标记这是一个控制器拥有的资源，布尔指针 true
	}

	// 创建 PVC 对象的定义
	pvc := &v1.PersistentVolumeClaim{
		ObjectMeta: metav1.ObjectMeta{
			Name:      name,      // PVC 的名称
			Namespace: namespace, // PVC 所在的命名空间
			Labels: map[string]string{
				"app": "mysql", // 添加标签，便于选择器或资源筛选
			},
			OwnerReferences: []metav1.OwnerReference{
				ownerRef, // 设置拥有者引用，绑定到 MysqlCluster
			},
		},
		Spec: v1.PersistentVolumeClaimSpec{
			AccessModes: []v1.PersistentVolumeAccessMode{
				v1.ReadWriteOnce, // 设置访问模式：只能被单个节点以读写方式挂载
			},
			Resources: v1.VolumeResourceRequirements{
				Requests: v1.ResourceList{
					v1.ResourceStorage: resource.MustParse(storageSize), // 请求的存储容量，解析字符串（如 "10Gi"）
				},
			},
			StorageClassName: &storageClassName, // 指定使用的存储类（StorageClass）
		},
	}

	// 使用 Kubernetes 客户端 API 创建 PVC
	if err := r.Create(ctx, pvc); err != nil {
		// 如果创建失败，记录错误日志并返回
		r.Log.Error(err, "Failed to create PVC", "PVC.Name", name)
		return err
	}

	// 创建成功，记录日志
	r.Log.Info("PVC created successfully", "PVC.Name", name)
	return nil
}

// 创建 Pod 的函数
func (r *MysqlClusterReconciler) createPod(ctx context.Context, name, image, configMapName, pvcName, namespace string, cluster *databasev1.MysqlCluster) error {
	// 检查 Pod 是否已经存在，避免重复创建
	existingPod := &v1.Pod{}
	err := r.Get(ctx, client.ObjectKey{Namespace: namespace, Name: name}, existingPod)
	if err == nil {
		// 如果已经存在，则记录日志并直接返回
		r.Log.Info("Pod already exists", "Pod.Name", name)
		return nil
	}

	// 定义 OwnerReference，使该 Pod 与 MysqlCluster 资源绑定
	ownerRef := metav1.OwnerReference{
		APIVersion: MysqlClusterAPIVersion,                 // 使用定义好的常量表示 API 版本
		Kind:       MysqlClusterKind,                       // 使用定义好的常量表示资源类型
		Name:       cluster.Name,                           // 所属 MysqlCluster 实例的名称
		UID:        cluster.UID,                            // 唯一标识该实例
		Controller: func(b bool) *bool { return &b }(true), // 表示该引用对象是控制器拥有者（关键！用于自动回收）
	}

	// 获取资源限制和请求，从 MysqlCluster 的 spec 中提取
	resources := v1.ResourceRequirements{
		Requests: v1.ResourceList{
			v1.ResourceCPU:    resource.MustParse(cluster.Spec.Resources.Requests.CPU),    // 请求的 CPU 资源
			v1.ResourceMemory: resource.MustParse(cluster.Spec.Resources.Requests.Memory), // 请求的内存资源
		},
		Limits: v1.ResourceList{
			v1.ResourceCPU:    resource.MustParse(cluster.Spec.Resources.Limits.CPU),    // 限制的 CPU
			v1.ResourceMemory: resource.MustParse(cluster.Spec.Resources.Limits.Memory), // 限制的内存
		},
	}

	// 创建 Pod 的定义
	pod := &v1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      name,      // Pod 名称
			Namespace: namespace, // 所在命名空间
			Labels: map[string]string{
				"app": "mysql", // 添加标签用于服务发现和选择器匹配
				//"role": role, // 暂未使用，将来用于主从标识
			},
			OwnerReferences: []metav1.OwnerReference{
				ownerRef, // 设置所属对象为当前的 MysqlCluster
			},
		},
		Spec: v1.PodSpec{
			Containers: []v1.Container{
				{
					Name:      "mysql",   // 容器名称
					Image:     image,     // 使用传入的镜像
					Resources: resources, // 使用资源限制与请求
					Env: []v1.EnvVar{
						{
							Name:  "MYSQL_ROOT_PASSWORD",
							Value: "password", // 设置 root 用户的初始密码
						},
					},
					Ports: []v1.ContainerPort{
						{
							Name:          "mysql",
							ContainerPort: 3306, // 暴露 3306 端口供访问
						},
					},
					VolumeMounts: []v1.VolumeMount{
						{
							Name:      "mysql-config",
							MountPath: "/etc/my.cnf", // 挂载 ConfigMap 到配置文件路径
							SubPath:   "my.cnf",      // 只挂载 my.cnf 文件而非整个目录
						},
						{
							Name:      "mysql-data",
							MountPath: "/var/lib/mysql", // 持久化数据挂载点
						},
					},
				},
			},
			Volumes: []v1.Volume{
				{
					Name: "mysql-config",
					VolumeSource: v1.VolumeSource{
						ConfigMap: &v1.ConfigMapVolumeSource{
							LocalObjectReference: v1.LocalObjectReference{
								Name: configMapName, // 使用指定 ConfigMap 提供 MySQL 配置
							},
						},
					},
				},
				{
					Name: "mysql-data",
					VolumeSource: v1.VolumeSource{
						PersistentVolumeClaim: &v1.PersistentVolumeClaimVolumeSource{
							ClaimName: pvcName, // 使用指定 PVC 存储 MySQL 数据
						},
					},
				},
			},
		},
	}

	// 创建 Pod 资源
	if err := r.Create(ctx, pod); err != nil {
		r.Log.Error(err, "Failed to create Pod", "Pod.Name", name)
		return err
	}
	r.Log.Info("Pod created successfully", "Pod.Name", name)

	// 等待 Pod 就绪，避免在 Pod 启动过程中进行主从设置导致失败
	podKey := client.ObjectKey{Namespace: namespace, Name: name}
	for {
		time.Sleep(5 * time.Second) // 每 5 秒检查一次 Pod 状态

		pod := &v1.Pod{}
		if err := r.Get(ctx, podKey, pod); err != nil {
			r.Log.Error(err, "Failed to get Pod", "Pod.Name", name)
			continue // 获取失败则继续等待
		}

		// 使用辅助函数判断 Pod 是否健康
		if isPodHealthy(*pod) {
			r.Log.Info("Pod is healthy", "Pod.Name", name)
			break // Pod 就绪则跳出循环
		} else {
			r.Log.Info("Waiting for Pod to become healthy", "Pod.Name", name)
		}
	}
	return nil
}

// 检查 Pod 是否健康的函数
func isPodHealthy(pod v1.Pod) bool {
	// 检查 Pod 的运行状态是否为 Running
	// 只有当 Pod 处于 Running 状态，才表示它已经调度到节点并开始运行
	return pod.Status.Phase == v1.PodRunning &&
		// 检查容器状态数组不为空，确保至少有一个容器
		len(pod.Status.ContainerStatuses) > 0 &&
		// 检查第一个容器的 Ready 状态为 true，表示容器已经就绪
		pod.Status.ContainerStatuses[0].Ready
}

// 获取或创建 Service 的函数
func (r *MysqlClusterReconciler) getOrCreateService(ctx context.Context, serviceName, role, namespace string, cluster databasev1.MysqlCluster) (*v1.Service, error) {
	// 定义一个空的 Service 对象，用于存放查询结果
	service := &v1.Service{}
	serviceKey := client.ObjectKey{Namespace: namespace, Name: serviceName}

	// 尝试从 Kubernetes 集群中获取已经存在的 Service
	if err := r.Get(ctx, serviceKey, service); err != nil {
		if errors.IsNotFound(err) {
			// 如果未找到（即 Service 不存在），则调用 createService 创建一个新的 Service 对象
			service = r.createService(serviceName, role, namespace, cluster)

			// 将新创建的 Service 对象提交到 Kubernetes 中
			if err := r.Create(ctx, service); err != nil {
				// 如果创建失败，返回错误
				return nil, err
			}
		} else {
			// 如果是其他类型的错误（非未找到），直接返回该错误
			return nil, err
		}
	}

	// 返回已存在或新创建的 Service 对象
	return service, nil
}

// 创建 Service 对象的辅助函数
func (r *MysqlClusterReconciler) createService(name, role, namespace string, cluster databasev1.MysqlCluster) *v1.Service {
	// 定义 OwnerReference，用于设置 Service 的所属资源，方便级联删除
	ownerRef := metav1.OwnerReference{
		APIVersion: MysqlClusterAPIVersion,                 // 设置 API 版本（常量）
		Kind:       MysqlClusterKind,                       // 设置 Kind，例如 "MysqlCluster"
		Name:       cluster.Name,                           // 设置拥有者资源的名称
		UID:        cluster.UID,                            // 设置拥有者资源的唯一标识符
		Controller: func(b bool) *bool { return &b }(true), // 标记该引用为控制器（关键）
	}

	// 返回一个新的 Service 对象
	return &v1.Service{
		ObjectMeta: metav1.ObjectMeta{
			Name:      name,      // 设置 Service 名称
			Namespace: namespace, // 设置命名空间
			Labels: map[string]string{ // 设置标签，用于 Pod 的选择器和分组
				"app":  "mysql", // 通常代表应用名称
				"role": role,    // 角色标签（例如 master 或 slave）
			},
			OwnerReferences: []metav1.OwnerReference{
				ownerRef, // 将上述定义的拥有者设置为当前 Service 的 Owner
			},
		},
		Spec: v1.ServiceSpec{
			Selector: map[string]string{ // 设置选择器，Service 会选择匹配这些标签的 Pod
				"app":  "mysql",
				"role": role,
			},
			Ports: []v1.ServicePort{
				{
					Port:       3306,                 // Service 暴露的端口
					TargetPort: intstr.FromInt(3306), // 容器内部的端口
					Protocol:   v1.ProtocolTCP,       // 使用 TCP 协议
				},
			},
			Type: v1.ServiceTypeClusterIP, // 设置 Service 类型为 ClusterIP，仅集群内部访问
		},
	}
}

// 制作主从同步的函数
func (r *MysqlClusterReconciler) setupMasterSlaveReplication(ctx context.Context, masterName string, slaveNames []string, cluster databasev1.MysqlCluster) error {
	log := log.FromContext(ctx)
	log.Info("setupMasterSlaveReplication函数", "masterName", masterName, "slaveNames", slaveNames)

	// 获取主库 Pod 对象，用于后续执行创建复制用户的命令
	masterPod := &v1.Pod{}
	masterPodKey := client.ObjectKey{Namespace: cluster.Namespace, Name: masterName}
	if err := r.Get(ctx, masterPodKey, masterPod); err != nil {
		// 获取失败则返回错误
		return fmt.Errorf("failed to get master pod %s: %v", masterName, err)
	}

	// 给主库 Pod 打上 role=master 的标签，方便被 Service 匹配到（通过 selector）
	// 非常重要，从库通过主库的 Service 来连接主库进行数据同步
	if err := r.labelPod(ctx, masterName, "master", cluster); err != nil {
		return fmt.Errorf("failed to label master pod %s: %v", masterName, err)
	}

	// 在主库中创建用于主从同步的账号 replica，并授予复制权限
	// 同时执行 STOP SLAVE，防止主库之前是从库残留状态
	masterCommand := fmt.Sprintf(
		"mysql -uroot -p%s -e \"CREATE USER IF NOT EXISTS 'replica'@'%%' IDENTIFIED BY 'password'; GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%%';STOP slave;\"",
		MySQLPassword,
	)
	if _, err := r.execCommandOnPod(masterPod, masterCommand); err != nil {
		// 执行命令失败则返回错误
		return fmt.Errorf("failed to execute command on master pod %s: %v", masterName, err)
	}

	// 遍历所有从库 Pod，进行主从复制配置
	for _, slaveName := range slaveNames {
		// 获取从库 Pod 对象
		slavePod := &v1.Pod{}
		slavePodKey := client.ObjectKey{Namespace: cluster.Namespace, Name: slaveName}
		if err := r.Get(ctx, slavePodKey, slavePod); err != nil {
			return fmt.Errorf("failed to get slave pod %s: %v", slaveName, err)
		}

		// 配置主从复制：
		// 1. 先停止 slave（防止之前配置冲突）
		// 2. 配置主库地址、同步账号密码等
		// 3. 启动 slave 开始复制
		masterServiceName := cluster.Spec.MasterService // 获取主库 Service 名（用于从库连接）
		slaveCommand := fmt.Sprintf(
			"mysql -uroot -p%s -e \"STOP SLAVE;CHANGE MASTER TO MASTER_HOST='%s', MASTER_USER='replica', MASTER_PASSWORD='password', MASTER_AUTO_POSITION=1; START SLAVE;\"",
			MySQLPassword,
			masterServiceName,
		)
		if _, err := r.execCommandOnPod(slavePod, slaveCommand); err != nil {
			// 执行命令失败返回错误
			return fmt.Errorf("failed to execute command on slave pod %s: %v", slaveName, err)
		}

		// 给从库 Pod 打上 role=slave 标签
		// 同样用于被 slave 类型的 Service 选择到
		if err := r.labelPod(ctx, slaveName, "slave", cluster); err != nil {
			return fmt.Errorf("failed to label slave pod %s: %v", slaveName, err)
		}
	}

	// 所有主从配置完成，返回 nil 表示成功
	return nil
}

// labelPod 为 Pod 打标签
func (r *MysqlClusterReconciler) labelPod(ctx context.Context, podName, role string, cluster databasev1.MysqlCluster) error {
	// 定义 Pod 对象，用于从 Kubernetes 获取指定的 Pod 实例
	pod := &v1.Pod{}
	podKey := client.ObjectKey{Namespace: cluster.Namespace, Name: podName}

	// 调用 r.Get 获取指定名称空间下的 podName 对应的 Pod
	if err := r.Get(ctx, podKey, pod); err != nil {
		// 如果获取失败（比如 Pod 不存在），返回错误
		return fmt.Errorf("failed to get pod %s: %v", podName, err)
	}

	// 检查 Pod 是否已有标签字段（防止为 nil）
	// 如果为空则初始化一个空的标签 map
	if pod.Labels == nil {
		pod.Labels = make(map[string]string)
	}

	// 设置标签 key 为 "role"，value 为传入的角色（如 master 或 slave）
	// 后续 Service 可以通过 selector: role=xxx 来选中对应的 Pod
	pod.Labels["role"] = role

	// 更新 Pod 对象，将修改后的标签同步到 Kubernetes 中
	if err := r.Update(ctx, pod); err != nil {
		// 如果更新失败则返回错误
		return fmt.Errorf("failed to update pod %s: %v", podName, err)
	}

	// 标签打成功，返回 nil 表示成功
	return nil
}

// execCommandOnPod 在指定 Pod 内执行命令，返回命令输出结果或错误
func (r *MysqlClusterReconciler) execCommandOnPod(pod *v1.Pod, command string) (string, error) {
	// 使用 InClusterConfig 获取集群内部访问 Kubernetes API 的配置（适用于程序在集群内部运行）
	// 如果程序运行在集群外，可以改为使用 clientcmd.BuildConfigFromFlags 加载 ~/.kube/config
	// Load kubeconfig from default location
	//kubeconfig := os.Getenv("KUBECONFIG")
	//if kubeconfig == "" {
	//	kubeconfig = "/root/.kube/config" // Fallback to default path
	//}
	//config, err := clientcmd.BuildConfigFromFlags("", KubeConfigPath) // 来自包："k8s.io/client-go/tools/clientcmd"
	config, err := rest.InClusterConfig() // 来自包："k8s.io/client-go/rest"
	if err != nil {
		// 获取配置失败，返回错误
		return "", err
	}

	// 创建 Kubernetes 客户端集
	kubeClient, err := kubernetes.NewForConfig(config)
	if err != nil {
		// 创建失败返回错误
		return "", err
	}

	// 创建用于执行命令的 REST 请求对象
	restClient := kubeClient.CoreV1().RESTClient()
	req := restClient.
		Post().                                          // 使用 POST 方法
		Resource("pods").                                // 针对的资源是 pods
		Name(pod.Name).                                  // 指定 Pod 名称
		Namespace(pod.Namespace).                        // 指定 Pod 所在命名空间
		SubResource("exec").                             // 使用 exec 子资源，即在容器中执行命令
		Param("stdin", "false").                         // 不使用标准输入
		Param("stdout", "true").                         // 启用标准输出
		Param("stderr", "true").                         // 启用标准错误输出
		Param("tty", "false").                           // 不分配 TTY
		Param("container", pod.Spec.Containers[0].Name). // 指定执行命令的容器名称（Pod 可能包含多个容器）
		Param("command", "/bin/sh").                     // 使用 shell 执行
		Param("command", "-c").                          // 使用 `-c` 参数传入命令字符串
		Param("command", command)                        // 实际要执行的命令字符串

	// 创建 SPDY 协议的执行器（executor），用于发起远程命令执行
	executor, err := remotecommand.NewSPDYExecutor(config, "POST", req.URL())
	if err != nil {
		// 创建执行器失败
		return "", err
	}

	// 执行命令，并把输出写入 output 变量
	var output strings.Builder
	err = executor.Stream(remotecommand.StreamOptions{
		Stdout: &output,   // 捕获标准输出
		Stderr: os.Stderr, // 标准错误直接输出到本地控制台（调试方便）
	})
	if err != nil {
		// 命令执行过程中出错
		return "", err
	}

	// 命令执行成功，返回输出结果字符串
	return output.String(), nil
}

// 通过master-service中的endpoint地址获取当前主库pod名
func (r *MysqlClusterReconciler) getMasterPodNameFromEndpoints(ctx context.Context, cluster databasev1.MysqlCluster) (string, error) {
	log := log.FromContext(ctx) // 从上下文中获取日志对象，用于后续记录日志信息

	// 创建一个Endpoints对象，用来存放从Kubernetes API获取的Service对应的Endpoints信息
	endpoints := &v1.Endpoints{}

	// 调用r.Get方法，从Kubernetes集群中获取指定namespace下，名称为cluster.Spec.MasterService的Endpoints资源
	// 如果获取失败，则返回错误，错误信息中带上具体的Service名称和错误内容
	if err := r.Get(ctx, client.ObjectKey{Name: cluster.Spec.MasterService, Namespace: cluster.Namespace}, endpoints); err != nil {
		return "", fmt.Errorf("failed to get endpoints for service %s: %v", cluster.Spec.MasterService, err)
	}

	// 判断获取到的Endpoints资源中是否有可用的子集（Subsets），以及子集内是否有地址（Addresses）
	// 如果没有，则说明该Service当前没有关联的Pod，返回相应错误信息
	if len(endpoints.Subsets) == 0 || len(endpoints.Subsets[0].Addresses) == 0 {
		return "", fmt.Errorf("no endpoints found for service %s", cluster.Spec.MasterService)
	}

	// 从Endpoints的第一个子集的第一个地址中获取TargetRef的Pod名称
	// TargetRef是指向Pod对象的引用，通过它可以拿到对应的Pod名字
	masterPodName := endpoints.Subsets[0].Addresses[0].TargetRef.Name

	// 记录日志，打印通过Endpoints找到的主库Pod名称
	log.Info("从master-service的endpoints获取主pod名", "masterPodName", masterPodName)

	// 返回找到的主库Pod名称，error为nil表示成功
	return masterPodName, nil
}

// 获取所有当前从pod的名字
func (r *MysqlClusterReconciler) getReplicaPodsNames(ctx context.Context, cluster databasev1.MysqlCluster, masterPodName string) ([]string, error) {
	// 调用r.getActualReplicaInfo获取当前集群中实际存在的副本数量和所有Pod名称列表
	actualReplicaCount, podNames := r.getActualReplicaInfo(ctx, cluster)

	// 如果实际副本数量为0，说明没有从库Pod，直接返回一个空数组和nil错误
	if actualReplicaCount == 0 {
		return nil, nil
	}

	// 下面是注释掉的代码，原本是重新通过endpoints获取主库Pod名称，但这里直接传入了masterPodName参数
	//masterPodName, err := r.getMasterPodNameFromEndpoints(ctx, cluster)
	//if err != nil {
	//	return nil, err
	//}

	// 创建一个空的切片，用于存放过滤后的从库Pod名称
	var replicaPodNames []string

	// 遍历所有Pod名称
	for _, podName := range podNames {
		// 过滤掉与主库Pod名称相同的Pod，因为主库不是从库
		if podName != masterPodName {
			// 将剩余的Pod名称添加到从库Pod名称列表中
			replicaPodNames = append(replicaPodNames, podName)
		}
	}

	// 返回过滤后的从库Pod名称列表，以及nil表示无错误
	return replicaPodNames, nil
}

```

### 6、reconcile_master_slave.go

```go
package controller

import (
	"context"                                   // 用于处理上下文，提供超时、取消等操作
	"fmt"                                       // 格式化I/O函数，如字符串格式化和打印
	"sigs.k8s.io/controller-runtime/pkg/client" // 提供与 Kubernetes API 交互的客户端
	"sigs.k8s.io/controller-runtime/pkg/log"
	"strings"

	databasev1 "github.com/xiaowu/api/v1" // 导入自定义的 MySQLCluster API 资源定义
	v1 "k8s.io/api/core/v1"               // 核心 Kubernetes API 对象，例如 Pod 和 Service
	ctrl "sigs.k8s.io/controller-runtime"
)

// 主从调谐逻辑，负责检测主库状态并确保主从关系正确
func (r *MysqlClusterReconciler) reconcileMasterSlave(ctx context.Context, cluster databasev1.MysqlCluster) (ctrl.Result, error) {
	// 1. 检查主库是否存活
	masterAlive, err := r.checkMasterStatus(ctx, cluster)
	if err != nil {
		// 如果检查主库状态出错，直接返回错误，调谐会被重新排队执行
		return ctrl.Result{}, err
	}

	if !masterAlive {
		// 2. 如果主库挂掉了，处理主库故障（选举新主库并重新配置）
		if err := r.handleMasterFailure(ctx, cluster); err != nil {
			// 处理故障时出错，同样返回错误重试
			return ctrl.Result{}, err
		}
	} else {
		// 3. 主库正常，检查所有从库的主从状态
		masterPodName, failedReplicas, err := r.checkReplicaStatus(ctx, cluster)
		if err != nil {
			// 检查从库状态失败，返回错误重试
			return ctrl.Result{}, err
		}

		// 4. 重新配置状态异常的从库，修复主从复制关系
		if err := r.setupMasterSlaveReplication(ctx, masterPodName, failedReplicas, cluster); err != nil {
			return ctrl.Result{}, err
		}

		// 5. 确保所有非主库 Pod 的标签 role 都是 slave，保证标识正确
		if err := r.ensureSlaveRoles(ctx, cluster); err != nil {
			return ctrl.Result{}, err
		}
	}

	// 6. 所有操作成功，返回成功结果
	return ctrl.Result{}, nil
}

// 通过查看 master-service 关联的 endpoint 是否为空来判断主库是否挂掉
func (r *MysqlClusterReconciler) checkMasterStatus(ctx context.Context, cluster databasev1.MysqlCluster) (bool, error) {
	// 创建 Endpoints 对象，用于存储 master-service 关联的 endpoint 信息
	endpoints := &v1.Endpoints{}

	// 获取指定的 Endpoints，名称为 cluster.Spec.MasterService，命名空间为 cluster.Namespace
	if err := r.Get(ctx, client.ObjectKey{Name: cluster.Spec.MasterService, Namespace: cluster.Namespace}, endpoints); err != nil {
		// 获取失败，返回错误
		return false, err
	}

	// 判断 Endpoints 是否包含地址，如果没有地址说明主库挂掉了
	if len(endpoints.Subsets) == 0 || len(endpoints.Subsets[0].Addresses) == 0 {
		return false, nil // 主库挂掉，返回 false
	}

	return true, nil // 主库正常，返回 true
}

// 当主库挂掉时，选举新的主库并重新配置主从关系
func (r *MysqlClusterReconciler) handleMasterFailure(ctx context.Context, cluster databasev1.MysqlCluster) error {
	log := log.FromContext(ctx)

	// 选举新的主库，返回新主库名称和剩余从库列表
	newMasterName, remainingSlaves, err := r.electNewMaster(ctx, cluster)
	if err != nil {
		// 选举失败，返回错误
		return err
	}

	// 记录日志，说明选举出的新主库和剩余从库
	log.Info("选举出新主库", "newMasterName", newMasterName, "remainingSlaves", remainingSlaves)

	// 重新配置主从复制关系，新主库与剩余从库建立主从关系
	err = r.setupMasterSlaveReplication(ctx, newMasterName, remainingSlaves, cluster)
	if err != nil {
		// 重新配置失败，返回错误
		return err
	}

	return nil // 成功处理主库故障
}

// 检查所有从库的主从状态，返回当前主库名和状态异常的从库名称列表
func (r *MysqlClusterReconciler) checkReplicaStatus(ctx context.Context, cluster databasev1.MysqlCluster) (string, []string, error) {
	/*
		1. 调用已有的函数获取主库名称和所有从库名称
		2. 对每个从库执行命令查看主从复制状态，判断 SQL 线程和 IO 线程是否都为 "Yes"
		3. 返回当前主库名称和所有状态异常的从库名称
	*/
	log := log.FromContext(ctx)

	// 获取主库 Pod 名称
	masterPodName, err := r.getMasterPodNameFromEndpoints(ctx, cluster)
	if err != nil {
		return "", nil, fmt.Errorf("failed to get master pod name: %v", err)
	}

	// 获取所有从库 Pod 名称（排除主库）
	replicaPodNames, err := r.getReplicaPodsNames(ctx, cluster, masterPodName)
	if err != nil {
		return "", nil, fmt.Errorf("failed to get replica pods: %v", err)
	}

	// 构造用于查询主从状态的 MySQL 命令，MySQLPassword 变量应为集群 root 密码
	sqlQuery := fmt.Sprintf(
		"mysql -uroot -p%s -e \"SHOW SLAVE STATUS \\G\"",
		MySQLPassword,
	)

	var failedReplicas []string // 用于记录主从状态异常的从库名称

	for _, replicaPodName := range replicaPodNames {
		// 获取当前从库对应的 Pod 对象
		pod := &v1.Pod{}
		if err := r.Get(ctx, client.ObjectKey{Name: replicaPodName, Namespace: cluster.Namespace}, pod); err != nil {
			return "", nil, fmt.Errorf("failed to get pod %s: %v", replicaPodName, err)
		}

		// 在该 Pod 上执行 MySQL 命令，获取主从复制状态信息
		output, err := r.execCommandOnPod(pod, sqlQuery)
		if err != nil {
			return "", nil, fmt.Errorf("failed to execute command on pod %s: %v", replicaPodName, err)
		}

		// 检查输出中是否包含 Slave_SQL_Running: Yes 和 Slave_IO_Running: Yes，表示复制正常
		sqlThread := strings.Contains(output, "Slave_SQL_Running: Yes")
		ioThread := strings.Contains(output, "Slave_IO_Running: Yes")

		// 如果任一线程未运行，则认为该从库状态异常，加入失败列表
		if !(sqlThread && ioThread) {
			failedReplicas = append(failedReplicas, replicaPodName)
		}
	}

	log.Info("主从状态检查完成", "主库", masterPodName, "状态失败的从库", failedReplicas)

	// 返回主库名称和所有异常的从库名称
	return masterPodName, failedReplicas, nil
}

// 确保所有非主库的 Pod 标签 role 都是 slave，标识从库身份
func (r *MysqlClusterReconciler) ensureSlaveRoles(ctx context.Context, cluster databasev1.MysqlCluster) error {
	// 获取主库 Pod 名称
	//endpoints := &v1.Endpoints{}
	//if err := r.Get(ctx, client.ObjectKey{Name: cluster.Spec.MasterService, Namespace: cluster.Namespace}, endpoints); err != nil {
	//	return err
	//}
	//masterPodName := endpoints.Subsets[0].Addresses[0].TargetRef.Name

	masterPodName, err := r.getMasterPodNameFromEndpoints(ctx, cluster)
	if err != nil {
		return err
	}

	// 查询当前命名空间下所有 app=mysql 标签的 Pod
	podList := &v1.PodList{}
	listOpts := []client.ListOption{
		client.InNamespace(cluster.Namespace),
		client.MatchingLabels{"app": "mysql"},
	}
	if err := r.List(ctx, podList, listOpts...); err != nil {
		return err
	}

	// 遍历所有 Pod
	for _, pod := range podList.Items {
		// 如果当前 Pod 不是主库，且 role 标签不是 slave，则更新标签为 slave
		if pod.Name != masterPodName && pod.Labels["role"] != "slave" {
			pod.Labels["role"] = "slave"
			// 更新 Pod 对象，保存修改后的标签
			if err := r.Update(ctx, &pod); err != nil {
				return err
			}
		}
	}

	return nil // 确保所有非主库 Pod 标签正确
}

```

### 7、reconcile_replicas.go

```go
package controller

import (
	"context"                                   // 用于处理上下文，提供超时、取消等操作
	"fmt"                                       // 格式化I/O函数，如字符串格式化和打印
	"k8s.io/apimachinery/pkg/labels"            // Kubernetes 标签选择器相关包
	"regexp"                                    // 正则表达式处理包
	"sigs.k8s.io/controller-runtime/pkg/client" // 提供与 Kubernetes API 交互的客户端
	"sigs.k8s.io/controller-runtime/pkg/log"    // 日志工具
	"strconv"                                   // 字符串与数字转换工具

	databasev1 "github.com/xiaowu/api/v1" // 导入自定义的 MysqlCluster API 资源定义
	v1 "k8s.io/api/core/v1"               // 核心 Kubernetes API 对象，例如 Pod 和 Service
	ctrl "sigs.k8s.io/controller-runtime" // controller-runtime 框架
)

// reconcileReplicas 是 MysqlCluster 的核心控制循环，负责保证副本数量与期望一致
func (r *MysqlClusterReconciler) reconcileReplicas(ctx context.Context, cluster databasev1.MysqlCluster) (ctrl.Result, error) {
	log := log.FromContext(ctx) // 从上下文中获取日志记录器

	// 获取当前实际的副本数和副本名称列表
	actualReplicaCount, actualReplicaNames := r.getActualReplicaInfo(ctx, cluster)
	expectedReplicaCount := cluster.Spec.Replicas // 期望的副本数

	// 判断实际副本数是否与期望一致
	if actualReplicaCount != expectedReplicaCount {
		log.Info("副本数与预期不符", "实际副本数", actualReplicaCount, "预期副本数", expectedReplicaCount)

		// 从实际存在的 Pod 名称中提取出编号（如 mysql-01 -> 01）
		actualPodNumbers := extractPodNumbers(actualReplicaNames)

		// 根据期望副本数，生成一个完整的编号列表（如 01, 02, 03）
		expectedPodNumbers := generateExpectedPodNumbers(int(expectedReplicaCount))

		// 找出期望编号中缺失的那些编号（即需要新建的 Pod 编号）
		missingPodNumbers := findMissingPodNumbers(expectedPodNumbers, actualPodNumbers)

		// 针对每个缺失的编号，创建对应的 PVC、ConfigMap 和 Pod
		for _, podNumber := range missingPodNumbers {
			// 拼接出 Pod 和 ConfigMap 的名称
			podName := fmt.Sprintf("mysql-%s", podNumber)
			configMapName := fmt.Sprintf("mysql-config-%s", podNumber)

			// PVC 名称与 Pod 名称一致
			pvcName := fmt.Sprintf("mysql-%s", podNumber)
			storageClassName := cluster.Spec.Storage.StorageClassName // 存储类名称
			storageSize := cluster.Spec.Storage.Size                  // 存储大小

			// 创建 PVC（持久化存储卷），如果已经存在则跳过
			if err := r.createPVC(ctx, pvcName, storageClassName, cluster.Namespace, storageSize, &cluster); err != nil {
				return ctrl.Result{}, err // 创建失败返回错误
			}

			// podNumber 是字符串，转成整数传给 createConfigMap 用作 serverId
			serverIdNum, _ := strconv.Atoi(podNumber)

			// 创建 ConfigMap，包含 MySQL 配置，绑定到对应 serverId
			if err := r.createConfigMap(ctx, configMapName, serverIdNum, cluster.Namespace, &cluster); err != nil {
				return ctrl.Result{}, err // 创建失败返回错误
			}

			// 创建 Pod，关联对应的镜像、ConfigMap 和 PVC
			if err := r.createPod(ctx, podName, cluster.Spec.Image, configMapName, pvcName, cluster.Namespace, &cluster); err != nil {
				return ctrl.Result{}, err // 创建失败返回错误
			}

			// 记录日志，方便调试和查看创建的 Pod
			log.Info("创建缺失的 Pod", "PodName", podName)
		}
	}

	// 副本数符合预期，或已完成创建缺失副本，正常结束
	return ctrl.Result{}, nil
}

// getActualReplicaInfo 查询当前集群 namespace 下，带有标签 app=mysql 的 Pod 列表
// 返回实际副本数量和 Pod 名称切片
func (r *MysqlClusterReconciler) getActualReplicaInfo(ctx context.Context, cluster databasev1.MysqlCluster) (int32, []string) {
	log := log.FromContext(ctx)

	podList := &v1.PodList{} // 创建 PodList 用于存储查询结果

	// 设置查询选项：命名空间、标签选择器，筛选 app=mysql 的 Pod
	listOptions := &client.ListOptions{
		Namespace:     cluster.Namespace,
		LabelSelector: labels.SelectorFromSet(map[string]string{"app": "mysql"}),
	}

	// 调用 client.List 方法，从 Kubernetes API 获取 Pod 列表
	if err := r.List(ctx, podList, listOptions); err != nil {
		log.Error(err, "获取 Pod 列表失败")
		return 0, nil // 出错返回空结果
	}

	// 解析 Pod 名称
	var podNames []string
	for _, pod := range podList.Items {
		podNames = append(podNames, pod.Name)
	}

	log.Info("当前副本情况", "副本数", len(podList.Items), "PodNames", podNames, "预期副本数", cluster.Spec.Replicas)

	// 返回实际副本数和 Pod 名称列表
	return int32(len(podList.Items)), podNames
}

// extractPodNumbers 从 Pod 名称列表中提取编号，格式如 mysql-01 提取 01
func extractPodNumbers(podNames []string) []string {
	var podNumbers []string
	// 通过正则表达式匹配 mysql-后面的数字编号
	var podNamePattern = regexp.MustCompile(`mysql-(\d+)`)

	for _, name := range podNames {
		// 查找匹配项，匹配成功则将编号追加到 podNumbers 列表
		if matches := podNamePattern.FindStringSubmatch(name); len(matches) > 1 {
			podNumbers = append(podNumbers, matches[1])
		}
	}
	return podNumbers
}

// generateExpectedPodNumbers 根据期望副本数量生成编号列表，编号从 01 开始，格式为两位数字符串
func generateExpectedPodNumbers(count int) []string {
	var podNumbers []string
	for i := 1; i <= count; i++ {
		// 格式化成 2 位数字字符串，比如 01, 02, 03...
		podNumbers = append(podNumbers, fmt.Sprintf("%02d", i))
	}
	return podNumbers
}

// findMissingPodNumbers 对比期望编号和实际编号，返回缺失的编号列表
func findMissingPodNumbers(expected, actual []string) []string {
	// 用 map 结构存储期望编号
	expectedSet := make(map[string]struct{}, len(expected))
	for _, pod := range expected {
		expectedSet[pod] = struct{}{}
	}

	// 从期望编号中删除实际存在的编号，剩余即为缺失的
	for _, pod := range actual {
		delete(expectedSet, pod)
	}

	var missingPodNumbers []string
	// 把缺失编号转成切片返回
	for pod := range expectedSet {
		missingPodNumbers = append(missingPodNumbers, pod)
	}
	return missingPodNumbers
}

```

### 8、elect_new_master.go

```go
package controller

import (
	"context" // 用于处理上下文，提供超时、取消等操作
	"fmt"     // 格式化I/O函数，如字符串格式化和打印
	"strconv"
	"strings"
	"time"

	"k8s.io/apimachinery/pkg/labels"
	"sigs.k8s.io/controller-runtime/pkg/client" // 提供与 Kubernetes API 交互的客户端

	databasev1 "github.com/xiaowu/api/v1" // 导入自定义的 MySQLCluster API 资源定义
	v1 "k8s.io/api/core/v1"               // 核心 Kubernetes API 对象，例如 Pod 和 Service
)

// 选主逻辑：
// MHA数据一致性和选择原则
/*
GTID模式：MHA会比较每个从库的GTID集合。选择包含主库GTID率最高的从库作为新的主库。
*/

// electNewMaster 选举一个新的主库，并返回新主库名称和剩余的从库名称列表
func (r *MysqlClusterReconciler) electNewMaster(ctx context.Context, cluster databasev1.MysqlCluster) (string, []string, error) {
	// 定义用于过滤从库Pod的选项（根据标签 role=slave）
	listOpts := client.ListOptions{
		Namespace:     cluster.Namespace,
		LabelSelector: labels.SelectorFromSet(map[string]string{"role": "slave"}),
	}

	// 列出所有符合条件的从库Pod
	slavePods := &v1.PodList{}
	if err := r.List(ctx, slavePods, &listOpts); err != nil {
		return "", nil, fmt.Errorf("failed to list slave pods: %v", err)
	}

	// 初始化选举变量：最佳从库和最高得分
	var bestSlave *v1.Pod
	var highestScore float64 = -1

	// 遍历所有从库Pod，挑选出数据得分最高的
	for _, pod := range slavePods.Items {
		// 跳过不健康的Pod
		if !isPodHealthy(pod) {
			continue
		}

		// 获取该从库的得分（包括GTID和数据量得分）
		dataScore, err := r.getDataScore(ctx, &pod)
		if err != nil {
			return "", nil, fmt.Errorf("failed to get data score for pod %s: %v", pod.Name, err)
		}

		// 如果当前从库得分比已有最高得分高，则更新最佳选择
		if dataScore > highestScore {
			highestScore = dataScore
			bestSlave = &pod
		}
	}

	// 如果找不到合适的从库，返回错误
	if bestSlave == nil {
		return "", nil, fmt.Errorf("no suitable slave found to be promoted to master")
	}

	// 记录选出的新主库名称
	newMasterName := bestSlave.Name

	// 过滤掉新主库，获得剩余的从库名称列表
	var remainingSlaves []string
	for _, pod := range slavePods.Items {
		if pod.Name != newMasterName {
			remainingSlaves = append(remainingSlaves, pod.Name)
		}
	}

	return newMasterName, remainingSlaves, nil
}

// getDataScore 计算某个Pod的数据得分，包含GTID完整度和数据量两个维度
func (r *MysqlClusterReconciler) getDataScore(ctx context.Context, pod *v1.Pod) (float64, error) {
	// 获取当前已缓存的主库GTID集合快照（全局快照）
	masterGTIDSet := r.MasterGTIDSnapshot

	// 获取该从库Pod的GTID集合
	slaveGTIDSet, err := r.getSlaveGTIDSet(pod)
	if err != nil {
		return 0.0, fmt.Errorf("failed to get slave GTID set: %v", err)
	}

	// 计算GTID完整度得分（主库与从库GTID的匹配程度）
	gtidScore := r.calculateGTIDScore(masterGTIDSet, slaveGTIDSet)

	// 获取该从库Pod的数据大小（字节数）
	dataSize, err := r.getDataSize(ctx, pod)
	if err != nil {
		return 0.0, fmt.Errorf("failed to get data size: %v", err)
	}

	// 计算数据量得分（这里直接用数据大小作为得分）
	dataScore := r.calculateDataScore(dataSize)

	// 最终得分是GTID得分和数据得分的叠加
	finalScore := gtidScore + dataScore

	return finalScore, nil
}

// getMasterGTIDSet 获取主库Pod的GTID集合（Executed_Gtid_Set）
func (r *MysqlClusterReconciler) getMasterGTIDSet(pod *v1.Pod) (string, error) {
	// 通过MySQL命令获取主库的Executed_Gtid_Set字段
	masterCommand := "mysql -uroot -p%s -e \"SHOW MASTER STATUS\\G\" | grep 'Executed_Gtid_Set:' | awk '{print $2}'"
	command := fmt.Sprintf(masterCommand, MySQLPassword)

	// 在Pod内执行命令，返回结果
	gtidSet, err := r.execCommandOnPod(pod, command)
	if err != nil {
		return "", err
	}
	return gtidSet, nil
}

// getSlaveGTIDSet 获取从库Pod的GTID集合（Retrieved_Gtid_Set）
func (r *MysqlClusterReconciler) getSlaveGTIDSet(pod *v1.Pod) (string, error) {
	// 通过MySQL命令获取从库的Retrieved_Gtid_Set字段
	slaveCommand := "mysql -uroot -p%s -e \"SHOW SLAVE STATUS\\G\" | grep 'Retrieved_Gtid_Set:' | awk '{print $2}'"
	command := fmt.Sprintf(slaveCommand, MySQLPassword)

	// 在Pod内执行命令，返回结果
	gtidSet, err := r.execCommandOnPod(pod, command)
	if err != nil {
		return "", err
	}
	return gtidSet, nil
}

// calculateGTIDScore 计算GTID完整度得分，衡量从库GTID集合覆盖主库GTID集合的程度
func (r *MysqlClusterReconciler) calculateGTIDScore(masterGTIDSet, slaveGTIDSet string) float64 {
	// 如果任意一方GTID集合为空，则得分为0
	if masterGTIDSet == "" || slaveGTIDSet == "" {
		return 0.0
	}

	// 将GTID字符串用逗号分隔成数组
	masterGTIDs := strings.Split(masterGTIDSet, ",")
	slaveGTIDs := strings.Split(slaveGTIDSet, ",")

	// 统计从库GTID集合包含的主库GTID数量
	count := 0
	for _, masterGTID := range masterGTIDs {
		for _, slaveGTID := range slaveGTIDs {
			if masterGTID == slaveGTID {
				count++
				break
			}
		}
	}

	// 得分为包含数量占主库GTID数量的比例（0-1之间）
	return float64(count) / float64(len(masterGTIDs))
}

// getDataSize 获取Pod中MySQL数据目录的大小（字节数）
func (r *MysqlClusterReconciler) getDataSize(ctx context.Context, pod *v1.Pod) (int64, error) {
	// MySQL数据目录路径（假设容器中固定此路径）
	dataDirPath := "/var/lib/mysql"

	// 使用du命令计算目录大小，单位为字节
	dataSizeCommand := fmt.Sprintf("du -sb %s | awk '{print $1}'", dataDirPath)

	// 在Pod内执行命令
	output, err := r.execCommandOnPod(pod, dataSizeCommand)
	if err != nil {
		return 0, err
	}

	// 将命令输出解析为整数
	dataSize, err := strconv.ParseInt(strings.TrimSpace(output), 10, 64)
	if err != nil {
		return 0, err
	}

	return dataSize, nil
}

// calculateDataScore 计算数据量得分，这里简单返回数据大小作为得分
func (r *MysqlClusterReconciler) calculateDataScore(dataSize int64) float64 {
	// 简单以数据大小直接作为得分
	return float64(dataSize)
}

// startAndUpdateGTIDSnapshot 启动一个协程，定期更新主库的GTID快照
func (r *MysqlClusterReconciler) startAndUpdateGTIDSnapshot(ctx context.Context, cluster databasev1.MysqlCluster) {
	go func() {
		for {
			// 定义查询条件，筛选带有 role=master 标签的Pod
			listOptions := &client.ListOptions{
				Namespace:     cluster.Namespace,
				LabelSelector: labels.SelectorFromSet(map[string]string{"role": "master"}),
			}

			// 获取Pod列表
			podList := &v1.PodList{}
			err := r.List(ctx, podList, listOptions)
			if err != nil {
				fmt.Printf("Error listing pods: %v\n", err)
				time.Sleep(1 * time.Second) // 出错时短暂等待后重试
				continue
			}

			// 如果没有找到主库Pod，则清空快照并等待下一次检查
			if len(podList.Items) == 0 {
				r.MasterGTIDSnapshot = ""
				time.Sleep(1 * time.Minute) // 每分钟检查一次
				continue
			}

			// 假设只有一个主库Pod，获取第一个Pod
			pod := &podList.Items[0]

			// 获取主库GTID集合快照
			gtidSet, err := r.getMasterGTIDSet(pod)
			if err != nil {
				fmt.Printf("Error getting master GTID set: %v\n", err)
			} else {
				// 更新全局GTID快照变量
				r.MasterGTIDSnapshot = gtidSet
				fmt.Printf("MasterGTIDSnapshot更新成功: %v\n", gtidSet)
			}

			// 每隔一分钟更新一次
			time.Sleep(1 * time.Minute)
		}
	}()
}

```

## 四、物理机运行测试

### 1、修改utlis.go文件

```go
	//"k8s.io/client-go/tools/clientcmd" // 本地连接：通过config.yml方式链接k8s
	"k8s.io/client-go/rest" // 集群内部连接：通过serviceaccess方式链接k8s


...
// kubectl exec 进入pod内执行命令
	//config, err := clientcmd.BuildConfigFromFlags("", KubeConfigPath) // 来自包："k8s.io/client-go/tools/clientcmd"
	config, err := rest.InClusterConfig() // 来自包："k8s.io/client-go/rest"
```

### 2、修改mysqlcluster_controller.go

```go
const (
...
	//KubeConfigPath         = "/root/.kube/config" // Hardcoded kubeconfig path
...
)
```

### 3、运行

```bash
make generate
make run
```

## 五、部署到k8s集群

```bash
make docker-build IMG=<your-operator-image>:<tag>
make docker-build IMG=myregistry/mysql-operator:v1.0.0
make docker-build IMG=myregistry/mysql-operator:v1.0.0
# 然后使用deployment或者statefulset服务运行该镜像
```

## 六、Kubebuilder 原理详解

### 1、Kubebuilder 的核心组件

#### 1.**Controller-runtime 的作用**

>**Controller-runtime** 是 Kubebuilder 构建 Kubernetes Operator 的核心库，它为开发者提供了一套标准化的控制器开发工具，简化了 Kubernetes 资源的管理工作。`controller-runtime` 的主要作用包括

##### 1）控制器管理

>- `controller-runtime` 提供了一个 **Manager**，负责启动和管理所有的控制器。Manager 会管理控制器的生命周期，并确保它们持续运行。
>- 通过 `Manager`，多个控制器可以并行运行，监控和处理不同资源的状态变化。

##### 2）**事件监听与处理**

>**Informer** 监听 Kubernetes 资源的变化事件（如创建、更新、删除等），`controller-runtime` 会通过这些事件驱动控制器的 **Reconcile** 函数。
>
>当资源的状态与期望不一致时，`Reconcile` 函数被触发，执行对应的逻辑进行同步。

##### 3）资源的缓存机制

>`controller-runtime` 使用缓存本地存储资源的状态，减少与 Kubernetes API Server 的交互。控制器可以优先从缓存中获取资源信息，这提高了系统性能并减少了对 API Server 的负载。

##### 4）**封装 Client 交互**

>`controller-runtime` 提供了一个简化的 **Client**，开发者可以通过它来对 Kubernetes 资源进行增删改查操作，而无需直接与 Kubernetes API Server 进行复杂的 HTTP 请求交互。
>
>例如，通过 Client 可以轻松创建或更新 Kubernetes 资源，如 Pods、Deployments、Services 等。

##### 5）为什么能做到上述这些

>**为什么 `controller-runtime` 能做这些？** `controller-runtime` 封装了复杂的 API 调用、缓存机制和事件监听系统，开发者只需编写核心业务逻辑，它会自动处理资源的状态同步、错误处理、重试等过程。它能够通过 Manager 和 Client 管理控制器的生命周期和资源交互，使得 Operator 能够以高效、优雅的方式与 Kubernetes 生态系统交互。

#### 2.**Code Generation（代码生成）的原理**

>**代码生成** 是 Kubebuilder 的一个重要特性，它通过自动生成大量样板代码，减少了开发者的重复工作量。Kubebuilder 的代码生成功能基于 Go 语言的代码注解，通过注解和命令行工具生成相应的 CRD 文件、深度拷贝函数、RBAC 权限等

##### 1）api生成

>当开发者定义自定义资源 (CRD) 的结构体时，Kubebuilder 会根据这些定义自动生成 Kubernetes 所需的 CRD 清单文件以及相应的 Go 代码。例如，你定义的资源 `MySQLSpec` 和 `MySQLStatus` 会生成相应的 `yaml` 文件，用于在集群中注册 CRD。

##### 2）**深度拷贝函数生成**

>Kubernetes 需要在内部对对象进行深度拷贝操作，Kubebuilder 提供了自动生成深度拷贝函数的能力。开发者只需定义资源的结构，生成工具会自动为这些结构体生成 `DeepCopy` 函数，确保对象可以安全地在不同线程和上下文中传递。

##### 3）**RBAC 权限生成**

>在控制器代码中，开发者可以通过注解（如 `+kubebuilder:rbac`）为控制器生成相应的 RBAC 配置。注解会告诉 Kubebuilder 控制器需要对哪些资源有权限，这样 Kubebuilder 会自动生成 Kubernetes 中的 RBAC 规则清单，确保控制器可以正确操作相关资源。

##### 4）为什么可以做到

>**为什么 Kubebuilder 能做到？** Kubebuilder 通过静态分析代码注解，结合 Kubernetes API 的需求，自动生成符合 Kubernetes 标准的资源清单和代码逻辑。它的底层依赖于 Kubernetes 的工具链（如 `controller-tools`）和 Go 语言的反射机制，开发者只需编写核心业务逻辑，Kubebuilder 就能自动生成复杂的样板代码。

### 2、Kubebuilder 的工作流程

>Kubebuilder 的工作流程包括初始化项目、生成 API 和控制器代码、编写业务逻辑，以及最终生成 Kubernetes 清单文件。以下是详细的工作流程。

#### 1.初始化项目

>开发者通过 `kubebuilder init` 命令初始化一个标准化的 Operator 项目。这个命令会生成项目的目录结构，包括 `api/`、`controllers/`、`config/` 等文件夹。

```bash
kubebuilder init --domain mydomain.com --repo github.com/myorg/my-operator
```

##### 1)生成的文件结构

>- `api/`：存放自定义资源的定义文件。
>- `controllers/`：存放控制器的逻辑。
>- `config/`：存放 Kubernetes 的清单文件（如 CRD、RBAC 配置）。

##### 2)开发者要做什么

>执行 `kubebuilder init`，然后根据生成的结构填充业务逻辑。此时，项目已经具备标准的目录结构。

#### 2.**创建 API 和控制器**

>开发者通过 `kubebuilder create api` 命令为自定义资源生成 API 结构和控制器骨架代码。

```bash
kubebuilder create api --group apps --version v1 --kind MySQL
```

##### 1)生成的文件

>- `api/v1/mysql_types.go`：自定义资源 API 结构的定义。
>- `controllers/mysql_controller.go`：控制器的骨架代码。

##### 2)开发者要做什么？

>在 `api/v1/mysql_types.go` 中定义自定义资源的 `Spec` 和 `Status`，在 `controllers/mysql_controller.go` 中编写业务逻辑（如如何创建、更新资源）。

#### 3.**编写控制器逻辑**

>控制器是 Operator 的核心，负责监听和处理资源的状态变化。开发者需要在控制器的 `Reconcile` 函数中编写核心逻辑，例如当检测到 MySQL CR 创建时，如何为它创建一个相应的 Kubernetes `Deployment`。

```go
func (r *MySQLReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 获取 MySQL 实例
    var mysql appsv1alpha1.MySQL
    if err := r.Get(ctx, req.NamespacedName, &mysql); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 检查是否需要创建或更新 Deployment
    // 处理业务逻辑
}

```

##### 1)开发者要做什么

>在 `controllers/mysql_controller.go` 文件中编写核心逻辑，如资源的创建、更新和状态同步。Kubebuilder 提供的骨架代码会自动调用 `Reconcile` 函数，开发者只需专注于业务逻辑。

#### 4.**生成代码和配置文件**

>开发者完成业务逻辑编写后，运行以下命令生成辅助代码和 Kubernetes 清单文件。
>
>```bash
>make generate
>make manifests
>```

##### 1)作用

>- **`make generate`**：生成深度拷贝函数、API 注册等必要的辅助代码。
>- **`make manifests`**：生成 CRD、RBAC、Deployment 等 Kubernetes 资源的清单文件。

##### 2）开发者要做什么

>通过 `make generate` 生成自动化的辅助代码，无需手动编写深度拷贝和序列化逻辑。通过 `make manifests` 生成可部署的 CRD 和 Operator 清单文件。

### 3、与 Kubernetes API 的集成

>Kubebuilder 通过 `controller-runtime` 进行与 Kubernetes API 的交互，使用 **ClientSet**、**Informer** 和 **Controller** 协同工作，确保资源状态与期望一致。

#### 1.**如何通过 ClientSet 进行资源管理**

>在 `controller-runtime` 中，**Client** 是与 Kubernetes API Server 交互的主要方式。开发者可以通过 `Client` 进行资源的 CRUD 操作。
>
>>- **获取资源**：通过 `r.Get` 获取当前的资源实例。
>>- **创建资源**：通过 `r.Create` 向 Kubernetes 集群中创建资源。
>>- **更新资源**：通过 `r.Update` 更新资源状态。
>>- **删除资源**：通过 `r.Delete` 删除不再需要的资源。

```go
var mysql appsv1alpha1.MySQL
if err := r.Get(ctx, req.NamespacedName, &mysql); err != nil {
    return ctrl.Result{}, client.IgnoreNotFound(err)
}
```

##### 1）开发者要做什么

>在控制器中使用 `Client` 来管理 Kubernetes 资源，编写 `Get`、`Create`、`Update` 和 `Delete` 的业务逻辑。`controller-runtime` 封装了与 API Server 的交互，简化了操作。

#### 2.**Informer、Controller 如何配合工作**

>**Informer** 监听 Kubernetes 中资源的变化（如创建、更新、删除），并将这些事件通知给相应的控制器。控制器通过 **Reconcile Loop** 响应这些事件，执行资源状态的同步和调整。

##### 1）**Informer 的工作流程**

>- 启动时，Informer 从 API Server 获取资源列表并缓存。
>- 通过 Watch 机制，Informer 监听资源的变化并更新缓存。
>- 当资源发生变化时，Informer 将事件通知给 Controller。

##### 2）**Controller 的工作流程**

>Controller 通过 `Reconcile` 函数响应来自 Informer 的事件。每当有新的事件发生，`Reconcile` 函数被调用，执行资源的状态更新或调整。

##### 3）开发者要做什么

>Kubebuilder 自动生成了 Informer 和 Controller 的配合逻辑。开发者只需在 `Reconcile` 函数中编写具体的业务逻辑，无需手动处理事件监听或缓存管理。

## 七、总结：开发者如何使用 Kubebuilder 开发 Operator

>**初始化项目**：使用 `kubebuilder init` 创建标准项目结构。
>
>**创建 API 和控制器**：通过 `kubebuilder create api` 生成 CRD 和控制器骨架代码。
>
>**编写 API 和控制器**：定义 CRD 的 `Spec` 和 `Status`，编写控制器的 `Reconcile` 逻辑。
>
>**生成代码和清单文件**：运行 `make generate` 和 `make manifests`，生成 Kubernetes 所需的清单文件和代码。
>
>**部署和测试 Operator**：通过 `make install` 部署 CRD，使用 `kubectl` 验证 Operator 的工作效果。

