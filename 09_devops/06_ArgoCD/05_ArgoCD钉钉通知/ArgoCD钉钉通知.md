# ArgoCD钉钉通知

## 一、背景需求

>有的时候我们可能希望将应用同步的状态发送到指定的渠道，这样方便我们了解部署流水线的结果
>
>- ArgoCD Notifications - Argo CD 通知系统，持续监控 Argo CD 应用程序，旨在与各种通知服务集成，例如 Slack、SMTP、Telegram、Discord 等。
>- Argo Kube Notifier - 通用 Kubernetes 资源控制器，允许监控任何 Kubernetes 资源并在满足配置的规则时发送通知。
>- Kube Watch - 可以向 Slack/hipchat/mattermost/flock 频道发布通知，它监视集群中的资源变更并通过 webhook 通知它们。

>​    Argo CD 本身是提供 resource hook 功能的，在资源同步前、中、后提供脚本来执行相应的动作, 那么想在资源同步后获取应用的状态，然后根据状态进行通知就非常简单了，通知可以是很简单的 curl 命令：
>
>- PreSync: 在同步之前执行相关操作，这个一般用于比如数据库操作等
>- Sync: 同步时执行相关操作，主要用于复杂应用的编排
>- PostSync: 同步之后且app状态为health执行相关操作
>- SyncFail: 同步失败后执行相关操作，同步失败一般不常见

>​    但是对于 `PostSync` 可以发送成功的通知，但对于状态为 Processing 的无法判断，而且通知还是没有办法做到谁执行的 pipeline 谁接收通知的原则，没有办法很好地进行更细粒度的配置。`ArgoCD Notifications` 就可以来解决我们的问题，这里我们就以 `ArgoCD Notifications` 为例来说明如何使用钉钉来通知 Argo CD 的同步状态通知。

## 二、查看集群中是否包含ArgoCD Notifications模块

```bash
root@k8s-master1:~/kubeconfig# kubectl get pod,configmap -n gitops | grep argocd-notifications
pod/argocd-notifications-controller-79c985586c-pw6kc    1/1     Running   0               14d
configmap/argocd-notifications-cm            0      14d
```

**如果没有可以通过下面的资源清单安装**

**2022年开始已经和argocd一起发布了**

>https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/stable/manifests/install.yaml

## 三、查看argocd的应用状态信息

>https://argocd.k8s.local/api/v1/stream/applications

