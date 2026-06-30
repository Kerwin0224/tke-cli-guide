# 其他存储卷使用说明

> 对照官方：[其他存储卷使用说明](https://cloud.tencent.com/document/product/457/31713) · page_id `31713`

## 概述

除 CSI 驱动的云存储外，TKE 还支持以下本地存储卷类型。这些卷无需安装 CSI 驱动组件，直接在工作负载 YAML 中定义即可使用。

| 数据卷类型 | Kubernetes 对应 | 描述 |
|-----------|----------------|------|
| 临时路径 | `emptyDir` | Pod 生命周期内的临时存储，Pod 删除后数据丢失 |
| 主机路径 | `hostPath` | 挂载宿主机文件目录到容器，指定源路径适用于持久化到特定宿主机 |
| NFS 盘 | `nfs` | 挂载已有 NFS 服务器路径，可使用 CFS 或自建 NFS |
| 已有 PVC | `persistentVolumeClaim` | 引用已创建的 PVC |
| ConfigMap | `configMap` | 以文件形式挂载配置项 |
| Secret | `secret` | 以文件形式挂载密钥 |

## 前置条件

- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达。以下 YAML 完整，但 `kubectl apply` 无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 挂载临时路径 | `kubectl apply -f <yaml>`（`volumes[].emptyDir`） | 是 |
| 挂载主机路径 | `kubectl apply -f <yaml>`（`volumes[].hostPath`） | 是 |
| 挂载 NFS | `kubectl apply -f <yaml>`（`volumes[].nfs`） | 是 |
| 挂载 ConfigMap | `kubectl apply -f <yaml>`（`volumes[].configMap`） | 是 |
| 挂载 Secret | `kubectl apply -f <yaml>`（`volumes[].secret`） | 是 |

## 操作步骤

### 临时路径（emptyDir）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-emptydir
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

Pod 删除后数据丢失。系统分配 `/var/lib/kubelet/pods/<pod>/volumes/kubernetes.io~empty-dir` 临时目录。

```bash
kubectl apply -f emptydir-pod.yaml
# expected: pod/test-emptydir created
```

### 主机路径（hostPath）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /data
      name: host-volume
  volumes:
  - name: host-volume
    hostPath:
      path: /var/lib/docker
      type: DirectoryOrCreate
```

`type` 检查类型：`DirectoryOrCreate`、`Directory`、`FileOrCreate`、`File`、`Socket`、`CharDevice`、`BlockDevice`。

```bash
kubectl apply -f hostpath-pod.yaml
# expected: pod/test-hostpath created
```

### NFS 盘

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /nfs-data
      name: nfs-volume
  volumes:
  - name: nfs-volume
    nfs:
      server: <nfs-server-ip>
      path: /
```

```bash
kubectl apply -f nfs-pod.yaml
# expected: pod/test-nfs created
```

### ConfigMap

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-configmap
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /etc/config
      name: config-volume
  volumes:
  - name: config-volume
    configMap:
      name: <configmap-name>
```

```bash
kubectl apply -f configmap-pod.yaml
# expected: pod/test-configmap created
```

### Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-secret
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /etc/secret
      name: secret-volume
  volumes:
  - name: secret-volume
    secret:
      secretName: <secret-name>
```

```bash
kubectl apply -f secret-pod.yaml
# expected: pod/test-secret created
```

## 验证

```bash
kubectl get pod test-emptydir test-hostpath test-nfs test-configmap test-secret
# expected: 所有 Pod Status 为 Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete pod test-emptydir test-hostpath test-nfs test-configmap test-secret
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| hostPath Pod 无法启动 | 宿主机路径不存在且 type 为 Directory | 创建对应目录或改为 `DirectoryOrCreate` |
| NFS Pod 挂载失败 | NFS 服务器不可达 | 确认 NFS 服务器 IP 在集群 VPC 内可达 |
| emptyDir 数据丢失 | Pod 删除 | `emptyDir` 生命周期与 Pod 绑定，改用 PVC 持久化 |

## 下一步

- [使用云硬盘 CBS](../使用云硬盘 CBS/云硬盘使用说明/tccli 操作.md)
- [使用文件存储 CFS](../使用文件存储 CFS/文件存储使用说明/tccli 操作.md)
- [PV 和 PVC 的绑定规则](../PV 和 PVC 的绑定规则/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 工作负载 → 新建](https://console.cloud.tencent.com/tke2/cluster)：在"数据卷（选填）"中添加数据卷，选择存储方式并配置挂载点。
