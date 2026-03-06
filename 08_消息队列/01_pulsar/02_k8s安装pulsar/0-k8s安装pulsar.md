# k8s安装pulsar

> pulsar-helm-chart 的版本为 3.0.0，该版本中 pulsar 的版本为 2.10.3, 尝试安装pulsar-manager 版本为 v0.2.0

## 一、依赖准备

### 1、安装helm3

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### 2、配置helm3源

#### 1.常见源

```bash
# 先添加常用的chart源
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add aliyuncs https://apphub.aliyuncs.com
helm repo add stable http://mirror.azure.cn/kubernetes/charts/
```

#### 2.添加pulsar源

```bash
helm repo add apache https://pulsar.apache.org/charts
helm repo update
```

### 3、下载pulsar-helm-chart-pulsar脚本

```bash
wget https://github.com/apache/pulsar-helm-chart/archive/refs/tags/pulsar-3.0.0.tar.gz
tar xf pulsar-helm-chart-pulsar-3.0.0.tar.gz
```

### 4、提前上传镜像到harbor

```bash
docker pull apachepulsar/pulsar-all:2.10.3
docker pull apachepulsar/pulsar-manager:v0.2.0
docker pull apachepulsar/pulsar-dashboard:2.8.1
docker pull postgres:15.2

docker tag apachepulsar/pulsar-all:2.10.3 harbor.local/public/apachepulsar/pulsar-all:2.10.3
docker tag apachepulsar/pulsar-manager:v0.2.0 harbor.local/public/apachepulsar/pulsar-manager:v0.2.0 
docker tag apachepulsar/pulsar-dashboard:2.8.1 harbor.local/public/apachepulsar/pulsar-dashboard:2.8.1
docker tag postgres:15.2 harbor.local/public/postgres:15.2

docker push harbor.local/public/apachepulsar/pulsar-all:2.10.3
docker push harbor.local/public/apachepulsar/pulsar-manager:v0.2.0
docker push harbor.local/public/apachepulsar/pulsar-dashboard:2.8.1
docker push harbor.local/public/postgres:15.2
```



## 二、创建动态存储

> 略

## 三、创建JWT认证所需的secret

>部署的 Pulsar 集群需要在安全上开通 JWT 认证。根据前面学习的内容，JWT 支持通过两种不同的秘钥生成和验证 Token:
>
>- 对称秘钥：
>
>​     使用单个 Secret key 来生成和验证 Token
>
>- 非对称秘钥：包含由私钥和公钥组成的一对密钥
>
>- 使用 Private key 生成 Token
>
>- 使用 Public key 验证 Token

>​    推荐使用非对称密钥的方式，需要先生成密钥对，再用秘钥生成 token。因为 Pulsar 被部署在 K8S 集群中，在 K8S 集群中存储这些秘钥和 Token 的最好的方式是使用 K8S 的 Secret。

>pulsar-helm-chart 专门提供了一个prepare_helm_release.sh脚本，可以用来生成这些 Secret。

### 1、修改common_auth.sh文件

>./scripts/pulsar/common_auth.sh

```bash
#!/usr/bin/env bash
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

if [ -z "$CHART_HOME" ]; then
    echo "error: CHART_HOME should be initialized"
    exit 1
fi

OUTPUT=${CHART_HOME}/output
OUTPUT_BIN=${OUTPUT}/bin
PULSARCTL_VERSION=v0.4.0
PULSARCTL_BIN=${HOME}/.pulsarctl/pulsarctl
export PATH=${HOME}/.pulsarctl/plugins:${PATH}

discoverArch() {
  ARCH=$(uname -m)
  case $ARCH in
    x86) ARCH="386";;
    x86_64) ARCH="amd64";;
    i686) ARCH="386";;
    i386) ARCH="386";;
  esac
}

discoverArch
OS=$(echo `uname`|tr '[:upper:]' '[:lower:]')

test -d "$OUTPUT_BIN" || mkdir -p "$OUTPUT_BIN"

function pulsar::verify_pulsarctl() {
    if test -x "$PULSARCTL_BIN"; then
        return
    fi
    return 1
}

function pulsar::ensure_pulsarctl() {
    if pulsar::verify_pulsarctl; then
        return 0
    fi
    echo "Get pulsarctl install.sh script ..."
    # 改动1指定脚本位置，不需要访问github拉取
    install_script=$OUTPUT_BIN/install.sh
    echo "$(mktemp)"
#     trap "test -f $install_script && rm $install_script" RETURN
    echo "${PULSARCTL_VERSION}"
#    curl --retry 10 -L -o $install_script https://raw.githubusercontent.com/streamnative/pulsarctl/master/install.sh
    chmod +x $install_script
    $install_script --user --version ${PULSARCTL_VERSION}
}
```

### 2、修改install.sh

>./output/bin/install.sh

```bash
#!/usr/bin/env bash
#
#/**
# * Licensed to the Apache Software Foundation (ASF) under one
# * or more contributor license agreements.  See the NOTICE file
# * distributed with this work for additional information
# * regarding copyright ownership.  The ASF licenses this file
# * to you under the Apache License, Version 2.0 (the
# * "License"); you may not use this file except in compliance
# * with the License.  You may obtain a copy of the License at
# *
# *     http://www.apache.org/licenses/LICENSE-2.0
# *
# * Unless required by applicable law or agreed to in writing, software
# * distributed under the License is distributed on an "AS IS" BASIS,
# * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# * See the License for the specific language governing permissions and
# * limitations under the License.
# */

set -e

if [[ "x${version}" == "x" ]]; then
    # 改动1 指定最新稳定版本
    version=v2.10.3.3
fi

discoverArch() {
  ARCH=$(uname -m)
  case $ARCH in
    x86) ARCH="386";;
    x86_64) ARCH="amd64";;
    i686) ARCH="386";;
    i386) ARCH="386";;
    arm64) ARCH="arm64";;
  esac
}

usage() {
    cat <<EOF
This script is used to install pulsarctl
Options:
       -h,--help               Prints the usage message
       -u,--user               Install to the user install directory for your platform. Typically '$HOME/.pulsarctl/pulsarctl'.
       -v,--version            Install a specific version of pulsarctl
EOF
}

userOnly=false

while [[ $# -gt 0 ]]
do
key="$1"

case $key in
    -u|--user)
    userOnly=true
    shift
    ;;
    -v|--version)
    version="$2"
    shift
    shift
    ;;
    -h|--help)
    usage
    exit 0
    ;;
    *)
    echo "unknown option: $key"
    usage
    exit 1
    ;;
esac
done


discoverArch
OS=$(echo `uname`|tr '[:upper:]' '[:lower:]')

copyBinary() {
  target_dir=/usr/local/bin
  if [[ "${userOnly}" == "true" ]]; then
      target_dir=${HOME}/.pulsarctl
      mkdir -p ${target_dir}
  fi
    
  target_binary_file=pulsarctl${version}
  target_binary_file_path=${target_dir}/${target_binary_file}
  if [[ -f ${target_binary_file_path} ]]; then
    rm ${target_binary_file_path}
  fi
  mv pulsarctl ${target_binary_file_path}
  chmod +x ${target_binary_file_path}
  if [[ -f ${target_dir}/pulsarctl ]];then
    rm ${target_dir}/pulsarctl
  fi
  ln -s ${target_binary_file} ${target_dir}/pulsarctl
}

installNew() {
  echo ${pwd}
  TARFILE=pulsarctl-${ARCH}-${OS}.tar.gz
  UNTARFILE=pulsarctl-${ARCH}-${OS}

  if [[ -f ${TARFILE} ]]; then
    rm -f ${TARFILE} 
  fi

  if [[ -d ${UNTARFILE} ]]; then
    rm -rf ${UNTARFILE} 
  fi

  if [[ -f ${UNTARFILE} ]]; then
    rm -f ${UNTARFILE} 
  fi

  curl --retry 10 -L -o ${TARFILE} https://github.com/streamnative/pulsarctl/releases/download/${version}/${TARFILE}
  tar -xzf ${TARFILE}

  pushd ${UNTARFILE}

  copyBinary

  local plugins_dir=${HOME}/.pulsarctl/plugins
  
  mkdir -p ${plugins_dir}
  cp -r plugins/* ${plugins_dir}
  rm -rf ${TARFILE}
  rm -rf ${UNTARFILE}

  echo "The plugins of pulsarctl ${version} are successfully installed under directory '${plugins_dir}'."
  echo
  echo "In order to use this plugins, please add the plugin directory '${plugins_dir}' to the system PATH. You can do so by adding the following line to your bash profile."
  echo
  echo 'export PATH=${PATH}:${HOME}/.pulsarctl/plugins'
  echo
  echo "Happy Pulsaring!"

  export PATH=${HOME}/.pulsarctl:${HOME}/.pulsarctl/plugins:${PATH}
  popd
}

installOld() {
  binary=pulsarctl-${ARCH}-${OS}

  if [[ -d ${binary} ]]; then
    rm -rf ${binary} 
  fi

  if [[ -f ${binary} ]]; then
    rm -f ${binary} 
  fi

  curl --retry 10 -L -o ${binary} https://github.com/streamnative/pulsarctl/releases/download/${version}/${binary}
  mv ${binary} pulsarctl

  copyBinary

  echo "Happy Pulsaring!"
}

case $version in
  v0.1.0)
    installOld
    ;;
  v0.2.0)
    installOld
    ;;
  v0.3.0)
    installOld
    ;;
  *)
    installNew
esac
```

### 3、生成密钥对和令牌

```bash
cd pulsar-helm-chart
./scripts/pulsar/prepare_helm_release.sh \
-n -dev \
-k pulsar \
-l
```

>-n：指定namespace命名
>-k： 指定helm部署时helm release的版本名称
>-l： 将生成的内容输出

### 4、k8s应用这个资源清单