## 四、修改ConfigMap用以支持钉钉

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.webhook.dingtalk: |
    url: https://oapi.dingtalk.com/robot/send?access_token=9a632a742960d1dd0943ebf21948140340354b4a2d06eb4399cab72984475de8
    headers:
      - name: Content-Type
        value: application/json
  context: |
    argocdUrl: https://101.43.196.155:32329
  template.app-sync-change: |
    webhook:
      dingtalk:
        method: POST
        body: |
          {
                "msgtype": "markdown",
                "markdown": {
                    "title":"ArgoCD应用状态",
                    "text": "### ArgoCD应用状态\n> - 应用名称111111: {{.app.metadata.name}}\n> - 同步状态: {{ .app.status.operationState.phase}}\n> - 时间:{{.app.status.operationState.finishedAt}}\n> - 应用URL: [点击跳转]({{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true) \n"
                }
          }
  template.app-sync-status-unknown: |
    webhook:
      dingtalk:
        method: POST
        body: |
          {
                "msgtype": "markdown",
                "markdown": {
                    "title":"ArgoCD应用Unknown",
                    "text": "### ArgoCD应用Unknown\n> - <font color=\"warning\">应用名称</font>: {{.app.metadata.name}}\n> - <font color=\"warning\">应用同步状态</font>: {{.app.status.sync.status}}\n> - <font color=\"warning\">应用健康状态</font>: {{.app.status.health.status}}\n> - <font color=\"warning\">时间</font>: {{.app.status.operationState.startedAt}}\n> - <font color=\"warning\">应用URL</font>: [点击跳转ArgoCD UI]({{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true)"
                }
          }
  template.app-sync-failed: |
    webhook:
      dingtalk:
        method: POST
        body: |
          {
                "msgtype": "markdown",
                "markdown": {
                    "title":"ArgoCD应用发布失败",
                    "text": "### ArgoCD应用发布失败\n> - <font color=\"danger\">应用名称</font>: {{.app.metadata.name}}\n> - <font color=\"danger\">应用同步状态</font>: {{.app.status.operationState.phase}}\n> - <font color=\"danger\">应用健康状态</font>: {{.app.status.health.status}}\n> - <font color=\"danger\">时间</font>: {{.app.status.operationState.startedAt}}\n> - <font color=\"danger\">应用URL</font>: [点击跳转ArgoCD UI]({{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true)"
                }
          }
  trigger.on-deployed: |
    - description: Application is synced and healthy. Triggered once per commit.
      oncePer: app.status.sync.revision
      send: [app-sync-change]
      # trigger condition
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
  trigger.on-health-degraded: |
    - description: Application has degraded
      send: [app-sync-change]
      when: app.status.health.status == 'Degraded'
  trigger.on-sync-failed: |
    - description: Application syncing has failed
      send: [app-sync-failed]
      when: app.status.operationState != nil and app.status.operationState.phase in ['Error','Failed']
  trigger.on-sync-status-unknown: |
    - description: Application status is 'Unknown'
      send: [app-sync-status-unknown]
      when: app.status.sync.status == 'Unknown'
  trigger.on-sync-running: |
    - description: Application is being synced
      send: [app-sync-change]
      when: app.status.operationState != nil and app.status.operationState.phase in ['Running']
  trigger.on-sync-succeeded: |
    - description: Application syncing has succeeded
      send: [app-sync-change]
      when: app.status.operationState != nil and app.status.operationState.phase in ['Succeeded']
  subscriptions: |
    - recipients: [dingtalk]
      triggers: [on-sync-failed, on-sync-succeeded, on-sync-status-unknown,on-deployed,on-health-degraded]
---
apiVersion: v1
data:
  policy.csv: |
    g, admin@example.com, role:admin
  policy.default: role:readonly
  scopes: '[email, group]'
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-rbac-cm
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.webhook.dingtalk: |
    url: https://oapi.dingtalk.com/robot/send?access_token=31429a8a66c8cd5beb7c4295ce592ac3221c47152085da006dd4556390d4d7e0
    headers:
      - name: Content-Type
        value: application/json
  context: |
    argocdUrl: http://argocd.k8s.local
  template.app-sync-change: |
    webhook:
      dingtalk:
        method: POST
        body: |
          {
                "msgtype": "markdown",
                "markdown": {
                    "title":"ArgoCD同步状态",
                    "text": "### ArgoCD同步状态\n> - app名称: {{.app.metadata.name}}\n> - app同步状态: {{ .app.status.operationState.phase}}\n> - 时间:{{.app.status.operationState.startedAt}}\n> - URL: [点击跳转ArgoCD]({{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true) \n"
                }
            }
  trigger.on-deployed: |
    - description: Application is synced and healthy. Triggered once per commit.
      oncePer: app.status.sync.revision
      send: [app-sync-change]  # template names
      # trigger condition
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
  trigger.on-health-degraded: |
    - description: Application has degraded
      send: [app-sync-change]
      when: app.status.health.status == 'Degraded'
  trigger.on-sync-failed: |
    - description: Application syncing has failed
      send: [app-sync-change]  # template names
      when: app.status.operationState.phase in ['Error', 'Failed']
  trigger.on-sync-running: |
    - description: Application is being synced
      send: [app-sync-change]  # template names
      when: app.status.operationState.phase in ['Running']
  trigger.on-sync-status-unknown: |
    - description: Application status is 'Unknown'
      send: [app-sync-change]  # template names
      when: app.status.sync.status == 'Unknown'
  trigger.on-sync-succeeded: |
    - description: Application syncing has succeeded
      send: [app-sync-change]  # template names
      when: app.status.operationState.phase in ['Succeeded']
  subscriptions: |
    - recipients: [dingtalk]  # 可能有bug，正常应该是webhook:dingtalk
      triggers: [on-sync-running, on-deployed, on-sync-failed, on-sync-succeeded]

```

