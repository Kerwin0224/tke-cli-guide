# 查看监控（tccli）

> 对照官方：[监控和告警](https://cloud.tencent.com/document/product/457/58212) · page_id `58212`

## 概述

TKE Serverless 集群通过云监控（Cloud Monitor）提供 4 级粒度的监控指标：集群级、工作负载级、Pod 级和容器级。监控命名空间为 `QCE/EKS`，支持通过 `tccli monitor` 拉取指标数据和管理告警策略。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已配置 `tccli`，可用 `tccli tke help` 验证
- 已有 TKE Serverless 集群（`ClusterId`），状态为 `运行中`（`Running`）
- 集群中已有运行中的工作负载/容器实例（否则大部分指标无数据）
- `tccli monitor help` 可用（需安装 monitor 产品 SDK）

> **注意：** kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）。所有监控查询均通过 tccli monitor API 完成。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群级监控 | `tccli monitor DescribeStatisticData --Namespace QCE/EKS --MetricName <Metric> --Period <Period> --Conditions <Conditions>` | 是 |
| 查看工作负载级监控 | 同上，额外指定 `--GroupBy tke_cls_workload` | 是 |
| 查看 Pod 级监控 | 同上，额外指定 `--GroupBy tke_cls_pod` | 是 |
| 查看容器级监控 | 同上，额外指定 `--GroupBy tke_cls_container` | 是 |
| 查看告警策略列表 | `tccli monitor DescribeAlarmPolicies --MonitorTypes '["MT_QCE"]' --Namespaces '["QCE/EKS"]'` | 是 |
| 创建告警策略 | `tccli monitor CreateAlarmPolicy --cli-input-json file://alarm-policy.json` | 否 |
| 查看告警历史 | `tccli monitor DescribeAlarmHistories` | 是 |

## 操作步骤

### 1. 查看集群级监控指标

集群级指标包含 CPU 使用量和内存使用量。

**集群 CPU 使用量（核）：**

```bash
tccli monitor DescribeStatisticData \
    --Module "monitor" \
    --Namespace "QCE/EKS" \
    --MetricName "K8sClusterCpuCoreUsed" \
    --Period "60" \
    --StartTime "$(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S+08:00)" \
    --EndTime "$(date +%Y-%m-%dT%H:%M:%S+08:00)" \
    --Conditions '[{"Key":"tke_cls_instance_id","Operator":"=","Value":["cls-example"]}]' \
    --region ap-guangzhou
```

```output
{
    "StartTime": "2024-07-17T09:00:00+08:00",
    "EndTime": "2024-07-17T10:00:00+08:00",
    "MetricName": "K8sClusterCpuCoreUsed",
    "Period": 60,
    "DataPoints": [
        {
            "Dimensions": [
                {"Name": "tke_cls_instance_id", "Value": "cls-example"}
            ],
            "Values": [2.4, 2.6, 2.3, 2.8, 2.5, 2.7, 2.4]
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

**集群内存使用量（GB）：**

```bash
tccli monitor DescribeStatisticData \
    --Module "monitor" \
    --Namespace "QCE/EKS" \
    --MetricName "K8sClusterMemUsageBytes" \
    --Period "60" \
    --StartTime "$(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S+08:00)" \
    --EndTime "$(date +%Y-%m-%dT%H:%M:%S+08:00)" \
    --Conditions '[{"Key":"tke_cls_instance_id","Operator":"=","Value":["cls-example"]}]' \
    --region ap-guangzhou
```

### 2. 查看工作负载级监控指标

工作负载级指标包括重启次数、CPU 使用量/利用率、内存使用量/利用率。通过 `GroupBy` 按工作负载维度聚合。

**工作负载重启次数：**

```bash
tccli monitor DescribeStatisticData \
    --Module "monitor" \
    --Namespace "QCE/EKS" \
    --MetricName "K8sWorkloadPodRestartTotal" \
    --Period "300" \
    --StartTime "$(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S+08:00)" \
    --EndTime "$(date +%Y-%m-%dT%H:%M:%S+08:00)" \
    --Conditions '[
        {"Key":"tke_cls_instance_id","Operator":"=","Value":["cls-example"]},
        {"Key":"namespace","Operator":"=","Value":["default"]}
    ]' \
    --GroupBy '["tke_cls_workload"]' \
    --region ap-guangzhou