```yaml
cat 03-pulsar-token-secret.yaml


apiVersion: v1
data:
  PRIVATEKEY: 
  ...............................
  PUBLICKEY: 
  ...............................
kind: Secret
metadata:
  creationTimestamp: null
  name: pulsar-token-asymmetric-key
  namespace: -dev
---
apiVersion: v1
data:
  TOKEN: 
  ...............................
  TYPE: YXN5bW1ldHJpYw==
kind: Secret
metadata:
  creationTimestamp: null
  name: pulsar-token-proxy-admin
  namespace: -dev
---
apiVersion: v1
data:
  TOKEN: 
  ...............................
  TYPE: YXN5bW1ldHJpYw==
kind: Secret
metadata:
  creationTimestamp: null
  name: pulsar-token-broker-admin
  namespace: -dev
---
apiVersion: v1
data:
  TOKEN: 
  .............................
  TYPE: YXN5bW1ldHJpYw==
kind: Secret
metadata:
  creationTimestamp: null
  name: pulsar-token-admin
  namespace: -dev
```

**应用**

```bash
# kubectl apply -f 01-pulsar-token-secret.yaml
secret/pulsar-token-asymmetric-key created
secret/pulsar-token-proxy-admin created
secret/pulsar-token-broker-admin created
secret/pulsar-token-admin created
```

### 6、查看创建结果

```bash
# kubectl -n -dev get secrets | grep pulsar
pulsar-token-admin            Opaque                                2      2m21s
pulsar-token-asymmetric-key   Opaque                                2      2m21s
pulsar-token-broker-admin     Opaque                                2      2m21s
pulsar-token-proxy-admin      Opaque                                2      2m21s
```

**解释**

>pulsar-token-asymmetric-key 这个 Secret 中是用于生成 Token 和验证 Token 的私钥和公钥
>
>pulsar-token-proxy-admin 这个 Secret 中是用于 proxy 的超级用户角色 Token
>
>pulsar-token-broker-admin 这个 Secret 中是用于 broker 的超级用户角色 Token
>
>pulsar-token-admin 这个 Secret 中是用于管理客户端的超级用户角色 Token



## 四、修改helm文件

### 1、复制values

```bash
cp 02-pulsar-helm-install/values.yaml 02-pulsar-helm-install/values.yaml_bak
```

### 2、修改values.yaml

