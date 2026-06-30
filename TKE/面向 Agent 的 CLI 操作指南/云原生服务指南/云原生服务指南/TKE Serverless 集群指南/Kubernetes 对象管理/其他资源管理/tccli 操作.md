# 其他资源管理（tccli）

> 对照官方：[其他资源管理](https://cloud.tencent.com/document/product/457/39818) · page_id `39818`

## 概述

在 TKE Serverless 集群中管理命名空间（Namespace）、配置资源（ConfigMap/Secret）、持久化存储（PVC）等 Kubernetes 对象。这些资源为工作负载提供运行环境、配置注入和数据持久化能力。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。

> **注意：** kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）。以下 `kubectl` 命令是正确的 CLI 操作方式，实际执行需解决 CAM 权限或使用 Cloud Shell 连接。

## 前置条件

- [环境准备](../../环境准备.md)
- 已创建 Serverless 集群且状态为 `Running`，参见[创建集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)
- 已[连接集群](../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md)，`kubectl` 可用

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-------------|------|
| 创建 Namespace | `kubectl create namespace <name>` | 否（已存在会报错） |
| 查看 Namespace | `kubectl get namespaces` | 是 |
| 创建 ConfigMap | `kubectl create configmap <name> --from-literal=<k>=<v>` | 否 |
| YAML 方式创建 ConfigMap | `kubectl apply -f <yaml>` | 是 |
| 创建 Secret | `kubectl create secret generic <name> --from-literal=<k>=<v>` | 否 |
| YAML 方式创建 Secret | `kubectl apply -f <yaml>` | 是 |
| 创建 PVC | `kubectl apply -f <yaml>` | 是 |
| 查看 PVC | `kubectl get pvc` | 是 |
| 删除资源 | `kubectl delete <type> <name>` | 是 |

## 操作步骤

### 1. Namespace（命名空间）

Namespace 用于隔离集群内的资源，不同 Namespace 下的资源互相不可见（默认）。

```bash
# 创建 Namespace
kubectl create namespace <namespace-name>

# 示例：创建 dev、staging、prod 三个命名空间
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod
```

```bash
# 查看所有 Namespace
kubectl get namespaces
```

示例输出：

```text
NAME              STATUS   AGE
default           Active   7d
dev               Active   5m
kube-node-lease   Active   7d
kube-public       Active   7d
kube-system       Active   7d
prod              Active   5m
staging           Active   5m
```

```bash
# 查看 Namespace 详情
kubectl describe namespace dev

# 删除 Namespace（会级联删除内部所有资源）
kubectl delete namespace dev
```

```text
Name:         ...
Status:       Running
...
```

### 2. ConfigMap（配置管理）

ConfigMap 用于存储非敏感的配置数据，以键值对形式注入到 Pod 中。

#### 从命令行字面值创建

```bash
kubectl create configmap <configmap-name> \
    --from-literal=<key1>=<value1> \
    --from-literal=<key2>=<value2> \
    --namespace=<namespace>
```

示例：

```bash
kubectl create configmap app-config \
    --from-literal=DB_HOST=10.0.0.50 \
    --from-literal=DB_PORT=3306 \
    --from-literal=LOG_LEVEL=info \
    --namespace=default
```

#### 从文件创建

```bash
kubectl create configmap app-config-file \
    --from-file=<path>/app.properties \
    --namespace=default
```

#### YAML 方式

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  DB_HOST: "10.0.0.50"
  DB_PORT: "3306"
  LOG_LEVEL: "info"
  app.properties: |
    server.port=8080
    server.context-path=/api
```

```bash
kubectl apply -f configmap.yaml
```

#### 在 Pod 中使用 ConfigMap

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
  namespace: default
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "1Gi"
spec:
  containers:
  - name: app
    image: nginx:1.25
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_PORT
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app/config
    resources:
      requests:
        cpu: "1"
        memory: "1Gi"
      limits:
        cpu: "1"
        memory: "1Gi"
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

#### 查看 ConfigMap

```bash
kubectl get configmaps --namespace=default
kubectl describe configmap app-config --namespace=default
```

示例输出（`kubectl get configmaps`）：

```text
NAME               DATA   AGE
app-config         4      5m
app-config-file    1      3m
```

### 3. Secret（密钥管理）

Secret 用于存储敏感数据（密码、Token、证书等），以 Base64 编码存储。

#### 从命令行创建

```bash
kubectl create secret generic <secret-name> \
    --from-literal=<key1>=<value1> \
    --from-literal=<key2>=<value2> \
    --namespace=<namespace>
```

示例：

```bash
kubectl create secret generic db-secret \
    --from-literal=username=admin \
    --from-literal=password='MyS3cur3P@ss' \
    --namespace=default
```

#### YAML 方式

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
type: Opaque
data:
  username: YWRtaW4=
  password: TXlTM2N1cjNQQHNz
```

> **注意：** YAML 中 `data` 字段的值必须是 Base64 编码。使用 `echo -n '<value>' | base64` 编码。

```bash
kubectl apply -f secret.yaml
```

#### 在 Pod 中使用 Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secret
  namespace: default
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "1Gi"
spec:
  containers:
  - name: app
    image: nginx:1.25
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    resources:
      requests:
        cpu: "1"
        memory: "1Gi"
      limits:
        cpu: "1"
        memory: "1Gi"
```

#### 查看 Secret

```bash
kubectl get secrets --namespace=default
kubectl describe secret db-secret --namespace=default
```

### 4. PVC（持久化存储卷声明）

PVC 用于请求持久化存储资源，TKE Serverless 支持 CBS（云硬盘）和 CFS（文件存储）。

#### CBS PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: cbs
  resources:
    requests:
      storage: 20Gi
```

#### CFS PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: cfs
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f pvc-cbs.yaml
kubectl apply -f pvc-cfs.yaml
```

#### 在 Pod 中使用 PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
  namespace: default
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "2Gi"
spec:
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: data
      mountPath: /data
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
      limits:
        cpu: "1"
        memory: "2Gi"
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: cbs-pvc
```

#### 查看 PVC

```bash
kubectl get pvc --namespace=default
```

示例输出：

```text
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cbs-pvc   Bound    pvc-abc123-def456                          20Gi       RWO            cbs            5m
cfs-pvc   Bound    pvc-ghi789-jkl012                          10Gi       RWX            cfs            5m
```

### 5. 删除资源

```bash
# 删除 ConfigMap
kubectl delete configmap app-config --namespace=default

# 删除 Secret
kubectl delete secret db-secret --namespace=default

# 删除 PVC（注意：PVC 删除后持久化数据将丢失）
kubectl delete pvc cbs-pvc --namespace=default

# 删除 Namespace
kubectl delete namespace dev
```

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| Namespace 已创建 | `kubectl get ns` | 目标命名空间在列表中，STATUS 为 `Active` |
| ConfigMap 已创建 | `kubectl get configmaps -n default` | 目标 ConfigMap 在列表中 |
| Secret 已创建 | `kubectl get secrets -n default` | 目标 Secret 在列表中 |
| PVC 已绑定 | `kubectl get pvc -n default` | STATUS 为 `Bound` |

## 清理

```bash
# 删除 ConfigMap
kubectl delete configmap <configmap-name> --namespace=<namespace>

# 删除 Secret
kubectl delete secret <secret-name> --namespace=<namespace>

# 删除 PVC（数据将丢失）
kubectl delete pvc <pvc-name> --namespace=<namespace>

# 删除 Namespace（级联删除内部所有资源）
kubectl delete namespace <namespace-name>
```

> **注意：** 删除 Namespace 会级联删除该 Namespace 下的所有资源（Pod、Service、ConfigMap、Secret 等），请谨慎操作。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| PVC 一直 `Pending` | 存储类未就绪或后端 CBS/CFS 异常 | `kubectl describe pvc <name>` 查看 Events |
| ConfigMap/Secret 更新后 Pod 未生效 | Pod 启动时一次性加载 | 重启 Pod（`kubectl rollout restart deployment <name>`） |
| Namespace 删除卡住 | Namespace 中资源清理延迟 | `kubectl get ns <name>` 查看状态，若长时间 `Terminating` 可 [在线咨询](https://cloud.tencent.com/online-service?from=doc_457) |
| `kubectl create namespace` 报 `AlreadyExists` | 同名 Namespace 已存在 | 使用 `kubectl get ns` 确认，复用已有或改名 |

## 下一步

- [工作负载管理](../工作负载管理/tccli%20操作.md) — 创建使用 ConfigMap、Secret 和 PVC 的工作负载
- [Annotation 说明](../Annotation%20说明/tccli%20操作.md) — 通过 Annotation 自定义资源配置

## 控制台替代

[容器服务控制台 → 集群 → 目标集群](https://console.cloud.tencent.com/tke2/cluster) — 左侧导航栏提供命名空间、配置管理、存储等资源管理入口。
