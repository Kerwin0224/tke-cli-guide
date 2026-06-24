# 维护窗口和排除项（tccli）

> 对照官方：[维护窗口和排除项](https://cloud.tencent.com/document/product/457/124478) · page_id `124478`

## 概述

通过 `CreateClusterMaintenanceWindowAndExclusions` 为集群设置维护时间窗口，结合排除项在窗口内剔除不宜升级的日期。维护窗口创建后，集群版本升级将在窗口内自动触发。

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
#    tke:CreateClusterMaintenanceWindowAndExclusions
#    tke:DescribeClusterMaintenanceWindowAndExclusions
#    tke:DeleteClusterMaintenanceWindowAndExclusion
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

# （可选）如需全局窗口：需 CentrallyManage 级 CAM 权限
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

# 5. 确认目标集群是否已有维护窗口
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> \
    --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'
# expected: TotalCount 为 0（可新建）或 1（已有，需先删除再重建）
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

## 关键字段说明

以下说明 `CreateClusterMaintenanceWindowAndExclusions` 的主要参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterID` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx`。每集群最多一个维护窗口 | 集群不存在 → `ResourceNotFound` |
| `MaintenanceTime` | String | 是 | 格式 `HH:MM:SS`，24 小时制。如 `04:00:00`（非 `4:00:00`） | 格式错误 → `InvalidParameter` |
| `Duration` | Integer | 是 | 持续**小时**数，整数。范围 1-24。**注意：单位是小时，不是分钟** | 填 120（分钟）→ `InvalidParameter` "must not exceed 24 hours"；填 0 → `InvalidParameter`；填 25 → `InvalidParameter` |
| `DayOfWeek` | Array of String | 是 | `MO`/`TU`/`WE`/`TH`/`FR`/`SA`/`SU`，严格双字母大写。不支持全拼（`Saturday`）、数字（`6`）、缩写（`Mon`） | 任何格式错误 → `InvalidParameter` "invalid day of week" |
| `Exclusions` | Array | 否 | 排除的时间段列表，每项含 `StartTime`、`EndTime`，格式 `YYYY-MM-DD HH:MM:SS` | StartTime > EndTime → `InvalidParameter` |

### DayOfWeek 对照表

| 星期 | 正确写法 | 常见错误写法 |
|------|---------|------------|
| 星期一 | `MO` | `Monday`、`monday`、`Mon`、`1` |
| 星期二 | `TU` | `Tuesday`、`tuesday`、`Tue`、`2` |
| 星期三 | `WE` | `Wednesday`、`wednesday`、`Wed`、`3` |
| 星期四 | `TH` | `Thursday`、`thursday`、`Thu`、`4` |
| 星期五 | `FR` | `Friday`、`friday`、`Fri`、`5` |
| 星期六 | `SA` | `Saturday`、`saturday`、`Sat`、`6` |
| 星期日 | `SU` | `Sunday`、`sunday`、`Sun`、`0`、`7` |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建维护窗口 | `tccli tke CreateClusterMaintenanceWindowAndExclusions --region <Region> --ClusterID CLUSTER_ID --MaintenanceTime 04:00:00 --Duration 4 --DayOfWeek '["SA"]'` | 否 |
| 查看维护窗口 | `tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region>` | 是 |
| 删除维护窗口 | `tccli tke DeleteClusterMaintenanceWindowAndExclusion --region <Region> --ClusterID CLUSTER_ID` | 是 |

## 操作步骤

### 1. 创建维护窗口

#### 选择依据

- **DayOfWeek**：选择业务低峰期对应的星期几。例如周六 `SA` 业务量最低。
- **MaintenanceTime**：建议凌晨时段，如 `04:00:00`。
- **Duration**：至少 4 小时确保升级有足够时间完成。单位是小时，不是分钟。

#### 最小创建（周六 4 点，4 小时）

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

#### 增强配置（多日窗口 + 排除项）

排除 2026-06-20 至 2026-06-21 期间不进行升级：

```bash
tccli tke CreateClusterMaintenanceWindowAndExclusions --region <Region> \
    --ClusterID CLUSTER_ID \
    --MaintenanceTime "03:00:00" \
    --Duration 6 \
    --DayOfWeek '["SA","SU"]' \
    --Exclusions '[{"StartTime":"2026-06-20 00:00:00","EndTime":"2026-06-21 23:59:59"}]'
```

### 2. 查看维护窗口

```bash
# 查看所有维护窗口
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region>
# expected: TotalCount >= 1
```

预期输出：

```json
{
    "MaintenanceWindowAndExclusions": [
        {
            "ClusterID": "cls-example",
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

按集群过滤：

```bash
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> \
    --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'
```

```json
{
  "MaintenanceWindowAndExclusions": [],
  "MaintenanceTime": "<MaintenanceTime>",
  "Duration": 0,
  "ClusterID": "<ClusterID>",
  "DayOfWeek": [],
  "Region": "<Region>",
  "ClusterName": "<ClusterName>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 3. 删除维护窗口

```bash
tccli tke DeleteClusterMaintenanceWindowAndExclusion --region <Region> \
    --ClusterID CLUSTER_ID
# expected: exit 0，返回 RequestId
```

## 验证

```bash
# 验证维护窗口存在且参数正确
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> \
    --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'
# expected: TotalCount: 1，DayOfWeek、Duration、MaintenanceTime 与创建参数一致
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
# 1. 清理前确认目标
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> \
    --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'
# expected: 确认是待删除的维护窗口

# 2. 删除维护窗口
tccli tke DeleteClusterMaintenanceWindowAndExclusion --region <Region> \
    --ClusterID CLUSTER_ID
# expected: exit 0

# 3. 验证已删除
tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> \
    --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'
# expected: TotalCount: 0
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

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DayOfWeek` 返回 `InvalidParameter` "invalid day of week: Saturday" | 检查 `--DayOfWeek` 参数值 | 使用了全拼 `Saturday` | 改用 `'["SA"]'` |
| `DayOfWeek` 返回 `InvalidParameter` "invalid day of week: 6" | 检查参数值 | 使用了数字 `6` | 改用 `'["SA"]'` |
| `DayOfWeek` 返回 `InvalidParameter` | 检查参数值 | 使用了小写 `monday` | 改用 `'["MO"]'` |
| `DayOfWeek` 返回 `InvalidParameter` | 检查参数值 | 使用了缩写 `Mon` | 改用 `'["MO"]'` |
| `Duration` 返回 `InvalidParameter` "must not exceed 24 hours" | 检查 `--Duration` 值 | 填了分钟数如 `120`（误以为单位是分钟） | Duration 单位是小时：`--Duration 2`（120 分钟 = 2 小时） |
| `Duration` 返回 `InvalidParameter` | 检查 `--Duration` 值 | 填了 `0`（最少 1 小时） | `--Duration 1` |
| `Duration` 返回 `InvalidParameter` | 检查 `--Duration` 值 | 填了 `25`（最多 24 小时） | `--Duration 24` |
| `MaintenanceTime` 返回 `InvalidParameter` | 检查时间格式 | 用了 `4:00:00` 而非 `04:00:00` | 使用两位小时，如 `04:00:00` |
| `CreateGlobalMaintenanceWindowAndExclusions` 返回 `UnauthorizedOperation.CamNoAuth` | 检查 CAM 策略 | 全局窗口 API 需 CentrallyManage 级权限（此为环境限制） | 改用集群级 `CreateClusterMaintenanceWindowAndExclusions` |
| `CreateClusterMaintenanceWindowAndExclusions` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查凭据 | 缺少 `tke:CreateClusterMaintenanceWindowAndExclusions` 权限 | 联系管理员授予该 CAM Action |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 维护窗口已存在时重复创建 | `tccli tke DescribeClusterMaintenanceWindowAndExclusions --region <Region> --Filters '[{"Name":"ClusterID","Values":["CLUSTER_ID"]}]'` 确认 | 每集群最多一个维护窗口 | `tccli tke DeleteClusterMaintenanceWindowAndExclusion --region <Region> --ClusterID CLUSTER_ID` 后重新创建 |

## 下一步

- [集群编排](../集群编排/tccli%20操作.md) — 定义批量升级顺序
- [计划升级流程](../计划升级流程/tccli%20操作.md) — 端到端升级步骤
- [计划升级概述](../计划升级概述/tccli%20操作.md) — 核心概念

## 控制台替代

控制台：集群 → 运维管理 → 计划升级 → 维护窗口 → 新建。