```yaml
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

###
### K8S Settings
###

### Namespace to deploy pulsar
# The namespace to use to deploy the pulsar components, if left empty
# will default to .Release.Namespace (aka helm --namespace).
# 修改1：指定名称空间，与之前创建secret的名称空间一致
namespace: "-dev"
namespaceCreate: false

## clusterDomain as defined for your k8s cluster
# 修改2：指定集群域名
clusterDomain: pulsar.-dev.local

###
### Global Settings
###

## Set to true on install
initialize: false

## Set cluster name
# clusterName:

## add custom labels to components of cluster
# labels:
#   environment: dev
#   customer: apache

## Pulsar Metadata Prefix
##
## By default, pulsar stores all the metadata at root path.
## You can configure to have a prefix (e.g. "/my-pulsar-cluster").
## If you do so, all the pulsar and bookkeeper metadata will
## be stored under the provided path
metadataPrefix: ""

## Port name prefix
##
## Used for Istio support which depends on a standard naming of ports
## See https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/#explicit-protocol-selection
## Prefixes are disabled by default

tcpPrefix: ""  # For Istio this will be "tcp-"
tlsPrefix: ""  # For Istio this will be "tls-"

## Persistence
##
## If persistence is enabled, components that have state will
## be deployed with PersistentVolumeClaims, otherwise, for test
## purposes, they will be deployed with emptyDir
##
## This is a global setting that is applied to all components.
## If you need to disable persistence for a component,
## you can set the `volume.persistence` setting to `false` for
## that component.
##
## Deprecated in favor of using `volumes.persistence`
persistence: true
## Volume settings
volumes:
  persistence: true
  # configure the components to use local persistent volume
  # the local provisioner should be installed prior to enable local persistent volume
  local_storage: false

## RBAC
##
## Configure settings related to RBAC such as limiting broker access to single
## namespece or enabling PSP

rbac:
  enabled: false
  psp: false
  limit_to_namespace: false


## AntiAffinity
##
## Flag to enable and disable `AntiAffinity` for all components.
## This is a global setting that is applied to all components.
## If you need to disable AntiAffinity for a component, you can set
## the `affinity.anti_affinity` settings to `false` for that component.
affinity:
  anti_affinity: true
  # Set the anti affinity type. Valid values:
  # requiredDuringSchedulingIgnoredDuringExecution - rules must be met for pod to be scheduled (hard) requires at least one node per replica
  # preferredDuringSchedulingIgnoredDuringExecution - scheduler will try to enforce but not guranentee
  type: requiredDuringSchedulingIgnoredDuringExecution

## Components
##
## Control what components of Apache Pulsar to deploy for the cluster
components:
  # zookeeper
  zookeeper: true
  # bookkeeper
  bookkeeper: true
  # bookkeeper - autorecovery
  autorecovery: true
  # broker
  broker: true
  # functions
  functions: true
  # proxy
  proxy: true
  # toolset
  toolset: true
  # pulsar manager
  pulsar_manager: true

## which extra components to deploy (Deprecated)
extra:
  # Pulsar proxy
  proxy: false
  # Bookkeeper auto-recovery
  autoRecovery: false
  # Pulsar dashboard
  # Deprecated
  # Replace pulsar-dashboard with pulsar-manager
  dashboard: false
  # pulsar manager
  pulsar_manager: false
  # Configure Kubernetes runtime for Functions
  functionsAsPods: false

# default image tag for pulsar images
# uses chart's appVersion when unspecified
# 修改：指定默认镜像版本
defaultPulsarImageTag: 2.10.3

## Images
##
## Control what images to use for each component
images:
  zookeeper:
    repository: harbor.local/public/apachepulsar/pulsar-all
    # uses defaultPulsarImageTag when unspecified
    tag:
    pullPolicy: IfNotPresent
  bookie:
    repository: harbor.local/public/apachepulsar/pulsar-all
    # uses defaultPulsarImageTag when unspecified
    tag:
    pullPolicy: IfNotPresent
  autorecovery:
    repository: harbor.local/public/apachepulsar/pulsar-all
    # uses defaultPulsarImageTag when unspecified
    tag:
    pullPolicy: IfNotPresent
  broker:
    repository: harbor.local/public/apachepulsar/pulsar-all
    # uses defaultPulsarImageTag when unspecified
    tag:
    pullPolicy: IfNotPresent
  proxy:
    repository: harbor.local/public/apachepulsar/pulsar-all
    # uses defaultPulsarImageTag when unspecified
    tag:
    pullPolicy: IfNotPresent
  functions:
    repository: harbor.local/public/apachepulsar/pulsar-all
    # uses defaultPulsarImageTag when unspecified
    tag:
  pulsar_manager:
    repository: harbor.local/public/apachepulsar/pulsar-manager
    tag: v0.3.0
    pullPolicy: IfNotPresent
    hasCommand: false

## TLS
## templates/tls-certs.yaml
##
## The chart is using cert-manager for provisioning TLS certs for
## brokers and proxies.
tls:
  enabled: false
  ca_suffix: ca-tls
  # common settings for generating certs
  common:
    # 90d
    duration: 2160h
    # 15d
    renewBefore: 360h
    organization:
      - pulsar
    keySize: 4096
    keyAlgorithm: RSA
    keyEncoding: PKCS8
  # settings for generating certs for proxy
  proxy:
    enabled: false
    cert_name: tls-proxy
  # settings for generating certs for broker
  broker:
    enabled: false
    cert_name: tls-broker
  # settings for generating certs for bookies
  bookie:
    enabled: false
    cert_name: tls-bookie
  # settings for generating certs for zookeeper
  zookeeper:
    enabled: false
    cert_name: tls-zookeeper
  # settings for generating certs for recovery
  autorecovery:
    cert_name: tls-recovery
  # settings for generating certs for toolset
  toolset:
    cert_name: tls-toolset

# Enable or disable broker authentication and authorization.
# 修改：开启jwt鉴权
auth:
  authentication:
    enabled: true
    provider: "jwt"
    jwt:
      # Enable JWT authentication
      # If the token is generated by a secret key, set the usingSecretKey as true.
      # If the token is generated by a private key, set the usingSecretKey as false.
      usingSecretKey: false
  authorization:
    enabled: true
  superUsers:
    # broker to broker communication
    broker: "broker-admin"
    # proxy to broker communication
    proxy: "proxy-admin"
    # pulsar-admin client to broker/proxy communication
    client: "admin"

######################################################################
# External dependencies
######################################################################

## cert-manager
## templates/tls-cert-issuer.yaml
##
## Cert manager is used for automatically provisioning TLS certificates
## for components within a Pulsar cluster
certs:
  internal_issuer:
    apiVersion: cert-manager.io/v1
    enabled: false
    component: internal-cert-issuer
    type: selfsigning
    # 90d
    duration: 2160h
    # 15d
    renewBefore: 360h
  issuers:
    selfsigning:

######################################################################
# Below are settings for each component
######################################################################

## Pulsar: Zookeeper cluster
## templates/zookeeper-statefulset.yaml
##
zookeeper:
  # use a component name that matches your grafana configuration
  # so the metrics are correctly rendered in grafana dashboard
  component: zookeeper
  # the number of zookeeper servers to run. it should be an odd number larger than or equal to 3.
  replicaCount: 1
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: Parallel
  # This is how prometheus discovers this component
  podMonitor:
    enabled: true
    interval: 10s
    scrapeTimeout: 10s
  # True includes annotation for statefulset that contains hash of corresponding configmap, which will cause pods to restart on configmap change
  restartPodsOnConfigMapChange: false
  ports:
    http: 8000
    client: 2181
    clientTls: 2281
    follower: 2888
    leaderElection: 3888
  # nodeSelector:
    # cloud.google.com/gke-nodepool: default-pool
  probe:
    liveness:
      enabled: true
      failureThreshold: 10
      initialDelaySeconds: 20
      periodSeconds: 30
      timeoutSeconds: 30
    readiness:
      enabled: true
      failureThreshold: 10
      initialDelaySeconds: 20
      periodSeconds: 30
      timeoutSeconds: 30
    startup:
      enabled: false
      failureThreshold: 30
      initialDelaySeconds: 20
      periodSeconds: 30
      timeoutSeconds: 30
  affinity:
    anti_affinity: true
    # Set the anti affinity type. Valid values:
    # requiredDuringSchedulingIgnoredDuringExecution - rules must be met for pod to be scheduled (hard) requires at least one node per replica
    # preferredDuringSchedulingIgnoredDuringExecution - scheduler will try to enforce but not guranentee
    type: requiredDuringSchedulingIgnoredDuringExecution
  annotations: {}
  tolerations: []
  gracePeriod: 30
  resources:
    requests:
      memory: 256Mi
      cpu: 0.1
  # extraVolumes and extraVolumeMounts allows you to mount other volumes
  # Example Use Case: mount ssl certificates
  # extraVolumes:
  #   - name: ca-certs
  #     secret:
  #       defaultMode: 420
  #       secretName: ca-certs
  # extraVolumeMounts:
  #   - name: ca-certs
  #     mountPath: /certs
  #     readOnly: true
  extraVolumes: []
  extraVolumeMounts: []
  # Ensures 2.10.0 non-root docker image works correctly.
  securityContext:
    fsGroup: 0
    fsGroupChangePolicy: "OnRootMismatch"
  volumes:
    # use a persistent volume or emptyDir
    # 修改：支持动态存储
    persistence: true
    data:
      name: data
      size: 60Gi
      local_storage: false
      storageClassName: "nfs-storage"
      ## If you already have an existent storage class and want to reuse it, you can specify its name with the option below
      ##
      # storageClassName: existent-storage-class
      #
      ## Instead if you want to create a new storage class define it below
      ## If left undefined no storage class will be defined along with PVC
      ##
      # storageClass:
        # type: pd-ssd
        # fsType: xfs
        # provisioner: kubernetes.io/gce-pd
      ## If you want to bind static persistent volumes via selectors, e.g.:
      # selector:
        # matchLabels:
        # app: pulsar-zookeeper
      selector: {}
  # External zookeeper server list in case of global-zk list to create zk cluster across zk deployed on different clusters/namespaces
  # Example value: "us-east1-pulsar-zookeeper-0.us-east1-pulsar-zookeeper.us-east1.svc.cluster.local:2888:3888,us-east1-pulsar-zookeeper-1.us-east1-pulsar-zookeeper.us-east1.svc.cluster.local:2888:3888,us-east1-pulsar-zookeeper-2.us-east1-pulsar-zookeeper.us-east1.svc.cluster.local:2888:3888,us-west1-pulsar-zookeeper-0.us-west1-pulsar-zookeeper.us-west1.svc.cluster.local:2888:3888,us-west1-pulsar-zookeeper-1.us-west1-pulsar-zookeeper.us-west1.svc.cluster.local:2888:3888,us-west1-pulsar-zookeeper-2.us-west1-pulsar-zookeeper.us-west1.svc.cluster.local:2888:3888"
  externalZookeeperServerList: ""
  ## Zookeeper configmap
  ## templates/zookeeper-configmap.yaml
  ##
  configData:
    PULSAR_MEM: >
      -Xms64m -Xmx128m
    PULSAR_GC: >
      -XX:+UseG1GC
      -XX:MaxGCPauseMillis=10
      -Dcom.sun.management.jmxremote
      -Djute.maxbuffer=10485760
      -XX:+ParallelRefProcEnabled
      -XX:+UnlockExperimentalVMOptions
      -XX:+DoEscapeAnalysis
      -XX:+DisableExplicitGC
      -XX:+ExitOnOutOfMemoryError
      -XX:+PerfDisableSharedMem
  ## Add a custom command to the start up process of the zookeeper pods (e.g. update-ca-certificates, jvm commands, etc)
  additionalCommand:
  ## Zookeeper service
  ## templates/zookeeper-service.yaml
  ##
  service:
    annotations:
      service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
  ## Zookeeper PodDisruptionBudget
  ## templates/zookeeper-pdb.yaml
  ##
  pdb:
    usePolicy: true
    maxUnavailable: 1

## Pulsar: Bookkeeper cluster
## templates/bookkeeper-statefulset.yaml
##
bookkeeper:
  # use a component name that matches your grafana configuration
  # so the metrics are correctly rendered in grafana dashboard
  component: bookie
  ## BookKeeper Cluster Initialize
  ## templates/bookkeeper-cluster-initialize.yaml
  metadata:
    ## Set the resources used for running `bin/bookkeeper shell initnewcluster`
    ##
    resources:
      # requests:
        # memory: 4Gi
        # cpu: 2
  replicaCount: 4
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: Parallel
  # This is how prometheus discovers this component
  podMonitor:
    enabled: true
    interval: 10s
    scrapeTimeout: 10s
  # True includes annotation for statefulset that contains hash of corresponding configmap, which will cause pods to restart on configmap change
  restartPodsOnConfigMapChange: false
  ports:
    http: 8000
    bookie: 3181
  # nodeSelector:
    # cloud.google.com/gke-nodepool: default-pool
  probe:
    liveness:
      enabled: true
      failureThreshold: 60
      initialDelaySeconds: 10
      periodSeconds: 30
      timeoutSeconds: 5
    readiness:
      enabled: true
      failureThreshold: 60
      initialDelaySeconds: 10
      periodSeconds: 30
      timeoutSeconds: 5
    startup:
      enabled: false
      failureThreshold: 30
      initialDelaySeconds: 60
      periodSeconds: 30
      timeoutSeconds: 5
  affinity:
    anti_affinity: true
    # Set the anti affinity type. Valid values:
    # requiredDuringSchedulingIgnoredDuringExecution - rules must be met for pod to be scheduled (hard) requires at least one node per replica
    # preferredDuringSchedulingIgnoredDuringExecution - scheduler will try to enforce but not guranentee
    type: requiredDuringSchedulingIgnoredDuringExecution
  annotations: {}
  tolerations: []
  gracePeriod: 30
  resources:
    requests:
      memory: 512Mi
      cpu: 0.2
  # extraVolumes and extraVolumeMounts allows you to mount other volumes
  # Example Use Case: mount ssl certificates
  # extraVolumes:
  #   - name: ca-certs
  #     secret:
  #       defaultMode: 420
  #       secretName: ca-certs
  # extraVolumeMounts:
  #   - name: ca-certs
  #     mountPath: /certs
  #     readOnly: true
  extraVolumes: []
  extraVolumeMounts: []
  # Ensures 2.10.0 non-root docker image works correctly.
  securityContext:
    fsGroup: 0
    fsGroupChangePolicy: "OnRootMismatch"
  volumes:
    # use a persistent volume or emptyDir
    # 修改：bookkeeper持久化
    persistence: true
    journal:
      name: journal
      size: 30Gi
      local_storage: false
      storageClassName: "nfs-storage"
      ## If you already have an existent storage class and want to reuse it, you can specify its name with the option below
      ##
      # storageClassName: existent-storage-class
      #
      ## Instead if you want to create a new storage class define it below
      ## If left undefined no storage class will be defined along with PVC
      ##
      # storageClass:
        # type: pd-ssd
        # fsType: xfs
        # provisioner: kubernetes.io/gce-pd
      ## If you want to bind static persistent volumes via selectors, e.g.:
      # selector:
        # matchLabels:
        # app: pulsar-bookkeeper-journal
      selector: {}
      useMultiVolumes: false
      multiVolumes:
        - name: journal0
          size: 10Gi
          # storageClassName: existent-storage-class
          mountPath: /pulsar/data/bookkeeper/journal0
        - name: journal1
          size: 10Gi
          # storageClassName: existent-storage-class
          mountPath: /pulsar/data/bookkeeper/journal1
    ledgers:
      name: ledgers
      size: 150Gi
      local_storage: false
      storageClassName: "nfs-storage"
      # storageClassName:
      # storageClass:
        # ...
      # selector:
        # ...
      useMultiVolumes: false
      multiVolumes:
        - name: ledgers0
          size: 10Gi
          # storageClassName: existent-storage-class
          mountPath: /pulsar/data/bookkeeper/ledgers0
        - name: ledgers1
          size: 10Gi
          # storageClassName: existent-storage-class
          mountPath: /pulsar/data/bookkeeper/ledgers1

    ## use a single common volume for both journal and ledgers
    useSingleCommonVolume: false
    common:
      name: common
      size: 60Gi
      local_storage: true
      # storageClassName:
      # storageClass: ## this is common too
        # ...
      # selector:
        # ...

  ## Bookkeeper configmap
  ## templates/bookkeeper-configmap.yaml
  ##
  configData:
    # we use `bin/pulsar` for starting bookie daemons
    PULSAR_MEM: >
      -Xms128m
      -Xmx256m
      -XX:MaxDirectMemorySize=256m
    PULSAR_GC: >
      -XX:+UseG1GC
      -XX:MaxGCPauseMillis=10
      -XX:+ParallelRefProcEnabled
      -XX:+UnlockExperimentalVMOptions
      -XX:+DoEscapeAnalysis
      -XX:ParallelGCThreads=4
      -XX:ConcGCThreads=4
      -XX:G1NewSizePercent=50
      -XX:+DisableExplicitGC
      -XX:-ResizePLAB
      -XX:+ExitOnOutOfMemoryError
      -XX:+PerfDisableSharedMem
      -Xlog:gc*
      -Xlog:gc::utctime
      -Xlog:safepoint
      -Xlog:gc+heap=trace
      -verbosegc
    # configure the memory settings based on jvm memory settings
    dbStorage_writeCacheMaxSizeMb: "32"
    dbStorage_readAheadCacheMaxSizeMb: "32"
    dbStorage_rocksDB_writeBufferSizeMB: "8"
    dbStorage_rocksDB_blockCacheSize: "8388608"
  ## Add a custom command to the start up process of the bookie pods (e.g. update-ca-certificates, jvm commands, etc)
  additionalCommand:
  ## Bookkeeper Service
  ## templates/bookkeeper-service.yaml
  ##
  service:
    spec:
      publishNotReadyAddresses: true
  ## Bookkeeper PodDisruptionBudget
  ## templates/bookkeeper-pdb.yaml
  ##
  pdb:
    usePolicy: true
    maxUnavailable: 1

## Pulsar: Bookkeeper AutoRecovery
## templates/autorecovery-statefulset.yaml
##
autorecovery:
  # use a component name that matches your grafana configuration
  # so the metrics are correctly rendered in grafana dashboard
  component: recovery
  replicaCount: 1
  # This is how prometheus discovers this component
  podMonitor:
    enabled: true
    interval: 10s
    scrapeTimeout: 10s
  # True includes annotation for statefulset that contains hash of corresponding configmap, which will cause pods to restart on configmap change
  restartPodsOnConfigMapChange: false
  ports:
    http: 8000
  # nodeSelector:
    # cloud.google.com/gke-nodepool: default-pool
  affinity:
    anti_affinity: true
    # Set the anti affinity type. Valid values:
    # requiredDuringSchedulingIgnoredDuringExecution - rules must be met for pod to be scheduled (hard) requires at least one node per replica
    # preferredDuringSchedulingIgnoredDuringExecution - scheduler will try to enforce but not guranentee
    type: requiredDuringSchedulingIgnoredDuringExecution
  annotations: {}
  # tolerations: []
  gracePeriod: 30
  resources:
    requests:
      memory: 64Mi
      cpu: 0.05
  ## Bookkeeper auto-recovery configmap
  ## templates/autorecovery-configmap.yaml
  ##
  configData:
    BOOKIE_MEM: >
      -Xms64m -Xmx64m
    PULSAR_PREFIX_useV2WireProtocol: "true"

## Pulsar Zookeeper metadata. The metadata will be deployed as
## soon as the last zookeeper node is reachable. The deployment
## of other components that depends on zookeeper, such as the
## bookkeeper nodes, broker nodes, etc will only start to be
## deployed when the zookeeper cluster is ready and with the
## metadata deployed
# 修改：初始化镜像修改
pulsar_metadata:
  component: pulsar-init
  image:
    # the image used for running `pulsar-cluster-initialize` job
    repository: harbor.local/public/apachepulsar/pulsar-all
    # uses defaultPulsarImageTag when unspecified
    tag:
    pullPolicy: IfNotPresent
  ## set an existing configuration store
  # configurationStore:
  configurationStoreMetadataPrefix: ""
  configurationStorePort: 2181
  ## optional you can specify a nodeSelector for all init jobs (pulsar-init & bookkeeper-init)
  # nodeSelector:
    # cloud.google.com/gke-nodepool: default-pool

  ## optional, you can provide your own zookeeper metadata store for other components
  # to use this, you should explicit set components.zookeeper to false
  #
  # userProvidedZookeepers: "zk01.example.com:2181,zk02.example.com:2181"

  ## optional, you can specify where to run pulsar-cluster-initialize job
  # nodeSelector:

# Can be used to run extra commands in the initialization jobs e.g. to quit istio sidecars etc.
extraInitCommand: ""

## Pulsar: Broker cluster
## templates/broker-statefulset.yaml
##
broker:
  # use a component name that matches your grafana configuration
  # so the metrics are correctly rendered in grafana dashboard
  component: broker
  replicaCount: 3
  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 3
    metrics: ~
  # This is how prometheus discovers this component
  podMonitor:
    enabled: true
    interval: 10s
    scrapeTimeout: 10s
  # True includes annotation for statefulset that contains hash of corresponding configmap, which will cause pods to restart on configmap change
  restartPodsOnConfigMapChange: false
  ports:
    http: 8080
    https: 8443
    pulsar: 6650
    pulsarssl: 6651
  # nodeSelector:
    # cloud.google.com/gke-nodepool: default-pool
  probe:
    liveness:
      enabled: true
      failureThreshold: 10
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
    readiness:
      enabled: true
      failureThreshold: 10
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
    startup:
      enabled: false
      failureThreshold: 30
      initialDelaySeconds: 60
      periodSeconds: 10
      timeoutSeconds: 5
  affinity:
    anti_affinity: true
    # Set the anti affinity type. Valid values:
    # requiredDuringSchedulingIgnoredDuringExecution - rules must be met for pod to be scheduled (hard) requires at least one node per replica
    # preferredDuringSchedulingIgnoredDuringExecution - scheduler will try to enforce but not guranentee
    type: preferredDuringSchedulingIgnoredDuringExecution
  annotations: {}
  tolerations: []
  gracePeriod: 30
  resources:
    requests:
      memory: 512Mi
      cpu: 0.2
  # extraVolumes and extraVolumeMounts allows you to mount other volumes
  # Example Use Case: mount ssl certificates
  # extraVolumes:
  #   - name: ca-certs
  #     secret:
  #       defaultMode: 420
  #       secretName: ca-certs
  # extraVolumeMounts:
  #   - name: ca-certs
  #     mountPath: /certs
  #     readOnly: true
  extraVolumes: []
  extraVolumeMounts: []
  extreEnvs: []
#    - name: POD_NAME
#      valueFrom:
#        fieldRef:
#          apiVersion: v1
#          fieldPath: metadata.name
  ## Broker configmap
  ## templates/broker-configmap.yaml
  ##
  configData:
    PULSAR_MEM: >
      -Xms128m -Xmx256m -XX:MaxDirectMemorySize=256m
    PULSAR_GC: >
      -XX:+UseG1GC
      -XX:MaxGCPauseMillis=10
      -Dio.netty.leakDetectionLevel=disabled
      -Dio.netty.recycler.linkCapacity=1024
      -XX:+ParallelRefProcEnabled
      -XX:+UnlockExperimentalVMOptions
      -XX:+DoEscapeAnalysis
      -XX:ParallelGCThreads=4
      -XX:ConcGCThreads=4
      -XX:G1NewSizePercent=50
      -XX:+DisableExplicitGC
      -XX:-ResizePLAB
      -XX:+ExitOnOutOfMemoryError
      -XX:+PerfDisableSharedMem
    managedLedgerDefaultEnsembleSize: "1"
    managedLedgerDefaultWriteQuorum: "1"
    managedLedgerDefaultAckQuorum: "1"
  ## Add a custom command to the start up process of the broker pods (e.g. update-ca-certificates, jvm commands, etc)
  additionalCommand:
  ## Broker service
  ## templates/broker-service.yaml
  ##
  service:
    annotations: {}
  ## Broker PodDisruptionBudget
  ## templates/broker-pdb.yaml
  ##
  pdb:
    usePolicy: true
    maxUnavailable: 1
  ### Broker service account
  ## templates/broker-service-account.yaml
  service_account:
    annotations: {}

## Pulsar: Functions Worker
## templates/function-worker-configmap.yaml
##
functions:
  component: functions-worker

## Pulsar: Proxy Cluster
## templates/proxy-statefulset.yaml
##
proxy:
  # use a component name that matches your grafana configuration
  # so the metrics are correctly rendered in grafana dashboard
  component: proxy
  replicaCount: 3
  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 3
    metrics: ~
  # This is how prometheus discovers this component
  podMonitor:
    enabled: true
    interval: 10s
    scrapeTimeout: 10s
  # True includes annotation for statefulset that contains hash of corresponding configmap, which will cause pods to restart on configmap change
  restartPodsOnConfigMapChange: false
  # nodeSelector:
    # cloud.google.com/gke-nodepool: default-pool
  probe:
    liveness:
      enabled: true
      failureThreshold: 10
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
    readiness:
      enabled: true
      failureThreshold: 10
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
    startup:
      enabled: false
      failureThreshold: 30
      initialDelaySeconds: 60
      periodSeconds: 10
      timeoutSeconds: 5
  affinity:
    anti_affinity: true
    # Set the anti affinity type. Valid values:
    # requiredDuringSchedulingIgnoredDuringExecution - rules must be met for pod to be scheduled (hard) requires at least one node per replica
    # preferredDuringSchedulingIgnoredDuringExecution - scheduler will try to enforce but not guranentee
    type: requiredDuringSchedulingIgnoredDuringExecution
  annotations: {}
  tolerations: []
  gracePeriod: 30
  resources:
    requests:
      memory: 128Mi
      cpu: 0.2
  # extraVolumes and extraVolumeMounts allows you to mount other volumes
  # Example Use Case: mount ssl certificates
  # extraVolumes:
  #   - name: ca-certs
  #     secret:
  #       defaultMode: 420
  #       secretName: ca-certs
  # extraVolumeMounts:
  #   - name: ca-certs
  #     mountPath: /certs
  #     readOnly: true
  extraVolumes: []
  extraVolumeMounts: []
  extreEnvs: []
#    - name: POD_IP
#      valueFrom:
#        fieldRef:
#          apiVersion: v1
#          fieldPath: status.podIP
  ## Proxy configmap
  ## templates/proxy-configmap.yaml
  ##
  configData:
    PULSAR_MEM: >
      -Xms64m -Xmx64m -XX:MaxDirectMemorySize=64m
    PULSAR_GC: >
      -XX:+UseG1GC
      -XX:MaxGCPauseMillis=10
      -Dio.netty.leakDetectionLevel=disabled
      -Dio.netty.recycler.linkCapacity=1024
      -XX:+ParallelRefProcEnabled
      -XX:+UnlockExperimentalVMOptions
      -XX:+DoEscapeAnalysis
      -XX:ParallelGCThreads=4
      -XX:ConcGCThreads=4
      -XX:G1NewSizePercent=50
      -XX:+DisableExplicitGC
      -XX:-ResizePLAB
      -XX:+ExitOnOutOfMemoryError
      -XX:+PerfDisableSharedMem
    httpNumThreads: "8"
  ## Add a custom command to the start up process of the proxy pods (e.g. update-ca-certificates, jvm commands, etc)
  additionalCommand:
  ## Proxy service
  ## templates/proxy-service.yaml
  ##
  ports:
    http: 80
    https: 443
    pulsar: 6650
    pulsarssl: 6651
  # 修改proxy集群映射方式
  service:
    annotations: {}
    type: NodePort
  ## Proxy ingress
  ## templates/proxy-ingress.yaml
  ##
  ingress:
    enabled: true
    annotations: {}
    tls:
      enabled: false

      ## Optional. Leave it blank if your Ingress Controller can provide a default certificate.
      secretName: ""

    hostname: "proxy.dev.pulsar"
    path: "/"
  ## Proxy PodDisruptionBudget
  ## templates/proxy-pdb.yaml
  ##
  pdb:
    usePolicy: true
    maxUnavailable: 1

## Pulsar Extra: Dashboard
## templates/dashboard-deployment.yaml
## Deprecated
## 修改：
dashboard:
  component: dashboard
  replicaCount: 1
  # nodeSelector:
    # cloud.google.com/gke-nodepool: default-pool
  annotations: {}
  tolerations: []
  gracePeriod: 0
  image:
    repository: harbor.local/public/apachepulsar/pulsar-dashboard
    tag: 2.8.1
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: 1Gi
      cpu: 250m
  ## Dashboard service
  ## templates/dashboard-service.yaml
  ##
  service:
    annotations: {}
    ports:
    - name: server
      port: 80
  ingress:
    enabled: true
    annotations: {}
    tls:
      enabled: false

      ## Optional. Leave it blank if your Ingress Controller can provide a default certificate.
      secretName: ""

    ## Required if ingress is enabled
    hostname: "dashboard.dev.pulsar"
    path: "/"
    port: 80


## Pulsar ToolSet
## templates/toolset-deployment.yaml
##
toolset:
  component: toolset
  useProxy: true
  replicaCount: 1
  # True includes annotation for statefulset that contains hash of corresponding configmap, which will cause pods to restart on configmap change
  restartPodsOnConfigMapChange: false
  # nodeSelector:
    # cloud.google.com/gke-nodepool: default-pool
  annotations: {}
  tolerations: []
  gracePeriod: 30
  resources:
    requests:
      memory: 256Mi
      cpu: 0.1
  # extraVolumes and extraVolumeMounts allows you to mount other volumes
  # Example Use Case: mount ssl certificates
  # extraVolumes:
  #   - name: ca-certs
  #     secret:
  #       defaultMode: 420
  #       secretName: ca-certs
  # extraVolumeMounts:
  #   - name: ca-certs
  #     mountPath: /certs
  #     readOnly: true
  extraVolumes: []
  extraVolumeMounts: []
  ## Bastion configmap
  ## templates/bastion-configmap.yaml
  ##
  configData:
    PULSAR_MEM: >
      -Xms64M
      -Xmx128M
      -XX:MaxDirectMemorySize=128M
  ## Add a custom command to the start up process of the toolset pods (e.g. update-ca-certificates, jvm commands, etc)
  additionalCommand:

#############################################################
### Monitoring Stack : kube-prometheus-stack chart
#############################################################

## Prometheus, Grafana, and the rest of the kube-prometheus-stack are managed by the dependent chart here:
## https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
## For sample values, please see their documentation.
# 修改
kube-prometheus-stack:
  enabled: false
  prometheus:
    enabled: true
  grafana:
    enabled: true
  prometheus-node-exporter:
    enabled: true
  alertmanager:
    enabled: false

## Components Stack: pulsar_manager
## templates/pulsar-manager.yaml
##
pulsar_manager:
  component: pulsar-manager
  replicaCount: 1
  # True includes annotation for statefulset that contains hash of corresponding configmap, which will cause pods to restart on configmap change
  restartPodsOnConfigMapChange: false
  # nodeSelector:
  # cloud.google.com/gke-nodepool: default-pool
  annotations: {}
  # tolerations: []
  gracePeriod: 30
  resources:
    requests:
      memory: 250Mi
      cpu: 0.1
  configData:
    REDIRECT_HOST: "http://127.0.0.1"
    REDIRECT_PORT: "9527"
    DRIVER_CLASS_NAME: org.postgresql.Driver
    URL: jdbc:postgresql://127.0.0.1:5432/pulsar_manager
    LOG_LEVEL: DEBUG
    ## If you enabled authentication support
    JWT_TOKEN: "ZXlKaGJHY2lPaUpTVXpJMU5pSXNJblI1Y0NJNklrcFhWQ0o5LmV5SnpkV0lpT2lKaFpHMXBiaUo5LkIyRXpfdl80LWhnOHZBa1JjNmg4a05lZ3VONVZnMndIU2piSGJMOC03eExRS25YUVFVZ3V4ZVhtM2IzbDhCNlJTSnF1UlRyWlpobWMyNktkUk83N2VBdGZXTHBGOGNpSlRjeHZGSEljSy1nZl8xWmJ4bTZXSDd4bWpEM0xKTHFsOEVpNThvbFcxbmdQZFVGQW9nMHpLTXltZjV4TkNoTmx3LUNBcEJLN2x5WGNNNG5TSGtDM1p1RXBETmt2MnA1QldyM2RZZDdrZTluMjFrOGhkQkxpN2tBUjFWLTlRcy1udFJCXzhIa05CMnRkdlo0a20yWHNQMW1FWWkxX0Z2cVE2SmM1Q2FTVG1kQW0yTW42TElMZkE1Zks1UjNadVdxaFlBUmRkUzhYM3JaQmVrM19fRm9zUlJ2VkRpVGtaUkloWWpXTUlqaDRNU3pHc253bU9ZVVRndw=="
    ## SECRET_KEY: data:base64,<secret key>
  ## Pulsar manager service
  ## templates/pulsar-manager-service.yaml
  ##
  service:
    type: NodePort
    port: 9527
    targetPort: 9527
    annotations: {}
  ## Pulsar manager ingress
  ## templates/pulsar-manager-ingress.yaml
  ##
  ingress:
    enabled: false
    annotations: {}
    tls:
      enabled: false

      ## Optional. Leave it blank if your Ingress Controller can provide a default certificate.
      secretName: ""

    hostname: "manager.dev.pulsar"
    path: "/"

  ## If set use existing secret with specified name to set pulsar admin credentials.
  existingSecretName:
  admin:
    user: pulsar
    password: pulsar

# These are jobs where job ttl configuration is used
# pulsar-helm-chart/charts/pulsar/templates/pulsar-cluster-initialize.yaml
# pulsar-helm-chart/charts/pulsar/templates/bookkeeper-cluster-initialize.yaml
job:
  ttl:
    enabled: false
    secondsAfterFinished: 3600
```

