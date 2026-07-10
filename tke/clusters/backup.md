---
doc_type: How-to
subtype: 6A
fused: false
---
# 集群备份存储位置

> 配置 TKE 集群备份的存储位置（基于 Velero + COS），用于集群灾备与迁移。
> 控制台: [容器服务 - 备份 - 存储仓库](https://console.cloud.tencent.com/tke2/backup)

> 本文档 Action 属 **TKE 2018-05-25**（旧版独有，新版无）。备份存储位置是全局命名资源，不绑定集群——`--ClusterId` 不是其入参。

## 概述

TKE 集群备份（基于 Velero）将集群资源（Deployment/Service/ConfigMap 等）备份到对象存储（COS）。备份存储位置（BackupStorageLocation）是备份的目标 COS 桶配置——配好位置后，集群备份会写入该桶。

> ⚠️ **产品边界（灾备生命周期三段，tcli 只覆盖第一段）**：
> - **配置存储位置** → tcli `CreateBackupStorageLocation`（本文档）
> - **执行一次备份** → **Velero 插件层**（`kubectl` 调 Velero CRD，非 tcli）
> - **从备份恢复** → **Velero 插件层**（`kubectl` 调 Velero CRD，非 tcli）
>
> ⚠️ **恢复边界**：当前**仅支持 Kubernetes 资源对象**的备份与恢复；**不支持**云硬盘 CBS、负载均衡 CLB 等云资源的恢复。跨集群恢复要求备份组件 ≥1.1.0。
>
> tke API 无 `ExecuteBackup`/`Restore` Action——备份/恢复执行由 Velero（集群内插件）承担。本文覆盖 tcli 能做的存储位置 CRUD，并指路 Velero kubectl 命令完成灾备闭环（见 [执行备份与恢复](#执行备份与恢复velero-边界) 段）。

> ⚠️ 备份存储位置是**全局命名资源**，不绑定集群（`--ClusterId` 不是其入参）。一个位置可被多个集群共用。多地域 TKE 集群备份到同一仓库无需重复创建。单账号最多创建 **100 个**备份仓库，超出需清理闲置仓库。

## 触发条件

- 生产/重要集群需灾备，但还没配备份存储位置（`DescribeBackupStorageLocations` 为空）— 用本文创建位置 + Velero 执行备份
- 需跨地域迁移集群（备份后在新地域恢复）— 先用本文配存储位置
- 已有位置需调整 COS 桶或路径 — 删旧位置重建（位置不可改名，覆盖式修改需先删后建）

## 准备工作

- 已创建 TKE 集群 + 已开通 COS 服务 (备份存 COS)
- 已配置 tccli 凭证 (见 [配置凭证](../../getting-started/credentials.md))


## 决策依据

### COS 桶选型

| 项 | 要求 | 说明 |
|:---|:-----|:-----|
| 桶名前缀 | **必须以 `tke-backup` 开头** | 非此前缀报 `InvalidParameter: The bucket prefix must be tke-backup` |
| `Provider` | `tencentcloud` | 枚举值（非 `qcloud`） |
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
| `Bucket` | Create | 是 | COS 桶名（须 `tke-backup` 前缀） |
| `Provider` | Create | 否 | `tencentcloud`（默认 `tencentcloud`，可省略） |
| `Path` | Create | 否 | 桶内路径 |
| `Names[]` | Describe | 否 | 按名查询（空则查全部） |

> `DescribeBackupStorageLocations` 用 `Names[]`，不接受 `--ClusterId`。

## 操作步骤

### 步骤 1：创建合规 COS 桶

在 COS 服务创建桶名以 `tke-backup` 开头的桶（如 `tke-backup-myteam-1250000000`，`1250000000` 为账号 APPID）。

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
# expected: exit 0, BackupStorageLocationSet 含新建位置, State=Available
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

> `State` 状态机：创建后 `Available`（桶可达且权限正常）/ `Unavailable`（桶不存在或无权限，查 `Message`）。**创建初期 `State` 可能为空**（异步校验未完成），稍后重查才有 `Available`/`Unavailable` 值——勿据空 `State` 判断创建失败。

## 验证

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 位置已创建 | `DescribeBackupStorageLocations` | `BackupStorageLocationSet` 含该 Name |
| 位置可用 | 同上 | `State=Available` |
| 桶可达 | `State=Available` 且 `Message` 空 | 桶存在且有权限 |
| 按名查询 | `DescribeBackupStorageLocations --Names '["<LOCATION_NAME>"]'` | 返回该位置 |

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGION>` | 调用地域 | 如 `ap-guangzhou` | `tccli cvm DescribeRegions` |
| `<STORAGE_REGION>` | COS 桶地域 | 与桶一致 | COS 控制台或 `coscli ls`（COS 独立工具） |
| `<BUCKET_NAME>` | COS 桶名 | **`tke-backup` 前缀** | COS 控制台或 `coscli ls`（TCCLI 无 cos 服务） |
| `<LOCATION_NAME>` | 存储位置名 | 全局唯一 | 自定义 |

## 执行备份与恢复（Velero 边界）

> 以下命令用 **Velero CLI**（非 tcli、非 kubectl 内置）——tke API 不提供执行备份/恢复的 Action，灾备执行由 Velero 集群内插件承担。前置：集群已装 Velero 插件（见 [插件管理](../addons/manage.md)）、本地装 Velero CLI（`velero` 命令，见 [Velero 安装文档](https://velero.io/docs/main/install-overview/)）、且配好本篇的存储位置。
>
> ⚠️ `kubectl create backup/restore` **不是有效命令**——`kubectl create` 仅支持内置资源，Velero 备份/恢复须用 `velero` CLI 或提交 Backup/Restore CRD YAML。

### 前置：确认 Velero 已装

```bash
kubectl get pods -n velero-system
# expected: velero pod Running（若不存在，先在 TKE 控制台或 addons 装 Velero 插件）
```

```bash
velero version --client-only
# expected: Velero CLI 版本号（若 command not found，装 Velero CLI）
```

### 执行一次备份

```bash
# 备份 default 命名空间所有资源到已配的存储位置
velero backup create <BACKUP_NAME> --include-namespaces default
# expected: Backup request submitted successfully
```

```bash
# 查看备份状态（BackupPhase 由 New → InProgress → Completed）
velero backup get <BACKUP_NAME>
# expected: STATUS=Completed
```

### 从备份恢复

> ⚠️ 恢复只会还原 K8s 对象（Deployment/Service/ConfigMap 等）。**PVC 背后的 CBS、Service/Ingress 关联的 CLB 不会随备份恢复**——须另行重建云资源或改 YAML 绑定已有资源。

```bash
# 从指定备份恢复（恢复到原命名空间）
velero restore create --from-backup <BACKUP_NAME>
# expected: Restore request submitted successfully

velero restore get
# expected: <RESTORE_NAME> STATUS=Completed
```

> Velero 是 K8s 原生灾备工具，命令以 Velero CRD（`kubectl get backup/restore`）操作，非 tcli。完整 Velero 文档见 [Velero 官方](https://velero.io/docs/)。tcli 仅管存储位置 CRUD。

## 清理

```bash
tccli tke DeleteBackupStorageLocation --region <REGION> --Name <LOCATION_NAME>
# expected: exit 0
```

> `DeleteBackupStorageLocation` 仅需 `Name`。删除位置不影响已备份到 COS 的数据（需到 COS 删桶）。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `InvalidParameter: The bucket prefix must be tke-backup` | 桶名非 `tke-backup` 前缀 | 改用合规桶名 |
| `State=Unavailable` | 桶不存在 / 无权限 | 检查 `Message`，确认桶存在且 TKE 有访问权限 |
| `Unknown options: --ClusterId` | 误传 `--ClusterId` | 该接口不绑集群，用 `Names[]` 查询 |
| `Provider` 无效 | 用了 `qcloud` 等错误值 | 改用 `tencentcloud` |
| 按名查询返回空 | `Names[]` 大小写/拼写错 | 核对 `Name` 与创建时一致 |

> 错误样本：桶名非前缀 → `code:InvalidParameter message:The bucket prefix must be tke-backup`。

## 收尾确认

```bash
# 衔接下一步前置：Velero 插件就绪（backup.md 声明的备份执行前置，Verify 查 State=Available 但没查 Velero 插件能否执行备份）
kubectl get pods -n velero-system
# expected: velero pod Running（Velero 插件就绪，可执行 velero backup create）

# 存储位置可用（步骤 3 Verify 已核 State=Available，此处核对接通性）
tccli tke DescribeBackupStorageLocations --region <REGION> \
  --Names '["<LOCATION_NAME>"]' \
  --filter "BackupStorageLocationSet[0].{name:Name,state:State,bucket:Bucket}" --output text
# expected: state=Available → 存储位置可达
```

> 存储位置 `State=Available` + Velero 插件 pod `Running` = 备份前置就绪，可衔接 [执行备份与恢复（Velero 边界）](#执行备份与恢复velero-边界) 段。**业务可用性边界**：tcli 仅管存储位置 CRUD，执行备份须 Velero 插件就绪（见 [§概述](#概述) 产品边界警告）；Velero 未装时存储位置配好也无法备份。

## 下一步

- 集群创建/删除（备份对象）：[创建集群](../clusters/create.md)
- 集群升级（升级前备份）：[升级集群版本](../clusters/upgrade.md)
- 集群状态查询：[查询集群](../clusters/query.md)
