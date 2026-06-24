# 其他存储卷使用说明

> 对照官方：[其他存储卷使用说明](https://cloud.tencent.com/document/product/457/31713) · page_id `31713` · tccli ≥3.1.107 · API 2019-07-19

## 概述

TKE 工作负载除了通过 PV/PVC 使用云存储（CBS/CFS/COS/GooseFS）外，还可以直接在 Pod 规格中定义多种**数据卷**类型。这些卷不属于 PV/PVC 体系，而是在工作负载 YAML 中直接声明，生命周期与 Pod 绑定。

六种数据卷类型：

| 数据卷类型 | Kubernetes 概念 | 持久化 | 典型场景 |
|-----------|----------------|:--:|------|
| 使用临时路径 | `emptyDir` | 否（Pod 删除则数据丢失） | 临时缓存、中间计算结果 |
| 使用主机路径 | `hostPath` | 是（节点级，节点故障则丢失） | 节点日志收集、DaemonSet 本地数据 |
| 使用 NFS 盘 | NFS（CFS 或自建） | 是 | 多 Pod 共享文件、已有 NFS 服务 |
| 使用已有 PVC | `persistentVolumeClaim` | 是 | 引用已创建的 PV/PVC（CBS/CFS 等） |
| 使用 ConfigMap | `configMap` | 是（配置级） | 配置文件挂载 |
| 使用 Secret | `secret` | 是（凭证级） | 敏感信息挂载 |

**关键约束**（来自官方文档）：

- 同一工作负载下，数据卷名称和容器挂载点路径不可重复
- 数据卷挂载未显式设置权限时，默认为**读写权限**
- emptyDir 未指定源路径时，系统自动分配节点临时目录（路径为 `/var/lib/kubelet/pods/<pod_name>/volumes/kubernetes.io~empty-dir`），生命周期与 Pod 一致

**建议**：生产环境持久化数据优先使用 PV/PVC 绑定云存储（CBS/CFS/COS），避免 HostPath 和 emptyDir 数据在节点故障时丢失。

## 前置条件

- [环境准备](../../../环境准备.md)
- kubectl 已连接目标集群：[连接集群](../../集群管理/连接集群/tccli%20操作.md)

### 环境检查

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'
# expected: ClusterStatus: "Running"

# 2. 确认 kubectl 可达
kubectl get ns
# expected: 返回命名空间列表

# 3. 需 CAM 权限（NFS 场景）
#    tke:DescribeClusters
tccli tke DescribeClusters --region <REGION>
# expected: exit 0，返回集群列表
```

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-------------|:--:|
| 查询集群状态 | `tccli tke DescribeClusters` | 是 |
| 创建含 emptyDir 的 Pod | `kubectl apply -f pod-emptydir.yaml` | 否 |
| 创建含 hostPath 的 Pod | `kubectl apply -f pod-hostpath.yaml` | 否 |
| 创建含 NFS 的 Pod | `kubectl apply -f pod-nfs.yaml` | 否 |
| 创建引用 PVC 的 Pod | `kubectl apply -f pod-pvc.yaml` | 否 |
| 创建含 ConfigMap 的 Pod | `kubectl apply -f pod-configmap.yaml` | 否 |
| 创建含 Secret 的 Pod | `kubectl apply -f pod-secret.yaml` | 否 |

## 操作步骤

以下为六种卷类型的 YAML 示例和关键参数说明。每个 YAML 对应 `spec.volumes` 和 `spec.containers.volumeMounts` 两部分。

### 类型 1：使用临时路径（emptyDir）

#### 选择依据

emptyDir 在 Pod 创建时从节点分配临时目录，Pod 删除后数据清除。适合缓存、中间计算结果等不要求持久化的场景。未指定 `medium` 时使用节点磁盘；`medium: Memory` 使用 tmpfs（内存）。

`pod-emptydir.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-emptydir
spec:
  containers:
  - name: app
    image: nginx:latest
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  volumes:
  - name: cache-volume
    emptyDir: {}
```

```bash
kubectl apply -f pod-emptydir.yaml
# expected: pod/test-emptydir created
```

### 类型 2：使用主机路径（hostPath）

#### 选择依据

hostPath 将节点上的目录直接挂载到容器。适用于 DaemonSet 日志采集、节点监控等场景。**注意**：节点故障或 Pod 重新调度到其他节点后，原节点数据丢失。

`pod-hostpath.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
spec:
  containers:
  - name: app
    image: nginx:latest
    volumeMounts:
    - name: host-log
      mountPath: /var/log/app
  volumes:
  - name: host-log
    hostPath:
      path: /data/logs
      type: DirectoryOrCreate
```

```bash
kubectl apply -f pod-hostpath.yaml
# expected: pod/test-hostpath created
```

### 类型 3：使用 NFS 盘

#### 选择依据

NFS 卷直接挂载 NFS 服务（可以是 CFS 或自建 NFS 服务），适合已有 NFS 基础设施的场景。**注意**：CFS 推荐通过 PV/PVC 方式使用（支持动态供给、StorageClass 管理）。

`pod-nfs.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs
spec:
  containers:
  - name: app
    image: nginx:latest
    volumeMounts:
    - name: nfs-volume
      mountPath: /mnt/nfs
  volumes:
  - name: nfs-volume
    nfs:
      server: <NFS_SERVER_IP>
      path: /<NFS_EXPORT_PATH>
```

```bash
kubectl apply -f pod-nfs.yaml
# expected: pod/test-nfs created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `NFS_SERVER_IP` | NFS 服务器 IP 地址 | CFS 场景：`tccli cfs DescribeMountTargets --region <REGION> --FileSystemId CFS_ID` |
| `NFS_EXPORT_PATH` | NFS 导出路径 | CFS 场景：固定为 `/` |

### 类型 4：引用已有 PVC

#### 选择依据

这是推荐的生产环境持久化方式。PVC 在 [使用云硬盘 CBS](../使用云硬盘CBS/PV和PVC管理云硬盘/tccli%20操作.md)、[使用文件存储 CFS](../使用文件存储CFS/PV和PVC管理文件存储/tccli%20操作.md) 等页面中创建。

`pod-pvc.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pvc
spec:
  containers:
  - name: app
    image: nginx:latest
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: <PVC_NAME>
```

```bash
kubectl apply -f pod-pvc.yaml
# expected: pod/test-pvc created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `PVC_NAME` | 已创建的 PVC 名称 | `kubectl get pvc -A` |

### 类型 5：使用 ConfigMap

#### 选择依据

ConfigMap 以文件系统形式挂载到 Pod 上，适合存储非敏感配置文件（如 Nginx 配置、应用参数文件）。支持两种挂载模式：全部 Key 挂载到目录，或指定部分 Key 挂载到特定路径（通过 `items` 字段）。ConfigMap 的生命周期独立于 Pod，删除 Pod 不会删除 ConfigMap。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-configmap
spec:
  containers:
  - name: app
    image: nginx:latest
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: <CONFIGMAP_NAME>
```

### 类型 6：使用 Secret

#### 选择依据

Secret 以文件系统形式挂载到 Pod 上，适合存储敏感信息（如密码、Token、证书私钥）。与 ConfigMap 类似，支持全部 Key 挂载或指定部分 Key 挂载。Secret 数据以 base64 编码存储，但不加密（如需加密需启用 ETCD 加密）。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-secret
spec:
  containers:
  - name: app
    image: nginx:latest
    volumeMounts:
    - name: secret
      mountPath: /etc/secret
  volumes:
  - name: secret
    secret:
      secretName: <SECRET_NAME>
```

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'` | `ClusterStatus: "Running"` |

### 数据面（kubectl）

```bash
# 确认 Pod 运行正常
kubectl get pod test-emptydir
# expected: STATUS: Running

# 检查挂载点
kubectl exec test-emptydir -- df -h /cache
# expected: 显示挂载点信息
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
# 删除测试 Pod
kubectl delete pod test-emptydir test-hostpath test-nfs test-pvc test-configmap test-secret --ignore-not-found
# expected: pod "test-*" deleted

# 验证已删除
kubectl get pod test-emptydir 2>&1
# expected: Error from server (NotFound)
```

```text
NAME  STATUS  AGE
...
```

**注意**：删除 Pod 后，emptyDir 数据丢失；hostPath 数据保留在节点上；NFS 和 PVC 数据保留在后端存储中（除非 PVC 的 reclaimPolicy 为 Delete）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 创建失败，Events 报目录不存在 | `kubectl describe pod POD_NAME` 查看 Events | hostPath `path` 在节点上不存在且 `type` 未设为 `DirectoryOrCreate` | 添加 `type: DirectoryOrCreate` 或手工在节点上创建对应目录 |
| Pod 挂载 NFS 超时 | `kubectl describe pod POD_NAME` 查看 Events | NFS 服务 IP 不可达或 NFS 端口（2049）被防火墙拦截 | 确认 NFS 服务和网络可达性：从同一 VPC 节点 `ping NFS_SERVER_IP` |
| Pod 引用 PVC 失败，Events 报 "claim not found" | `kubectl get pvc PVC_NAME` | PVC 不存在或不在 Pod 所在命名空间 | `kubectl get pvc -A` 确认 PVC 存在且命名空间正确 |

### 选型误区

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 使用 hostPath 的 Pod 重新调度后数据丢失 | hostPath 数据在节点本地磁盘，不跟随 Pod 迁移 | 错误使用 hostPath 做跨节点持久化 | 改用 PVC 绑定云存储（CBS/CFS）实现跨节点持久化 |
| emptyDir 用完后 Pod 被驱逐 | `kubectl describe pod POD_NAME` 查看 `Evicted` 原因 | emptyDir 空间耗尽导致节点磁盘压力 | 为 emptyDir 设置 `sizeLimit`：`emptyDir: {sizeLimit: "500Mi"}` |

## 下一步

- [使用云硬盘 CBS](../使用云硬盘CBS/PV和PVC管理云硬盘/tccli%20操作.md) — 生产环境持久化推荐
- [使用文件存储 CFS](../使用文件存储CFS/PV和PVC管理文件存储/tccli%20操作.md) — 多 Pod 共享存储
- [使用对象存储 COS](../使用对象存储COS/tccli%20操作.md) — 海量静态资源
- [PV 和 PVC 的绑定规则](../PV和PVC的绑定规则/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 工作负载 → 新建](https://console.cloud.tencent.com/tke2/cluster) 在「数据卷」向导中添加卷并配置挂载点。