### 3、修改charm.yml文件

```yaml
root@k8s-isvc-master-01:~/k8s-yaml.d/ELKBF/03-pulsar# cat pulsar-helm-chart-pulsar-3.0.0/charts/pulsar/Chart.yaml
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

apiVersion: v2
appVersion: "2.10.2"
description: Apache Pulsar Helm chart for Kubernetes
name: pulsar
version: 3.0.0
home: https://pulsar.apache.org
sources:
- https://github.com/apache/pulsar
- https://github.com/apache/pulsar-helm-chart
icon: https://pulsar.apache.org/img/pulsar.svg
maintainers:
- name: The Apache Pulsar Team
  email: dev@pulsar.apache.org
# 修改：取消prometheus依赖
#dependencies:
#  - name: kube-prometheus-stack
#    version: 41.x.x
#    repository: https://prometheus-community.github.io/helm-charts
#    condition: kube-prometheus-stack.enabled
```

### 4、修改每一个k8s控制器模板的亲和(需要配亲和再改)

> 以zookeeper为例
>
> cat pulsar-helm-chart-pulsar-3.0.0/charts/pulsar/templates/zookeeper-statefulset.yaml

```yaml
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

# deploy zookeeper only when `components.zookeeper` is true
{{- if .Values.components.zookeeper }}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: "{{ template "pulsar.fullname" . }}-{{ .Values.zookeeper.component }}"
  namespace: {{ template "pulsar.namespace" . }}
  labels:
    {{- include "pulsar.standardLabels" . | nindent 4 }}
    component: {{ .Values.zookeeper.component }}
spec:
  serviceName: "{{ template "pulsar.fullname" . }}-{{ .Values.zookeeper.component }}"
  replicas: {{ .Values.zookeeper.replicaCount }}
  selector:
    matchLabels:
      {{- include "pulsar.matchLabels" . | nindent 6 }}
      component: {{ .Values.zookeeper.component }}
  updateStrategy:
{{ toYaml .Values.zookeeper.updateStrategy | indent 4 }}
  podManagementPolicy: {{ .Values.zookeeper.podManagementPolicy }}
  template:
    metadata:
      labels:
        {{- include "pulsar.template.labels" . | nindent 8 }}
        component: {{ .Values.zookeeper.component }}
      annotations:
        {{- if .Values.zookeeper.restartPodsOnConfigMapChange }}
        checksum/config: {{ include (print $.Template.BasePath "/zookeeper-configmap.yaml") . | sha256sum }}
        {{- end }}
{{ toYaml .Values.zookeeper.annotations | indent 8 }}
    spec:
    {{- if .Values.zookeeper.nodeSelector }}
      nodeSelector:
{{ toYaml .Values.zookeeper.nodeSelector | indent 8 }}
    {{- end }}
    {{- if .Values.zookeeper.tolerations }}
      tolerations:
{{ toYaml .Values.zookeeper.tolerations | indent 8 }}
    {{- end }}
      # 修改：容忍污点，配置亲和
      # 容忍污点
      tolerations:
        - key: "key1"
          operator: "Equal"
          value: "value1"
          effect: "NoSchedule"
      affinity:
        # node亲和
        nodeAffinity:
          # 硬亲和
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: project
                    operator: In
                    values:
                      - elk
          # 软亲和
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              preference:
                matchExpressions:
                  - key: kubernetes.io/hostname
                    operator: In
                    values:
                      - 172.16.2.35
            - weight: 40
              preference:
                matchExpressions:
                  - key: kubernetes.io/hostname
                    operator: In
                    values:
                      - 172.16.2.36
        {{- if and .Values.affinity.anti_affinity .Values.zookeeper.affinity.anti_affinity}}
        podAntiAffinity:
          {{ if eq .Values.zookeeper.affinity.type "requiredDuringSchedulingIgnoredDuringExecution"}}
          {{ .Values.zookeeper.affinity.type }}:
          - labelSelector:
              matchExpressions:
              - key: "app"
                operator: In
                values:
                - "{{ template "pulsar.name" . }}"
              - key: "release"
                operator: In
                values:
                - {{ .Release.Name }}
              - key: "component"
                operator: In
                values:
                - {{ .Values.zookeeper.component }}
            topologyKey: "kubernetes.io/hostname"
        {{ else }}
          {{ .Values.zookeeper.affinity.type }}:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: "app"
                      operator: In
                      values:
                      - "{{ template "pulsar.name" . }}"
                    - key: "release"
                      operator: In
                      values:
                      - {{ .Release.Name }}
                    - key: "component" 
                      operator: In
                      values:
                      - {{ .Values.zookeeper.component }}
                topologyKey: "kubernetes.io/hostname"
        {{ end }}
        {{- end }}
      terminationGracePeriodSeconds: {{ .Values.zookeeper.gracePeriod }}
    {{- if and .Values.rbac.enabled .Values.rbac.psp }}
      serviceAccountName: "{{ template "pulsar.fullname" . }}-{{ .Values.zookeeper.component }}"
    {{- end }}
      {{- if .Values.zookeeper.securityContext }}
      securityContext:
{{ toYaml .Values.zookeeper.securityContext | indent 8 }}
      {{- end }}
      containers:
      - name: "{{ template "pulsar.fullname" . }}-{{ .Values.zookeeper.component }}"
        image: "{{ template "pulsar.imageFullName" (dict "image" .Values.images.zookeeper "root" .) }}"
        imagePullPolicy: {{ .Values.images.zookeeper.pullPolicy }}
      {{- if .Values.zookeeper.resources }}
        resources:
{{ toYaml .Values.zookeeper.resources | indent 10 }}
      {{- end }}
        command: ["sh", "-c"]
        args:
        - >
        {{- if .Values.zookeeper.additionalCommand }}
          {{ .Values.zookeeper.additionalCommand }}
        {{- end }}
          bin/apply-config-from-env.py conf/zookeeper.conf;
          {{- include "pulsar.zookeeper.tls.settings" . | nindent 10 }}
          bin/generate-zookeeper-config.sh conf/zookeeper.conf;
          OPTS="${OPTS} -Dlog4j2.formatMsgNoLookups=true" exec bin/pulsar zookeeper;
        ports:
        # prometheus needs to access /metrics endpoint
        - name: http
          containerPort: {{ .Values.zookeeper.ports.http }}
        - name: client
          containerPort: {{ .Values.zookeeper.ports.client }}
        - name: follower
          containerPort: {{ .Values.zookeeper.ports.follower }}
        - name: leader-election
          containerPort: {{ .Values.zookeeper.ports.leaderElection }}
        {{- if and .Values.tls.enabled .Values.tls.zookeeper.enabled }}
        - name: client-tls
          containerPort: {{ .Values.zookeeper.ports.clientTls }}
        {{- end }}
        env:
         - name: ZOOKEEPER_SERVERS
        {{- if .Values.zookeeper.externalZookeeperServerList }}
           value: {{ .Values.zookeeper.externalZookeeperServerList }}
        {{- else }}
           {{- $global := . }}
           value: {{ range $i, $e := until (.Values.zookeeper.replicaCount | int) }}{{ if ne $i 0 }},{{ end }}{{ template "pulsar.fullname" $global }}-{{ $global.Values.zookeeper.component }}-{{ printf "%d" $i }}{{ end }}
        {{- end }}
         - name: EXTERNAL_PROVIDED_SERVERS
        {{- if .Values.zookeeper.externalZookeeperServerList }}
           value: "true"
        {{- else }}
           value: "false"
        {{- end }}
        envFrom:
        - configMapRef:
            name: "{{ template "pulsar.fullname" . }}-{{ .Values.zookeeper.component }}"
        {{- $zkConnectCommand := "" -}}
        {{- if and .Values.tls.enabled .Values.tls.zookeeper.enabled }}
        {{- $zkConnectCommand = print "openssl s_client -quiet -crlf -connect localhost:" .Values.zookeeper.ports.clientTls " -cert /pulsar/certs/zookeeper/tls.crt -key /pulsar/certs/zookeeper/tls.key" -}}
        {{- else -}}
        {{- $zkConnectCommand = print "nc -q 1 localhost " .Values.zookeeper.ports.client -}}
        {{- end }}
        {{- if .Values.zookeeper.probe.readiness.enabled }}
        {{- if and .Values.rbac.enabled .Values.rbac.psp }}
        securityContext:
          readOnlyRootFilesystem: false
        {{- end}}
        readinessProbe:
          exec:
            command:
            - timeout
            - "{{ .Values.zookeeper.probe.readiness.timeoutSeconds }}"
            - bash
            - -c
            - 'echo ruok | {{ $zkConnectCommand }} | grep imok'
          initialDelaySeconds: {{ .Values.zookeeper.probe.readiness.initialDelaySeconds }}
          periodSeconds: {{ .Values.zookeeper.probe.readiness.periodSeconds }}
          timeoutSeconds: {{ .Values.zookeeper.probe.readiness.timeoutSeconds }}
          failureThreshold: {{ .Values.zookeeper.probe.readiness.failureThreshold }}
        {{- end }}
        {{- if .Values.zookeeper.probe.liveness.enabled }}
        livenessProbe:
          exec:
            command:
            - timeout
            - "{{ .Values.zookeeper.probe.liveness.timeoutSeconds }}"
            - bash
            - -c
            - 'echo ruok | {{ $zkConnectCommand }} | grep imok'
          initialDelaySeconds: {{ .Values.zookeeper.probe.liveness.initialDelaySeconds }}
          periodSeconds: {{ .Values.zookeeper.probe.liveness.periodSeconds }}
          timeoutSeconds: {{ .Values.zookeeper.probe.liveness.timeoutSeconds }}
          failureThreshold: {{ .Values.zookeeper.probe.liveness.failureThreshold }}
        {{- end }}
        {{- if .Values.zookeeper.probe.startup.enabled }}
        startupProbe:
          exec:
            command:
            - timeout
            - "{{ .Values.zookeeper.probe.startup.timeoutSeconds }}"
            - bash
            - -c
            - 'echo ruok | {{ $zkConnectCommand }} | grep imok'
          initialDelaySeconds: {{ .Values.zookeeper.probe.startup.initialDelaySeconds }}
          periodSeconds: {{ .Values.zookeeper.probe.startup.periodSeconds }}
          timeoutSeconds: {{ .Values.zookeeper.probe.startup.timeoutSeconds }}
          failureThreshold: {{ .Values.zookeeper.probe.startup.failureThreshold }}
        {{- end }}
        volumeMounts:
        - name: "{{ template "pulsar.fullname" . }}-{{ .Values.zookeeper.component }}-{{ .Values.zookeeper.volumes.data.name }}"
          mountPath: /pulsar/data
        {{- if and .Values.tls.enabled .Values.tls.zookeeper.enabled }}
        - mountPath: "/pulsar/certs/zookeeper"
          name: zookeeper-certs
          readOnly: true
        - mountPath: "/pulsar/certs/ca"
          name: ca
          readOnly: true
        - name: keytool
          mountPath: "/pulsar/keytool/keytool.sh"
          subPath: keytool.sh
        {{- end }}
        {{- if .Values.zookeeper.extraVolumeMounts }}
{{ toYaml .Values.zookeeper.extraVolumeMounts | indent 8 }}
        {{- end }}
      volumes:
      {{- if not (and (and .Values.volumes.persistence .Values.volumes.persistence) .Values.zookeeper.volumes.persistence) }}
      - name: "{{ template "pulsar.fullname" . }}-{{ .Values.zookeeper.component }}-{{ .Values.zookeeper.volumes.data.name }}"
        emptyDir: {}
      {{- end }}
      {{- if .Values.zookeeper.extraVolumes }}
{{ toYaml .Values.zookeeper.extraVolumes | indent 6 }}
      {{- end }}
      {{- if and .Values.tls.enabled .Values.tls.zookeeper.enabled }}
      - name: zookeeper-certs
        secret:
          secretName: "{{ .Release.Name }}-{{ .Values.tls.zookeeper.cert_name }}"
          items:
            - key: tls.crt
              path: tls.crt
            - key: tls.key
              path: tls.key
      - name: ca
        secret:
          secretName: "{{ .Release.Name }}-{{ .Values.tls.ca_suffix }}"
          items:
            - key: ca.crt
              path: ca.crt
      - name: keytool
        configMap:
          name: "{{ template "pulsar.fullname" . }}-keytool-configmap"
          defaultMode: 0755
      {{- end}}
      {{- include "pulsar.imagePullSecrets" . | nindent 6}}
{{- if and (and .Values.persistence .Values.volumes.persistence) .Values.zookeeper.volumes.persistence }}
  volumeClaimTemplates:
  - metadata:
      name: "{{ template "pulsar.fullname" . }}-{{ .Values.zookeeper.component }}-{{ .Values.zookeeper.volumes.data.name }}"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: {{ .Values.zookeeper.volumes.data.size }}
    {{- if .Values.zookeeper.volumes.data.storageClassName }}
      storageClassName: "{{ .Values.zookeeper.volumes.data.storageClassName }}"
    {{- else if and (not (and .Values.volumes.local_storage .Values.zookeeper.volumes.data.local_storage)) .Values.zookeeper.volumes.data.storageClass }}
      storageClassName: "{{ template "pulsar.fullname" . }}-{{ .Values.zookeeper.component }}-{{ .Values.zookeeper.volumes.data.name }}"
    {{- else if and .Values.volumes.local_storage .Values.zookeeper.volumes.data.local_storage }}
      storageClassName: "local-storage"
    {{- end }}
    {{- with .Values.zookeeper.volumes.data.selector }}
      selector:
        {{- toYaml . | nindent 8 }}
    {{- end }}
{{- end }}
{{- end }}
```



