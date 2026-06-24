# 计划升级概述

> 对照官方：[计划升级概述](https://cloud.tencent.com/document/product/457/124477) · page_id `124477`

## 概述

TKE 计划升级功能允许您在指定的维护时间窗口内自动升级集群版本，避免业务高峰期发生意外中断。通过 `tccli tke` 的 `MaintenanceWindow` 和 `RollOutSequence` 系列 API 完成维护窗口配置和集群编排。

**核心概念**：

| 概念 | 说明 | 对应 API |
|------|------|---------|
| 维护窗口（MaintenanceWindow） | 定义允许升级的时间段（周几、起始时间、持续时长） | `CreateClusterMaintenanceWindowAndExclusions` |
| 排除项（Exclusions） | 在维护窗口内排除的时间段，避免特定日期升级 | `CreateClusterMaintenanceWindowAndExclusions --Exclusions` |
| 集群编排（RollOutSequence） | 定义多个集群的升级顺序和策略 | `CreateRollOutSequence` |

**全局维护窗口 vs 集群维护窗口**：集群级维护窗口（`CreateClusterMaintenanceWindowAndExclusions`）为单个集群设置；全局维护窗口（`CreateGlobalMaintenanceWindowAndExclusions`）为所有集群设置默认策略。全局窗口 API 需要 CentrallyManage 级权限，普通账号可能无权限。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:CreateClusterMaintenanceWindowAndExclusions
#    tke:DescribeClusterMaintenanceWindowAndExclusions
#    tke:DeleteClusterMaintenanceWindowAndExclusion
#    tke:CreateRollOutSequence
#    tke:DescribeRollOutSequences
#    tke:UpdateClusterVersion
# 验证：
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region>
# expected: exit 0，返回窗口列表（可为空）

> **说明**：示例中 `REGION` 替换为 `ap-guangzhou`，`CLUSTER_ID` 替换为 `cls-xxxxxxxx`。

预期输出（尚未配置维护窗口）：

```json
{
    "MaintenanceWindowAndExclusions": [],
    "TotalCount": 0,
    "RequestId": "..."
}
```

### 资源检查

```bash
# 4. 确认目标集群存在
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"

预期输出：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-test-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterType": "MANAGED_CLUSTER"
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

# 5. 查询可用 K8s 版本
tccli tke DescribeAvailableClusterVersion --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 列出可升级的版本列表

预期输出：

```json
{
    "Versions": ["1.30.0", "1.31.0", "1.32.0"],
    "RequestId": "..."
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建维护窗口 | `tccli tke CreateClusterMaintenanceWindowAndExclusions --region <Region> --ClusterID CLUSTER_ID --MaintenanceTime 04:00:00 --Duration 4 --DayOfWeek '["SA"]'` | 否 |
| 查看维护窗口 | `tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region>` | 是 |
| 删除维护窗口 | `tccli tke DeleteClusterMaintenanceWindowAndExclusion --region <Region> --ClusterID CLUSTER_ID` | 是 |
| 创建集群编排 | `tccli tke CreateRollOutSequence --region <Region> --Name SEQUENCE_NAME --SequenceFlows SEQUENCE_FLOWS --Enabled true` | 否 |
| 查看集群编排 | `tccli tke DescribeRollOutSequences --region <Region>` | 是 |

## 操作步骤

### 概念 1：维护窗口

维护窗口定义了集群允许升级的时间段。每个集群最多一个维护窗口，包含以下要素：

- **DayOfWeek**：允许升级的星期几，格式为 `MO`/`TU`/`WE`/`TH`/`FR`/`SA`/`SU`（严格双字母大写）
- **MaintenanceTime**：当天起始时间，格式 `HH:MM:SS`
- **Duration**：持续时长，单位**小时**（整数，≤ 24）

### 概念 2：排除项

排除项（Exclusions）在维护窗口内排除特定时间段，用于跳过节假日、大促等不宜升级的日期。

### 概念 3：集群编排

集群编排定义多个集群的批量升级策略和顺序，按 SequenceFlow 依次升级。

## 验证

```bash
# 验证维护窗口已创建
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> \
    --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'
# expected: TotalCount >= 1
```

```json
{
  "MaintenanceWindowAndExclusions": "<MaintenanceWindowAndExclusions>",
  "MaintenanceTime": "<MaintenanceTime>",
  "Duration": "<Duration>",
  "ClusterID": "<ClusterID>",
  "DayOfWeek": "<DayOfWeek>",
  "Region": "<Region>"
}
```

## 清理

```bash
# 删除集群维护窗口
tccli tke DeleteClusterMaintenanceWindowAndExclusion --region <Region> \
    --ClusterID CLUSTER_ID
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DayOfWeek` 参数报错 "invalid day of week" | 检查 DayOfWeek 值格式 | 必须为 `MO`/`TU`/`WE`/`TH`/`FR`/`SA`/`SU` 严格双字母大写，不可用 `Monday`/`Saturday` 或数字 `1`/`6` | 使用 `SA` 等标准双字母格式 |
| `Duration` 参数报错 "must not exceed 24 hours" | 检查 Duration 值和单位 | Duration 单位是**小时**（不是分钟），值需 ≤ 24 | 将分钟数换算为小时，如 120 分钟 → 2 小时 |
| `CreateGlobalMaintenanceWindowAndExclusions` 返回 `UnauthorizedOperation.CamNoAuth` | 检查 CAM 策略 | 全局窗口 API 需要 CentrallyManage 级权限（此为环境限制，非命令错误） | 改用集群级 `CreateClusterMaintenanceWindowAndExclusions` |
| 维护窗口已存在时重复创建 | `tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'` | 每集群最多一个维护窗口 | 先 `DeleteClusterMaintenanceWindowAndExclusion` 再重建 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 维护窗口存在但升级未触发 | `tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region>` 检查窗口配置 | 当前时间不在维护窗口内，或集群未有待升级版本 | 在窗口内手动触发 `UpdateClusterVersion`，或调整窗口时间覆盖当前时段 |

## 下一步

- [维护窗口和排除项](../维护窗口和排除项/tccli%20操作.md) — API 操作详解与参数排障
- [集群编排](../集群编排/tccli%20操作.md) — 定义批量升级策略
- [计划升级流程](../计划升级流程/tccli%20操作.md) — 端到端升级步骤

## 控制台替代

控制台：集群 → 运维管理 → 计划升级 → 维护窗口。
