# gerrit代码同步到gitlab

## 一、仓库准备

>两边都创建好仓库，仓库路径尽量一致

## 二、.ssh和秘钥准备

### 1、获取known_hosts

```bash
ssh-keyscan -p 8022 gitlab-devops-k8s-local.gmbaifa.online > ~/.ssh/git_known_hosts
ssh-keyscan -p 29418 gerrit-devops-k8s-local.gmbaifa.online >> ~/.ssh/git_known_hosts
```

### 2、将秘钥对创建成secrets

```bash
kubectl create secret generic git-ssh-key-known-hosts \
  --from-file=id_ed25519=$HOME/.ssh/id_ed25519 \
  --from-file=id_ed25519.pub=$HOME/.ssh/id_ed25519.pub \
  --from-file=known_hosts=$HOME/.ssh/git_known_hosts \
  -n git-repository
```

### 3、gerrit .ssh目录创建

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: gerrit
spec:
  serviceName: gerrit
  replicas: 1
  selector:
    matchLabels:
      app: gerrit
  template:
    metadata:
      labels:
        app: gerrit
    spec:
      securityContext:
        runAsUser: 1000       # Gerrit 用户 UID
        runAsGroup: 1000      # Gerrit 用户 GID
        fsGroup: 1000
      initContainers:
        - name: init-ssh
          image: busybox
          command:
            - sh
            - -c
            - |
              mkdir -p /var/gerrit/.ssh
              cp /ssh-key/id_ed25519 /var/gerrit/.ssh/id_ed25519
              cp /ssh-key/id_ed25519.pub /var/gerrit/.ssh/id_ed25519.pub
              cp /ssh-key/known_hosts /var/gerrit/.ssh/known_hosts
              chmod 600 /var/gerrit/.ssh/id_ed25519
              chmod 644 /var/gerrit/.ssh/known_hosts
          volumeMounts:
            - name: ssh-secret
              mountPath: /ssh-key
            - name: ssh-empty
              mountPath: /var/gerrit/.ssh
      containers:
        - name: gerrit
          image: gerritcodereview/gerrit:3.12.1
          ports:
            - containerPort: 29418
            - containerPort: 8080
          env:
            - name: CANONICAL_WEB_URL
              value: "http://gerrit-devops-k8s-local.gmbaifa.online"
            - name: TZ
              value: Asia/Shanghai
          volumeMounts:
            - name: etc
              mountPath: /var/gerrit/etc
            - name: git
              mountPath: /var/gerrit/git
            - name: db
              mountPath: /var/gerrit/db
            - name: index
              mountPath: /var/gerrit/index
            - name: cache
              mountPath: /var/gerrit/cache
            - name: ssh-empty
              mountPath: /var/gerrit/.ssh
      volumes:
        - name: ssh-secret
          secret:
            secretName: git-ssh-key-known-hosts
        - name: ssh-empty
          emptyDir: {}
  volumeClaimTemplates:
    - metadata:
        name: etc
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 10Gi
        storageClassName: rook-cephfs
    - metadata:
        name: git
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 100Gi
        storageClassName: rook-cephfs
    - metadata:
        name: db
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 100Gi
        storageClassName: rook-cephfs
    - metadata:
        name: index
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 100Gi
        storageClassName: rook-cephfs
    - metadata:
        name: cache
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 100Gi
        storageClassName: rook-cephfs