## 五、集群部署及管理

### 1、使用helm命令部署

```bash
helm install \
--values ./02-pulsar-helm-install/values.yaml \
--set initialize=true \
--namespace -dev \
pulsar ./02-pulsar-helm-install
```

```bash
helm install \
--values ./03-pulsar-helm-chart-pulsar-3.0.0/charts/pulsar/values.yaml \
--set initialize=true \
--namespace op-elk \
pulsar ./03-pulsar-helm-chart-pulsar-3.0.0/charts/pulsar
```

### 2、启动集群

```bash
helm install \
--values ./02-pulsar-helm-install/values.yaml \
--namespace -dev \
pulsar ./02-pulsar-helm-install
```

```bash
helm install \
--values ./03-pulsar-helm-chart-pulsar-3.0.0/charts/pulsar/values.yaml \
--namespace op-elk \
pulsar ./03-pulsar-helm-chart-pulsar-3.0.0/charts/pulsar
```

### 3、修改过values.yaml进行集群更新

```bash
helm upgrade \
--values ./02-pulsar-helm-install/values.yaml  \
--namespace -dev \
pulsar ./02-pulsar-helm-install
```

```bash
helm upgrade \
--values ./03-pulsar-helm-chart-pulsar-3.0.0/charts/pulsar/values.yaml \
--namespace op-elk \
pulsar ./03-pulsar-helm-chart-pulsar-3.0.0/charts/pulsar
```

