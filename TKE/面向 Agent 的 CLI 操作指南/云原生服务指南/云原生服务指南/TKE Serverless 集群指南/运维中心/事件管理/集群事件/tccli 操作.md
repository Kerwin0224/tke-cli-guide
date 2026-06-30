# 集群事件（tccli）

> 对照官方：[事件日志](https://cloud.tencent.com/document/product/457/50988) · page_id `50988`

## 概述

TKE Serverless 集群事件日志通过 CLS（日志服务）持久化存储，支持跨地域投递。开启后自动为日志主题启用索引，可配合 CLS SearchLog 检索事件。每个日志集最多创建 500 个日志/指标主题。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已配置 `tccli`，可用 `tccli tke help` 验证
- 已有 TKE Serverless 集群（`ClusterId`），状态为 `Running`
- 已开通 CLS 服务（可在 [CLS 控制台](https://console.cloud.tencent.com/cls) 确认）
- `tccli cls help` 可用（需安装 CLS 产品 SDK）

> **注意：** kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）。事件日志的管理和检索均通过 tccli 和 CLS API 完成。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 开通事件日志 | `tccli tke EnableEventPersistence --ClusterId <ClusterId> --LogsetId <LogsetId> --TopicId <TopicId>` | 是 |
| 创建日志集 | `tccli cls CreateLogset --LogsetName <LogsetName> --region <Region>` | 否 |
| 创建日志主题 | `tccli cls CreateTopic --LogsetId <LogsetId> --TopicName <TopicName> --region <Region>` | 否 |
| 更新日志集 | `tccli cls ModifyLogset --LogsetId <LogsetId> --LogsetName <NewName> --region <Region>` | 是 |
| 更新日志主题 | `tccli cls ModifyTopic --TopicId <TopicId> --TopicName <NewName> --region <Region>` | 是 |
| 查询日志主题 | `tccli cls DescribeTopics --Filters '[{"Key":"logsetId","Values":["<LogsetId>"]}]' --region <Region>` | 是 |
| 检索事件 | `tccli cls SearchLog --TopicId <TopicId> --Query "<Query>" --From <From> --To <To> --region <Region>` | 是 |
| 关闭事件日志 | `tccli tke DisableEventPersistence --ClusterId <ClusterId>` | 是 |
| 删除日志主题 | `tccli cls DeleteTopic --TopicId <TopicId> --region <Region>` | 否 |

## 操作步骤

### 1. 选择日志投递地域并创建日志集

CLS 支持跨地域投递，可选择与集群不同地域的 CLS。以下以广州地域为例。

```bash
tccli cls CreateLogset \
    --LogsetName "tke-serverless-events-logset-example" \
    --region ap-guangzhou
```

```output
{
    "LogsetId": "logset-example",
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 2. 创建日志主题

在日志集下创建日志主题。注意：每个日志集最多创建 500 个日志/指标主题。

```bash
tccli cls CreateTopic \
    --LogsetId "logset-example" \
    --TopicName "tke-serverless-cluster-events-topic-example" \
    --region ap-guangzhou
```

```output
{
    "TopicId": "topic-example",
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

### 3. 开启集群事件日志持久化

将集群事件关联到已创建的 CLS 日志主题。开启后，索引将自动启用。

```bash
tccli tke EnableEventPersistence \
    --ClusterId "cls-example" \
    --LogsetId "logset-example" \
    --TopicId "topic-example" \
    --region ap-guangzhou
```

```output
{
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 4. 查看事件日志

使用 CLS SearchLog 检索集群事件。时间范围使用 Unix 时间戳（秒）。

**示例 1：检索最近 1 小时所有 Warning 级别事件**

```bash
tccli cls SearchLog \
    --TopicId "topic-example" \
    --Query "type:Warning" \
    --From $(date -d '1 hour ago' +%s) \
    --To $(date +%s) \
    --Limit 20 \
    --region ap-guangzhou
```

```output
{
    "Context": "",
    "ListOver": true,
    "Analysis": false,
    "Results": [
        {
            "Time": 1689600001,
            "TopicId": "topic-example",
            "TopicName": "tke-serverless-cluster-events-topic-example",
            "LogJson": "{\"type\":\"Warning\",\"reason\":\"FailedScheduling\",\"message\":\"0/3 nodes are available: 3 Insufficient cpu.\",\"namespace\":\"default\",\"name\":\"nginx-deployment-7b8f9c6d5-abcde\",\"firstTimestamp\":\"2024-07-17T10:00:01Z\",\"lastTimestamp\":\"2024-07-17T10:00:01Z\",\"count\":1}"
        }
    ],
    "RequestId": "d4e5f6a7-b8c9-0123-defa-123456789013"
}
```

**示例 2：检索 Pod CrashLoopBackOff 事件**

```bash
tccli cls SearchLog \
    --TopicId "topic-example" \
    --Query "reason:BackOff OR reason:CrashLoopBackOff" \
    --From $(date -d '24 hours ago' +%s) \
    --To $(date +%s) \
    --Limit 50 \
    --Sort "desc" \
    --region ap-guangzhou
```

```output
{
    "Context": "",
    "ListOver": true,
    "Analysis": false,
    "Results": [
        {
            "Time": 1689686401,
            "TopicId": "topic-example",
            "TopicName": "tke-serverless-cluster-events-topic-example",
            "LogJson": "{\"type\":\"Warning\",\"reason\":\"BackOff\",\"message\":\"Back-off restarting failed container nginx in pod nginx-deployment-7b8f9c6d5-xyz\",\"namespace\":\"default\",\"name\":\"nginx-deployment-7b8f9c6d5-xyz\",\"firstTimestamp\":\"2024-07-17T12:00:01Z\",\"lastTimestamp\":\"2024-07-17T14:00:01Z\",\"count\":15}"
        }
    ],
    "RequestId": "e5f6a7b8-c9d0-1234-efab-123456789014"
}
```

**示例 3：检索指定命名空间的事件**

```bash
tccli cls SearchLog \
    --TopicId "topic-example" \
    --Query "namespace:kube-system" \
    --From $(date -d '1 hour ago' +%s) \
    --To $(date +%s) \
    --Limit 20 \
    --region ap-guangzhou
```

### 5. 更新日志集或日志主题（可选）

如需更名或调整配置：

```bash
# 更新日志集名称
tccli cls ModifyLogset \
    --LogsetId "logset-example" \
    --LogsetName "tke-serverless-events-v2" \
    --region ap-guangzhou

# 更新日志主题名称
tccli cls ModifyTopic \
    --TopicId "topic-example" \
    --TopicName "tke-serverless-cluster-events-v2" \
    --region ap-guangzhou
```

### 6. 关闭事件日志

```bash
tccli tke DisableEventPersistence \
    --ClusterId "cls-example" \
    --region ap-guangzhou
```

```output
{
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-123456789015"
}
```

## 验证

### Control plane (tccli)

**验证事件持久化状态：**

```bash
tccli tke DescribeEKSClusters \
    --ClusterIds '["cls-example"]' \
    --region ap-guangzhou \
    | jq '.Clusters[0].EnableEventPersistence'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": [],
  "K8SVersion": "<K8SVersion>",
  "Status": "<Status>"
}
```

预期返回 `true`。

**验证日志主题可检索：**

```bash
tccli cls DescribeTopics \
    --Filters '[{"Key":"logsetId","Values":["logset-example"]}]' \
    --region ap-guangzhou \
    | jq '.Topics[] | select(.TopicId == "topic-example") | .TopicName'
```

预期返回日志主题名称。

**验证事件已写入：**

```bash
tccli cls SearchLog \
    --TopicId "topic-example" \
    --Query "*" \
    --From $(date -d '24 hours ago' +%s) \
    --To $(date +%s) \
    --Limit 1 \
    --region ap-guangzhou \
    | jq '.Results | length'
```

预期返回 `> 0`（若有事件产生）。

## 清理

### Control plane (tccli)

**关闭事件持久化，清理 CLS 资源：**

```bash
# 1. 关闭事件持久化
tccli tke DisableEventPersistence --ClusterId "cls-example" --region ap-guangzhou

# 2. 删除日志主题
tccli cls DeleteTopic --TopicId "topic-example" --region ap-guangzhou

# 3. 删除日志集（需确保日志集下无其他主题）
tccli cls DeleteLogset --LogsetId "logset-example" --region ap-guangzhou
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableEventPersistence` 报 `InvalidParameterValue.ClusterId` | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'` 检查集群 | ClusterId 不正确或集群状态非 `Running` | 确认 ClusterId 正确且集群状态为 `Running` |
| `CreateTopic` 报 `LimitExceeded` | `tccli cls DescribeTopics --Filters '[{"Key":"logsetId","Values":["<LogsetId>"]}]'` 查数 | 日志集下主题数达 500 上限（此为环境限制） | 删除不再使用的主题或创建新日志集 |
| `SearchLog` 返回空结果 | `tccli tke DescribeEKSClusters --ClusterIds '["<ClusterId>"]'` 检查 `EnableEventPersistence` | 事件持久化未开启，或查询时间范围未覆盖事件产生时间 | 确认事件持久化已开启为 `true`；新开启后等待数分钟再查询 |
| `SearchLog` 报 `InvalidParameterValue.TopicId` | `tccli cls DescribeTopics --TopicId <TopicId>` 确认存在 | TopicId 不正确或日志主题已被删除 | 用 `tccli cls DescribeTopics` 重新获取正确 TopicId |
| `DescribeTopics` 未找到日志主题 | `tccli cls DescribeLogsets` 列出所有日志集 | `logsetId` 参数不正确 | 确认 `Filters` 中 `logsetId` 正确 |
| 跨地域投递失败 | `tccli cls DescribeLogsets --region <TargetRegion>` 检查 CLS 服务 | 目标地域 CLS 服务未开通 | 在目标地域开通 CLS 服务后再配置事件投递 |

## 下一步

- [查看监控](../../监控和告警/查看监控/tccli%20操作.md) — 查看 TKE Serverless 集群监控指标
- [使用 CLS 告警异常资源](../../../../实践教程/日志/使用%20CLS%20告警异常资源/tccli%20操作.md) — 基于事件日志创建告警

## 控制台替代

容器服务控制台 → 集群 → 目标 Serverless 集群 → 基本信息 → 事件日志。CLS 控制台 → 检索分析 → 选择日志主题 → 输入检索语句。
