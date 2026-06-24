# 事件日志（tccli）

> 对照官方：[事件日志](https://cloud.tencent.com/document/product/457/32091) · page_id `32091`

## 概述

Kubernetes Events 记录集群运行和各类资源调度情况。TKE 支持开启事件持久化功能，将集群事件实时导出到 CLS（日志服务），支持在控制台进行检索分析。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已在 CLS 中创建或准备好日志集和日志主题
- 集群为标准集群

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
| 开启事件日志 | `tccli tke EnableEventPersistence` | 是 |
| 查询事件日志状态 | `tccli tke DescribeLogSwitches` | 是 |
| 关闭事件日志 | `tccli tke DisableEventPersistence` | 是 |

## 操作步骤

### 开启事件日志

```bash
tccli tke EnableEventPersistence --ClusterId cls-xxxxxxxx \
  --LogsetId logset-xxxxxxxx \
  --TopicId topic-xxxxxxxx \
  --TopicRegion ap-guangzhou \
  --region ap-guangzhou
```

```json
{
  "RequestId": "c9d0e1f2-a3b4-5678-9012-345678901234"
}
```

参数说明：
- `--LogsetId`: CLS 日志集 ID
- `--TopicId`: CLS 日志主题 ID
- `--TopicRegion`: CLS 日志所在地域（与集群地域一致）

### 查看事件日志状态

```bash
tccli tke DescribeLogSwitches --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "d0e1f2a3-b4c5-6789-0123-456789012345",
  "SwitchSet": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "AuditLog": {
        "LogsetId": "",
        "TopicId": "",
        "Enable": false
      },
      "EventLog": {
        "LogsetId": "logset-xxxxxxxx",
        "TopicId": "topic-xxxxxxxx",
        "Enable": true,
        "TopicRegion": "ap-guangzhou"
      }
    }
  ]
}
```

### 关闭事件日志

```bash
tccli tke DisableEventPersistence --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "RequestId": "e1f2a3b4-c5d6-7890-1234-567890123456"
}
```

如需同时删除 CLS 日志数据，添加 `--DeleteLogSetAndTopic true`。

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

确认 `EventLog.Enable` 为 `true`（开启后）或 `false`（关闭后）。

## 清理

### Control plane (tccli)

```bash
tccli tke DisableEventPersistence --ClusterId cls-xxxxxxxx --DeleteLogSetAndTopic true --region ap-guangzhou
```

## 排障

| 现象 | 处理 |
|------|------|
| 无法开启 | 确认集群为标准集群；每个日志集下最多 500 个日志主题 |
| 开启后无日志 | 确认 CLS 地域和日志集/主题配置正确；开启事件日志时默认为 Topic 开启索引 |
| 日志主题被覆盖 | 事件、审计、业务日志不可共用同一 topic |
| CLS logset 创建需 Tags | 子账号创建 CLS 资源可能受 CAM condition 标签策略限制（strategyId 274251632），创建时需带 `--Tags '[{"Key":"billing","Value":"<user>"}]'` |
| EnableEventPersistence 真跑验证 | 完整闭环：CreateLogset -> CreateTopic -> EnableEventPersistence 成功 -> DisableEventPersistence 成功 -> DeleteTopic + DeleteLogset |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达；事件日志操作仅通过 tccli TKE API |

## 下一步

- [事件仪表盘](../事件仪表盘/tccli 操作.md) — 事件可视化仪表盘
- [审计日志](../../审计管理/审计日志/tccli 操作.md) — 集群审计日志

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 选择集群 > 日志 > 事件日志 > 开启/关闭/检索分析。
