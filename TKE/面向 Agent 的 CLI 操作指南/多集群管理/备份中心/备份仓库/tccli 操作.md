# 备份仓库（tccli）

> 对照官方：[备份仓库](https://cloud.tencent.com/document/product/457/90491) · page_id `90491`

## 概述

备份仓库（BackupStorageLocation）是备份数据的存储位置，对应腾讯云 COS 存储桶。创建备份前必须先创建备份仓库。通过 `tccli tke` 的 `BackupStorageLocation` 系列 API 完成仓库的创建、查看和删除。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:CreateBackupStorageLocation
#    tke:DescribeBackupStorageLocations
#    tke:DeleteBackupStorageLocation
# 验证：
tccli tke DescribeBackupStorageLocations --region <Region>
# expected: exit 0，返回仓库列表（可为空）

# 4. 检查 COS 服务可用
tccli cos ListBuckets --region <Region>
# expected: exit 0，返回桶列表
```

```json
{
  "BackupStorageLocationSet": "<BackupStorageLocationSet>",
  "Name": "<Name>",
  "StorageRegion": "<StorageRegion>",
  "Provider": "<Provider>",
  "Bucket": "<Bucket>",
  "Path": "<Path>"
}
```

> **说明**：示例中 `REGION` 替换为 `ap-guangzhou`，`CLUSTER_ID` 替换为 `cls-xxxxxxxx`。

### 资源检查

```bash
# 5. 确认目标 COS 桶存在（桶名必须以 tke-backup 为前缀）
tccli cos HeadBucket --region <Region> --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID'
# expected: exit 0，桶存在
```

## 关键字段说明

以下说明 `CreateBackupStorageLocation` 的主要参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `StorageRegion` | String | 是 | COS 存储桶所在地域，如 `ap-guangzhou`。需与 COS 桶地域一致 | 地域不匹配 → 仓库 State 非 Available |
| `Bucket` | String | 是 | COS 桶完整名称，格式 `BUCKET_NAME-APPID`。桶名必须以 `tke-backup` 为前缀 | 桶名不以 `tke-backup` 开头 → `InvalidParameter` |
| `Name` | String | 是 | 备份仓库名称，集群内唯一。长度 1-63 字符 | 重名 → 创建失败 |
| `Provider` | String | 否 | 存储提供商，默认 `COS`。当前仅支持 COS | 填其他值无效果 |
| `Path` | String | 否 | COS 桶内子路径。以 `/` 开头，如 `/backups/cluster-1` | 路径不存在会自动创建 |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建备份仓库 | `tccli tke CreateBackupStorageLocation --region <Region> --StorageRegion REGION --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID' --Name LOCATION_NAME` | 否 |
| 查看仓库列表 | `tccli tke DescribeBackupStorageLocations --region <Region>` | 是 |
| 删除仓库 | `tccli tke DeleteBackupStorageLocation --region <Region> --Name LOCATION_NAME` | 是 |
| 查看仓库详情（kubectl） | `kubectl get backupstoragelocations -n velero` | 是 |

## 操作步骤

### 1. 创建 COS 存储桶

备份仓库依赖 COS 存储桶，桶名必须以 `tke-backup` 为前缀。

```bash
tccli cos PutBucket --region <Region> \
    --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID'
# expected: exit 0
```

### 2. 创建备份仓库

```bash
tccli tke CreateBackupStorageLocation --region <Region> \
    --StorageRegion REGION \
    --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID' \
    --Name LOCATION_NAME
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "b3f35196-1fc5-4c26-8025-0f74378c054e"
}
```

### 3. 查看备份仓库列表

```bash
tccli tke DescribeBackupStorageLocations --region <Region>
# expected: BackupStorageLocationSet 不为空
```

预期输出：

```json
{
    "BackupStorageLocationSet": [
        {
            "Name": "LOCATION_NAME",
            "Bucket": "tke-backup-example-1234567890",
            "StorageRegion": "ap-guangzhou",
            "State": "Available",
            "Provider": "COS"
        }
    ],
    "RequestId": "..."
}
```

按名称过滤：

```bash
tccli tke DescribeBackupStorageLocations --region <Region> \
    --Names '["LOCATION_NAME"]'
# expected: 仅返回目标仓库
```

```json
{
  "BackupStorageLocationSet": "<BackupStorageLocationSet>",
  "Name": "<Name>",
  "StorageRegion": "<StorageRegion>",
  "Provider": "<Provider>",
  "Bucket": "<Bucket>",
  "Path": "<Path>"
}
```

### 4. 通过 kubectl 查看 BSL（可选）

> **前提**：需 Velero 在集群内安装，kubectl 端点可达。

```bash
kubectl get backupstoragelocations -n velero
# expected: PHASE 为 Available
```

预期输出：

```text
NAME               PHASE       LAST VALIDATED   AGE
LOCATION_NAME      Available   10s              2m
```

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。

## 验证

```bash
# 验证备份仓库状态为 Available
tccli tke DescribeBackupStorageLocations --region <Region> \
    --Names '["LOCATION_NAME"]'
# expected: State: "Available"
```

```json
{
  "BackupStorageLocationSet": "<BackupStorageLocationSet>",
  "Name": "<Name>",
  "StorageRegion": "<StorageRegion>",
  "Provider": "<Provider>",
  "Bucket": "<Bucket>",
  "Path": "<Path>"
}
```

## 清理

> **警告**：删除备份仓库不会删除 COS 中的备份文件。如需彻底清理，请先清空 COS 桶内容。

```bash
# 1. 确认目标仓库
tccli tke DescribeBackupStorageLocations --region <Region> \
    --Names '["LOCATION_NAME"]'
# expected: 记录 Name 和 Bucket

# 2. 删除备份仓库
tccli tke DeleteBackupStorageLocation --region <Region> \
    --Name LOCATION_NAME
# expected: exit 0，返回 RequestId

# 3. 验证已删除
tccli tke DescribeBackupStorageLocations --region <Region> \
    --Names '["LOCATION_NAME"]'
# expected: BackupStorageLocationSet 返回空列表
```

预期输出（验证已删除）：

```json
{
    "BackupStorageLocationSet": [],
    "RequestId": "..."
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateBackupStorageLocation` 返回 `InvalidParameter` "bucket prefix must be tke-backup" | 检查 `--Bucket` 参数值 | COS 桶名格式不满足 `tke-backup` 前缀硬约束 | `tccli cos PutBucket --region <Region> --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID'` |
| `CreateBackupStorageLocation` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查凭据 | 缺少 `tke:CreateBackupStorageLocation` 权限 | 联系管理员授予 TKE 备份仓库管理权限 |
| COS 桶不存在 | `tccli cos HeadBucket --region <Region> --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID'` | 未创建 COS 桶或桶名拼写错误 | `tccli cos PutBucket --region <Region> --Bucket 'tke-backup-BUCKET_SUPPLIER-APPID'` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeBackupStorageLocations` 返回空列表 | 不带 `--Names` 查询全部仓库 | 仓库可能已被删除或 `--Names` 过滤参数拼写错误 | `tccli tke DescribeBackupStorageLocations --region <Region>` 不带过滤参数确认 |
| 仓库 State 非 Available | `tccli tke DescribeBackupStorageLocations --region <Region> --Names '["LOCATION_NAME"]'` 查看 State | COS 桶不存在或 CAM 权限不足 | 确认 COS 桶存在、地域匹配、CAM 策略允许访问 |

## 下一步

- [备份管理](../备份管理/tccli%20操作.md) — 创建备份和定时备份策略
- [恢复管理](../恢复管理/tccli%20操作.md) — 从备份恢复集群资源

## 控制台替代

控制台：集群 → 备份中心 → 备份仓库 → 新建。
