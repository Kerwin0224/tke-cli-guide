---
doc_type: How-to
subtype: 6A
fused: false
---
# 集群备份存储位置

> 配置 TKE 集群备份的存储位置（基于 Velero + COS），用于集群灾备与迁移。
> 控制台: [容器服务 - 备份 - 存储仓库](https://console.cloud.tencent.com/tke2/backup)

## 概述

TKE 集群备份（基于 Velero）将集群资源（Deployment/Service/ConfigMap 等）备份到对象存储（COS）。备份存储位置（BackupStorageLocation）是备份的目标 COS 桶配置——配好位置后，集群备份会写入该桶。

> ⚠️ 备份存储位置是**全局命名资源**，不绑定集群（`--ClusterId` 不是其入参）。一个位置可被多个集群共用。

## 决策依据

### COS 桶选型

| 项 | 要求 | 说明 |
|:---|:-----|:-----|
| 桶名前缀 | **必须以 `tke-backup` 开头** | 实测：非此前缀报 `InvalidParameter: The bucket prefix must be tke-backup` |
| `Provider` | `tencentcloud` | 实测枚举值（非 `qcloud`） |
| `StorageRegion` | COS 桶所在地域 | 如 `ap-guangzhou` |
| `Path` | 桶内路径 | 如 `/backup` |

> ⚠️ **桶名前缀是硬约束（实测）**：COS 桶名必须以 `tke-backup` 开头。先在 COS 服务创建合规桶，再配存储位置。

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
| `Provider` | Create | 是 | `tencentcloud` |
| `Path` | Create | 否 | 桶内路径 |
| `Names[]` | Describe | 否 | 按名查询（空则查全部） |

> 参数名实测自各 Action `--generate-cli-skeleton`（P7）。`DescribeBackupStorageLocations` 用 `Names[]`，不接受 `--ClusterId`。

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

> `State` 状态机：创建后 `Available`（桶可达且权限正常）/ `Unavailable`（桶不存在或无权限，查 `Message`）。

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
| `<STORAGE_REGION>` | COS 桶地域 | 与桶一致 | `coscli ls（COS 独立工具）或 COS 控制台 |
| `<BUCKET_NAME>` | COS 桶名 | **`tke-backup` 前缀** | `coscli ls 或 COS 控制台（tccli 无 cos 服务） |
| `<LOCATION_NAME>` | 存储位置名 | 全局唯一 | 自定义 |

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

> 实测错误样本：桶名非前缀 → `code:InvalidParameter message:The bucket prefix must be tke-backup`。

## 下一步

- 集群创建/删除（备份对象）：[创建集群](../clusters/create.md)
- 集群升级（升级前备份）：[升级集群版本](../clusters/upgrade.md)
- 集群状态查询：[查询集群](../clusters/query.md)

## Action 清单

| Action | 类型 | 跨产品 | 说明 |
|:-------|:-----|:------:|:-----|
| `CreateBackupStorageLocation` | 主操作 | cos | 创建存储位置（Bucket 须 tke-backup 前缀） |
| `DescribeBackupStorageLocations` | 验证 | — | 查询位置（Names[]，不绑集群） |
| `DeleteBackupStorageLocation` | 清理 | — | 删除位置（仅 Name） |
