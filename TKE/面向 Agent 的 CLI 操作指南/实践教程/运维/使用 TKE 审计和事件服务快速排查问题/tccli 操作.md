# 使用 TKE 审计和事件服务快速排查问题（tccli）

> 对照官方：[使用 TKE 审计和事件服务快速排查问题](https://cloud.tencent.com/document/product/457/50677) · page_id `50677`

## 概述

同时启用集群审计（EnableClusterAudit）和事件持久化（EnableEventPersistence），结合 CLS 检索快速定位集群问题。审计记录 API 操作，事件记录 Pod/Node 生命周期变化。

## 前置条件

- 已开通 [CLS](https://console.cloud.tencent.com/cls)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 开启审计 | `tccli tke EnableClusterAudit --ClusterId <ClusterId>` | 是 |
| 开启事件持久化 | `tccli tke EnableEventPersistence --ClusterId <ClusterId>` | 是 |
| 查看事件 | `kubectl get events -A --sort-by=.lastTimestamp` | 是 |
| 查看审计日志 | CLS 检索 | 是 |

## 操作步骤

### 1. 开启审计

```bash
tccli tke EnableClusterAudit --region ap-guangzhou --ClusterId <ClusterId> --LogsetId <LogsetId> --TopicId <TopicId>
```

### 2. 开启事件持久化

```bash
tccli tke EnableEventPersistence --region ap-guangzhou --ClusterId <ClusterId> --LogsetId <LogsetId> --TopicId <TopicId>
```

### 3. 常见排查场景

| 问题 | CLS 检索 | 说明 |
|------|---------|------|
| Pod 被谁删除 | `objectRef.resource:pods AND verb:delete` | 审计日志 |
| 镜像拉取失败 | `reason:Failed OR reason:ErrImagePull` | 事件 |
| 节点 NotReady | `involvedObject.kind:Node AND reason:NodeNotReady` | 事件 |

## 验证

```bash
tccli tke DescribeLogSwitches --region ap-guangzhou --ClusterId <ClusterId>
kubectl get events -n default --sort-by=.lastTimestamp | tail -5
```

```json
{
  "SwitchSet": [],
  "Audit": "<Audit>",
  "Enable": true,
  "ErrorMsg": "<ErrorMsg>",
  "LogsetId": "<LogsetId>",
  "Status": "<Status>",
  "TopicId": "<TopicId>",
  "TopicRegion": "<TopicRegion>"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableClusterAudit` 返回 `LogSetNotFound` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | 集群未 Running 或 CLS Logset/Topic 未创建 | 确认集群 Running；在 CLS 控制台创建 Logset/Topic 后重试 |
| CLS 检索无审计日志 | `tccli tke DescribeLogSwitches --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 审计未真正开启或时间范围错误 | 确认 `DescribeLogSwitches` 返回已开启；扩大 CLS 检索时间范围 |
| `kubectl get events` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl |

## 清理

```bash
tccli tke DisableClusterAudit --region ap-guangzhou --ClusterId <ClusterId>
tccli tke DisableEventPersistence --region ap-guangzhou --ClusterId <ClusterId>
```

## 下一步

- [使用集群审计排查问题](../使用集群审计排查问题/tccli%20操作.md)
- [CLS 告警异常资源](../../日志/使用%20CLS%20告警异常资源/tccli%20操作.md)

## 控制台替代

控制台：集群 → 日志管理 → 审计日志 / 事件日志。