### 4、集群卸载

```bash
helm delete \
--namespace -dev \
pulsar
```

```bash
helm delete \
--namespace op-elk \
pulsar
```



## 六、pulsar集群部署验证

### 1、查看pod

>注意：完成发布后pulsar-bookie-init和pulsar-pulsar-init的状态是completed

```bash
kubectl get pod -A -o wide -n dev | grep pulsar
-dev    pulsar-bookie-0                             1/1     Running     0               30m     10.200.169.163   172.16.3.12   <none>           <none>
-dev    pulsar-bookie-1                             1/1     Running     0               30m     10.200.107.244   172.16.3.13   <none>           <none>
-dev    pulsar-bookie-2                             1/1     Running     0               30m     10.200.36.72     172.16.3.11   <none>           <none>
-dev    pulsar-bookie-3                             0/1     Pending     0               30m     <none>           <none>        <none>           <none>
-dev    pulsar-broker-0                             1/1     Running     3 (25m ago)     30m     10.200.169.157   172.16.3.12   <none>           <none>
-dev    pulsar-broker-1                             1/1     Running     3 (25m ago)     30m     10.200.107.241   172.16.3.13   <none>           <none>
-dev    pulsar-broker-2                             1/1     Running     2 (25m ago)     30m     10.200.36.92     172.16.3.11   <none>           <none>
-dev    pulsar-proxy-0                              1/1     Running     1               30m     10.200.169.153   172.16.3.12   <none>           <none>
-dev    pulsar-proxy-1                              1/1     Running     0               30m     10.200.107.242   172.16.3.13   <none>           <none>
-dev    pulsar-proxy-2                              1/1     Running     0               30m     10.200.36.105    172.16.3.11   <none>           <none>
-dev    pulsar-pulsar-manager-79f6495b4f-26bbk      1/1     Running     0               30m     10.200.107.239   172.16.3.13   <none>           <none>
-dev    pulsar-recovery-0                           1/1     Running     0               30m     10.200.169.165   172.16.3.12   <none>           <none>
-dev    pulsar-toolset-0                            1/1     Running     0               30m     10.200.169.160   172.16.3.12   <none>           <none>
-dev    pulsar-zookeeper-0                          1/1     Running     0               30m     10.200.169.164   172.16.3.12   <none>           <none>
```