```

### 4、仓库同步配置

```yaml
root@k8s-node-04:/fs/rook-ceph/volumes/csi/csi-vol-78cd2316-21f1-4aa3-bd75-ca967508835e/22b4fea7-ecb4-41be-8e1f-d7e61587dcd9# cat replication.config 
[remote "gitlab-study"]
    url = ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/${name}.git
    push = +refs/heads/*:refs/heads/*
    push = +refs/tags/*:refs/tags/*
    push = +refs/changes/*:refs/changes/*
    timeout = 30
    threads = 3
```

### 5、gerrit重启

### 6、gerrit代码提交并查看日志

```bash
bash-5.1$ tail -f replication_log 

[2025-09-19 17:53:11,464] scheduling replication app1:[refs/changes/61/61/1, refs/changes/61/61/meta] => ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git [CONTEXT PLUGIN="replication" RECEIVE_ID="app1-1758275590434-ff25541f" TRACE_ID="1758275590421-ff25541f" project="app1" request="GIT_RECEIVE" ]
[2025-09-19 17:53:11,471] scheduled app1:[refs/changes/61/61/1, refs/changes/61/61/meta] => [b4d7b3db] push ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git [refs/changes/61/61/1 refs/changes/61/61/meta] to run after 15s [CONTEXT PLUGIN="replication" RECEIVE_ID="app1-1758275590434-ff25541f" TRACE_ID="1758275590421-ff25541f" project="app1" request="GIT_RECEIVE" ]
[2025-09-19 17:53:26,477] Replication to ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git started... [CONTEXT PLUGIN="replication" RECEIVE_ID="app1-1758275590434-ff25541f" TRACE_ID="1758275590421-ff25541f" project="app1" pushOneId="b4d7b3db" request="GIT_RECEIVE" ]
[2025-09-19 17:53:26,521] Push to ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git references: RemoteRefUpdate{refSpec=refs/changes/61/61/1:refs/changes/61/61/1, status=NOT_ATTEMPTED, id=(null)..AnyObjectId[ce5ab49c0dff48c663480c72ec6090e4227e1690], force=yes, delete=no, ffwd=no}, RemoteRefUpdate{refSpec=refs/changes/61/61/meta:refs/changes/61/61/meta, status=NOT_ATTEMPTED, id=(null)..AnyObjectId[ac0935468a78478815dcc555cb5d768b465e3167], force=yes, delete=no, ffwd=no} [CONTEXT PLUGIN="replication" RECEIVE_ID="app1-1758275590434-ff25541f" TRACE_ID="1758275590421-ff25541f" project="app1" pushOneId="b4d7b3db" request="GIT_RECEIVE" ]
[2025-09-19 17:53:30,512] Replication to ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git completed in 4033ms, 15007ms delay, 0 retries [CONTEXT PLUGIN="replication" RECEIVE_ID="app1-1758275590434-ff25541f" TRACE_ID="1758275590421-ff25541f" project="app1" pushOneId="b4d7b3db" request="GIT_RECEIVE" ]
[2025-09-19 17:55:08,087] scheduling replication app1:[refs/changes/61/61/meta] => ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git [CONTEXT PLUGIN="replication" TRACE_ID="1758275707891-db7c1f03" project="app1" request="REST /changes/*/revisions/*/review" ]
[2025-09-19 17:55:08,089] scheduled app1:[refs/changes/61/61/meta] => [3428a3f0] push ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git [refs/changes/61/61/meta] to run after 15s [CONTEXT PLUGIN="replication" TRACE_ID="1758275707891-db7c1f03" project="app1" request="REST /changes/*/revisions/*/review" ]
[2025-09-19 17:55:10,924] scheduling replication app1:[refs/heads/master, refs/changes/61/61/meta] => ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git [CONTEXT PLUGIN="replication" SUBMISSION_ID="61-1758275710612-08958c1b" TRACE_ID="1758275710604-08958c1b" project="app1" request="REST /changes/*/revisions/*/submit" ]
[2025-09-19 17:55:10,924] consolidated app1:[refs/heads/master, refs/changes/61/61/meta] => [3428a3f0] push ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git [refs/heads/master refs/changes/61/61/meta] with an existing pending push [CONTEXT PLUGIN="replication" SUBMISSION_ID="61-1758275710612-08958c1b" TRACE_ID="1758275710604-08958c1b" project="app1" request="REST /changes/*/revisions/*/submit" ]
[2025-09-19 17:55:23,090] Replication to ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git started... [CONTEXT PLUGIN="replication" TRACE_ID="1758275707891-db7c1f03" project="app1" pushOneId="3428a3f0" request="REST /changes/*/revisions/*/review" ]
[2025-09-19 17:55:23,096] Push to ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git references: RemoteRefUpdate{refSpec=refs/heads/master:refs/heads/master, status=NOT_ATTEMPTED, id=(null)..AnyObjectId[ce5ab49c0dff48c663480c72ec6090e4227e1690], force=yes, delete=no, ffwd=no}, RemoteRefUpdate{refSpec=refs/changes/61/61/meta:refs/changes/61/61/meta, status=NOT_ATTEMPTED, id=(null)..AnyObjectId[f627b1c317d473da9f03025535d606dfdb265ff4], force=yes, delete=no, ffwd=no} [CONTEXT PLUGIN="replication" TRACE_ID="1758275707891-db7c1f03" project="app1" pushOneId="3428a3f0" request="REST /changes/*/revisions/*/review" ]
[2025-09-19 17:55:25,905] Replication to ssh://git@gitlab-devops-k8s-local.gmbaifa.online:8022/study/app1.git completed in 2814ms, 15002ms delay, 0 retries [CONTEXT PLUGIN="replication" TRACE_ID="1758275707891-db7c1f03" project="app1" pushOneId="3428a3f0" request="REST /changes/*/revisions/*/review" ]
```

