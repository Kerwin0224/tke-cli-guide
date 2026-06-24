# PV 和 PVC 的绑定规则

> 对照官方：[PV 和 PVC 的绑定规则](https://cloud.tencent.com/document/product/457/47014) · page_id `47014`

## 概述

PV（PersistentVolume）与 PVC（PersistentVolumeClaim）的绑定遵循 Kubernetes 标准规则。理解这些规则是排查存储问题的关键。

### PV 状态

| 状态 | 描述 |
|------|------|
| `Available` | PV 创建后未与任何 PVC 绑定 |
| `Bound` | PV 已与 PVC 成功绑定 |
| `Released` | 绑定的 PVC 被删除后（回收策略为 Retain），PV 转入此状态 |

Released 状态的 PV 若要重新绑定，必须手动删除 YAML 中的 `claimRef` 字段。

### PVC 状态

| 状态 | 描述 |
|------|------|
| `Pending` | 集群中不存在满足条件的 PV |
| `Bound` | PVC 已成功绑定 PV |

## 前置条件

- kubectl 可达集群以执行查询

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达。以下命令格式完整，但无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看 PV 列表及状态 | `kubectl get pv` | 是 |
| 查看 PVC 列表及状态 | `kubectl get pvc` | 是 |
| 查看 PV 详情 | `kubectl describe pv <name>` | 是 |
| 查看 PVC 详情 | `kubectl describe pvc <name>` | 是 |

## 操作步骤

### 绑定规则

PVC 绑定 PV 时，系统依据以下 4 个参数筛选：

| 参数 | 匹配规则 | 示例 |
|------|---------|------|
| `VolumeMode` | PV 与 PVC 的 `volumeMode` 必须一致 | `Filesystem` 对 `Filesystem`，`Block` 对 `Block` |
| `StorageClass` | PV 与 PVC 的 `storageClassName` 必须相同（或两者同时为空） | 均为 `cbs`，或均为 `""` |
| `AccessMode` | PV 与 PVC 的访问模式必须相同 | `ReadWriteOnce`、`ReadOnlyMany`、`ReadWriteMany` |
| `Size` | PVC 请求容量 <= PV 容量，系统选择满足条件的最小 PV | PVC 请求 10Gi，PV 有 10Gi 和 100Gi 两个，选 10Gi |

当集群现有 PV 资源不足时，系统可基于 StorageClass 动态创建 PV。

### 查看 PV 状态

```bash
kubectl get pv
# expected: 列出所有 PV 及其状态
```

预期输出：

```text
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM              STORAGECLASS
cbs-pv     10Gi       RWO            Retain           Bound       default/cbs-pvc    cbs
cfs-pv     10Gi       RWX            Retain           Available                       cfs
cos-pv     10Gi       RWX            Retain           Released                       ""
```

### 查看 PVC 状态

```bash
kubectl get pvc -A
# expected: 列出所有命名空间的 PVC 及其状态
```

预期输出：

```text
NAMESPACE   NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS
default     cbs-pvc    Bound    cbs-pv    10Gi       RWO            cbs
default     cfs-pvc    Pending                                     cfs
```

### 排查 PVC Pending

```bash
kubectl describe pvc <pvc-name>
# expected: Events 中会说明为何无法绑定（如 "no persistent volumes available"）
```

```text
NAME  STATUS  AGE
...
```

### 修复 Released 状态 PV

```bash
# 查看 PV 的 claimRef
kubectl get pv <pv-name> -o yaml | grep -A 5 claimRef

# 编辑 PV，删除 claimRef 字段
kubectl edit pv <pv-name>
# 删除 spec.claimRef 整个块
```

```text
NAME  STATUS  AGE
...
```

## 验证

```bash
# 检查 PV 绑定状态
kubectl get pv --sort-by=.spec.capacity.storage
# 检查 PVC 绑定状态
kubectl get pvc -A --sort-by=.spec.resources.requests.storage
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页为概念说明页，无资源需清理。

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| PVC Pending，Events 显示 "no persistent volumes available" | 没有满足条件的 PV | 创建匹配的 PV 或 StorageClass 支持动态创建 |
| PV Available 但 PVC 无法绑定 | StorageClass 不匹配 | 检查 PV 和 PVC 的 `storageClassName` 是否一致 |
| PV Available 但 PVC 无法绑定 | AccessMode 不匹配 | 检查 PV 和 PVC 的 `accessModes`，如 PV 为 RWO 但 PVC 要求 RWX |
| PV Available 但 PVC 无法绑定 | PVC 请求容量超过 PV | 调大 PV 容量或减小 PVC 请求 |
| Released PV 无法重新绑定 | PV 中残留 `claimRef` | `kubectl edit pv <name>` 删除 `spec.claimRef` |
| 动态创建的 PV 与预期不符 | StorageClass parameters 配置错误 | 检查 StorageClass 参数（可用区、云盘类型等） |

## 下一步

- [使用云硬盘 CBS](../使用云硬盘 CBS/云硬盘使用说明/tccli 操作.md)
- [使用文件存储 CFS](../使用文件存储 CFS/文件存储使用说明/tccli 操作.md)
- [使用对象存储 COS](../使用对象存储 COS/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 存储](https://console.cloud.tencent.com/tke2/cluster)：查看 PV/PVC 列表及状态，创建时自动遵循绑定规则。
