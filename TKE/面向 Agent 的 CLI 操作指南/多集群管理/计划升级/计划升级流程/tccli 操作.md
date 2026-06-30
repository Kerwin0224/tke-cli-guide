# 计划升级流程

> 对照官方：[计划升级流程](https://cloud.tencent.com/document/product/457/124480) · page_id `124480`

## 概述

端到端计划升级流程：设置维护窗口、创建集群升级编排、执行集群版本升级。通过 CLI 完成从维护窗口配置到升级触发的全链路操作。

**流程概览**：

```
1. 设置维护窗口 (tccli)
   CreateClusterMaintenanceWindowAndExclusions
2. 创建升级编排 (tccli)
   CreateRollOutSequence
3. 触发集群升级 (tccli)
   UpdateClusterVersion（在维护窗口内执行）
4. 监控升级进度 (tccli)
   DescribeClusterStatus
```

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
#    tke:DescribeClusterStatus
# 验证：
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region>
# expected: exit 0

> **说明**：示例中 `REGION` 替换为 `ap-guangzhou`，`CLUSTER_ID` 替换为 `cls-xxxxxxxx`。

tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，返回集群信息

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

### 资源检查

```bash
# 4. 查询可用目标版本
tccli tke DescribeAvailableClusterVersion --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 列出可升级版本列表，含最新稳定版

# 5. 确认当前版本低于目标版本
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterVersion 低于目标版本
```

## 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `DayOfWeek` | Array | 是 | `MO`/`TU`/`WE`/`TH`/`FR`/`SA`/`SU` 严格双字母大写 | 全拼/数字/缩写 → `InvalidParameter` "invalid day of week" |
| `Duration` | Integer | 是 | 单位**小时**，整数，1-24。**非分钟** | 填分钟数（如 120）→ `InvalidParameter` "must not exceed 24 hours" |
| `MaintenanceTime` | String | 是 | 格式 `HH:MM:SS`，24 小时制，两位小时 | `4:00:00` → `InvalidParameter` |
| `DstVersion` | String | 是 | 必须为 `DescribeAvailableClusterVersion` 返回的有效版本号 | 无效版本 → `InvalidParameterValue.Version` |
| `SequenceFlows[].Order` | Integer | 是 | 升级顺序，数字越小越优先 | Order 重复 → 并发升级（可为预期行为） |
| `Enabled` | Boolean | 是 | `true` 启用编排，`false` 停用 | `false` → 编排不自动执行 |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 设置维护窗口 | `tccli tke CreateClusterMaintenanceWindowAndExclusions --region <Region> --ClusterID CLUSTER_ID --MaintenanceTime 04:00:00 --Duration 4 --DayOfWeek '["SA"]'` | 否 |
| 创建升级编排 | `tccli tke CreateRollOutSequence --region <Region> --cli-input-json file://sequence.json` | 否 |
| 触发版本升级 | `tccli tke UpdateClusterVersion --region <Region> --ClusterId CLUSTER_ID --DstVersion TARGET_VERSION` | 否 |
| 查看升级进度 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 取消升级 | `tccli tke CancelClusterRelease --region <Region> --ClusterId CLUSTER_ID` | 否 |

## 操作步骤

### 步骤 1：设置维护窗口

#### 选择依据

- **时间选择**：周六凌晨 4 点（业务低峰期），持续 4 小时保证升级有充足时间。
- **集群级 vs 全局级**：使用集群级 `CreateClusterMaintenanceWindowAndExclusions`，避免全局级 API 的 CentrallyManage 权限限制。

```bash
tccli tke CreateClusterMaintenanceWindowAndExclusions --region <Region> \
    --ClusterID CLUSTER_ID \
    --MaintenanceTime "04:00:00" \
    --Duration 4 \
    --DayOfWeek '["SA"]'
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

验证：

```bash
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> \
    --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'
# expected: TotalCount: 1，MaintenanceTime: "04:00:00"，Duration: 4
```

### 步骤 2：创建集群升级编排

#### 选择依据

- **灰度升级**：测试集群 Order=1 先升级，验证通过后生产集群 Order=2 后升级。
- **启用**：`Enabled: true` 立即生效。

`sequence.json`：

```json
{
    "Name": "canary-upgrade-seq",
    "SequenceFlows": [
        {"ClusterID": "CLUSTER_ID_1", "Order": 1},
        {"ClusterID": "CLUSTER_ID_2", "Order": 2}
    ],
    "Enabled": true
}
```

```bash
tccli tke CreateRollOutSequence --region <Region> \
    --cli-input-json file://sequence.json
# expected: exit 0，返回 ID
```

预期输出：

```json
{
    "ID": 1,
    "RequestId": "..."
}
```

验证：

```bash
tccli tke DescribeRollOutSequences --region <Region>
# expected: Sequences 包含 canary-upgrade-seq，Enabled: true
```

### 步骤 3：触发集群升级

```bash
# 查看可用目标版本
tccli tke DescribeAvailableClusterVersion --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 列出可升级版本列表
```

预期输出：

```json
{
    "Versions": ["1.30.0", "1.31.0", "1.32.0"],
    "RequestId": "..."
}
```

```bash
# 触发升级
tccli tke UpdateClusterVersion --region <Region> \
    --ClusterId CLUSTER_ID \
    --DstVersion TARGET_VERSION
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `TARGET_VERSION` | 目标 K8s 版本 | 必须是 `DescribeAvailableClusterVersion` 返回的有效版本 | `tccli tke DescribeAvailableClusterVersion --region <Region> --ClusterId CLUSTER_ID` |

### 步骤 4：监控升级进度

```bash
# 查看集群状态
tccli tke DescribeClusterStatus --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterState 为 Upgrading 或 Running
```

预期输出：

```json
{
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterState": "Running",
            "ClusterInstanceState": ""
        }
    ],
    "RequestId": "..."
}
```

轮询直至升级完成：

```bash
tccli tke DescribeClusterStatus --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    --waiter '{"expr":"ClusterStatusSet[0].ClusterState","to":"Running","timeout":1800,"interval":60}'
# expected: ClusterState 由 Upgrading 变为 Running，超时 1800s（30 分钟）
```

### 步骤 5：取消升级（如需）

```bash
tccli tke CancelClusterRelease --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0
```

## 验证

```bash
# 验证维护窗口
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> \
    --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'
# expected: TotalCount >= 1

# 验证集群版本已升级
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterVersion 为 TARGET_VERSION

# 验证集群运行状态
tccli tke DescribeClusterStatus --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterState: "Running"
```

## 清理

```bash
# 1. 删除维护窗口
tccli tke DeleteClusterMaintenanceWindowAndExclusion --region <Region> \
    --ClusterID CLUSTER_ID
# expected: exit 0

# 2. 停用编排（DeleteRollOutSequence 未暴露）
tccli tke CreateRollOutSequence --region <Region> \
    --cli-input-json file://sequence-disable.json
# expected: exit 0（sequence-disable.json 设置 Enabled: false）
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DayOfWeek` 参数报错 "invalid day of week" | 检查 DayOfWeek 值格式 | 使用了全拼（`Saturday`）或数字（`6`） | 改用 `'["SA"]'`（详见 [维护窗口排障](../维护窗口和排除项/tccli%20操作.md)） |
| `Duration` 报错 "must not exceed 24 hours" | 检查值和单位 | 填入了分钟数（如 `120`） | 换算为小时（如 `--Duration 2`） |
| `MaintenanceTime` 报错 `InvalidParameter` | 检查时间格式 | 用了单数字段如 `4:00:00` | 使用 `04:00:00`（两位小时） |
| `UpdateClusterVersion` 返回 `InvalidParameterValue.Version` | `tccli tke DescribeAvailableClusterVersion --region <Region> --ClusterId CLUSTER_ID` | 目标版本号不存在或集群不支持 | 从返回的 Versions 列表中选择有效版本 |
| `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查凭据 | 缺少升级相关权限 | 联系管理员授予 `tke:UpdateClusterVersion` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 升级长时间未开始 | `tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'` 检查窗口 | 当前时间不在维护窗口内 | 等待维护窗口到来，或 `tccli tke DeleteClusterMaintenanceWindowAndExclusion --region <Region> --ClusterID CLUSTER_ID` 移除窗口后直接调用 `UpdateClusterVersion` 立即升级 |
| 升级状态长时间 Upgrading | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'` 轮询状态 | 升级过程耗时（正常）或卡住（异常） | 持续轮询；超过 30 分钟则保留 region、ClusterId、RequestId → 登录控制台查看详细状态 → 仍无法解决则提交工单 |

## 下一步

- [维护窗口和排除项](../维护窗口和排除项/tccli%20操作.md) — DayOfWeek 格式详解与排障
- [集群编排](../集群编排/tccli%20操作.md) — 批量升级策略配置

## 控制台替代

控制台：集群 → 运维管理 → 计划升级 → 升级集群。