### 2、验证创建租户，命名空间和主题是否正常

```bash
# kubectl exec -it -n -dev pulsar-toolset-0 -- bash
I have no name!@pulsar-toolset-0:/pulsar$ bin/pulsar-admin tenants create test-tenant
I have no name!@pulsar-toolset-0:/pulsar$
I have no name!@pulsar-toolset-0:/pulsar$
I have no name!@pulsar-toolset-0:/pulsar$
I have no name!@pulsar-toolset-0:/pulsar$ bin/pulsar-admin tenants list
public
pulsar
test-tenant
I have no name!@pulsar-toolset-0:/pulsar$ bin/pulsar-admin namespaces create test-tenant/test-ns
I have no name!@pulsar-toolset-0:/pulsar$ bin/pulsar-admin namespaces list test-tenant
test-tenant/test-ns
I have no name!@pulsar-toolset-0:/pulsar$ bin/pulsar-admin topics create-partitioned-topic test-tenant/test-ns/test-topic -p 3
I have no name!@pulsar-toolset-0:/pulsar$ bin/pulsar-admin topics list-partitioned-topics test-tenant/test-ns
persistent://test-tenant/test-ns/test-topic

bin/pulsar-admin topics create default-tenant/ticket-dispatch-namespace/ticket-dispatch-topic
```

## 七、支持aop、mop、kop镜像制作

>原先使用的其他mq，转移到pulsar需要改动自己代码适配pulsar的协议。
>
>或者，添加aop、mop、kop插件到pulsar镜像，使用自己的pulsar镜像。

>可以自己拉取源码自己构建，也可以用编译好插件。

### 1、aop插件

>源码：https://github.com/streamnative/aop

>插件地址：https://github.com/streamnative/aop/releases/download/v2.10.3.3/pulsar-protocol-handler-amqp-2.10.3.3.nar

### 2、mop插件

>源码：https://github.com/streamnative/mop

>插件地址：https://github.com/streamnative/mop/releases/download/v2.10.3.3/pulsar-protocol-handler-mqtt-2.10.3.3.nar

### 3、kop插件

>源码：https://github.com/streamnative/kop

>插件地址：https://github.com/streamnative/kop/releases/download/v2.10.3.3/pulsar-protocol-handler-kafka-2.10.3.3.nar

### 4、制作镜像

#### 1.写Dockerfile

```dockerfile
FROM apachepulsar/pulsar-all:2.10.3
USER 10000
WORKDIR /pulsar
COPY protocols/*.nar /pulsar/protocols/
ENV PATH=/pulsar/bin:/pulsar/plugins:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

#### 2.制作并上传

```bash
docker build -t harbor.local/public/apachepulsar/pulsar-all:2.10.3-v1 .
docker push harbor.local/public/apachepulsar/pulsar-all:2.10.3-v1
```



## 八、角色token创建

### 1、创建test角色

```bash
bin/pulsar tokens create --private-key file:///pulsar/keys/token/secret.key  --subject test
# 会生成token
```

### 2、授权角色

#### 1.名称空间授权

```bash
bin/pulsar-admin namespaces grant-permission test-tenant/test-ns \
            --role test \
            --actions produce,consume
```

#### 2.topic授权

```bash
bin/pulsar-admin topics grant-permission default-tenant/robot-request-namespace/lanes-resource-topic \
            --role rrrr \
            --actions produce,consume
```



## 九、部署pulsar-manager

### 1、部署pgsql

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: -dev
  labels:
    app: postgres
spec:
  selector:
    app: postgres
  ports:
  - name: pgport
    port: 5432
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: -dev
spec:
  selector:
    matchLabels:
      app: postgres
  serviceName: "postgres-service"
  template:
    metadata:
      labels:
        app: postgres
    spec:
      terminationGracePeriodSeconds: 10
      nodeSelector:
        public: ops
      containers:
      - name: postgres
        image: harbor.local/public/postgres:15.2
        imagePullPolicy: IfNotPresent
        env:
        # 配置postgres的密码
        - name: POSTGRES_PASSWORD
          value: "pulsar"
        - name: ALLOW_IP_RANGE
          value: "0.0.0.0/0"
        ports:
        - name: pgport
          protocol: TCP
          containerPort: 5432
        #resources:
        #  limits:
        #    cpu: 8000m
        #    memory: 8Gi
        #  requests:
        #    cpu: 1000m
        #    memory: 1Gi
        volumeMounts:
        - name: postgresdata
          mountPath: /var/lib/postgresql
      securityContext:
        runAsUser: 999
        fsGroup: 999
  volumeClaimTemplates:
  - metadata:
      name: postgresdata
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs-storage"
      resources:
        requests:
          storage: 50Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgressql-service
  namespace: -dev
  labels:
    app: postgressql
spec:
  selector:
    app: postgressql
  ports:
  - name: pgport
    port: 5432
  clusterIP: None
```

### 2、pgsql库表初始化

#### 1.进入容器

```bash
kubectl exec -it -n -dev postgres-0 bash
```

#### 2.登录数据库

```bash
注意：登录pulsar需要切换到postgres用户
su postgres
psql -U postgres -W
输入密码：pulsar
```

#### 3.查看当前数据库

```bash
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(3 rows)
```

#### 4.执行建库建表语句

