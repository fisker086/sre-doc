# Kustomize 入门与实战详解

在 Kubernetes 应用管理中，我们常常需要部署相同的应用到多个环境（如开发、测试、生产），但每个环境的配置可能略有不同。例如：镜像标签不同、副本数量不同、资源限制不同等。为了解决这个问题，Kubernetes 社区推出了 **Kustomize** —— 一个强大且原生支持的配置管理工具。

本文将从零开始，详细讲解 Kustomize 的核心概念、工作原理，并通过图解和实例帮助你彻底掌握其使用方法。

## 一、什么是 Kustomize？

**Kustomize** 是一个用于定制化 Kubernetes 资源清单（Manifest）的命令行工具（CLI）。它允许你在不修改原始 YAML 文件的前提下，对任意字段进行“复写”（Override），从而实现多环境配置管理。

> **关键特性**：
>
> - 不依赖模板引擎（如 Helm 中的 Go template）
> - 直接操作标准 YAML/JSON 格式的资源文件
> - 内置于 `kubectl` 工具中（自 v1.14 起），无需额外安装

```bash
# 使用 kubectl 直接调用 kustomize
kubectl apply -k ./overlays/production
```

## 二、Kustomize 的核心架构：Base + Overlays 模式

Kustomize 的设计理念基于 **“基础配置 + 覆盖层”** 的模式：

- `base/`：存放所有环境中通用的基础资源文件（如 Deployment、Service）
- `overlays/`：按环境划分的覆盖目录（如 dev、staging、prod），包含该环境特有的修改

### 1、目录结构示意图

```
my-app/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml    ← 基础资源配置声明
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── redis-deployment.yaml
    └── prod/
        ├── kustomization.yaml
        └── patches.yaml
```

> **说明**：  
> 所有 `kustomization.yaml` 文件是 Kustomize 的入口文件，定义了当前目录下要加载哪些资源、如何打补丁、是否生成 Secret 等。

## 三、kustomization.yaml 核心字段详解

这是 Kustomize 的配置中枢，决定了最终输出的资源集合。

### 1、示例：base/kustomization.yaml

```yaml
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

### 2、示例：overlays/prod/kustomization.yaml

```yaml
bases:
  - ../../base

resources:
  - postgres-secret.yaml

patchesStrategicMerge:
  - patch-deployment.yaml

images:
  - name: myapp:v1.0
    newName: myapp:v2.1

secretGenerator:
  - name: db-password
    literals:
      - password=secure123
```

### 3、字段解释（配图文说明）

| 字段                    | 功能                         | 图示               |
| ----------------------- | ---------------------------- | ------------------ |
| `resources`             | 引入本目录下的资源文件       | 🟩🟦 ➝ 🧩（拼装积木） |
| `bases`                 | 继承 base 目录中的所有资源   | 🔗 base → overlay   |
| `patchesStrategicMerge` | 对已有对象打补丁（推荐方式） | 🛠️ 修改特定字段     |
| `images`                | 修改容器镜像名称或标签       | 🖼️ 替换 image:tag   |
| `secretGenerator`       | 自动生成带随机后缀的 Secret  | 🔐 自动生成加密数据 |

> 💡 **图解说明**：想象你有一辆“基础汽车”（base），然后你可以给它换轮胎（patch）、加装音响（new resource）、更换发动机型号（image）——这些都在 `overlays` 中完成。

## 四、实战演示：构建 Dev 与 Prod 环境

### 步骤 1：创建 Base 配置

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote-app
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: app
          image: gcr.io/example/vote:v1.0
```

```yaml
# base/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml
```

### 步骤 2：创建 Production 覆盖层

```yaml
# overlays/prod/kustomization.yaml
bases:
  - ../../base

patchesStrategicMerge:
  - replica-patch.yaml

images:
  - name: gcr.io/example/vote
    newName: gcr.io/example/vote
    newTag: v2.1-prod

secretGenerator:
  - name: db-secret
    env: db.env
```

```yaml
# overlays/prod/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote-app
spec:
  replicas: 5  # 生产环境更多副本
```

> ✅ 最终效果：生产环境使用 v2.1 镜像、5 个副本、并自动创建数据库密码 Secret。

## 五、高级功能：Patch 补丁机制详解

Kustomize 支持三种 Patch 方式：

| 类型                    | 描述                     | 使用场景                         |
| ----------------------- | ------------------------ | -------------------------------- |
| `patchesStrategicMerge` | 战略性合并补丁（最常用） | 修改字段值（如 replicas、image） |
| `patchesJson6902`       | JSON Patch（RFC 6902）   | 精确控制增删改操作               |
| `patches`（旧版）       | 已弃用，请优先使用前两者 | ——                               |

### 示例：使用 JSON Patch 添加标签

```yaml
patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: vote-app
    path: add-label.json
```

```json
// add-label.json
[
  {
    "op": "add",
    "path": "/spec/template/metadata/labels",
    "value": { "env": "production" }
  }
]
```

> 🎯效果：向 Pod 模板添加 `env=production` 标签，便于调度或监控。

## 六、Kustomize vs Helm：何时选择谁？

| 特性         | Kustomize               | Helm                            |
| ------------ | ----------------------- | ------------------------------- |
| 学习成本     | 低（纯 YAML）           | 较高（需学模板语法）            |
| 灵活性       | 高（可改任何字段）      | 中等（受限于 values 设计）      |
| 分发能力     | 弱（依赖 Git 目录结构） | 强（Chart Repo + Artifact Hub） |
| 适用人群     | 开发者 / 运维工程师     | 终端用户 / 平台团队             |
| 是否隐藏细节 | 否（暴露全部 API）      | 是（封装复杂逻辑）              |

> ✅ **建议组合使用**：  
> 用 Helm 安装通用组件（如 MySQL、Redis），再用 Kustomize 进行微调！

```yaml
# 在 kustomization.yaml 中引用 Helm Chart
helmCharts:
  - name: redis
    repo: https://charts.bitnami.com/bitnami
    version: 16.8.0
    releaseName: my-redis
    valuesInline:
      cluster.enabled: false
      replicaCount: 3
```

---

## 七、总结：为什么需要 Kustomize？

尽管 Helm 很流行，但 Kustomize 仍有不可替代的优势：

1. **无需学习模板语法**：直接写 YAML，适合熟悉 Kubernetes API 的开发者。
2. **精准控制每一个字段**：即使是 annotations、tolerations 等冷门字段也能轻松修改。
3. **天然支持多环境管理**：通过 base + overlays 实现高效复用。
4. **与 kubectl 深度集成**：一条命令即可部署，无需额外工具链。

> ⚠️ 缺点提醒：  
>
> - 目录结构必须规范，否则难以维护  
> - 缺乏统一的发布仓库，分发不如 Helm 方便  

## 结语

Kustomize 是现代 Kubernetes 配置管理的重要组成部分。它不是为了取代 Helm，而是提供了一种更透明、更灵活的配置方式。对于希望深入掌控应用部署细节的工程师来说，掌握 Kustomize 是必不可少的一环。

> **下一步建议**：尝试将你的现有项目改造成 Kustomize 结构，体验 base/overlays 的威力！