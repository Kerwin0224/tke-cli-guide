# 使用数据加速器 GooseFS

> 对照官方：[使用数据加速器 GooseFS](https://cloud.tencent.com/document/product/457/116405) · page_id `116405`

## 概述

TKE 支持通过 GooseFS-CSI 驱动将腾讯云数据加速器 GooseFS 挂载为 Pod 数据卷。GooseFS 相比 COSFS 提供更高的大文件读写速度，不受本地磁盘性能限制，建议容器内存大于 2G 时使用。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 GooseFS-CSI 组件（addon: `goosefs`）
- 已在数据加速器控制台创建 GooseFS 集群，并创建命名空间
- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达。以下 PV/PVC/Pod 创建步骤 YAML 完整，但 `kubectl apply` 无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 安装 GooseFS-CSI 组件 | `tccli tke InstallAddon --AddonName goosefs` | 否 |
| 查看已安装组件 | `tccli tke DescribeAddon` | 是 |
| 创建 PV（静态） | `kubectl apply -f <yaml>` | 否 |
| 创建 PVC | `kubectl apply -f <yaml>` | 否 |
| 创建工作负载挂载 PVC | `kubectl apply -f <yaml>` | 是 |

## 操作步骤

### 安装 GooseFS-CSI 组件

```bash
tccli tke InstallAddon --region <Region> \
    --cli-input-json '{"ClusterId":"<ClusterId>","AddonName":"goosefs","AddonVersion":"<AddonVersion>"}'
# expected: exit 0
```

### 创建 PV（静态配置）

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: goosefs-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.goosefs
    volumeAttributes:
      goosefsPath: /<namespace>-<appid>
      javaOptions: >-
        -Dgoosefs.master.embedded.journal.addresses=<master-ip-1>:9202,<master-ip-2>:9202,<master-ip-3>:9202
        -Xms4g -Xmx8g -XX:MaxDirectMemorySize=8g -XX:+UseG1GC
    volumeHandle: goosefs-pv
  mountOptions:
  - allow_other
  - direct_io
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
```

GooseFS FUSE JVM 参数：

| 参数 | 说明 | 默认值 | 必填 |
|------|------|--------|:--:|
| `-Xms4g` | JVM 最小内存 | 4g | 否 |
| `-Xmx8g` | JVM 最大内存 | 8g | 否 |
| `-XX:MaxDirectMemorySize` | 堆外最大内存 | 8g | 否 |
| `-XX:+UseG1GC` | JVM GC 算法 | G1 | 否 |

```bash
kubectl apply -f goosefs-pv.yaml
# expected: persistentvolume/goosefs-pv created
```

### 创建 PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: goosefs-pvc
  namespace: kube-system
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""
  volumeMode: Filesystem
  volumeName: goosefs-pv
```

```bash
kubectl apply -f goosefs-pvc.yaml
# expected: persistentvolumeclaim/goosefs-pvc created
```

### 创建 Pod 使用 GooseFS PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-goosefs
spec:
  containers:
  - name: pod-goosefs
    command: ["tail", "-f", "/etc/hosts"]
    image: centos:latest
    volumeMounts:
    - mountPath: /data
      name: goosefs
    resources:
      requests:
        memory: "128Mi"
        cpu: "0.1"
  volumes:
  - name: goosefs
    persistentVolumeClaim:
      claimName: goosefs-pvc
```

```bash
kubectl apply -f pod-goosefs.yaml
# expected: pod/pod-goosefs created
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.Addons[] | select(.AddonName == "goosefs") | {AddonName, AddonVersion, Status}'
# expected: Status 为 "Enabled"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 数据面（kubectl）

```bash
kubectl get pv goosefs-pv
kubectl get pvc goosefs-pvc -n kube-system
kubectl get pod pod-goosefs
# expected: PV/PVC Status 为 Bound
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete pod pod-goosefs
kubectl delete pvc goosefs-pvc -n kube-system
kubectl delete pv goosefs-pv
```

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName goosefs
# expected: exit 0
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| Fuse Pod 资源不足 | 默认请求 2GB 内存 | 修改 configmap `goosefs-csi-fuse-config` 降低 fuse pod 资源需求 |
| 同一 GooseFS 集群多个 PVC | 每个 volumeHandle 创建一个 fuse pod，浪费资源 | 同一 GooseFS 集群使用同一个 PVC，复用 fuse pod |
| Fuse Pod 被驱逐 | 节点内存紧张 | 为 fuse pod 配置 `priorityClassName: system-node-critical` |

## 下一步

- [使用文件存储 CFS](../使用文件存储 CFS/文件存储使用说明/tccli 操作.md)
- [使用对象存储 COS](../使用对象存储 COS/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 存储 → PersistentVolume](https://console.cloud.tencent.com/tke2/cluster)：新建 PV，选择"数据加速 GooseFS" Provisioner，选择 GooseFS 集群和命名空间。
