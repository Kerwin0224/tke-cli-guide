# 告警历史（tccli）

> 对照官方：[告警历史](https://cloud.tencent.com/document/product/457/71904) · page_id `71904`

## 概述

通过 `tccli monitor DescribeAlarmHistories` 查询 Prometheus 监控服务产生的告警历史记录，支持按告警策略、监控类型、时间范围等条件过滤。告警历史记录了告警从触发到恢复的完整生命周期状态变化。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已创建 Prometheus 监控实例（参见 [创建监控实例](../创建监控实例/tccli%20操作.md)）
- 目标集群已关联到监控实例（参见 [关联集群](../关联集群/tccli%20操作.md)）
- 已配置数据采集规则（参见 [数据采集配置](../数据采集配置/tccli%20操作.md)）
- 已配置告警规则（参见 [新建告警策略](../新建告警策略/tccli%20操作.md)）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看告警历史 | `tccli monitor DescribeAlarmHistories --cli-input-json file://query.json` | 是 |
| 查看告警策略列表 | `tccli monitor DescribePrometheusAlertPolicy --InstanceId <InstanceId>` | 是 |
| 查看告警规则 | `tccli monitor DescribeAlertRules --InstanceId <InstanceId>` | 是 |
| 获取告警指标 | `tccli monitor DescribeAlarmMetrics --MonitorType <MonitorType>` | 是 |

## 操作步骤

### 1. 查看告警策略列表

```bash
tccli monitor DescribePrometheusAlertPolicy --region ap-guangzhou --InstanceId prom-example01 --filter "Policies[].Name, Policies[].Status"
```

```json
{
    "Response": {
        "Policies": [
            {
                "Name": "HighCPUUsage",
                "Status": 1,
                "PolicyId": "policy-example01"
            },
            {
                "Name": "PodRestartAlert",
                "Status": 1,
                "PolicyId": "policy-example02"
            }
        ],
        "RequestId": "abc123-..."
    }
}
```

`Status: 1` 表示启用，`Status: 2` 表示已停用。

### 2. 按监控类型查告警历史

```bash
# 查询 Prometheus 监控类型的告警历史
tccli monitor DescribeAlarmHistories --region ap-guangzhou --cli-unfold-argument \
    --MonitorTypes '["MT_QCE"]' \
    --StartTime $(date -v-1d +%s) \
    --EndTime $(date +%s) \
    --Limit 10
```

```json
{
    "Response": {
        "Histories": [
            {
                "AlarmId": "alarm-example01",
                "MonitorType": "MT_QCE",
                "Namespace": "QCE/PROMETHEUS",
                "PolicyName": "HighCPUUsage",
                "Content": "CPU 使用率 > 80%，当前值 85.3%",
                "FirstOccurTime": "2026-06-16 08:30:00",
                "LastOccurTime": "2026-06-16 09:00:00",
                "AlarmStatus": "ALARM",
                "AlarmObject": "prom-example01/cls-example/node-10.0.0.1"
            },
            {
                "AlarmId": "alarm-example02",
                "MonitorType": "MT_QCE",
                "Namespace": "QCE/PROMETHEUS",
                "PolicyName": "PodRestartAlert",
                "Content": "Pod 重启次数超过阈值，当前值 5 次",
                "FirstOccurTime": "2026-06-16 08:45:00",
                "LastOccurTime": "2026-06-16 08:45:00",
                "AlarmStatus": "OK",
                "AlarmObject": "prom-example01/cls-example/default/my-app-7d4f8b9c-x2k9m"
            }
        ],
        "TotalCount": 2,
        "RequestId": "def456-..."
    }
}
```

告警状态说明：

| 状态 | 含义 |
|------|------|
| `ALARM` | 告警中（未恢复） |
| `OK` | 已恢复 |
| `NO_CONF` | 已失效 |
| `NO_DATA` | 数据不足 |

### 3. 按告警策略过滤

```bash
tccli monitor DescribeAlarmHistories --region ap-guangzhou --cli-unfold-argument \
    --MonitorTypes '["MT_QCE"]' \
    --PolicyIds '["policy-example01"]' \
    --StartTime $(date -v-7d +%s) \
    --EndTime $(date +%s) \
    --Limit 20 \
    --filter "Histories[].AlarmId, Histories[].Content, Histories[].AlarmStatus, Histories[].FirstOccurTime"
```

### 4. 按告警状态过滤

```bash
# 仅查看当前仍在告警中的记录
tccli monitor DescribeAlarmHistories --region ap-guangzhou --cli-unfold-argument \
    --MonitorTypes '["MT_QCE"]' \
    --AlarmStatus '["ALARM"]' \
    --StartTime $(date -v-1d +%s) \
    --EndTime $(date +%s) \
    --Limit 10
```

### 5. 查看告警策略详情

```bash
tccli monitor DescribePrometheusAlertPolicy --region ap-guangzhou --InstanceId prom-example01
```

```json
{
    "Response": {
        "Policies": [
            {
                "PolicyId": "policy-example01",
                "Name": "HighCPUUsage",
                "Status": 1,
                "Rules": [
                    {
                        "Name": "cpu-usage-rule",
                        "Rule": 80,
                        "TemplateId": "temp-xxx",
                        "Describe": "CPU 使用率超过 80% 持续 5 分钟则告警",
                        "UpdateTime": "2026-06-10 10:00:00"
                    }
                ]
            }
        ],
        "RequestId": "ghi789-..."
    }
}
```

### 6. 查看关联实例的告警规则

```bash
tccli monitor DescribeAlertRules --region ap-guangzhou --InstanceId prom-example01 --filter "AlertRuleSet[].AlertName, AlertRuleSet[].AlertState"
```

## 验证

```bash
# 确认最近 24 小时是否有告警记录
tccli monitor DescribeAlarmHistories --region ap-guangzhou --cli-unfold-argument \
    --MonitorTypes '["MT_QCE"]' \
    --StartTime $(date -v-1d +%s) \
    --EndTime $(date +%s) \
    --filter "TotalCount"
```

## 清理

告警历史为只读数据，无法通过 CLI 删除。历史记录由系统自动保留，到期后清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 子用户查询不到告警历史 | `tccli configure list` 检查当前凭据 | 子用户只能查询被授权项目下的告警历史，CAM 项目权限不足（此为环境限制，非命令错误） | 联系主账号在 [CAM 控制台](https://console.cloud.tencent.com/cam) 为子用户添加相关项目的只读权限 |
| `DescribeAlarmHistories` 返回为空 | 检查 `--StartTime` 和 `--EndTime` 时间范围 | `MonitorTypes` 不正确或时间范围内无告警产生 | 确认 `MonitorTypes` 为 `["MT_QCE"]`，调整时间范围扩大搜索 |
| `DescribeAlertRules` 返回为空 | `tccli monitor DescribePrometheusAlertPolicy --InstanceId <InstanceId>` 确认策略配置 | 该实例尚未配置告警规则 | 执行 [新建告警策略](../新建告警策略/tccli%20操作.md) 创建告警规则 |

## 下一步

- [新建告警策略](../新建告警策略/tccli%20操作.md) 配置告警规则
- [暂停告警策略](../暂停告警策略/tccli%20操作.md) 临时关闭不必要告警
- [精简监控指标](../精简监控指标/tccli%20操作.md) 过滤指标减少无效告警

## 控制台替代

控制台：Prometheus 监控 → 实例详情 → 告警配置 → 告警历史 → 可按告警状态、时间范围过滤查看。
