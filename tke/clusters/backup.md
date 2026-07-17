---
doc_type: How-to
subtype: 6A
fused: false
---
# 集群备份存储位置

> 配置 TKE 集群备份的存储位置（基于 Velero + COS），用于集群灾备与迁移。
> 控制台: [容器服务 - 备份 - 存储仓库](https://console.cloud.tencent.com/tke2/backup)

> 本文档 Action 属 **TKE 2018-05-25**（旧版独有，新版无）。备份存储位置是全局命名资源，不绑定集群——`--ClusterId` 不是其入参。

> 官方文档：[基本概念](https://cloud.tencent.com/document/product/457/45598) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 概述

TKE 集群备份组件以 Velero 为基础，将 Kubernetes 资源对象备份到对象存储（COS）。备份存储位置（BackupStorageLocation）是备份的目标 COS 桶配置。控制台“备份中心”中的“备份仓库/存储仓库”对应本文的 `BackupStorageLocation`；备份与恢复由集群内的 `tke-backup` 组件通过 TKE 自定义 `Backup`、`BackupSchedule`、`Restore` 资源执行，不是同一个 TKE API 资源。

> ⚠️ **产品边界（灾备生命周期三段，TCCLI 只覆盖第一段）**：
> - **配置存储位置** → tccli `CreateBackupStorageLocation`（本文档）
> - **执行备份** → TKE 控制台“运维中心 > 备份中心”创建备份
> - **从备份恢复** → TKE 控制台“运维中心 > 备份中心”创建恢复
>
> ⚠️ **恢复边界**：当前**仅支持 Kubernetes 资源对象**的备份与恢复；**不支持**云硬盘 CBS、负载均衡 CLB 等云资源的恢复。跨集群恢复要求备份组件 ≥1.1.0。
>
> TKE API 无 `ExecuteBackup`/`Restore` Action。本文覆盖 TCCLI 能完成的存储位置 CRUD；备份与恢复应通过 TKE 备份中心执行。

> ⚠️ 备份存储位置是**全局命名资源**，不绑定集群（`--ClusterId` 不是其入参）。一个位置可被多个集群共用。多地域 TKE 集群备份到同一仓库无需重复创建。单账号最多创建 **100 个**备份仓库，超出需清理闲置仓库。

> 配额：COS 桶名须以 `tke-backup` 开头；单账号最多创建 **100** 个备份仓库。国内地域与国外地域不能共用同一个备份仓库。

## 触发条件

- 生产/重要集群需灾备，但还没配备份存储位置（`DescribeBackupStorageLocations` 为空）— 用本文创建位置，再通过 TKE 备份中心执行备份
- 需跨地域迁移集群（备份后在新地域恢复）— 先用本文配存储位置
- 已有位置需调整 COS 桶或路径 — 删旧位置重建（位置不可改名，覆盖式修改需先删后建）

## 准备工作

- 已创建 TKE 集群 + 已开通 COS 服务 (备份存 COS)
- 已配置 tccli 凭证 (见 [配置凭证](../../getting-started/credentials.md))


## 决策依据

### COS 桶选型

| 项 | 要求 | 说明 |
|:---|:-----|:-----|
| 桶名前缀 | **必须以 `tke-backup` 开头** | 不符合前缀约束时，服务端返回 `InvalidParameter` |
| `Provider` | `tencentcloud` | 默认值，可省略 |
| `StorageRegion` | COS 桶所在地域 | 如 `ap-guangzhou` |
| `Path` | 桶内路径 | 如 `/backup` |

> ⚠️ **桶名前缀是硬约束**：COS 桶名必须以 `tke-backup` 开头。先在 COS 服务创建合规桶，再配存储位置。

### 是否需要备份

| 场景 | 是否配置 | 说明 |
|:-----|:--------:|:-----|
| 生产集群 | ✅ | 灾备，资源可恢复 |
| 临时/测试集群 | ❌ | 备份有 COS 存储成本 |
| 跨地域迁移 | ✅ | 备份后在新地域恢复 |

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `Name` | Create/Delete | 是 | 存储位置名（全局唯一） |
| `StorageRegion` | Create | 是 | COS 桶地域 |
| `Bucket` | Create | 是 | COS 桶名：须 `tke-backup` 前缀；help 声明字符长度 19（创建前以 `tccli tke CreateBackupStorageLocation help --detail` 为准） |
| `Provider` | Create | 否 | `tencentcloud`（默认 `tencentcloud`，可省略） |
| `Path` | Create | 否 | 桶内路径 |
| `Names[]` | Describe | 否 | 按名查询（空则查全部） |

> `DescribeBackupStorageLocations` 用 `Names[]`，不接受 `--ClusterId`。

## 操作步骤

> ⚠️ **高危操作**：恢复覆盖现有资源不可逆；备份过期清理策略不当致数据丢失；跨 Region 恢复需验证兼容性（当前仅支持 K8s 资源对象，不支持 CBS/CLB 云资源恢复）。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

### 步骤 1：创建合规 COS 桶

在 COS 服务创建桶名以 `tke-backup` 开头的桶。help 声明 `Bucket` 字符长度 19（含前缀）；创建前以 `tccli tke CreateBackupStorageLocation help --detail` 与 COS 实际命名规则为准。

### 步骤 2：创建备份存储位置

```bash
tccli tke CreateBackupStorageLocation --region <REGION> \
  --StorageRegion <STORAGE_REGION> \
  --Bucket <BUCKET_NAME> \
  --Name <LOCATION_NAME> \
  --Provider tencentcloud \
  --Path /backup
# expected: exit 0
```

### 步骤 3：查询确认

```bash
tccli tke DescribeBackupStorageLocations --region <REGION>
# expected: exit 0；无位置时 BackupStorageLocationSet 可能为 null（非 []）；有位置时为数组，含新建项且 State=Available
```
```json
{
    "BackupStorageLocationSet": [
        {
            "Name": "test-example",
            "StorageRegion": "ap-guangzhou",
            "Provider": "tencentcloud",
            "Bucket": "tke-backup-example-1250000000",
            "Path": "/backup",
            "State": "Available",
            "Message": "",
            "LastValidationTime": "2026-01-01 00:00:00 +0800 CST"
        }
    ]
}
```

> `State` 可取 `Available` 或 `Unavailable`；不可用时检查 `Message`，并核对 COS 桶、TKE 服务角色授权和仓库配置。

## 验证

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 位置已创建 | `DescribeBackupStorageLocations` | `BackupStorageLocationSet` 含该 Name |
| 位置可用 | 同上 | `State=Available` |
| 不可用诊断 | 同上 | `State=Unavailable` 时检查 `Message`、COS 桶、服务角色授权和仓库配置 |
| 按名查询 | `DescribeBackupStorageLocations --Names '["<LOCATION_NAME>"]'` | 返回该位置 |

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGION>` | 调用地域 | 如 `ap-guangzhou` | `tccli cvm DescribeRegions` |
| `<STORAGE_REGION>` | COS 桶地域 | 与桶一致 | COS 控制台或 `coscli ls`（COS 独立工具） |
| `<BUCKET_NAME>` | COS 桶名 | **`tke-backup` 前缀** | COS 控制台或 `coscli ls`（TCCLI 无 cos 服务） |
| `<LOCATION_NAME>` | 存储位置名 | 全局唯一 | 自定义 |

## 执行备份与恢复（TKE 备份中心边界）

TCCLI 负责备份存储位置 CRUD。创建存储位置后，在 TKE 控制台进入“运维中心 > 备份中心”，使用 TKE 备份组件创建备份或恢复。TKE 备份组件位于 `tke-backup` namespace，其中 Deployment 和 Service 名称均为 `tke-backup`。

<!-- kubectl检查TKE备份组件，tccli无K8s工作负载管理能力，非tccli边界 -->
```bash
kubectl get deployment,service -n tke-backup tke-backup
# expected: Deployment 与 Service 均存在；Deployment AVAILABLE 为 1 或更大
```

跨集群恢复前，确认目标集群的备份组件版本为 1.1.0 或更高。恢复仅包含 Kubernetes 资源对象；PVC 背后的 CBS、Service/Ingress 关联的 CLB 不随备份恢复，需要另行重建或重新绑定。

> TKE 使用自定义 `Backup`、`BackupSchedule`、`Restore` 资源。本文不使用标准 Velero CLI 或未确认的资源短名直接操作这些资源；备份和恢复以 TKE 备份中心提供的操作路径为准。

## 清理

```bash
tccli tke DeleteBackupStorageLocation --region <REGION> --Name <LOCATION_NAME>
# expected: exit 0
```

> `DeleteBackupStorageLocation` 仅需 `Name`。删除位置不影响已备份到 COS 的数据（需到 COS 删桶）。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `InvalidParameter` | 桶名不符合 `tke-backup` 前缀约束 | 改用合规桶名 |
| `State=Unavailable` | 仓库当前不可用 | 检查 `Message`，核对桶、TKE 服务角色授权和仓库配置 |
| `Unknown options: --ClusterId` | 误传 `--ClusterId` | 该接口不绑集群，用 `Names[]` 查询 |
| 按名查询返回空 | `Names[]` 大小写/拼写错 | 核对 `Name` 与创建时一致 |

## 收尾确认

<!-- kubectl检查TKE备份组件，tccli无K8s工作负载管理能力，非tccli边界 -->
```bash
# 核对 TKE 备份组件对象
kubectl get deployment,service -n tke-backup tke-backup
# expected: Deployment 与 Service 均存在；Deployment AVAILABLE 为 1 或更大

# 核对存储位置状态
tccli tke DescribeBackupStorageLocations --region <REGION> \
  --Names '["<LOCATION_NAME>"]' \
  --filter "BackupStorageLocationSet[0].{name:Name,state:State,bucket:Bucket}" --output text
# expected: state=Available
```

> 存储位置为 `Available` 且 `tke-backup` Deployment 可用后，在 TKE 控制台“运维中心 > 备份中心”继续创建备份或恢复。**能力边界**：TCCLI 仅管理备份存储位置 CRUD，不直接执行备份或恢复。

## 下一步

- 集群创建/删除（备份对象）：[创建集群](create.md)
- 集群升级（升级前备份）：[升级集群版本](upgrade.md)
- 集群状态查询：[查询集群](query.md)

## 精确 Action 字段契约

| 字段 | 所属 Action | 必填 | 说明 |
|:---|:---|:---:|:---|
| `Bucket` | `CreateBackupStorageLocation` | 是 | 备份存储桶 |
