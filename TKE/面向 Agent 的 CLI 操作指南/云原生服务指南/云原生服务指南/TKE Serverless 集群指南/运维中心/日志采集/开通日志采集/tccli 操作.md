# 开通日志采集（tccli）

> 对照官方：[开启日志采集](https://cloud.tencent.com/document/product/457/61092) · page_id `61092`

## 概述

为 TKE Serverless 集群开通日志采集功能。开通后，集群内 Pod 的标准输出（stdout/stderr）和容器内文件日志可被采集到 CLS（日志服务）或外部 Kafka，供检索、分析和告警。首次使用需要为 TKE 完成 CAM 服务授权（首次授权），再对目标集群开启日志采集开关。

开通日志采集涉及两个阶段：

1. **首次授权**：CAM 角色 `TKE_QCSLinkedRoleInEKSLog` 绑定策略 `QcloudAccessForTKELinkedRoleInEKSLog`，授权 TKE 向 CLS 写入日志。
2. **开启日志采集**：在目标 Serverless 集群上启用日志采集功能，关联 CLS 日志集和日志主题。

由于 TKE Serverless 集群无独立 Worker 节点，日志采集由托管采集组件自动完成，无需用户部署 DaemonSet。

## 前置条件

- 已创建 TKE Serverless 集群且状态为 `Running`，参见 [创建集群](../../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)。
- [环境准备](../../../../环境准备.md)：`tccli` 已安装并配置 `region`（本文以 `ap-guangzhou` 为例）。
- 已开通 [CLS（日志服务）](https://console.cloud.tencent.com/cls)，目标地域有可用 CLS 服务。
- 如果使用子账号操作，需 CAM 权限 `cam:CreateRole`、`cam:AttachRolePolicy`、`cls:CreateLogset`、`cls:CreateTopic`、`tke:EnableEksLog`。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查看当前集群日志状态 | `DescribeEKSClusters` → `LogCollector` 字段 | 是 |
| 首次授权（CAM 服务角色） | `cam CreateRole` + `cam AttachRolePolicy` | 是（角色已存在则跳过） |
| 查看授权状态 | `cam GetRole --RoleName TKE_QCSLinkedRoleInEKSLog` | 是 |
| 创建 CLS 日志集 | `cls CreateLogset` | 否（同名可能冲突） |
| 创建 CLS 日志主题 | `cls CreateTopic` | 否（同名可能冲突） |
| 开启集群日志采集 | `EnableEKSLog`（控制台操作，CLI 适配） | 否（重复开启幂等） |

## 操作步骤

### 1. 首次授权：CAM 服务角色

首次为 TKE Serverless 集群启用日志采集时，需在 CAM 中创建服务角色 `TKE_QCSLinkedRoleInEKSLog` 并绑定策略 `QcloudAccessForTKELinkedRoleInEKSLog`。

#### 1.1 检查角色是否已存在

```bash
tccli cam GetRole \
    --RoleName TKE_QCSLinkedRoleInEKSLog \
    --region ap-guangzhou \
    --output json
```

角色已存在时的示例输出：

```json
{
    "RoleId": "4611686018427387904",
    "RoleName": "TKE_QCSLinkedRoleInEKSLog",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"action\":\"sts:AssumeRole\",\"effect\":\"allow\",\"principal\":{\"service\":\"eks.tke.cloud.tencent.com\"}}]}",
    "Description": "TKE Serverless 日志采集服务角色",
    "AddTime": "2024-01-15T10:30:00Z",
    "UpdateTime": "2024-01-15T10:30:00Z"
}
```

若返回 `RoleName`，跳至 **步骤 2**。若返回 `ResourceNotFound`，继续执行 **步骤 1.2**。

#### 1.2 创建服务角色

```bash
tccli cam CreateRole \
    --RoleName TKE_QCSLinkedRoleInEKSLog \
    --PolicyDocument '{"version":"2.0","statement":[{"action":"sts:AssumeRole","effect":"allow","principal":{"service":"eks.tke.cloud.tencent.com"}}]}' \
    --Description "TKE Serverless 日志采集服务角色" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "RoleId": "4611686018427387904",
    "RequestId": "abc123-def456-ghi789"
}
```

#### 1.3 绑定策略

```bash
tccli cam AttachRolePolicy \
    --PolicyId 0 \
    --AttachRoleName TKE_QCSLinkedRoleInEKSLog \
    --PolicyName QcloudAccessForTKELinkedRoleInEKSLog \
    --region ap-guangzhou \
    --output json
```

> **注意**：策略 `QcloudAccessForTKELinkedRoleInEKSLog` 为腾讯云预设策略，包含 `cls:PushLog`、`cls:CreateLogset`、`cls:CreateTopic` 等权限。若 `AttachRolePolicy` 需要 `PolicyId`，请先通过 `cam ListPolicies --keyword QcloudAccessForTKELinkedRoleInEKSLog` 获取。

示例输出：

```json
{
    "RequestId": "abc123-def456-ghi789"
}
```

### 2. 准备 CLS 日志集与日志主题

在开启日志采集前，需确保目标地域已有 CLS 日志集和日志主题。日志集用于组织日志主题，日志主题是日志数据存储和检索的最小单元。

#### 2.1 查看已有日志集

```bash
tccli cls DescribeLogsets \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TotalCount": 1,
    "Logsets": [
        {
            "LogsetId": "logset-example-001",
            "LogsetName": "tke-serverless-logs",
            "CreateTime": "2024-01-15T10:00:00Z",
            "TopicCount": 0
        }
    ]
}
```

#### 2.2 创建日志集（如需要）

```bash
tccli cls CreateLogset \
    --LogsetName "tke-serverless-logs" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "LogsetId": "logset-example-001",
    "RequestId": "abc123-def456-ghi789"
}
```

#### 2.3 创建日志主题（如需要）

日志主题的日志保存时间默认为 30 天，可根据需求调整。

```bash
tccli cls CreateTopic \
    --LogsetId "logset-example-001" \
    --TopicName "tke-serverless-stdout" \
    --Period 30 \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TopicId": "topic-example-001",
    "RequestId": "abc123-def456-ghi789"
}
```

### 3. 开启集群日志采集

日志采集功能为集群级别开关。确认集群状态正常后，在控制台 **集群 → 运维中心 → 日志采集** 中开启。由于 TKE Serverless 的日志采集由托管组件接管，不暴露独立的 `EnableEKSLog` API 给外部调用者，当前开启操作主要通过控制台执行。

#### 3.1 确认集群信息

```bash
tccli tke DescribeEKSClusters \
    --ClusterIds '["cls-example"]' \
    --region ap-guangzhou \
    --output json
```

示例输出（关注 `LogCollector` 相关字段）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "my-serverless-cluster",
            "Status": "Running",
            "VpcId": "vpc-example",
            "K8SVersion": "1.28.3",
            "CreatedTime": "2024-01-15T10:30:00Z"
        }
    ]
}
```

#### 3.2 控制台操作路径

1. 进入 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 集群列表。
2. 点击目标 Serverless 集群 ID（`cls-example`）。
3. 左侧导航栏选择 **运维中心 → 日志采集**。
4. 点击 **开启日志采集**，选择 CLS 日志集和日志主题，确认开启。

> **控制台操作说明：** 当前 `tccli tke` 未暴露独立的 `EnableEKSLog` 或 `ModifyEKSClusterLog` 接口，因此集群级别的日志采集开关只能通过控制台操作。日志采集规则（CRD LogConfig）可通过 `kubectl` 管理，见 [使用日志采集](../使用日志采集/tccli%20操作.md)（page_id `56751`）。

## 验证

### Control plane (tccli)

- `cam GetRole --RoleName TKE_QCSLinkedRoleInEKSLog` 返回角色信息，确认授权完成。
- `cls DescribeLogsets` 返回目标日志集，确认 CLS 资源就绪。
- `cls DescribeTopics --LogsetId logset-example-001` 返回目标日志主题。
- 在控制台 **日志采集** 页面确认日志采集开关为 **已开启**。

### Data plane (kubectl)

集群内 Pod 的标准输出日志应自动上报到 CLS：

```bash
kubectl get logconfigs -A
```

```text
NAME  STATUS  AGE
...
```

`kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）` 时，通过控制台 CLS 检索页面验证日志数据：

```bash
tccli cls SearchLog \
    --TopicId "topic-example-001" \
    --From 0 \
    --To $(date +%s)000 \
    --Query "" \
    --Limit 1 \
    --region ap-guangzhou \
    --output json
