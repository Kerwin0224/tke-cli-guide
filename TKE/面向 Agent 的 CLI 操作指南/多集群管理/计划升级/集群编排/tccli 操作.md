# 集群编排（tccli）

> 对照官方：[集群编排](https://cloud.tencent.com/document/product/457/124479) · page_id `124479`

## 概述

集群编排（RollOutSequence）定义多个集群的批量升级顺序和策略。通过 `CreateRollOutSequence` 创建编排，指定集群升级先后顺序、并发策略和启停状态。

> **注意**：当前 `tccli tke` 未暴露 `DeleteRollOutSequence` API。如需删除编排，可通过控制台操作或创建同名编排覆盖。

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
#    tke:CreateRollOutSequence
#    tke:DescribeRollOutSequences
# 验证：
tccli tke DescribeRollOutSequences --region <Region>
# expected: exit 0，返回编排列表（可为空）

> **说明**：示例中 `REGION` 替换为 `ap-guangzhou`，`CLUSTER_ID` 替换为 `cls-xxxxxxxx`。
```

```json
{
  "Sequences": "<Sequences>",
  "Name": "<Name>",
  "SequenceFlows": "<SequenceFlows>",
  "Tags": "<Tags>",
  "Key": "<Key>",
  "Value": "<Value>"
}
```

### 资源检查

```bash
# 4. 验证编排中涉及的集群均存在且运行中
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID_1","CLUSTER_ID_2"]'
# expected: 每个集群 ClusterStatus: "Running"

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

# 5. 确认各集群已设置维护窗口
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> \
    --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'
# expected: TotalCount >= 1

预期输出：

```json
{
    "MaintenanceWindowAndExclusions": [
        {
            "ClusterID": "cls-xxxxxxxx",
            "MaintenanceTime": "04:00:00",
            "Duration": 4,
            "DayOfWeek": ["SA"],
            "Exclusions": []
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

## 关键字段说明

以下说明 `CreateRollOutSequence` 的主要参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `Name` | String | 是 | 编排名称，全局唯一。长度 1-63 字符 | 重名 → 创建失败 |
| `SequenceFlows` | Array | 是 | 升级流程定义数组。每项含 `ClusterID`、`Order` | ClusterID 不存在 → `InvalidParameterValue.ClusterID` |
| `SequenceFlows[].ClusterID` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameterValue.ClusterID` |
| `SequenceFlows[].Order` | Integer | 是 | 升级顺序，数字越小越先执行。相同 Order 的集群可并发升级 | Order 重复 → 两个集群将并发升级（可能是预期行为） |
| `Enabled` | Boolean | 是 | `true` 启用编排，`false` 停用。停用后升级不会按编排自动执行 | `false` 导致编排不生效 |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建编排 | `tccli tke CreateRollOutSequence --region <Region> --cli-input-json file://sequence.json` | 否 |
| 查看编排列表 | `tccli tke DescribeRollOutSequences --region <Region>` | 是 |
| 启用/停用编排 | 修改编排后重新创建（更新 `--Enabled`） | 否 |

## 操作步骤

### 1. 创建集群升级编排

#### 选择依据

- **升级顺序**：测试集群 Order=1 先升级，验证通过后生产集群 Order=2 后升级（灰度策略）。
- **并发**：同 Order 值的集群并发升级。不同 Order 值则串行等待。
- **Enabled**：创建后立即启用试 `true`，先验证编排再开 `false`。

#### 最小创建（串行灰度编排）

`sequence.json`：

```json
{
    "Name": "rolling-upgrade-seq",
    "SequenceFlows": [
        {
            "ClusterID": "CLUSTER_ID_1",
            "Order": 1
        },
        {
            "ClusterID": "CLUSTER_ID_2",
            "Order": 2
        },
        {
            "ClusterID": "CLUSTER_ID_3",
            "Order": 3
        }
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

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID_1` | 测试/预发集群 ID | 必须存在且 Running | `tccli tke DescribeClusters --region <Region>` |
| `CLUSTER_ID_2` | 第一批生产集群 ID | 必须存在且 Running | 同上 |
| `CLUSTER_ID_3` | 第二批生产集群 ID | 必须存在且 Running | 同上 |

### 2. 查看集群编排列表

```bash
tccli tke DescribeRollOutSequences --region <Region>
# expected: TotalCount >= 1
```

预期输出：

```json
{
    "Sequences": [
        {
            "ID": 1,
            "Name": "rolling-upgrade-seq",
            "Enabled": true,
            "SequenceFlows": [
                {"ClusterID": "cls-example-1", "Order": 1},
                {"ClusterID": "cls-example-2", "Order": 2},
                {"ClusterID": "cls-example-3", "Order": 3}
            ]
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

分页查询：

```bash
tccli tke DescribeRollOutSequences --region <Region> \
    --Offset 0 --Limit 10
# expected: Sequences 数组，最多 10 条
```

### 3. 编排策略说明

| 策略要素 | 说明 | 配置方式 |
|---------|------|---------|
| 升级顺序 | 集群按 `Order` 字段从小到大依次升级 | `SequenceFlows[].Order` |
| 并发控制 | 同一 Order 的集群可并发升级 | 设置相同 Order 值 |
| 暂停策略 | 在某个集群升级完后暂停，等待人工确认 | 编排一期暂不支持 CLI 配置，可通过控制台设置 |
| 启用/停用 | `Enabled=false` 暂停整个编排 | 修改 `--Enabled` 后重建 |

## 验证

```bash
# 验证编排存在且启用
tccli tke DescribeRollOutSequences --region <Region>
# expected: Sequences 包含目标编排，Enabled: true，SequenceFlows 数组与创建参数一致
```

## 清理

> **注意**：当前 `tccli tke` 未暴露 `DeleteRollOutSequence` API。如需删除编排，可通过控制台操作或停用（设置 `Enabled=false`）。

```bash
# 停用编排（重新创建同名编排，Enabled=false）
tccli tke CreateRollOutSequence --region <Region> \
    --cli-input-json file://sequence-disable.json
```

`sequence-disable.json`：

```json
{
    "Name": "rolling-upgrade-seq",
    "SequenceFlows": [
        {"ClusterID": "CLUSTER_ID_1", "Order": 1}
    ],
    "Enabled": false
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateRollOutSequence` 返回 `InvalidParameterValue.ClusterID` | 检查 SequenceFlows 中每个 ClusterID | 某个集群 ID 不存在或拼写错误 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 验证集群存在 |
| `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查凭据 | 缺少 `tke:CreateRollOutSequence` 权限 | 联系管理员授予编排管理权限 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 编排创建后未见生效 | `tccli tke DescribeRollOutSequences --region <Region>` 检查 Enabled | `Enabled` 为 `false` 或集群未设置维护窗口 | 确认 `--Enabled true`，且各集群已设置维护窗口 |
| 同名编排已存在 | `tccli tke DescribeRollOutSequences --region <Region>` 确认 | 编排名称全局唯一，不可重复 | 通过控制台删除旧编排，或使用不同 Name |

## 下一步

- [计划升级流程](../计划升级流程/tccli%20操作.md) — 端到端升级步骤
- [维护窗口和排除项](../维护窗口和排除项/tccli%20操作.md) — 设置集群维护窗口

## 控制台替代

控制台：集群 → 运维管理 → 计划升级 → 集群编排 → 新建编排。
