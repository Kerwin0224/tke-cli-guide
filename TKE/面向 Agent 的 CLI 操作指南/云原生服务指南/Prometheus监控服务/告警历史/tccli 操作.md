# 告警历史

> 对照官方：[告警历史](https://cloud.tencent.com/document/product/457/71904) · page_id `71904`

## 概述

通过 `DescribePrometheusAlertHistory` 查询 Prometheus 监控实例的告警历史记录。支持按时间段、告警状态、告警名称等条件过滤。告警记录包含触发时间、告警规则名称、告警内容、当前状态等信息，可用于分析告警趋势、排查历史问题。

**告警状态说明**：

| 状态 | 说明 |
|------|------|
| `firing` | 告警触发中 |
| `resolved` | 告警已恢复 |
| `pending` | 告警等待中（待确认） |

## 前置条件

- [环境准备](../../../环境准备.md)
- 已有 Prometheus 监控实例，状态为 `Running`，且已配置告警规则

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    monitor:DescribePrometheusAlertHistory
#    monitor:DescribePrometheusInstances
# 验证
tccli monitor DescribePrometheusInstances --region <Region>
# expected: exit 0，返回实例列表
```

### 资源检查

```bash
# 4. 确认实例 Running
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: InstanceStatus: 2 (Running)
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看告警历史 | `tccli monitor DescribePrometheusAlertHistory --region <Region> --cli-input-json file://input.json` | 是 |

## 操作步骤

### 步骤：查询告警历史

#### 选择依据

- **时间范围**：`StartTime` 和 `EndTime` 为 Unix 秒级时间戳，建议按需设置以提升查询性能。不指定时默认返回最近 24 小时。
- **过滤条件**：可按 `AlertName` 过滤特定告警规则，按 `Status` 过滤 `firing`/`resolved`，按 `Content` 过滤告警内容中的关键词。
- **分页**：通过 `Offset` 和 `Limit` 控制分页，默认 `Limit` 为 20。

#### 最小查询（最近告警记录）

```bash
# 查询最近告警历史，默认返回 20 条
tccli monitor DescribePrometheusAlertHistory --region <Region> \
    --InstanceId '<InstanceId>'
# expected: exit 0，返回告警历史列表
```

预期输出：

```json
{
    "AlertHistorySet": [
        {
            "AlertName": "CPUThrottlingHigh",
            "Content": "alertname=CPUThrottlingHigh, instance=10.0.1.5:8080, job=kubernetes-pods, namespace=default, severity=warning",
            "StartTime": 1718700000,
            "EndTime": 1718700360,
            "Status": "resolved",
            "RuleExpr": "rate(container_cpu_cfs_throttled_seconds_total[5m]) > 0.01",
            "NotifyWay": ["EMAIL"],
            "Duration": "6m0s"
        },
        {
            "AlertName": "MemoryUsageWarning",
            "Content": "alertname=MemoryUsageWarning, instance=10.0.1.5:8080, job=kubernetes-pods, namespace=production, severity=critical",
            "StartTime": 1718705000,
            "EndTime": 0,
            "Status": "firing",
            "RuleExpr": "container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.9",
            "NotifyWay": ["SMS", "EMAIL"],
            "Duration": "15m0s"
        }
    ],
    "TotalCount": 2,
    "RequestId": "..."
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<InstanceId>` | Prometheus 实例 ID | `tccli monitor DescribePrometheusInstances --region <Region>` |
| `REGION` | 地域 | `tccli configure list` |

#### 增强查询（时间段 + 状态过滤）

```bash
tccli monitor DescribePrometheusAlertHistory --region <Region> \
    --InstanceId '<InstanceId>' \
    --StartTime '<START_TIMESTAMP>' \
    --EndTime '<END_TIMESTAMP>' \
    --Status '<ALERT_STATUS>' \
    --Filters '[{"Name":"AlertName","Values":["CPUThrottlingHigh"]}]' \
    --Offset 0 \
    --Limit 50
# expected: exit 0，返回过滤后的告警历史
```

| 参数 | 说明 | 约束 | 示例 |
|------|------|------|------|
| `--StartTime` | 查询开始时间 | Unix 秒级时间戳 | `1718613600` |
| `--EndTime` | 查询结束时间 | Unix 秒级时间戳 | `1718700000` |
| `--Status` | 告警状态 | `firing` / `resolved` / `pending` | `firing` |
| `--Filters` | 过滤条件 | 支持 `AlertName` 等字段过滤 | `[{"Name":"AlertName","Values":["CPUThrottlingHigh"]}]` |
| `--Offset` | 偏移量 | 分页起始位置 | `0` |
| `--Limit` | 返回条数 | 1-100 | `50` |

### 常用查询场景

**场景 1：查询当前活跃（firing）的告警**

```bash
tccli monitor DescribePrometheusAlertHistory --region <Region> \
    --InstanceId '<InstanceId>' \
    --Status firing
# expected: 返回所有 firing 状态的告警
```

**场景 2：查询最近 1 小时的告警**

```bash
END_TIME=$(date +%s)
START_TIME=$((END_TIME - 3600))
tccli monitor DescribePrometheusAlertHistory --region <Region> \
    --InstanceId '<InstanceId>' \
    --StartTime $START_TIME \
    --EndTime $END_TIME
# expected: 返回最近 1 小时内的告警记录
```

**场景 3：查询特定告警规则的历史**

```bash
tccli monitor DescribePrometheusAlertHistory --region <Region> \
    --InstanceId '<InstanceId>' \
    --Filters '[{"Name":"AlertName","Values":["MemoryUsageWarning"]}]' \
    --Limit 100
# expected: 返回 MemoryUsageWarning 规则的告警记录
```

## 验证

```bash
# 验证告警历史接口可访问
tccli monitor DescribePrometheusAlertHistory --region <Region> \
    --InstanceId '<InstanceId>' \
    --Limit 1
# expected: exit 0，返回 TotalCount 和 AlertHistorySet（可能为空）

# 解析返回结构
tccli monitor DescribePrometheusAlertHistory --region <Region> \
    --InstanceId '<InstanceId>' | jq '. | {TotalCount, FirstAlert: .AlertHistorySet[0].AlertName}'
# expected: 输出 TotalCount 和第一条告警名称（如有）
```

## 清理

> **注意**：告警历史记录是只读数据，CLI 无删除告警历史的 API。告警记录按实例数据保留时长自动清理（基础版 15 天，标准版 30 天）。
> **清理告警规则**：如需清理告警规则（停止产生新告警），需通过控制台或 `ModifyPrometheusAlertPolicy`（如支持）操作。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribePrometheusAlertHistory` 返回 `ResourceNotFound.InstanceNotFound` | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 验证实例 | InstanceId 不存在或已销毁 | 确认 InstanceId 正确，实例状态为 Running |
| `DescribePrometheusAlertHistory` 返回 `InvalidParameter.StartTime` | 检查时间戳格式 | 时间戳格式错误（非 Unix 秒级时间戳或使用了毫秒） | 使用秒级时间戳：`date +%s` 获取当前时间戳 |
| `DescribePrometheusAlertHistory` 返回 `UnauthorizedOperation.CamNoAuth` | 检查 CAM 策略 | 缺少 `monitor:DescribePrometheusAlertHistory` 权限 | 联系管理员授权 `monitor:DescribePrometheusAlertHistory` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 查询返回空结果但已知有告警 | 检查时间范围和状态过滤条件 | 时间范围过窄或状态过滤排除掉了目标告警 | 扩大时间范围，移除 Status 过滤重新查询 |
| `TotalCount` 很大但实际只返回了 Limit 条 | 检查 `Offset` 和 `Limit` 参数 | 分页限制了返回条数 | 增加 `Limit` 或翻页（调整 `Offset`） |
| 告警 `EndTime` 为 0 | 正常行为 | 告警仍在 firing 状态，尚未恢复 | 无 — 0 表示告警仍在持续中 |
| 返回的 `Content` 中包含不完整的标签 | 正常行为 | `Content` 为告警触发时的标签快照 | 结合 `RuleExpr` 和告警触发时刻的上下文综合判断 |

## 下一步

- [数据采集配置](../数据采集配置/tccli%20操作.md) — 确保告警规则所需的指标被正确采集
- [精简监控指标](../精简监控指标/tccli%20操作.md) — 精简指标但保留告警规则所需的指标
- [销毁监控实例](../销毁监控实例/tccli%20操作.md) — 清理监控实例

## 控制台替代

控制台：容器服务控制台 → 运维功能 → Prometheus 监控 → 选择实例 → 告警管理 → 告警历史。
