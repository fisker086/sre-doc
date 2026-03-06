# Harbor高可用安装

## 一、helm源配置

```bash
helm repo add harbor https://helm.goharbor.io
```

## 二、下载chart

```bash
helm fetch harbor/harbor --untar
```

## 三、修改values.yaml

```yaml
fullnameOverride: harbor
expose:
...
  type: ingress
...
externalURL: https://core-harbor-devops-k8s-local.gmbaifa.online
...
persistence:
  enabled: true
  # Setting it to "keep" to avoid removing PVCs during a helm delete
  # operation. Leaving it empty will delete PVCs after the chart deleted
  # (this does not apply for PVCs that are created for internal database
  # and redis components, i.e. they are never deleted automatically)
  resourcePolicy: "keep"
  persistentVolumeClaim:
    registry:
      # Use the existing PVC which must be created manually before bound,
      # and specify the "subPath" if the PVC is shared with other components
      existingClaim: ""
      # Specify the "storageClass" used to provision the volume. Or the default
      # StorageClass will be used (the default).
      # Set it to "-" to disable dynamic provisioning
      storageClass: "rook-cephfs"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 100Gi
      annotations: {}
    jobservice:
      jobLog:
        existingClaim: ""
        storageClass: "rook-cephfs"
        subPath: ""
        accessMode: ReadWriteOnce
        size: 1Gi
        annotations: {}
    # If external database is used, the following settings for database will
    # be ignored
    database:
      existingClaim: ""
      storageClass: "rook-cephfs"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 1Gi
      annotations: {}
    # If external Redis is used, the following settings for Redis will
    # be ignored
    redis:
      existingClaim: ""
      storageClass: "rook-cephfs"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 1Gi
      annotations: {}
    trivy:
      existingClaim: ""
      storageClass: "rook-cephfs"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 5Gi
      annotations: {}
```

