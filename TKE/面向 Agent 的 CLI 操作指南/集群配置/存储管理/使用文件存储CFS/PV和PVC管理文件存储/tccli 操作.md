# PV 和 PVC 管理文件存储

> 对照官方：[PV 和 PVC 管理文件存储](https://cloud.tencent.com/document/product/457/44236) · page_id `44236`

## 概述

通过已有 CFS 文件系统创建静态 PV，再通过 PVC 绑定后挂载到 Pod。适用于存量 CFS 实例复用场景。

**与动态创建对比**：

| 维度 | 静态创建（本页） | 动态创建（StorageClass） |
|------|-------------|---------------------|
| CFS 来源 | 提前手动创建 CFS | PVC 创建时自动新建 |
| 文件系统 ID / IP | 手动填入 PV YAML | 自动分配 |
| 控制粒度 | 精确控制每个 PV 参数 | 统一模板 |
| 适合场景 | 存量迁移、手动管控 | 按需供给、新建 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 CFS-CSI 组件：[文件存储使用说明](../文件存储使用说明/tccli%20操作.md)
- 已有 CFS 文件系统：在 [CFS 控制台](https://console.cloud.tencent.com/cfs) 或通过 `tccli cfs CreateCfsFileSystem` 创建
- kubectl 已连接目标集群

### 环境检查

```bash
# 1. 确认 CFS-CSI 已安装
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName CFS
# expected: Phase: "Succeeded", Status: "Running"

# 2. 确认 CFS 文件系统存在且可用
tccli cfs DescribeCfsFileSystems --region <Region>
# expected: 返回 CFS 实例列表，记录 FileSystemId 和 IpAddress

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
| 查询 CFS 实例 | `cfs DescribeCfsFileSystems` | 是 |
| 查询挂载点 | `cfs DescribeMountTargets` | 是 |
| 创建 PV | `kubectl apply -f cfs-pv.yaml` | 是（同名覆盖） |
| 创建 PVC | `kubectl apply -f cfs-pvc.yaml` | 是（同名覆盖） |

## 关键字段说明

以下说明 CFS 静态 PV YAML 中的关键参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `csi.driver` | String | 是 | 固定值 `com.tencent.cloud.csi.cfs` | 填错 → PV 创建成功但无法被 CSI 使用 |
| `csi.volumeAttributes.host` | String | 是 | CFS 挂载点 IP 地址，如 `10.0.1.5` | IP 错误 → Pod 挂载超时 |
| `csi.volumeAttributes.vers` | String | 否 | `"3.0"`（NFSv3）或 `"4.0"`（NFSv4），需与 CFS 实例协议一致 | 版本不匹配 → 挂载失败 |
| `csi.volumeHandle` | String | 是 | CFS 文件系统 ID，格式 `cfs-xxxxxxxx` | ID 错误 → Pod 挂载失败 |
| `spec.storageClassName` | String | 是 | 静态 PV 设为 `""`（空字符串），表示不属于任何 StorageClass | 设为某个 SC 名 → PVC 可能被其他动态卷抢占 |

## 操作步骤

### 步骤 1：查询 CFS 实例信息

#### 选择依据

需要获取两个关键字段：`FileSystemId`（`cfs-xxxxxxxx` 格式）和 `IpAddress`（挂载点 IP）。

```bash
tccli cfs DescribeCfsFileSystems --region <Region>
# expected: 返回 CFS 实例列表，每个实例含 FileSystemId、IpAddress、LifeCycleState
```

**预期输出**：

```json
{
    "FileSystems": [
        {
            "FileSystemId": "cfs-example01",
            "FsName": "my-cfs",
            "LifeCycleState": "available",
            "Protocol": "NFS",
            "IpAddress": "10.0.1.5",
            "VpcId": "vpc-example"
        }
    ]
}
```

### 步骤 2：创建 PV

#### 选择依据

- **accessModes**：`ReadWriteMany`，CFS 的核心优势。
- **persistentVolumeReclaimPolicy**：`Retain`（推荐），删除 PVC 时保留 CFS 实例和数据。设为 `Delete` 会删除 CFS 实例。
- **storageClassName**：设为 `""`（空字符串），表示此 PV 为静态 PV，不会被 StorageClass 动态供给抢占。

#### 最小配置

`cfs-pv.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cfs-pv-static
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 100Gi
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeAttributes:
      host: CFS_IP_ADDRESS
      vers: "3.0"
    volumeHandle: CFS_FILE_SYSTEM_ID
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
```

```bash
kubectl apply -f cfs-pv.yaml
# expected: persistentvolume/cfs-pv-static created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `CFS_IP_ADDRESS` | CFS 文件系统挂载点 IP | `tccli cfs DescribeCfsFileSystems --region <Region>` 输出中 `IpAddress` 字段 |
| `CFS_FILE_SYSTEM_ID` | CFS 文件系统 ID，格式 `cfs-xxxxxxxx` | 同上，`FileSystemId` 字段 |

### 步骤 3：创建 PVC 绑定

`cfs-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfs-pvc-static
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: ""
  volumeName: cfs-pv-static
```

```bash
kubectl apply -f cfs-pvc.yaml
# expected: persistentvolumeclaim/cfs-pvc-static created
```

### 步骤 4：验证绑定状态

```bash
kubectl get pv cfs-pv-static
# expected: STATUS: Bound

kubectl get pvc cfs-pvc-static
# expected: STATUS: Bound, VOLUME: cfs-pv-static
```

```text
NAME  STATUS  AGE
...
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| PV 状态 | `kubectl get pv cfs-pv-static` | STATUS: `Bound` |
| PVC 状态 | `kubectl get pvc cfs-pvc-static` | STATUS: `Bound`，VOLUME 指向 `cfs-pv-static` |
| CFS 实例状态 | `tccli cfs DescribeCfsFileSystems --region <Region>` | LifeCycleState: `available` |

## 清理

> **警告**：`persistentVolumeReclaimPolicy: Retain` 时，删除 PVC 后 PV 变为 `Released` 状态但 **CFS 实例和数据保留**。如需清理 CFS，需在 [CFS 控制台](https://console.cloud.tencent.com/cfs) 手动删除，否则持续产生 CFS 存储费用。
>
> `persistentVolumeReclaimPolicy: Delete` 时，删除 PVC 会**同步删除 CFS 文件系统及所有数据**，不可恢复。

### 1. 清理前状态检查

```bash
kubectl get pv cfs-pv-static
kubectl get pvc cfs-pvc-static
# 确认 reclaimPolicy 和绑定状态
```

```text
NAME  STATUS  AGE
...
```

### 2. 删除 PVC

```bash
kubectl delete pvc cfs-pvc-static
# expected: persistentvolumeclaim "cfs-pvc-static" deleted
```

### 3. 删除 PV

```bash
kubectl delete pv cfs-pv-static
# expected: persistentvolume "cfs-pv-static" deleted
```

### 4. 手动清理 CFS 实例（Retain 策略时）

```bash
# 确认 CFS 仍存在
tccli cfs DescribeCfsFileSystems --region <Region>
# 如需删除，在 CFS 控制台或通过命令行操作
```

### 5. 验证已删除

```bash
kubectl get pv cfs-pv-static 2>&1
# expected: Error from server (NotFound)

kubectl get pvc cfs-pvc-static 2>&1
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
| PV 创建成功但 PVC 不绑定 | `kubectl describe pvc cfs-pvc-static` 查看 Events | PVC 的 `storageClassName` 或 `volumeName` 与 PV 不匹配 | 确认 PVC 中 `storageClassName: ""` 与 PV 一致，且 `volumeName: cfs-pv-static` 正确 |
| `kubectl apply -f cfs-pv.yaml` 报驱动未找到 | `kubectl get csidriver \| grep cfs` | CFS-CSI 组件未安装 | 参考 [文件存储使用说明](../文件存储使用说明/tccli%20操作.md) 安装 |

### PVC 绑定后 Pod 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 挂载 CFS 超时或失败 | `kubectl describe pod POD_NAME` 查看 Events | CFS IP 不可达，可能 IP 填错或 VPC 网络不通 | 确认 `volumeAttributes.host` 与 `DescribeCfsFileSystems` 输出的 `IpAddress` 一致 |
| Pod 挂载成功但读写报 "Permission denied" | `kubectl exec POD_NAME -- ls /mnt/cfs` | NFS 挂载权限问题 | 确认 Pod 的 `securityContext` 中 `fsGroup` 与 CFS 文件权限一致 |
| PVC Bound 但 CFS 实例与期望不符 | `tccli cfs DescribeCfsFileSystems --region <Region>` 对比 | `volumeHandle` 填错导致绑定到错误的 CFS 实例 | 删除 PV/PVC，修正 `cfs-pv.yaml` 中的 `volumeHandle` 后重建 |

## 下一步

- [StorageClass 管理文件存储模板](../StorageClass管理文件存储模板/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../../PV和PVC的绑定规则/tccli%20操作.md)
- [使用云硬盘 CBS](../../使用云硬盘CBS/PV和PVC管理云硬盘/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 存储 → PersistentVolume](https://console.cloud.tencent.com/tke2/cluster) 创建 PV，选择已有 CFS。