```

```output
{
    "StartTime": "2024-07-17T09:00:00+08:00",
    "EndTime": "2024-07-17T10:00:00+08:00",
    "MetricName": "K8sWorkloadPodRestartTotal",
    "Period": 300,
    "DataPoints": [
        {
            "Dimensions": [
                {"Name": "tke_cls_instance_id", "Value": "cls-example"},
                {"Name": "tke_cls_workload", "Value": "Deployment/nginx-deployment"}
            ],
            "Values": [0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0]
        },
        {
            "Dimensions": [
                {"Name": "tke_cls_instance_id", "Value": "cls-example"},
                {"Name": "tke_cls_workload", "Value": "Deployment/api-server"}
            ],
            "Values": [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

**工作负载 CPU 利用率（%）：**

```bash
tccli monitor DescribeStatisticData \
    --Module "monitor" \
    --Namespace "QCE/EKS" \
    --MetricName "K8sWorkloadRateCpuCoreUsedRequest" \
    --Period "300" \
    --StartTime "$(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S+08:00)" \
    --EndTime "$(date +%Y-%m-%dT%H:%M:%S+08:00)" \
    --Conditions '[
        {"Key":"tke_cls_instance_id","Operator":"=","Value":["cls-example"]},
        {"Key":"namespace","Operator":"=","Value":["default"]}
    ]' \
    --GroupBy '["tke_cls_workload"]' \
    --region ap-guangzhou
```

### 3. 查看 Pod 级监控指标

Pod 级提供 9 个指标：CPU 使用量/利用率、内存使用量/利用率、网络入/出流量、网络入/出带宽、磁盘使用量。

**Pod CPU 利用率（%）：**

```bash
tccli monitor DescribeStatisticData \
    --Module "monitor" \
    --Namespace "QCE/EKS" \
    --MetricName "K8sPodRateCpuCoreUsedRequest" \
    --Period "300" \
    --StartTime "$(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S+08:00)" \
    --EndTime "$(date +%Y-%m-%dT%H:%M:%S+08:00)" \
    --Conditions '[
        {"Key":"tke_cls_instance_id","Operator":"=","Value":["cls-example"]},
        {"Key":"namespace","Operator":"=","Value":["default"]}
    ]' \
    --GroupBy '["tke_cls_pod"]' \
    --region ap-guangzhou
```

```output
{
    "StartTime": "2024-07-17T09:00:00+08:00",
    "EndTime": "2024-07-17T10:00:00+08:00",
    "MetricName": "K8sPodRateCpuCoreUsedRequest",
    "Period": 300,
    "DataPoints": [
        {
            "Dimensions": [
                {"Name":"tke_cls_instance_id","Value":"cls-example"},
                {"Name":"tke_cls_pod","Value":"nginx-deployment-7b8f9c6d5-abcde"}
            ],
            "Values": [12.5, 15.3, 11.8, 14.2, 13.7, 16.1, 12.9, 15.8, 14.5, 13.2, 11.6, 12.4]
        },
        {
            "Dimensions": [
                {"Name":"tke_cls_instance_id","Value":"cls-example"},
                {"Name":"tke_cls_pod","Value":"api-server-5c6d8e4f2-fghij"}
            ],
            "Values": [45.2, 42.1, 48.3, 44.7, 41.9, 43.5, 47.8, 46.3, 42.6, 44.1, 40.8, 43.2]
        }
    ],
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

**Pod 网络入流量（Bytes）：**

```bash
tccli monitor DescribeStatisticData \
    --Module "monitor" \
    --Namespace "QCE/EKS" \
    --MetricName "K8sPodNetworkReceiveBytes" \
    --Period "300" \
    --StartTime "$(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S+08:00)" \
    --EndTime "$(date +%Y-%m-%dT%H:%M:%S+08:00)" \
    --Conditions '[
        {"Key":"tke_cls_instance_id","Operator":"=","Value":["cls-example"]},
        {"Key":"namespace","Operator":"=","Value":["default"]}
    ]' \
    --GroupBy '["tke_cls_pod"]' \
    --region ap-guangzhou
```

### 4. 查看容器级监控指标

容器级提供 5 个指标：CPU 使用量/利用率、内存使用量/利用率、磁盘使用量。

```bash
tccli monitor DescribeStatisticData \
    --Module "monitor" \
    --Namespace "QCE/EKS" \
    --MetricName "K8sContainerRateCpuCoreUsedRequest" \
    --Period "300" \
    --StartTime "$(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S+08:00)" \
    --EndTime "$(date +%Y-%m-%dT%H:%M:%S+08:00)" \
    --Conditions '[
        {"Key":"tke_cls_instance_id","Operator":"=","Value":["cls-example"]},
        {"Key":"namespace","Operator":"=","Value":["default"]}
    ]' \
    --GroupBy '["tke_cls_container"]' \
    --region ap-guangzhou
```

### 5. 查看告警策略列表

```bash
tccli monitor DescribeAlarmPolicies \
    --Module "monitor" \
    --MonitorTypes '["MT_QCE"]' \
    --Namespaces '["QCE/EKS"]' \
    --region ap-guangzhou
```

```output
{
    "TotalCount": 2,
    "Policies": [
        {
            "PolicyId": "policy-abc123",
            "PolicyName": "Pod CPU利用率告警",
            "MonitorType": "MT_QCE",
            "Namespace": "QCE/EKS",
            "Enable": 1,
            "Conditions": {
                "IsUnionRule": 0,
                "Rules": [
                    {"MetricName": "K8sPodRateCpuCoreUsedRequest", "Period": 60, "Operator": "gt", "Value": "80"}
                ]
            },
            "NoticeIds": ["notice-xyz456"]
        }
    ],
    "RequestId": "d4e5f6a7-b8c9-0123-defa-123456789013"
}
```

### 6. 创建告警策略

创建针对 Pod 的告警策略，监控 CPU 利用率、内存利用率、重启次数和 Pod Ready 状态。

创建输入 JSON 文件 `alarm-policy.json`：

```json
{
    "Module": "monitor",
    "PolicyName": "TKE-Serverless-Pod-Alert-Example",
    "MonitorType": "MT_QCE",
    "Namespace": "QCE/EKS",
    "Remark": "TKE Serverless Pod 级别告警策略",
    "Enable": 1,
    "ProjectId": 0,
    "ConditionTemplateId": 0,
    "Conditions": {
        "IsUnionRule": 0,
        "Rules": [
            {
                "MetricName": "K8sPodRateCpuCoreUsedRequest",
                "Period": 60,
                "Operator": "gt",
                "Value": "80",
                "ContinuePeriod": 2,
                "NoticeFrequency": 300
            },
            {
                "MetricName": "K8sPodRateMemUsageRequest",
                "Period": 60,
                "Operator": "gt",
                "Value": "85",
                "ContinuePeriod": 2,
                "NoticeFrequency": 300
            },
            {
                "MetricName": "K8sPodRestartTotal",
                "Period": 300,
                "Operator": "ge",
                "Value": "1",
                "ContinuePeriod": 1,
                "NoticeFrequency": 300
            }
        ]
    }
}
```

```bash
tccli monitor CreateAlarmPolicy \
    --cli-input-json file://alarm-policy.json \
    --region ap-guangzhou
```

```output
{
    "PolicyId": "policy-def789",
    "OriginId": "1234567",
    "RequestId": "e5f6a7b8-c9d0-1234-efab-123456789014"
}
```

### 7. 监控指标速查表

#### 集群级（QCE/EKS）

| 指标英文名 | 指标中文名 | 单位 |
|-----------|-----------|------|
| `K8sClusterCpuCoreUsed` | CPU 使用量 | 核 |
| `K8sClusterMemUsageBytes` | 内存使用量 | MB |
| `K8sClusterRateCpuCoreUsed` | CPU 利用率 | % |
| `K8sClusterRateMemUsageBytes` | 内存利用率 | % |

#### 工作负载级（QCE/EKS）

| 指标英文名 | 指标中文名 | 单位 |
|-----------|-----------|------|
| `K8sWorkloadPodRestartTotal` | Pod 重启次数 | 次 |
| `K8sWorkloadCpuCoreUsed` | CPU 使用量 | 核 |
| `K8sWorkloadMemUsageBytes` | 内存使用量 | MB |
| `K8sWorkloadRateCpuCoreUsedRequest` | CPU 利用率（占 Request） | % |
| `K8sWorkloadRateMemUsageRequest` | 内存利用率（占 Request） | % |

#### Pod 级（QCE/EKS）

| 指标英文名 | 指标中文名 | 单位 |
|-----------|-----------|------|
| `K8sPodCpuCoreUsed` | CPU 使用量 | 核 |
| `K8sPodMemUsageBytes` | 内存使用量 | MB |
| `K8sPodRateCpuCoreUsedRequest` | CPU 利用率（占 Request） | % |
| `K8sPodRateMemUsageRequest` | 内存利用率（占 Request） | % |
| `K8sPodNetworkReceiveBytes` | 网络入流量 | Bytes |
| `K8sPodNetworkTransmitBytes` | 网络出流量 | Bytes |
| `K8sPodNetworkReceiveBytesBw` | 网络入带宽 | Bps |
| `K8sPodNetworkTransmitBytesBw` | 网络出带宽 | Bps |
| `K8sPodDiskUsage` | 磁盘使用量 | MB |

#### 容器级（QCE/EKS）

| 指标英文名 | 指标中文名 | 单位 |
|-----------|-----------|------|
| `K8sContainerCpuCoreUsed` | CPU 使用量 | 核 |
| `K8sContainerMemUsageBytes` | 内存使用量 | MB |
| `K8sContainerRateCpuCoreUsedRequest` | CPU 利用率（占 Request） | % |
| `K8sContainerRateMemUsageRequest` | 内存利用率（占 Request） | % |
| `K8sContainerDiskUsage` | 磁盘使用量 | MB |

#### Pod 告警指标

| 指标英文名 | 指标中文名 | 建议阈值 |
|-----------|-----------|---------|
| `K8sPodRateCpuCoreUsedRequest` | CPU 利用率 | > 80% |
| `K8sPodRateMemUsageRequest` | 内存利用率 | > 85% |
| `K8sPodRestartTotal` | 重启次数 | >= 1 次/5分钟 |
| `K8sPodStatusReady` | Pod Ready 状态 | != 1 |

## 验证

### Control plane (tccli)

**验证 DescribeStatisticData 返回数据：**

```bash
tccli monitor DescribeStatisticData \
    --Module "monitor" \
    --Namespace "QCE/EKS" \
    --MetricName "K8sClusterCpuCoreUsed" \
    --Period "60" \
    --StartTime "$(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S+08:00)" \
    --EndTime "$(date +%Y-%m-%dT%H:%M:%S+08:00)" \
    --Conditions '[{"Key":"tke_cls_instance_id","Operator":"=","Value":["cls-example"]}]' \
    --region ap-guangzhou \
    | jq '.DataPoints | length'
```

预期返回 `> 0`。

**验证告警策略已创建：**

```bash
tccli monitor DescribeAlarmPolicies \
    --Module "monitor" \
    --MonitorTypes '["MT_QCE"]' \
    --Namespaces '["QCE/EKS"]' \
    --region ap-guangzhou \
    | jq '.Policies[] | select(.PolicyName == "TKE-Serverless-Pod-Alert-Example") | .PolicyId'
```

预期返回策略 ID。

## 清理

### Control plane (tccli)

监控指标数据无需清理（由云监控自动管理过期）。

**删除创建的告警策略：**

```bash
# 先获取告警策略 ID
POLICY_ID=$(tccli monitor DescribeAlarmPolicies \
    --Module "monitor" \
    --MonitorTypes '["MT_QCE"]' \
    --Namespaces '["QCE/EKS"]' \
    --region ap-guangzhou \
    | jq -r '.Policies[] | select(.PolicyName == "TKE-Serverless-Pod-Alert-Example") | .PolicyId')

# 删除告警策略
tccli monitor DeleteAlarmPolicy \
    --Module "monitor" \
    --PolicyIds "[\"${POLICY_ID}\"]" \
    --region ap-guangzhou
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeStatisticData` 返回空 `DataPoints` | `date -d '1 hour ago' +%s ; date +%s` 验证时间戳范围 | 时间范围无效、`tke_cls_instance_id` 错误，或新集群数据上报有 5-10 分钟延迟 | 检查时间范围覆盖有效区间；确认 InstanceId 正确；新集群等待 5-10 分钟 |
| 工作负载级/Pod 级无数据 | `tccli tke DescribeEKSContainerInstances --Filters '[{"Name":"Status","Values":["Running"]}]'` 检查 Pod | 无运行中 Pod，或 `GroupBy` 维度与 `Conditions` 维度不一致 | 确保有 Running 状态 Pod；检查 `GroupBy` 参数与 `Conditions` 中 Key 一致 |
| `CreateAlarmPolicy` 报 `InvalidParameterValue` | 检查 `MetricName` 是否在 QCE/EKS 指标列表内 | MetricName 不在 QCE/EKS 命名空间下，或 `Period` 不是 60/300 整数倍 | 对照本文「监控指标速查表」确认 MetricName 正确；Period 设为 60/300 |
| `DescribeAlarmPolicies` 返回空 | `tccli monitor DescribeAlarmPolicies --Namespaces '["QCE/EKS"]'` 全量查询 | `Namespaces` 参数格式错误 | 确认 `Namespaces` 格式为 `["QCE/EKS"]`（JSON 数组字符串） |
| CPU/内存使用量为 0 | `tccli tke DescribeEKSContainerInstances --Filters '[{"Name":"Status","Values":["Running"]}]'` | 集群中无运行中 Pod 或容器实例 | 确认有实际负载运行；参见 [容器实例](../../容器实例/tccli%20操作.md) 创建测试实例 |

## 下一步

- [容器实例](../../容器实例/tccli%20操作.md) — 创建和管理弹性容器实例
- [日志采集](../../日志采集/开通日志采集/tccli%20操作.md) — 开通日志采集功能
- [Prometheus 监控服务](../../Prometheus%20监控服务/数据采集配置/tccli%20操作.md) — 使用 Prometheus 采集自定义指标

## 控制台替代

容器服务控制台 → 集群 → 目标 Serverless 集群 → 监控（查看集群/节点/Pod 指标）。云监控控制台 → 告警管理 → 告警配置 → 告警策略 → 新建（创建告警策略，产品类型选择「容器服务（Serverless）」）。
