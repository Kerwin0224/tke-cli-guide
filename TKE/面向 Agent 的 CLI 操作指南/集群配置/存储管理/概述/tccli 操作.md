# 存储管理概述

> 对照官方：[存储管理概述](https://cloud.tencent.com/document/product/457/46962) · page_id `46962`

## 概述

TKE 为工作负载提供四种持久化存储方案，覆盖块存储、文件存储、对象存储和分布式加速四个层级。

| 维度 | 云硬盘 CBS | 文件存储 CFS | 对象存储 COS | 数据加速器 GooseFS |
|------|-----------|-------------|-------------|-------------------|
| 存储层级 | 块级 | 文件级（NFS/CIFS） | 对象级 | 分布式加速文件 |
| 访问模式 | ReadWriteOnce | ReadWriteMany | ReadWriteMany | ReadWriteMany |
| 动态卷供给 | 支持 | 支持 | 不支持（仅静态） | 不支持（仅静态） |
| 典型场景 | 数据库、频繁细粒度更新 | 多 Pod 共享数据、媒体处理 | 海量文件上传下载、静态资源 | AI 训练数据加速、多机同时读 |
| 跨可用区挂载 | 不支持 | 支持 | 支持 | 依赖底层集群 |
| 默认 StorageClass | `cbs`（高性能云硬盘、按量计费） | 需自行创建 | 不适用 | 不适用 |

**核心概念**：

- **PV（PersistentVolume）**：集群内存储资源对象，独立于 Pod 生命周期。可由管理员静态创建，也可由 StorageClass 动态供给。
- **PVC（PersistentVolumeClaim）**：用户对存储的请求声明。当集群内无满足条件的 PV 时，若 StorageClass 支持动态供给则自动创建 PV。
- **StorageClass**：定义存储"等级"模板，决定动态创建 PV 的类型与参数。

**建议**：生产环境优先使用云存储服务（CBS/CFS/COS），避免使用 HostPath 等本地存储——节点异常时本地存储数据无法恢复。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon, tke:DescribeClusterAddons
#    cbs:DescribeDisks, cfs:DescribeCfsFileSystems
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <REGION>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"

# 5. 查看集群已安装的存储相关 Addon
tccli tke DescribeClusterAddons --region <REGION> --ClusterId <CLUSTER_ID>
# expected: 返回 Addons 列表，检查 CBS-CSI / CFS-CSI / COS-CSI / GooseFS-CSI 的 Status 和 Phase
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表及状态 | `DescribeClusters` | 是 |
| 查看集群已安装组件 | `DescribeClusterAddons` | 是 |
| 查看可用 CBS 云硬盘 | `cbs DescribeDisks` | 是 |
| 查看可用 CFS 文件系统 | `cfs DescribeCfsFileSystems` | 是 |
| 安装存储组件（CBS-CSI 等） | `InstallAddon` | 否 |
| 查看 StorageClass | `kubectl get storageclass` | 是 |
| 查看 PV/PVC 状态 | `kubectl get pv` / `kubectl get pvc -A` | 是 |

## 操作步骤

本页为概念说明页，仅含查询操作。各存储类型的安装和配置操作见对应子页。

### 步骤 1：确认目标集群状态

```bash
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'
# expected: ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.30.0"
        }
    ],
    "TotalCount": 1
}
```

### 步骤 2：查看集群已安装的存储组件

```bash
tccli tke DescribeClusterAddons --region <REGION> --ClusterId <CLUSTER_ID>
# expected: 返回 Addons 列表，Status 为 "Running" 表示已安装
```

**预期输出**：

```json
{
    "Addons": [
        {"AddonName": "CBS-CSI", "Status": "Running", "Phase": "Succeeded"},
        {"AddonName": "CFS-CSI", "Status": "Running", "Phase": "Succeeded"}
    ]
}
```

### 步骤 3：查询可用 CBS 云硬盘（静态 PV 场景）

```bash
tccli cbs DescribeDisks --region <REGION> --Limit 20
# expected: 返回 DiskSet，DiskState: "UNATTACHED" 的云盘可用于静态 PV
```

**预期输出**：

```json
{
    "DiskSet": [
        {"DiskId": "disk-example01", "DiskSize": 100, "DiskType": "CLOUD_PREMIUM", "DiskState": "UNATTACHED"},
        {"DiskId": "disk-example02", "DiskSize": 200, "DiskType": "CLOUD_SSD", "DiskState": "UNATTACHED"}
    ],
    "TotalCount": 2
}
```

### 步骤 4：查询可用 CFS 文件系统

```bash
tccli cfs DescribeCfsFileSystems --region <REGION> --Offset 0 --Limit 20
# expected: 返回 FileSystems 列表，LifeCycleState: "available" 的文件系统可用于 PV
```

**预期输出**：

```json
{
    "FileSystems": [
        {"FileSystemId": "cfs-example01", "LifeCycleState": "available", "Protocol": "NFS", "SizeByte": 107374182400},
        {"FileSystemId": "cfs-example02", "LifeCycleState": "available", "Protocol": "CIFS", "SizeByte": 53687091200}
    ],
    "TotalCount": 2
}
```

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'` | `ClusterStatus: "Running"` |
| 存储组件 | `tccli tke DescribeClusterAddons --region <REGION> --ClusterId <CLUSTER_ID>` | CBS-CSI / CFS-CSI 等组件 `Phase: "Succeeded"` |
| CBS 存量 | `tccli cbs DescribeDisks --region <REGION>` | 可列出 UNATTACHED 云盘 |
| CFS 存量 | `tccli cfs DescribeCfsFileSystems --region <REGION>` | 可列出 available 文件系统 |

### 数据面（kubectl）

```bash
kubectl get storageclass
# expected: 列出集群内 StorageClass（至少含默认 cbs）
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页仅执行只读查询操作，不会创建或变更云资源，无需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回空或集群不存在 | `tccli configure list` 检查 region 是否为目标地域 | region 与集群所在地域不一致 | 修改 `tccli configure set region <REGION>` 为目标地域 |
| `DescribeClusterAddons` 无 CSI 组件 | 命令返回空列表或仅含非存储组件 | 集群尚未安装对应 Addon | 进入对应子页，通过 `InstallAddon` 安装：`tccli tke InstallAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName CBS` |
| `cbs DescribeDisks` 返回 `UnauthorizedOperation` | 检查 CAM 策略是否包含 `cbs:DescribeDisks` | CAM 权限不足（此为环境限制） | 联系 CAM 管理员授予 `cbs:DescribeDisks` 权限 |
| `cfs DescribeCfsFileSystems` 返回 `UnauthorizedOperation` | 检查 CAM 策略是否包含 `cfs:DescribeCfsFileSystems` | CAM 权限不足（此为环境限制） | 联系 CAM 管理员授予 `cfs:DescribeCfsFileSystems` 权限 |

### 选型判断

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 数据库类工作负载写入慢 | `kubectl top pod POD_NAME` 查看 IO 指标 | 文件存储（CFS/NFS）延迟高于块存储 | 改用 CBS SSD 云硬盘，通过 `StorageClass` 指定 `diskType: CLOUD_SSD` |
| 多 Pod 需共享读写同一份数据 | CBS PVC 挂载到第二个 Pod 时报 `Multi-Attach` 错误 | CBS 仅支持 ReadWriteOnce | 改用 CFS（ReadWriteMany），参考 [文件存储使用说明](../使用文件存储CFS/文件存储使用说明/tccli%20操作.md) |
| COS PVC 创建后无法 Bound | `kubectl describe pvc PVC_NAME` 查看 Events | COS 仅支持静态 PV，不存在动态 Provisioner | 先手动创建 PV（绑定 COS 存储桶），再创建 PVC 指定 `volumeName`，参考 [使用对象存储 COS](../使用对象存储COS/tccli%20操作.md) |

## 下一步

按存储类型进入对应子页：

- [云硬盘使用说明](../使用云硬盘CBS/云硬盘使用说明/tccli%20操作.md) · [StorageClass 管理云硬盘模板](../使用云硬盘CBS/StorageClass管理云硬盘模板/tccli%20操作.md) · [PV 和 PVC 管理云硬盘](../使用云硬盘CBS/PV和PVC管理云硬盘/tccli%20操作.md)
- [文件存储使用说明](../使用文件存储CFS/文件存储使用说明/tccli%20操作.md) · [StorageClass 管理文件存储模板](../使用文件存储CFS/StorageClass管理文件存储模板/tccli%20操作.md) · [PV 和 PVC 管理文件存储](../使用文件存储CFS/PV和PVC管理文件存储/tccli%20操作.md)
- [使用对象存储 COS](../使用对象存储COS/tccli%20操作.md)
- [使用数据加速器 GooseFS](../使用数据加速器GooseFS/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../PV和PVC的绑定规则/tccli%20操作.md)
- [其他存储卷使用说明](../其他存储卷使用说明/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 存储](https://console.cloud.tencent.com/tke2/cluster) 查看 PersistentVolume、PersistentVolumeClaim、StorageClass 列表及状态。