```postgresql
 
CREATE DATABASE pulsar_manager;
\c pulsar_manager;  # 输入密码也是pulsar
 
CREATE TABLE IF NOT EXISTS environments (
  name varchar(256) NOT NULL,
  broker varchar(1024) NOT NULL,
  CONSTRAINT PK_name PRIMARY KEY (name),
  UNIQUE (broker)
);
 
CREATE TABLE IF NOT EXISTS topics_stats (
  topic_stats_id BIGSERIAL PRIMARY KEY,
  environment varchar(255) NOT NULL,
  cluster varchar(255) NOT NULL,
  broker varchar(255) NOT NULL,
  tenant varchar(255) NOT NULL,
  namespace varchar(255) NOT NULL,
  bundle varchar(255) NOT NULL,
  persistent varchar(36) NOT NULL,
  topic varchar(255) NOT NULL,
  producer_count BIGINT,
  subscription_count BIGINT,
  msg_rate_in double precision  ,
  msg_throughput_in double precision    ,
  msg_rate_out double precision ,
  msg_throughput_out double precision   ,
  average_msg_size double precision     ,
  storage_size double precision ,
  time_stamp BIGINT
);
 
CREATE TABLE IF NOT EXISTS publishers_stats (
  publisher_stats_id BIGSERIAL PRIMARY KEY,
  producer_id BIGINT,
  topic_stats_id BIGINT NOT NULL,
  producer_name varchar(255) NOT NULL,
  msg_rate_in double precision  ,
  msg_throughput_in double precision    ,
  average_msg_size double precision     ,
  address varchar(255),
  connected_since varchar(128),
  client_version varchar(36),
  metadata text,
  time_stamp BIGINT,
  CONSTRAINT fk_publishers_stats_topic_stats_id FOREIGN KEY (topic_stats_id) References topics_stats(topic_stats_id)
);
 
CREATE TABLE IF NOT EXISTS replications_stats (
  replication_stats_id BIGSERIAL PRIMARY KEY,
  topic_stats_id BIGINT NOT NULL,
  cluster varchar(255) NOT NULL,
  connected BOOLEAN,
  msg_rate_in double precision  ,
  msg_rate_out double precision ,
  msg_rate_expired double precision     ,
  msg_throughput_in double precision    ,
  msg_throughput_out double precision   ,
  msg_rate_redeliver double precision   ,
  replication_backlog BIGINT,
  replication_delay_in_seconds BIGINT,
  inbound_connection varchar(255),
  inbound_connected_since varchar(255),
  outbound_connection varchar(255),
  outbound_connected_since varchar(255),
  time_stamp BIGINT,
  CONSTRAINT FK_replications_stats_topic_stats_id FOREIGN KEY (topic_stats_id) References topics_stats(topic_stats_id)
);
 
CREATE TABLE IF NOT EXISTS subscriptions_stats (
  subscription_stats_id BIGSERIAL PRIMARY KEY,
  topic_stats_id BIGINT NOT NULL,
  subscription varchar(255) NULL,
  msg_backlog BIGINT,
  msg_rate_expired double precision     ,
  msg_rate_out double precision ,
  msg_throughput_out double precision   ,
  msg_rate_redeliver double precision   ,
  number_of_entries_since_first_not_acked_message BIGINT,
  total_non_contiguous_deleted_messages_range BIGINT,
  subscription_type varchar(16),
  blocked_subscription_on_unacked_msgs BOOLEAN,
  time_stamp BIGINT,
  UNIQUE (topic_stats_id, subscription),
  CONSTRAINT FK_subscriptions_stats_topic_stats_id FOREIGN KEY (topic_stats_id) References topics_stats(topic_stats_id)
);
 
CREATE TABLE IF NOT EXISTS consumers_stats (
  consumer_stats_id BIGSERIAL PRIMARY KEY,
  consumer varchar(255) NOT NULL,
  topic_stats_id BIGINT NOT NUll,
  replication_stats_id BIGINT,
  subscription_stats_id BIGINT,
  address varchar(255),
  available_permits BIGINT,
  connected_since varchar(255),
  msg_rate_out double precision ,
  msg_throughput_out double precision   ,
  msg_rate_redeliver double precision   ,
  client_version varchar(36),
  time_stamp BIGINT,
  metadata text
);
 
CREATE TABLE IF NOT EXISTS tokens (
  token_id BIGSERIAL PRIMARY KEY,
  role varchar(256) NOT NULL,
  description varchar(128),
  token varchar(1024) NOT NUll,
  UNIQUE (role)
);
 
CREATE TABLE IF NOT EXISTS users (
  user_id BIGSERIAL PRIMARY KEY,
  access_token varchar(256),
  name varchar(256) NOT NULL,
  description varchar(128),
  email varchar(256),
  phone_number varchar(48),
  location varchar(256),
  company varchar(256),
  expire BIGINT,
  password varchar(256),
  UNIQUE (name)
);
 
CREATE TABLE IF NOT EXISTS roles (
  role_id BIGSERIAL PRIMARY KEY,
  role_name varchar(256) NOT NULL,
  role_source varchar(256) NOT NULL,
  description varchar(128),
  resource_id BIGINT NOT NULL,
  resource_type varchar(48) NOT NULL,
  resource_name varchar(48) NOT NULL,
  resource_verbs varchar(256) NOT NULL,
  flag INT NOT NULL
);
 
CREATE TABLE IF NOT EXISTS tenants (
  tenant_id BIGSERIAL PRIMARY KEY,
  tenant varchar(255) NOT NULL,
  admin_roles varchar(255),
  allowed_clusters varchar(255),
  environment_name varchar(255),
  UNIQUE(tenant)
);
 
CREATE TABLE IF NOT EXISTS namespaces (
  namespace_id BIGSERIAL PRIMARY KEY,
  tenant varchar(255) NOT NULL,
  namespace varchar(255) NOT NULL,
  UNIQUE(tenant, namespace)
);
 
CREATE TABLE IF NOT EXISTS role_binding(
  role_binding_id BIGSERIAL PRIMARY KEY,
  name varchar(256) NOT NULL,
  description varchar(256),
  role_id BIGINT NOT NULL,
  user_id BIGINT NOT NULL
);
```

#### 5.查看结果

```bash
pulsar_manager=# \l
 postgres       | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 pulsar_manager | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
                |          |          |            |            |            |                 | postgres=CTc/postgres
 template1      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
                |          |          |            |            |            |                 | postgres=CTc/postgres

pulsar_manager=# \d
 public | consumers_stats                               | table    | postgres
 public | consumers_stats_consumer_stats_id_seq         | sequence | postgres
 public | environments                                  | table    | postgres
 public | namespaces                                    | table    | postgres
 public | namespaces_namespace_id_seq                   | sequence | postgres
 public | publishers_stats                              | table    | postgres
 public | publishers_stats_publisher_stats_id_seq       | sequence | postgres
 public | replications_stats                            | table    | postgres
 public | replications_stats_replication_stats_id_seq   | sequence | postgres
 public | role_binding                                  | table    | postgres
 public | role_binding_role_binding_id_seq              | sequence | postgres
 public | roles                                         | table    | postgres
 public | roles_role_id_seq                             | sequence | postgres
 public | subscriptions_stats                           | table    | postgres
 public | subscriptions_stats_subscription_stats_id_seq | sequence | postgres
 public | tenants                                       | table    | postgres
 public | tenants_tenant_id_seq                         | sequence | postgres
 public | tokens                                        | table    | postgres
 public | tokens_token_id_seq                           | sequence | postgres
 public | topics_stats                                  | table    | postgres
 public | topics_stats_topic_stats_id_seq               | sequence | postgres
 public | users                                         | table    | postgres
 public | users_user_id_seq                             | sequence | postgres
```

### 3、manager部署

#### 1.部署

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pulsar-manager-pvc
  namespace: -dev
spec:
  storageClassName: "nfs-storage"
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pulsar-manager
  namespace: -dev
  labels:
    app: pulsar-manager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pulsar-manager
  template:
    metadata:
      name: pulsar-manager
      labels:
        app: pulsar-manager
    spec:
      containers:
      - name: pulsar-manager
        image: harbor.local/public/apachepulsar/pulsar-manager:v0.2.0
        imagePullPolicy: Always
        ports:
        - name: sport
          containerPort: 7750
        - name: nport
          containerPort: 9527
        env:
        - name: REDIRECT_HOST
          value: "http://127.0.0.1"
        - name: REDIRECT_PORT
          value: "9527"
        - name: DRIVER_CLASS_NAME
          value: "org.postgresql.Driver"
        - name: URL
          value: "jdbc:postgresql://postgres-service.-dev.svc:5432/pulsar_manager"
        - name: USERNAME
          value: "postgres"
        - name: PASSWORD
          value: "pulsar"
        - name: LOG_LEVEL
          value: "DEBUG"
        # 开启JWT认证后, 这里需要指定pulsar-token-admin这个Secret中的JWTtoken
        - name: JWT_TOKEN
          value: "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiJ9.B2Ez_v_4-hg8vAkRc6h8kNeguN5Vg2wHSjbHbL8-7xLQKnXQQUguxeXm3b3l8B6RSJquRTrZZhmc26KdRO77eAtfWLpF8ciJTcxvFHIcK-gf_1Zbxm6WH7xmjD3LJLql8Ei58olW1ngPdUFAog0zKMymf5xNChNlw-CApBK7lyXcM4nSHkC3ZuEpDNkv2p5BWr3dYd7ke9n21k8hdBLi7kAR1V-9Qs-ntRB_8HkNB2tdvZ4km2XsP1mEYi1_FvqQ6Jc5CaSTmdAm2Mn6LILfA5fK5R3ZuWqhYARddS8X3rZBek3__FosRRvVDiTkZRIhYjWMIjh4MSzGsnwmOYUTgw"
        volumeMounts:
          - name: data
            mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: pulsar-manager-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: pulsar-manager
  namespace: -dev
  labels:
    app: pulsar-manager
  annotations:
    pulsar-manager.io/scrape: 'true'
spec:
  selector:
    app: pulsar-manager
  ports:
  - name: sport
    port: 7750
  - name: nport
    port: 9527
---
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: pulsar-manager-ingress
  namespace: -dev
spec:
  ingressClassName: nginx
# 创建证书
# openssl genrsa -out pulsarmanager.key 2048
# openssl req -new -x509 -key pulsarmanager.key -out pulsarmanager.crt -subj /C=CN/ST=ShangHai/L=ShangHai/O=Ingress/CN=pulsarmanager.-dev.local
# kubectl -n -dev create secret tls pulsarmanager-tls --cert=pulsarmanager.crt --key=pulsarmanager.key
  tls:
    - hosts:
      - pulsarmanager.-dev.local
      secretName: pulsarmanager-tls
  rules:
  - host: pulsarmanager.-dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: pulsar-manager
            port:
              number: 9527
```

#### 2.添加admin账户

```bash
CSRF_TOKEN=$(curl http://pulsar-manager.-dev.svc:7750/pulsar-manager/csrf-token)
 
curl \
    -H "X-XSRF-TOKEN: $CSRF_TOKEN" \
    -H "Cookie: XSRF-TOKEN=$CSRF_TOKEN;" \
    -H 'Content-Type: application/json' \
    -X PUT http://pulsar-manager.-dev.svc:7750/pulsar-manager/users/superuser \
    -d '{"name": "admin", "password": "pulsar", "description": "test", "email": "username@test.org"}'
```

#### 3.访问

>https://pulsarmanager.-dev.local/

配置

>dev-pulsar-cluster
>
>http://pulsar-proxy.-dev.svc:80



