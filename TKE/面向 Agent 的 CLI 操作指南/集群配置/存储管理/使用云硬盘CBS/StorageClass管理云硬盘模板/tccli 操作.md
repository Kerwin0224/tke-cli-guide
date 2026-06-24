# StorageClass 管理云硬盘模板

> 对照官方：[StorageClass 管理云硬盘模板](https://cloud.tencent.com/document/product/457/44239) · page_id `44239`

## 概述

通过创建 CBS 类型的 StorageClass，实现 PVC 动态创建 CBS 云硬盘并绑定到 Pod。TKE 托管集群默认预装了名为 `cbs` 的 StorageClass（高性能云硬盘、按量计费），可直接使用。

**与静态 PV 对比**：

| 维度 | 动态创建（StorageClass） | 静态创建（已有云盘） |
|------|---------------------|----------------|
| 云硬盘来源 | PVC 创建时自动新建 | 需提前在 CBS 创建 |
| 可用区匹配 | `WaitForFirstConsumer` 确保与 Pod 同可用区 | 需手动确认 |
| 管控粒度 | StorageClass 统一模板 | 每个 PV 独立配置 |
| 适合场景 | 按需自动供给 | 存量云盘复用、数据迁移 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 CBS-CSI 组件：[云硬盘使用说明](../云硬盘使用说明/tccli%20操作.md)
- kubectl 已连接目标集群

### 环境检查

```bash
# 1. 确认 CBS-CSI 已安装
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName CBS
# expected: Phase: "Succeeded", Status: "Running"

# 2. 确认 kubectl 可达
kubectl get ns
# expected: 返回命名空间列表

# 3. 查看默认 cbs StorageClass
kubectl get sc cbs
# expected: 如已存在则显示默认 cbs SC
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 安装 CBS-CSI | `InstallAddon --AddonName CBS` | 否 |
| 创建 StorageClass | `kubectl apply -f cbs-sc.yaml` | 否 |
| 创建 PVC | `kubectl apply -f cbs-pvc.yaml` | 否 |
| 扩容 PVC | `kubectl patch pvc PVC_NAME -p '{"spec":{"resources":{"requests":{"storage":"SIZE"}}}}'` | 否 |

## 关键字段说明

以下说明 CBS StorageClass YAML 中的关键参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `provisioner` | String | 是 | 固定值 `com.tencent.cloud.csi.cbs` | 填错 → PVC 一直 Pending |
| `parameters.diskType` | String | 是 | `CLOUD_PREMIUM`（高性能）、`CLOUD_SSD`（SSD）、`CLOUD_BSSD`（通用型 SSD）、`CLOUD_HSSD`（增强型 SSD） | 类型不存在 → CBS 创建失败 |
| `parameters.diskChargeType` | String | 否 | `POSTPAID_BY_HOUR`（按量计费，默认）或 `PREPAID`（包年包月） | `PREPAID` 需额外配置续费参数 |
| `reclaimPolicy` | String | 否 | `Delete`（PVC 删则云盘删）或 `Retain`（保留云盘） | `Delete` 误删 PVC → 云盘数据丢失 |
| `volumeBindingMode` | String | 否 | `Immediate`（立即创建）或 `WaitForFirstConsumer`（Pod 调度后创建，推荐） | `Immediate` 可能创建在错误的可用区，导致 Pod 无法挂载 |
| `allowVolumeExpansion` | Boolean | 否 | `true`（允许在线扩容）或 `false`（默认） | 设为 `false` 后 PVC 无法扩容 |

## 操作步骤

### 步骤 1：创建 StorageClass

#### 选择依据

- **diskType**：数据库类负载选 `CLOUD_SSD`（低延迟），常规负载选 `CLOUD_PREMIUM`（成本优化）。
- **volumeBindingMode**：**强烈推荐 `WaitForFirstConsumer`**。`Immediate` 模式在 PVC 创建时立即创建云盘，但此时 Pod 尚未调度，云盘可用区可能不匹配。
- **allowVolumeExpansion**：推荐 `true`，允许后续在线扩容。
- **reclaimPolicy**：测试环境 `Delete`，生产环境推荐 `Retain`。

#### 最小配置

`cbs-sc.yaml`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cbs-standard
provisioner: com.tencent.cloud.csi.cbs
parameters:
  diskType: CLOUD_PREMIUM
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f cbs-sc.yaml
# expected: storageclass.storage.k8s.io/cbs-standard created
```

#### 增强配置（SSD + 标签）

`cbs-sc-enhanced.yaml`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cbs-ssd
  labels:
    env: production
provisioner: com.tencent.cloud.csi.cbs
parameters:
  diskType: CLOUD_SSD
  diskChargeType: POSTPAID_BY_HOUR
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### 步骤 2：创建 PVC 自动供应云硬盘

#### 选择依据

- **accessModes**：`ReadWriteOnce`，CBS 云硬盘不支持多节点同时挂载。
- **storageClassName**：使用步骤 1 创建的 `cbs-standard`。

#### 最小配置

`cbs-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc-dynamic
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: cbs-standard
```

```bash
kubectl apply -f cbs-pvc.yaml
# expected: persistentvolumeclaim/cbs-pvc-dynamic created
```

### 步骤 3：验证

```bash
kubectl get pvc cbs-pvc-dynamic
# expected: STATUS: Pending（WaitForFirstConsumer 模式，需 Pod 消费后才创建云盘）
```

```text
NAME  STATUS  AGE
...
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| StorageClass 存在 | `kubectl get sc cbs-standard` | NAME: `cbs-standard`，PROVISIONER: `com.tencent.cloud.csi.cbs` |
| PVC 创建 | `kubectl get pvc cbs-pvc-dynamic` | STATUS: `Pending`（WaitForFirstConsumer 模式预期行为） |
| PVC Bound（Pod 消费后） | `kubectl get pvc cbs-pvc-dynamic` | STATUS: `Bound` |
| 云硬盘创建 | `tccli cbs DescribeDisks --region <Region>` | 出现新云盘，DiskState: `ATTACHED` |

## 清理

> **警告**：`reclaimPolicy: Delete` 时，删除 PVC 会同步删除 CBS 云硬盘及所有数据，不可恢复。生产环境建议使用 `Retain`。
>
> CBS 云硬盘按量计费持续产生费用，未绑定的云硬盘也计费。`Retain` 策略下删除 PVC 后，需在 [CBS 控制台](https://console.cloud.tencent.com/cvm/cbs) 手动退还云硬盘。

### 1. 清理前状态检查

```bash
kubectl get pvc cbs-pvc-dynamic
kubectl get pv | grep cbs-pvc-dynamic
# 确认 reclaimPolicy
```

```text
NAME  STATUS  AGE
...
```

### 2. 删除 PVC

```bash
kubectl delete pvc cbs-pvc-dynamic
# expected: persistentvolumeclaim "cbs-pvc-dynamic" deleted
# reclaimPolicy: Delete → PV 和云盘同步删除
# reclaimPolicy: Retain → PV 保留（Released），云盘保留
```

### 3. 删除 StorageClass（可选）

```bash
kubectl delete sc cbs-standard
# expected: storageclass.storage.k8s.io "cbs-standard" deleted
```

### 4. 验证已删除

```bash
kubectl get pvc cbs-pvc-dynamic 2>&1
# expected: Error from server (NotFound)
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f cbs-sc.yaml` 创建成功但 PVC 一直 Pending（非 WaitForFirstConsumer 原因） | `kubectl describe pvc cbs-pvc-dynamic` 查看 Events | Provisioner 未就绪（CBS-CSI 组件未安装） | 参考 [云硬盘使用说明](../云硬盘使用说明/tccli%20操作.md) 安装 CBS-CSI |
| `kubectl apply -f cbs-sc.yaml` 报 provisioner 未找到 | `kubectl get csidriver \| grep cbs` | CBS-CSI 组件未安装 | 安装 CBS-CSI 组件 |

### PVC 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC Pending 且 Events 报 "diskType not supported" | 检查 `cbs-sc.yaml` 中的 `diskType` 值 | `diskType` 值拼写错误或该地域不支持该类型 | 改为有效值：`CLOUD_PREMIUM` / `CLOUD_SSD` / `CLOUD_BSSD` / `CLOUD_HSSD` |
| PVC Bound 但 Pod 挂载失败（可用区不匹配） | `kubectl describe pod POD_NAME` 查看 Events，检查 Node 可用区 | `volumeBindingMode: Immediate` 导致云盘创建在非预期的可用区 | 删除 PVC，改用 `WaitForFirstConsumer` 模式重建 |
| PVC 绑定了不符合预期的 StorageClass | `kubectl get pvc cbs-pvc-dynamic -o yaml \| grep storageClassName` | 默认 `cbs` StorageClass 优先级更高 | 在 PVC 中明确指定 `storageClassName: cbs-standard` |

## 下一步

- [PV 和 PVC 管理云硬盘](../PV和PVC管理云硬盘/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../../PV和PVC的绑定规则/tccli%20操作.md)
- [使用文件存储 CFS](../../使用文件存储CFS/StorageClass管理文件存储模板/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 存储 → StorageClass](https://console.cloud.tencent.com/tke2/cluster) 新建 StorageClass，选择 CBS 模板。
