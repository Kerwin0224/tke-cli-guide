# 审计仪表盘

> 对照官方：[审计仪表盘](https://cloud.tencent.com/document/product/457/58243) · page_id `58243`

## 概述

TKE Serverless 集群的审计仪表盘提供 Kubernetes 审计日志的可视化展示，帮助运维人员快速了解集群操作行为和安全态势。

审计功能覆盖：
- **操作审计**：记录对 K8s API Server 的所有请求（谁、何时、对什么资源做了什么操作）。
- **安全分析**：识别异常操作、高频失败请求、未授权访问尝试。
- **合规审计**：满足安全合规要求的操作记录保留和检索。
- **趋势分析**：操作频率趋势、请求来源分布、操作类型分布。

> **依赖**：审计仪表盘依赖集群已开启[审计日志](../审计日志/tccli%20操作.md)采集功能。未开启时仪表盘无数据。

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

# 3. 确认集群存在
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，集群存在
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看审计仪表盘概览 | 无直接 CLI API（控制台可视化） | — |
| 查看集群基本信息 | `DescribeEKSClusters` | 是 |
| 查看审计日志详情 | 无 CLI API（依赖 CLS 审计日志） | — |
| 开启审计日志采集 | 控制台操作（无对应 CLI API） | — |
| 配置审计日志投递（CLS） | 控制台操作（无对应 CLI API） | — |

> **注意**：审计仪表盘为控制台纯可视化功能，无直接 CLI/Dashboard API 可获取仪表盘数据。以下 tccli 和 kubectl 操作用于确认审计采集状态和数据面验证。

## 操作步骤

### 步骤 1：确认集群状态正常

```bash
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0] | {ClusterId, Status, K8SVersion}'
# expected: Status == "Running"，集群正常运行
```

**预期输出**：

```json
{
    "ClusterId": "CLUSTER_ID",
    "Status": "Running",
    "K8SVersion": "1.30.0"
}
```

### 步骤 2：检查审计相关组件状态

> **需 kubectl 可达环境**

```bash
# 检查 kube-system 命名空间中的审计相关组件
kubectl get pods -n kube-system | grep -E 'audit|cls'
# expected: 如有审计日志采集组件，显示 Running 状态

# 查看审计配置
kubectl get configmap -n kube-system audit-policy -o yaml 2>/dev/null
# expected: 返回审计策略配置（如已配置）
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：通过 kubectl 查看近期的审计事件（间接验证）

> 审计数据存储在 CLS（日志服务），需通过 CLS API 或控制台查询。以下为通过 kubectl 间接验证集群操作活动的方法：

```bash
# 查看集群 Events（非审计日志，但可验证活动）
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
# expected: 返回集群最近的 Events

# 查看 API Server 版本（确认审计功能可用）
kubectl version
# expected: 返回 Client 和 Server K8s 版本
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：审计仪表盘指标类别

| 类别 | 展示内容 | 来源 |
|------|---------|------|
| 操作总览 | 总操作次数、成功/失败占比 | CLS 审计日志 |
| 操作类型分布 | Create/Delete/Update/Get/List 占比 | CLS 审计日志 |
| 用户操作 Top | Top 操作用户/ServiceAccount | CLS 审计日志 |
| 资源操作 Top | 被操作最多的资源类型 | CLS 审计日志 |
| 请求来源分布 | 源 IP、User-Agent 分布 | CLS 审计日志 |
| 失败请求分析 | 失败请求原因（403/404/409） | CLS 审计日志 |
| 操作趋势 | 按时间聚合的操作量趋势 | CLS 审计日志 |

## 验证

```bash
# 验证集群可正常访问
kubectl get ns
# expected: 返回命名空间列表

# 验证近期有 API 操作记录（通过 Events 间接验证）
kubectl get events -A --sort-by='.lastTimestamp' | wc -l
# expected: 返回 Events 数量（大于 0 说明集群有活动）

# 验证集群状态正常
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].Status'
# expected: "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

## 清理

本页面为只读操作，无需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 审计仪表盘页面无数据 | 检查是否已开启[审计日志采集](../审计日志/tccli%20操作.md) | 集群未开启审计日志采集功能 | 在控制台开启审计日志采集，并配置投递到 CLS |
| `DescribeEKSClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeEKSClusters` 权限 | 联系主账号授予相应 CAM 权限 |
| kubectl 报 `Unable to connect to the server` | `cat ~/.kube/config` 检查 server 地址 | kubeconfig 未配置或过期 | 重新获取：`tccli tke DescribeEKSClusterCredential --ClusterId CLUSTER_ID --region <Region>` |
| `kubectl get events` 返回空 | 集群无近期活动或 Events 已过期 | Events 默认保留 1 小时，过期自动清理 | 正常现象。执行任意 kubectl 操作后可观察新 Events 生成 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 开启了审计日志但仪表盘数据延迟大 | 检查 CLS 日志投递延迟 | CLS 日志采集和索引有 1-3 分钟延迟 | 等待 3-5 分钟后刷新仪表盘 |
| 仪表盘展示的统计与预期不符 | 检查审计策略是否排除了某些操作 | 审计策略配置了过滤，部分操作未被审计 | 检查并调整审计策略配置（控制台操作） |
| 审计日志存储成本过高 | 检查 CLS 存储用量 | 集群操作频繁产生大量审计日志 | 在 CLS 控制台设置日志保留期和索引优化 |

## 下一步

- [审计日志](../审计日志/tccli%20操作.md) — 配置审计日志采集和查询
- [事件仪表盘](../../事件管理/事件仪表盘/tccli%20操作.md) — 查看集群 Events 仪表盘
- [事件日志](../../事件管理/事件日志/tccli%20操作.md) — 查看集群事件日志

## 控制台替代

[TKE 控制台 → Serverless 集群 → 运维中心 → 审计仪表盘](https://console.cloud.tencent.com/tke2/ecluster)：控制台提供完整的审计仪表盘可视化界面，展示操作总览、趋势、Top 分析等。需先开启审计日志采集功能。
