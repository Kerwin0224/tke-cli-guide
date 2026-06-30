# PV 和 PVC 的绑定规则

> 对照官方：[PV 和 PVC 的绑定规则](https://cloud.tencent.com/document/product/457/47014) · page_id `47014`

## 概述

Kubernetes 中 PV（PersistentVolume）与 PVC（PersistentVolumeClaim）通过一组匹配规则实现绑定。理解这些规则是排查 "PVC Pending" 问题的关键。

**绑定核心逻辑**：PVC 创建后，Kubernetes 控制平面遍历集群中所有 Available 状态的 PV，寻找满足全部匹配条件的 PV 进行一对一绑定。

## 前置条件

- [环境准备](../../../环境准备.md)
- kubectl 已连接目标集群

### 环境检查

```bash
# 1. 确认 kubectl 可达
kubectl get ns
# expected: 返回命名空间列表

# 2. 查看当前集群存储资源概况
kubectl get sc
# expected: 列出 StorageClass 列表

kubectl get pv
# expected: 列出 PV 列表（可为空）

kubectl get pvc -A
# expected: 列出所有命名空间的 PVC（可为空）
```

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-------------|:--:|
| 查看 StorageClass | `kubectl get sc` | 是 |
| 查看 PV | `kubectl get pv` | 是 |
| 查看 PV 详情 | `kubectl describe pv PV_NAME` | 是 |
| 查看 PVC | `kubectl get pvc -A` | 是 |
| 查看 PVC 详情 | `kubectl describe pvc PVC_NAME -n NAMESPACE` | 是 |

## 关键字段说明

PV 和 PVC 匹配时，以下字段决定绑定成败。

| 字段 | 位置 | 匹配规则 | 不匹配后果 |
|------|------|------|------|
| `storageClassName` | PV 和 PVC 都有 | 必须完全相等（空字符串 `""` 表示"无 StorageClass"） | PVC 永远 Pending |
| `accessModes` | PV 和 PVC 都有 | PVC 的 accessMode 必须是 PV 的子集（PV 支持 ReadWriteMany 时可满足 ReadWriteOnce 的 PVC） | PVC Pending，Events 提示 "no matching PV" |
| `capacity.storage` | PV 声明，PVC 请求 | PV 的 `capacity.storage` >= PVC 的 `resources.requests.storage` | PVC Pending |
| `spec.volumeName` | PVC 可选 | 若指定，则只绑定该名称的 PV，绕过匹配规则 | 指定的 PV 不存在 → PVC Pending |
| `matchLabels` / `matchExpressions` | PVC 可选（selector） | PV 的 `labels` 必须全部匹配 | PVC Pending |

### 生命周期

```
PV 创建 → Available
    ↓ PVC 创建，匹配全部条件
  Bound（一对一绑定）
    ↓ PVC 删除，根据 PV 的 persistentVolumeReclaimPolicy:
    ├─ Delete: PV 删除，后端存储删除
    ├─ Retain: PV 状态变为 Released（保留数据，需手动清理）
    └─ Recycle: PV 数据清理后重置为 Available（已废弃，不推荐）
```

## 操作步骤

### 查看 PV 和 PVC 的绑定状态

```bash
# 查看所有 PV 及其状态
kubectl get pv
# expected: STATUS 列显示 Available / Bound / Released / Failed
```

**预期输出**：

```text
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS
cbs-pv-static   20Gi       RWO            Retain           Bound                   
cfs-pv-static   100Gi      RWX            Retain           Available
```

```bash
# 查看所有 PVC 及其绑定的 PV
kubectl get pvc -A
# expected: STATUS: Bound（正常）或 Pending（未匹配到 PV）
```

```text
NAME  STATUS  AGE
...
```

### 诊断 PVC 未绑定的原因

当 PVC 状态为 Pending 时，最有效的诊断命令是 `describe`：

```bash
kubectl describe pvc <PVC_NAME> -n <NAMESPACE>
# expected: Events 区域显示匹配失败的具体原因
```

关键诊断点（从 Events 中解读）：

1. **"no persistent volumes available for this claim"** → 无 Available 状态的 PV 满足条件。检查 PV 的 `storageClassName`、`accessModes`、`capacity`。
2. **"no volume plugin matched"** → `storageClassName` 指定的 StorageClass 不存在或无 Provisioner。
3. **"waiting for a volume to be created"** → 动态供给正在创建 PV（正常的中间状态）。

### 匹配示例：正确绑定

以下示例展示 PVC 如何正确绑定到 PV：

**PV**（注意 `storageClassName: ""` 且 `capacity: 20Gi`）：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cbs-pv-example
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 20Gi
  storageClassName: ""
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: com.tencent.cloud.csi.cbs
    volumeHandle: disk-example01
    fsType: ext4
```

**PVC**（`storageClassName: ""` 匹配，`requests: 10Gi` <= 20Gi，`accessModes: ReadWriteOnce` 匹配）：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""
```

```bash
kubectl apply -f pv-example.yaml
kubectl apply -f pvc-example.yaml
# expected: PVC STATUS: Bound, VOLUME: cbs-pv-example
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| PV 状态 | `kubectl get pv` | 目标 PV STATUS: `Bound` |
| PVC 状态 | `kubectl get pvc PVC_NAME -n NAMESPACE` | STATUS: `Bound`，VOLUME 列指向目标 PV |
| 绑定详情 | `kubectl describe pv PV_NAME` | 可见 "Claim: NAMESPACE/PVC_NAME" |
| 存储后端 | 各存储类型对应控制面命令 | CBS: `cbs DescribeDisks` / CFS: `cfs DescribeCfsFileSystems` |

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get pv` 无任何 PV 可查 | `kubectl api-resources \| grep persistentvolume` 确认 API 资源存在 | 集群可能非标准或 RBAC 限制 | 确认有 `get persistentvolumes` 权限 |

### PVC 绑定失败

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC Pending，Events 报 "no persistent volumes available" | `kubectl describe pvc PVC_NAME -n NAMESPACE` 查看 Events | 无 Available 状态的 PV 满足 PVC 的所有条件 | 逐项检查：1) `storageClassName` 一致；2) PV `capacity` >= PVC `requests`；3) `accessModes` 匹配；4) 如有 selector，PV labels 匹配 |
| PVC Pending，有 PV 但全是 Bound | `kubectl get pv \| grep Available` | 所有 PV 已被绑定（一对一的限制） | 创建新的 PV，或删除不再使用的 PVC 释放 PV |
| PVC 绑定到错误的 PV | `kubectl get pvc PVC_NAME -o yaml \| grep volumeName` 确认绑定目标 | 未指定 `volumeName`，匹配规则匹配到了非预期 PV | 在 PVC 中明确指定 `spec.volumeName: PV_NAME` 精确绑定 |
| 动态 StorageClass 场景 PVC Pending | `kubectl describe pvc PVC_NAME` 查看 Events | Provisioner 未运行（CSI 组件未安装） | 确认对应 CSI 组件已安装：`tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName CBS` |

