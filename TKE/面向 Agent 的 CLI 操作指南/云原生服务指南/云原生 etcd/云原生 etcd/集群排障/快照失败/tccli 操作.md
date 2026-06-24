# 快照失败

> 对照官方：[快照失败](https://cloud.tencent.com/document/product/457/109849) · page_id `109849`

## 概述

etcd 快照策略已开启但快照备份文件未按预期生成。最常见原因是 etcd 密码被修改后快照备份任务无法认证。此问题无法自助修复，需提交工单由腾讯云运维处理。

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要 cetcd:DescribeEtcdSnapshots
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看快照策略 | `DescribeEtcdSnapshotPolicy` | 是 |
| 查看快照列表 | `DescribeEtcdSnapshots` | 是 |
| 查看实例详情 | `DescribeEtcdInstance` | 是 |
| 手动创建快照 | `CreateEtcdSnapshot` | 否 |

## 操作步骤

### 步骤 1：检查快照策略配置

```bash
tccli cetcd DescribeEtcdSnapshotPolicy --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回备份间隔和最大备份个数
```

**预期输出**：

```json
{
    "BackupSettings": {
        "Interval": 6,
        "MaxCount": 100
    }
}
```

### 步骤 2：查看快照列表，确认最后一次成功快照时间

```bash
tccli cetcd DescribeEtcdSnapshots --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回快照列表
```

**预期输出（快照失败场景）**：

```json
{
    "TotalCount": 3,
    "SnapshotSet": [
        {
            "SnapshotId": "snap-example-1",
            "SnapshotName": "auto-backup-20240601",
            "Status": "Success",
            "CreateTime": "2024-06-01T10:00:00Z"
        },
        {
            "SnapshotId": "snap-example-2",
            "SnapshotName": "auto-backup-20240602",
            "Status": "Failed",
            "CreateTime": "2024-06-02T10:00:00Z"
        }
    ]
}
```

> 对比成功快照的时间线与快照策略间隔，确认快照是否在预期时间点停止生成。

### 步骤 3：尝试手动创建快照

```bash
tccli cetcd CreateEtcdSnapshot --region <Region> \
    --InstanceId INSTANCE_ID \
    --SnapshotName manual-verify-SNAPSHOT_DATE
# expected: 如成功则密码不是根因；如失败且近期改过密码，则确认为密码变更所致
```

### 步骤 4：确认根因并处理

检查以下可能原因：

| 检查项 | 排查命令 | 结论 |
|--------|---------|------|
| 是否修改过 etcd 密码 | 无 CLI 命令可查历史，需人工确认 | **是** → 此问题无法自助修复 |
| 快照策略是否已关闭 | `DescribeEtcdSnapshotPolicy` | Interval 为空或 0 → 重新开启策略 |
| COS 存储是否异常 | COS 桶状态 | 如 COS 异常 → 联系 COS 支持 |

## 验证

- `DescribeEtcdSnapshots` 中最新快照状态为 `Success`
- 快照生成时间线恢复正常（与快照策略间隔一致）

## 清理

本页面为排障页，无需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEtcdSnapshots` 返回空列表 | `DescribeEtcdSnapshotPolicy` 检查策略 | 未曾创建过快照策略 | 先执行 `CreateEtcdSnapshotPolicy` 创建策略 |
| `CreateEtcdSnapshot` 返回 `FailedOperation` | 检查 etcd 密码修改记录 | etcd 密码变更后快照任务无法认证（此为平台限制，非命令错误） | 无法自助修复 → [提交工单](https://console.cloud.tencent.com/workorder)，提供 `InstanceId`、最近快照失败的 `SnapshotId`、密码修改时间 |
| `CreateEtcdSnapshot` 返回 `LimitExceeded` | `DescribeEtcdSnapshots` 查看快照总数 | 快照数量已达 MaxCount 上限（此为平台限制） | 删除过期快照释放配额，或增加 MaxCount 上限 |

### 快照失败但命令执行成功

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 快照列表中出现 `Status: "Failed"` 的快照 | 查看快照失败时间与 etcd 密码变更时间是否匹配 | etcd 密码被修改 | 此为环境问题，非 CLI 错误 → [提交工单](https://console.cloud.tencent.com/workorder)，附 InstanceId 和失败 SnapshotId |
| 快照间隔期间没有自动快照生成 | `DescribeEtcdSnapshotPolicy` 确认策略是否开启 | 策略被关闭 | 重新开启快照策略或创建新策略 |
| 已超过 MaxCount 的快照未自动清理 | 检查 COS 桶状态 | COS 桶可能存在访问问题 | 手动删除过期快照释放空间 |

## 下一步

- [快照管理](../../快照管理/tccli%20操作.md) — 重新配置快照策略
- [数据量异常](../../集群排障/数据量异常/tccli%20操作.md) — 排查数据量异常
- [监控和告警配置](../../监控和告警配置/tccli%20操作.md) — 监控集群状态

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，点击实例进入 快照管理 页面查看快照列表和状态。
