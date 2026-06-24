# 快照管理

> 对照官方：[快照管理](https://cloud.tencent.com/document/product/457/58182) · page_id `58182`

## 概述

云原生 etcd 支持自动定期备份和手动立即备份。备份结果以快照形式存储在 COS（对象存储）桶中，可用于恢复集群到历史状态。创建 etcd 集群时自动创建 COS 桶并开启默认备份策略。

快照存储路径：`tencentcloud-tke-etcd-backups-REGION-APPID/INSTANCE_ID/`

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已有运行中的 etcd 实例

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要 cetcd:CreateEtcdSnapshot, cetcd:DescribeEtcdSnapshots, cetcd:CreateEtcdSnapshotPolicy
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0

# 4. 确认目标实例运行中
tccli cetcd DescribeEtcdInstance --region <Region> --InstanceId INSTANCE_ID | jq '.Status'
# expected: "Running"
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看快照列表 | `DescribeEtcdSnapshots` | 是 |
| 查看快照策略 | `DescribeEtcdSnapshotPolicy` | 是 |
| 创建快照策略 | `CreateEtcdSnapshotPolicy` | 否 |
| 修改快照策略 | `ModifyEtcdSnapshotPolicy` | 否 |
| 关闭快照策略 | `DisableEtcdSnapshotPolicy` | 否 |
| 手动创建快照 | `CreateEtcdSnapshot` | 否 |
| 从快照恢复 | `RestoreEtcdInstance` | 否 |
| 删除快照 | `DeleteEtcdSnapshot` | 否 |
| 导入快照文件 | `ImportEtcdSnapshot` | 否 |

## 关键字段说明

以下说明快照管理相关的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `InstanceId` | String | 是 | 目标 etcd 实例 ID | 不存在 → `InvalidParameter` |
| `SnapshotName` | String | 创建快照时必填 | 快照名称 | - |
| `BackupSettings.Interval` | Integer | 策略配置时必填 | 备份间隔（小时），如 6 | - |
| `BackupSettings.MaxCount` | Integer | 策略配置时必填 | 最大备份个数。上限 1000。超出后新快照无法上传至 COS | 超出 1000 → `InvalidParameter` |

## 操作步骤

### 步骤 1：查看当前快照策略

```bash
tccli cetcd DescribeEtcdSnapshotPolicy --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回备份间隔、最大保留数等
```

### 步骤 2：配置/修改快照策略

#### 选择依据

- **备份间隔**：生产环境建议 6 小时一次。间隔越短，数据丢失窗口越小，但快照存储成本越高。
- **最大备份个数**：建议 100-200。COS 按实际用量计费，保留过多快照会增加存储成本。

`snapshot-policy.json`：

```json
{
    "InstanceId": "INSTANCE_ID",
    "BackupSettings": {
        "Interval": 6,
        "MaxCount": 100
    }
}
```

```bash
tccli cetcd CreateEtcdSnapshotPolicy --region <Region> \
    --cli-input-json file://snapshot-policy.json
# expected: exit 0
```

### 步骤 3：手动创建快照

```bash
tccli cetcd CreateEtcdSnapshot --region <Region> \
    --InstanceId INSTANCE_ID \
    --SnapshotName SNAPSHOT_NAME
# expected: exit 0，返回 SnapshotId
```

**预期输出**：

```json
{
    "SnapshotId": "snap-example",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 4：查看快照列表

```bash
tccli cetcd DescribeEtcdSnapshots --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回快照列表
```

**预期输出**：

```json
{
    "TotalCount": 3,
    "SnapshotSet": [
        {
            "SnapshotId": "snap-example-1",
            "SnapshotName": "manual-backup-20240617",
            "Status": "Success",
            "CreateTime": "2024-06-17T10:00:00Z",
            "Size": 1048576
        }
    ]
}
```

### 步骤 5：从快照恢复集群

> **警告**：恢复过程会用快照数据覆盖集群现有数据，恢复期间集群暂时不可用。建议先执行手动备份保存当前状态。

```bash
tccli cetcd RestoreEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID \
    --SnapshotId SNAPSHOT_ID
# expected: exit 0
```

### 步骤 6：导入外部快照文件

> **警告**：快照文件不能通过直接复制 etcd 服务的 db 文件生成——写入过程中的直接复制会损坏数据。必须使用 `etcdctl snapshot save` 生成快照，并用 `etcdctl snapshot status` 验证完整性。

```bash
# 先在源集群生成快照
etcdctl --endpoints=SOURCE_ENDPOINT \
    --cacert=/path/to/ca.crt \
    --cert=/path/to/client.crt \
    --key=/path/to/client.key \
    snapshot save source-etcd.db

# 验证快照完整性
etcdctl snapshot status source-etcd.db --write-out=table
# expected: 显示 hash、revision、totalKey、totalSize

# 导入到腾讯云 etcd
tccli cetcd ImportEtcdSnapshot --region <Region> \
    --InstanceId INSTANCE_ID \
    --SnapshotName IMPORTED_SNAPSHOT_NAME \
    --SnapshotFile file://source-etcd.db
# expected: exit 0
```

### 步骤 7：删除快照

```bash
tccli cetcd DeleteEtcdSnapshot --region <Region> \
    --InstanceId INSTANCE_ID \
    --SnapshotId SNAPSHOT_ID
# expected: exit 0
```

> **警告**：删除操作不可逆，快照会同时从列表和 COS 桶中移除。

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `INSTANCE_ID` | etcd 实例 ID | 格式 `etcd-xxxxxxxx` | `tccli cetcd DescribeEtcdInstances` |
| `SNAPSHOT_NAME` | 快照名称 | 自定义 | 用户定义 |
| `SNAPSHOT_ID` | 快照 ID | 格式 `snap-xxxxxxxx` | `tccli cetcd DescribeEtcdSnapshots` |

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| 快照策略生效 | `DescribeEtcdSnapshotPolicy` | Interval 和 MaxCount 与配置一致 |
| 快照成功 | `DescribeEtcdSnapshots` | 状态为 `Success` |
| 恢复完成 | `DescribeEtcdInstance` | `Status: "Running"` |

```bash
# 确认快照策略
tccli cetcd DescribeEtcdSnapshotPolicy --region <Region> \
    --InstanceId INSTANCE_ID
# expected: Interval=6, MaxCount=100

# 确认最新快照状态
tccli cetcd DescribeEtcdSnapshots --region <Region> \
    --InstanceId INSTANCE_ID \
    | jq '.SnapshotSet[0].Status'
# expected: "Success"
```

## 清理

- 关闭快照策略（不删除已有快照）：`DisableEtcdSnapshotPolicy`
- 删除单个快照：`DeleteEtcdSnapshot`（不可逆）
- 关闭快照策略不会删除已创建的快照，需手动逐一删除

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEtcdSnapshot` 返回 `FailedOperation` | 检查实例状态和快照并发数 | 实例状态异常或已有快照正在生成 | 等待实例 Running 且当前无进行中的快照操作 |
| `RestoreEtcdInstance` 返回 `InvalidParameter` | 检查 SnapshotId 是否存在 | 快照已被删除或 ID 错误 | 用 `DescribeEtcdSnapshots` 确认快照存在 |
| `ImportEtcdSnapshot` 返回 `InvalidParameter` | 用 `etcdctl snapshot status FILE` 检查文件 | 快照文件损坏或格式不符 | 重新生成快照文件，确保用 `etcdctl snapshot save` 生成且 `etcdctl snapshot status` 通过 |
| `CreateEtcdSnapshotPolicy` 的 `MaxCount` 超过 1000 | 检查 JSON 中的值 | MaxCount 上限 1000 | 改为 ≤ 1000 的值 |

### 快照异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 快照列表为空但策略已配置 | `DescribeEtcdSnapshotPolicy` 确认策略状态 | 策略可能已关闭或首个快照尚未生成 | 等待下一个备份周期，或执行 `CreateEtcdSnapshot` 手动创建 |
| 快照状态为 `Failed` | 查看快照详情中的失败原因 | etcd 密码变更导致快照失败（常见） | 参见 [快照失败](../集群排障/快照失败/tccli%20操作.md) 排障 |

## 下一步

- [快照失败](../集群排障/快照失败/tccli%20操作.md) — 快照创建失败排障
- [数据同步](../数据同步/tccli%20操作.md) — 通过快照同步数据
- [删除集群](../删除集群/tccli%20操作.md) — 删除集群前执行最终快照
- [COS 对象存储控制台](https://console.cloud.tencent.com/cos) — 查看快照文件在 COS 中的存储

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，点击实例进入 快照管理 页面。