### 绑定后异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PV Released 后无法重新绑定 | `kubectl get pv PV_NAME` 显示 STATUS: `Released` | `Retain` 策略的 PV 在 PVC 删除后变成 Released，不自动变回 Available | 手动清理 PV 的后端存储数据（如云盘数据），然后 `kubectl delete pv PV_NAME` 重建 |
| PVC 删除后 PV 未释放（Retain 策略） | `kubectl get pv PV_NAME` 显示 STATUS: `Released` | `persistentVolumeReclaimPolicy: Retain`，这是预期行为 | 如不需要此 PV，执行 `kubectl delete pv PV_NAME`，并在存储控制台手动清理后端资源 |

## 清理

本页为概念说明与查询操作，所执行的均为只读命令（`kubectl get pv` / `kubectl get pvc` / `kubectl describe`），不会创建或变更任何云资源，无需清理。

若需清理各存储类型下的实际测试资源（PV、PVC、云硬盘、文件系统等），请参见各存储子页的清理章节。

## 下一步

- [使用云硬盘 CBS](../使用云硬盘%20CBS/PV%20和%20PVC%20管理云硬盘/tccli%20操作.md)
- [使用文件存储 CFS](../使用文件存储%20CFS/PV%20和%20PVC%20管理文件存储/tccli%20操作.md)
- [使用对象存储 COS](../使用对象存储%20COS/tccli%20操作.md)
- [StorageClass 管理云硬盘模板](../使用云硬盘%20CBS/StorageClass%20管理云硬盘模板/tccli%20操作.md) — 动态供给场景

## 控制台替代

[控制台 → 集群 → 存储](https://console.cloud.tencent.com/tke2/cluster) 查看 PersistentVolume 和 PersistentVolumeClaim 列表及绑定关系。
