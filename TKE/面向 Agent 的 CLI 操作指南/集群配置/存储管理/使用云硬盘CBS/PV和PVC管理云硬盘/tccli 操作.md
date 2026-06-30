# PV 和 PVC 管理云硬盘

> 对照官方：[PV 和 PVC 管理云硬盘](https://cloud.tencent.com/document/product/457/44240) · page_id `44240`

## 概述

通过已有 CBS 云硬盘创建静态 PV，再通过 PVC 绑定后挂载到 Pod。适用于存量云盘数据迁移和手动管控场景。支持在线扩容已挂载的云硬盘（需 StorageClass 启用 `allowVolumeExpansion`）。

**关键限制**：

- CBS 仅支持 **ReadWriteOnce**，一个云硬盘同时只能挂载到一个节点的一个 Pod。
- 云硬盘**不支持跨可用区挂载**，Pod 所在节点必须与云硬盘在同一可用区。
- 静态 PV 扩容后，需在 CBS 控制台或通过 API 同步执行云盘扩容操作。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 CBS-CSI 组件：[云硬盘使用说明](../云硬盘使用说明/tccli%20操作.md)
- 已有 CBS 云硬盘（状态为 `UNATTACHED` 或已从原节点卸载）
- kubectl 已连接目标集群

### 环境检查

```bash
# 1. 确认 CBS-CSI 已安装
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName CBS
# expected: Phase: "Succeeded", Status: "Running"

# 2. 查询可用的 UNATTACHED 云硬盘
tccli cbs DescribeDisks --region <Region>
# expected: DiskSet 列表，记录 DiskId、DiskSize、DiskType、DiskState

# 3. 确认 kubectl 可达
kubectl get ns
# expected: 返回命名空间列表
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查询云硬盘 | `cbs DescribeDisks` | 是 |
| 创建 PV | `kubectl apply -f cbs-pv.yaml` | 是（同名覆盖） |
| 创建 PVC | `kubectl apply -f cbs-pvc.yaml` | 是（同名覆盖） |
| 扩容 PVC | `kubectl patch pvc PVC_NAME -p '{"spec":{"resources":{"requests":{"storage":"SIZE"}}}}'` | 否 |

## 关键字段说明

以下说明 CBS 静态 PV YAML 中的关键参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `csi.driver` | String | 是 | 固定值 `com.tencent.cloud.csi.cbs` | 填错 → PV 无法被 CBS-CSI 识别 |
| `csi.fsType` | String | 否 | 文件系统类型：`ext4`（默认，推荐）或 `xfs` | 类型与云盘现有文件系统不匹配 → 挂载失败 |
| `csi.volumeAttributes.diskType` | String | 是 | `CLOUD_PREMIUM` / `CLOUD_SSD` / `CLOUD_BSSD` / `CLOUD_HSSD`，需与云盘实际类型一致 | 填错 → 挂载失败 |
| `csi.volumeHandle` | String | 是 | CBS 云硬盘 ID，格式 `disk-xxxxxxxx` | ID 错误 → 挂载失败 |
| `spec.storageClassName` | String | 是 | 设为 `""`（空字符串），表示此为静态 PV | 设非空值 → 可能被 StorageClass 匹配导致意外行为 |
| `persistentVolumeReclaimPolicy` | String | 否 | `Retain`（推荐，保留云盘）或 `Delete`（PVC 删则云盘删） | `Delete` 误删 PVC → 云盘数据丢失 |

## 操作步骤

### 步骤 1：查询可用云硬盘

```bash
tccli cbs DescribeDisks --region <Region>
# expected: 返回 DiskSet，找出 DiskState 为 "UNATTACHED" 的目标云盘
```

**预期输出**：

```json
{
    "DiskSet": [
        {
            "DiskId": "disk-example01",
            "DiskSize": 20,
            "DiskType": "CLOUD_PREMIUM",
            "DiskState": "UNATTACHED",
            "Placement": {"Zone": "ap-guangzhou-3"}
        }
    ]
}
```

### 步骤 2：创建 PV

#### 选择依据

- **csi.fsType**：`ext4`，Linux 标准文件系统，兼容性最好。
- **persistentVolumeReclaimPolicy**：`Retain`（推荐），删除 PVC 时保留云盘及数据。`Delete` 会同步删除云盘。
- **storageClassName**：`""`（空字符串），避免与动态 StorageClass 冲突。

#### 最小配置

`cbs-pv.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cbs-pv-static
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 20Gi
  csi:
    driver: com.tencent.cloud.csi.cbs
    fsType: ext4
    volumeAttributes:
      diskType: CLOUD_PREMIUM
    volumeHandle: DISK_ID
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
```

```bash
kubectl apply -f cbs-pv.yaml
# expected: persistentvolume/cbs-pv-static created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `DISK_ID` | CBS 云硬盘 ID，格式 `disk-xxxxxxxx` | `tccli cbs DescribeDisks --region <Region>` 输出中 `DiskId` 字段 |

### 步骤 3：创建 PVC 绑定

`cbs-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc-static
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: ""
  volumeName: cbs-pv-static
```

```bash
kubectl apply -f cbs-pvc.yaml
# expected: persistentvolumeclaim/cbs-pvc-static created
```

### 步骤 4：验证绑定

```bash
kubectl get pv cbs-pv-static
# expected: STATUS: Bound

kubectl get pvc cbs-pvc-static
# expected: STATUS: Bound, VOLUME: cbs-pv-static
```

```text
NAME  STATUS  AGE
...
```

### 步骤 5：在线扩容（可选）

#### 前置条件

PV 对应的 StorageClass 需启用 `allowVolumeExpansion: true`。静态 PV 扩容需先修改 PVC 的 `storage` 字段，再在 CBS 侧同步扩容。

```bash
# 扩大 PVC 存储请求（新值必须大于当前值）
kubectl patch pvc cbs-pvc-static -p '{"spec":{"resources":{"requests":{"storage":"40Gi"}}}}'
# expected: persistentvolumeclaim/cbs-pvc-static patched

kubectl get pvc cbs-pvc-static
# expected: CAPACITY 逐步变为 40Gi
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 云盘状态 | `tccli cbs DescribeDisks --region <Region> --DiskIds '["DISK_ID"]'` | DiskState: `ATTACHED`（如已挂载 Pod） |
| 云盘大小 | 同上 | DiskSize 与 PV/PVC 声明一致 |

### 数据面（kubectl）

```bash
kubectl get pv cbs-pv-static
# expected: STATUS: Bound

kubectl get pvc cbs-pvc-static
# expected: STATUS: Bound, CAPACITY: 20Gi
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：`persistentVolumeReclaimPolicy: Retain` 时，删除 PVC/PV 后云硬盘**保留且持续计费**。需在 [CBS 控制台](https://console.cloud.tencent.com/cvm/cbs) 手动退还。
>
> `persistentVolumeReclaimPolicy: Delete` 时，删除 PVC 会**同步删除 CBS 云硬盘及所有数据**，不可恢复。

### 1. 清理前状态检查

```bash
kubectl get pv cbs-pv-static
kubectl get pvc cbs-pvc-static
# 确认 reclaimPolicy 和绑定状态
```

```text
NAME  STATUS  AGE
...
```

### 2. 删除 PVC

```bash
kubectl delete pvc cbs-pvc-static
# expected: persistentvolumeclaim "cbs-pvc-static" deleted
```

### 3. 删除 PV

```bash
kubectl delete pv cbs-pv-static
# expected: persistentvolume "cbs-pv-static" deleted
```

### 4. 手动清理云硬盘（Retain 策略时）

```bash
# 确认云盘仍存在
tccli cbs DescribeDisks --region <Region> --DiskIds '["DISK_ID"]'
# 需在 CBS 控制台手动退还（卸载后销毁）
```

### 5. 验证已删除

```bash
kubectl get pv cbs-pv-static 2>&1
# expected: Error from server (NotFound)

kubectl get pvc cbs-pvc-static 2>&1
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
| PV 创建成功但 PVC 不绑定 | `kubectl describe pvc cbs-pvc-static` 查看 Events | PVC 的 `volumeName` 或 `storageClassName` 与 PV 不匹配 | 确认 PVC 的 `volumeName: cbs-pv-static` 与 PV 的 `name` 一致，且 `storageClassName: ""` 匹配 |
| `kubectl apply -f cbs-pv.yaml` 报驱动未找到 | `kubectl get csidriver \| grep cbs` | CBS-CSI 组件未安装 | 参考 [云硬盘使用说明](../云硬盘使用说明/tccli%20操作.md) 安装 |

### PVC 绑定后 Pod 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 挂载失败（`Multi-Attach` 错误） | `kubectl describe pod POD_NAME` 查看 Events | 同一 PVC 被多个 Pod 挂载（不同节点） | CBS 仅支持 ReadWriteOnce。确保只有一个 Pod 使用该 PVC，或多副本使用 StatefulSet 各副本独立 PVC |
| Pod 挂载失败（可用区不匹配） | `tccli cbs DescribeDisks --region <Region> --DiskIds '["DISK_ID"]'` 查看 `Placement.Zone` | 云盘可用区与 Pod 节点可用区不一致 | 删除 Pod，确保其调度到与云盘同可用区的节点（使用 nodeSelector 指定 zone） |
| `kubectl patch pvc` 扩容后不生效 | `kubectl get pvc cbs-pvc-static -o yaml` 检查条件 `FileSystemResizePending` | 云盘本身未扩容或文件系统未扩展 | 确认 StorageClass 已启用 `allowVolumeExpansion: true`，重启 Pod 触发文件系统 resize |

## 下一步

- [StorageClass 管理云硬盘模板](../StorageClass管理云硬盘模板/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../../PV和PVC的绑定规则/tccli%20操作.md)
- [使用文件存储 CFS](../../使用文件存储CFS/PV和PVC管理文件存储/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 存储 → PersistentVolume](https://console.cloud.tencent.com/tke2/cluster) 创建 PV，选择已有 CBS 云硬盘。
