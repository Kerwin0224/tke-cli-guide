# 审计日志（tccli）

> 对照官方：[审计日志](https://cloud.tencent.com/document/product/457/48346) · page_id `48346`

## 概述

TKE 审计日志基于 Kubernetes Audit 对 kube-apiserver 产生的可配置策略的 JSON 结构日志进行记录、存储及检索。记录每一次对 kube-apiserver 的访问事件（用户、管理员或系统组件影响集群的活动）。开启审计日志需重启 kube-apiserver，建议不要频繁开关。

每一条审计日志包含元数据（metadata）、请求内容（requestObject）和响应内容（responseObject）三部分。审计级别有 None、Metadata、Request、RequestResponse 四个级别。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已在 CLS 中创建或准备好日志集和日志主题
- 独立集群需保证 Master 节点约 1GiB 本地存储

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "1e8b2c3d-4a5f-6789-abcd-ef0123456789",
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
| 开启审计日志 | `tccli tke EnableClusterAudit` | 是 |
| 关闭审计日志 | `tccli tke DisableClusterAudit` | 是 |
| 查看日志开关状态 | `tccli tke DescribeLogSwitches` | 是 |

## 操作步骤

### 开启审计日志

```bash
tccli tke EnableClusterAudit --ClusterId cls-xxxxxxxx \
  --LogsetId logset-xxxxxxxx \
  --TopicId topic-xxxxxxxx \
  --region ap-guangzhou
```

```json
{
  "RequestId": "e5f6a7b8-c9d0-1234-5678-901234567890"
}
```

### 查看审计日志状态

```bash
tccli tke DescribeLogSwitches --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "f6a7b8c9-d0e1-2345-6789-012345678901",
  "SwitchSet": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "AuditLog": {
        "LogsetId": "logset-xxxxxxxx",
        "TopicId": "topic-xxxxxxxx",
        "Enable": true
      }
    }
  ]
}
```

### 关闭审计日志

```bash
tccli tke DisableClusterAudit --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "RequestId": "a7b8c9d0-e1f2-3456-7890-123456789012"
}
```

如需同时删除 CLS 中的日志数据：

```bash
tccli tke DisableClusterAudit --ClusterId cls-xxxxxxxx --DeleteLogSetAndTopic true --region ap-guangzhou
```

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeLogSwitches --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
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

确认 `AuditLog.Enable` 为 `true`（开启后）。

## 清理

### Control plane (tccli)

```bash
tccli tke DisableClusterAudit --ClusterId cls-xxxxxxxx --DeleteLogSetAndTopic true --region ap-guangzhou
```

## 排障

| 现象 | 处理 |
|------|------|
| 开启需重启 kube-apiserver | 建议不要频繁开关；独立集群占用 Master 约 1GiB 本地存储 |
| 某些操作不记录日志 | TKE 默认策略：get/list/watch 记录 Request 级别；对 secrets/configmaps/tokenreviews 记录 Metadata 级别；某些 kube-proxy/kubelet 操作不记录 |
| 日志主题被覆盖 | 事件、审计、业务日志不可共用同一 topic |
| 审计级别说明 | None 不记录；Metadata 记录元数据；Request 包含请求消息体；RequestResponse 包含所有信息 |
| EnableClusterAudit 真跑失败 | 真跑返回可能报 **FailedOperation.KubernetesPatchOperationError**："failed to patch resource from user kube-apiserver (failed to apply master CR)"。根因：托管集群 audit 配置需通过 kube-apiserver CR 下发，当前 API server CR 不可写（集群刚创建，控制面组件未完全初始化）。建议等待集群完全就绪后重试 |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达；审计日志操作仅通过 tccli TKE API |

## 下一步

- [审计仪表盘](../审计仪表盘/tccli 操作.md) — 审计日志可视化仪表盘
- [事件日志](../../事件管理/事件日志/tccli 操作.md) — 集群事件持久化

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 选择集群 > 日志 > 审计日志 > 开启/关闭。
