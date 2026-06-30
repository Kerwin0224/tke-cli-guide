# 成本洞察（新版）（tccli）

> 对照官方：[成本洞察（新版）](https://cloud.tencent.com/document/product/457/75471) · page_id `75471`

## 概述

成本洞察（新版）基于 CLS（日志服务）和 COS（对象存储）账单数据，提供命名空间维度的成本可视化能力。通过采集集群中节点/Pod 的计费数据，按命名空间聚合展示资源消耗和成本分布，支持成本分摊和趋势分析。依赖 CLS 日志集和 CAM 服务角色授权。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已开通 CLS（日志服务）和 COS（对象存储）服务
- 集群已授予 CAM 角色 `TKE_QCSRole` 并绑定 CLS/COS 相关策略

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterNodeNum": 2
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建日志集 | `tccli cls CreateLogset --LogsetName <Name> --region ap-guangzhou` | 否 |
| 查询日志集 | `tccli cls DescribeLogsets --region ap-guangzhou` | 是 |
| 创建日志主题 | `tccli cls CreateTopic --LogsetId <LogsetId> --TopicName <Name> --region ap-guangzhou` | 否 |
| 查询日志主题 | `tccli cls DescribeTopics --region ap-guangzhou` | 是 |
| 查看 CAM 角色 | `tccli cam GetRole --RoleName TKE_QCSRole --region ap-guangzhou` | 是 |
| 查看角色策略列表 | `tccli cam ListAttachedRolePolicies --RoleName TKE_QCSRole --region ap-guangzhou` | 是 |
| 查看 Pod 资源用量 | `kubectl top pods -A` | 是 |
| 查看节点列表 | `kubectl get nodes` | 是 |
| 查看命名空间 | `kubectl get ns` | 是 |

## 操作步骤

### 1. 检查 CAM 服务角色

成本洞察（新版）依赖 `TKE_QCSRole` 角色授权访问 CLS 和 COS：

```bash
tccli cam GetRole --RoleName TKE_QCSRole --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "RoleId": "4611686018427387xxx",
  "RoleName": "TKE_QCSRole",
  "Description": "TKE 服务角色，用于访问 CLS、COS 等资源",
  "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"action\":\"name/sts:AssumeRole\",\"effect\":\"allow\",\"principal\":{\"service\":\"ccs.qcloud.com\"}}]}"
}
```

### 2. 查看角色绑定的策略

```bash
tccli cam ListAttachedRolePolicies --RoleName TKE_QCSRole --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "List": [
    {
      "PolicyId": 123456,
      "PolicyName": "QcloudCOSFullAccess",
      "AddTime": "2024-01-01 00:00:00",
      "CreateMode": 2
    },
    {
      "PolicyId": 123457,
      "PolicyName": "QcloudCLSFullAccess",
      "AddTime": "2024-01-01 00:00:00",
      "CreateMode": 2
    }
  ]
}
```

### 3. 创建 CLS 日志集

成本数据以日志形式写入 CLS，需先创建日志集：

```bash
tccli cls CreateLogset --LogsetName "tke-cost-insight" --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 4. 查询已创建的日志集

```bash
tccli cls DescribeLogsets --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "Logsets": [
    {
      "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "LogsetName": "tke-cost-insight",
      "CreateTime": "2024-01-01 00:00:00",
      "TopicCount": 0
    }
  ],
  "TotalCount": 1
}
```

### 5. 创建日志主题

在日志集下创建日志主题用于存储成本数据：

```bash
tccli cls CreateTopic --LogsetId <LogsetId> --TopicName "tke-cost-data" --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 6. 查询日志主题

```bash
tccli cls DescribeTopics --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "Topics": [
    {
      "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "TopicName": "tke-cost-data",
      "CreateTime": "2024-01-01 00:00:00"
    }
  ],
  "TotalCount": 1
}
```

### 7. 查看集群命名空间分布

```bash
kubectl get ns
```

```text
NAME              STATUS   AGE
default           Active   30d
kube-system       Active   30d
kube-public       Active   30d
crane-system      Active   4d
app-production    Active   7d
```

### 8. 查看命名空间级别 Pod 资源用量

成本洞察按命名空间聚合，以命名空间为维度查看 Pod 用量：