```

示例输出（若有日志数据）：

```json
{
    "Results": [
        {
            "Time": 1705310400000,
            "TopicId": "topic-example-001",
            "LogJson": "{\"cluster_id\":\"cls-example\",\"container_name\":\"nginx\",\"namespace\":\"default\",\"pod_name\":\"nginx-7b9f8c6d5-x2k9m\",\"content\":\"2024/01/15 10:40:00 [notice] 1#1: start worker process 2\"}"
        }
    ],
    "TotalCount": 1
}
```

## 清理

日志采集功能为集群级开关，关闭后日志采集组件将停止工作，已采集的日志数据不影响生命周期。若需清理 CLS 日志主题与日志集：

```bash
tccli cls DeleteTopic \
    --TopicId "topic-example-001" \
    --region ap-guangzhou \
    --output json

tccli cls DeleteLogset \
    --LogsetId "logset-example-001" \
    --region ap-guangzhou \
    --output json
```

> **注意**：删除日志主题将永久删除其中的日志数据，请确认已备份或不再需要。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| `cam CreateRole` 报 `RoleNameInUse` | 角色 `TKE_QCSLinkedRoleInEKSLog` 已存在 | 跳过创建，直接绑定策略；或执行 `cam GetRole` 确认角色状态 |
| `cam AttachRolePolicy` 报 `PolicyIdInvalid` | 未指定正确的 `PolicyId` | 执行 `cam ListPolicies --keyword QcloudAccessForTKELinkedRoleInEKSLog --output json` 获取 `PolicyId` |
| `cls CreateTopic` 报 `LogsetNotExist` | `LogsetId` 不存在 | 确认 `LogsetId` 格式正确，或先创建日志集 |
| 控制台日志采集开关置灰不可点击 | 集群状态非 `Running`、或 CLS 服务未开通 | 确认集群 `Running`，确认 CLS 已开通 |
| 开启后 CLS 中无日志数据 | LogConfig CRD 未创建或采集规则不匹配 | 检查 LogConfig 资源，见 [日志采集配置](../日志采集配置/tccli%20操作.md)（page_id `56750`） |
| 子账号无权限 | CAM 策略不足 | 授予 `cam:CreateRole`、`cam:AttachRolePolicy`、`cls:*`、`tke:DescribeEKSClusters` |

## 下一步

- [使用日志采集](../使用日志采集/tccli%20操作.md)（page_id `56751`） — 配置 CRD LogConfig 将日志投递到 Kafka
- [采集容器内日志](../采集容器内日志/tccli%20操作.md)（page_id `56320`） — 通过控制台/CLI 配置采集规则
- [日志采集配置](../日志采集配置/tccli%20操作.md)（page_id `56750`） — YAML CRD 配置 CLS 日志采集
- [监控和告警](../../监控和告警/查看监控/tccli%20操作.md) — 配置日志监控指标与告警

## 控制台替代

[容器服务控制台 → 集群 → 运维中心 → 日志采集](https://console.cloud.tencent.com/tke2/cluster) — 点击 **开启日志采集**，选择 CLS 日志集与日志主题后确认。