```bash
kubectl top pods -A --sort-by=cpu | head -15
```

```text
NAMESPACE       NAME                              CPU(cores)   MEMORY(bytes)
app-production  api-server-5c8d7f9b6-xyzab        250m         512Mi
default         nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
kube-system     coredns-6d8cf9b84-xm9lk           8m           45Mi
crane-system    craned-6d8cf9b84-xm9lk            10m          80Mi
```

### 9. 查看节点资源容量（成本分摊基础）

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}CPU: {.status.allocatable.cpu}{"\t"}Memory: {.status.allocatable.memory}{"\n"}{end}'
```

```text
node-10-0-1-5    CPU: 4    Memory: 8192Mi
node-10-0-1-6    CPU: 4    Memory: 8192Mi
node-10-0-1-7    CPU: 8    Memory: 16384Mi
```

## 验证

### Control plane (tccli)

```bash
# 确认 CAM 角色存在且绑定策略
tccli cam GetRole --RoleName TKE_QCSRole --region ap-guangzhou && \
tccli cam ListAttachedRolePolicies --RoleName TKE_QCSRole --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "RoleId": "4611686018427387xxx",
  "RoleName": "TKE_QCSRole"
}
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "List": [
    {
      "PolicyName": "QcloudCOSFullAccess"
    },
    {
      "PolicyName": "QcloudCLSFullAccess"
    }
  ]
}
```

```bash
# 确认 CLS 日志集和主题已创建
tccli cls DescribeLogsets --region ap-guangzhou && \
tccli cls DescribeTopics --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "Logsets": [
    {
      "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "LogsetName": "tke-cost-insight"
    }
  ],
  "TotalCount": 1
}
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "Topics": [
    {
      "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "TopicName": "tke-cost-data"
    }
  ],
  "TotalCount": 1
}
```

### Data plane (kubectl)

```bash
kubectl get ns && kubectl top pods -A 2>/dev/null | head -10
```

```text
NAME              STATUS   AGE
app-production    Active   7d
crane-system      Active   4d
default           Active   30d
kube-public       Active   30d
kube-system       Active   30d

NAMESPACE       NAME                              CPU(cores)   MEMORY(bytes)
app-production  api-server-5c8d7f9b6-xyzab        250m         512Mi
default         nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
```

## 清理

### Control plane (tccli)

成本洞察（新版）产生的 CLS 和 COS 资源需手动清理：

```bash
# 删除日志主题
tccli cls DeleteTopic --TopicId <TopicId> --region ap-guangzhou

# 删除日志集
tccli cls DeleteLogset --LogsetId <LogsetId> --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> 注意：COS 存储桶中账单数据需前往 [COS 控制台](https://console.cloud.tencent.com/cos) 手动清理。

## 排障

| 现象 | 处理 |
|------|------|
| 成本数据为空 | 确认 CLS 日志集/主题已正确创建且在控制台已关联；确认 CAM 角色 TKE_QCSRole 已绑定 QcloudCLSFullAccess 和 QcloudCOSFullAccess |
| 无命名空间成本分布 | 确认集群有命名空间和 Pod 在运行；成本数据需等待账单数据写入 CLS 后才有展示 |
| CAM 角色不存在 | 首次使用需在控制台授权 TKE 服务角色，执行 `tccli cam GetRole --RoleName TKE_QCSRole` 确认状态 |
| CLS 日志集/主题创建失败 | 确认已开通 CLS 服务且账户余额充足 |
| 旧版成本洞察功能缺失 | 新版聚焦命名空间级成本分摊，与旧版（节点/Pod 级）互补；如需节点详细计费，请结合 [成本洞察](../成本洞察/tccli 操作.md) 使用 |

## 下一步

- [成本洞察](../成本洞察/tccli 操作.md) — 旧版成本洞察，节点/Pod 级成本可视化
- [Node Map](../Node Map/tccli 操作.md) — 节点可视化热力图
- [Workload Map](../Workload Map/tccli 操作.md) — 工作负载可视化热力图
- [Request 智能推荐](../成本优化/Request 智能推荐/tccli 操作.md) — 智能推荐资源配置

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > TKE Insight > 成本洞察（新版），配置 CLS 日志集和 COS 存储桶，查看命名空间级成本可视化面板。
