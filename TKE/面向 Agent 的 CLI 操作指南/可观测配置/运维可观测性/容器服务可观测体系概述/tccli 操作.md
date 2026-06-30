# 容器服务可观测体系概述（tccli）

> 对照官方：[容器服务可观测体系概述](https://cloud.tencent.com/document/product/457/118975) · page_id `118975`

## 概述

TKE 容器服务可观测体系覆盖五大层次：日志管理、审计管理、事件管理、监控管理和告警管理。各层数据均可通过 tccli 或 kubectl 操作，日志/审计/事件数据统一投递到 CLS（日志服务），监控告警基于腾讯云可观测平台（TCOP）或自建 Prometheus。

| 层次 | 功能 | 核心 API | 说明 |
|------|------|---------|------|
| 日志管理 | 容器日志采集 | `tke:InstallLogAgent` / `tke:CreateCLSLogConfig` | LogListener DaemonSet，支持 CLS/Kafka 消费端，可通过控制台或 CRD 配置 |
| 审计管理 | kube-apiserver 审计 | `tke:EnableClusterAudit` / `tke:DisableClusterAudit` | 记录对 apiserver 的访问事件，四级审计策略（None/Metadata/Request/RequestResponse） |
| 事件管理 | Kubernetes Events 持久化 | `tke:EnableEventPersistence` / `tke:DisableEventPersistence` | 将集群 Event 实时导出到 CLS，支持检索和仪表盘 |
| 监控管理 | 基础监控 + Prometheus | `tke:DescribePrometheusInstancesOverview` / `monitor:DescribeAlarmPolicies` | TCOP 基础监控（CPU/内存/网络等），Prometheus 自定义监控 |
| 告警管理 | 告警策略 + 风险推送 | `monitor:CreateAlarmPolicy` / `tke:UpdateAddon` (kubejarvis) | TCOP 告警策略 + kubejarvis 健康检查风险推送 |

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。

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
| 查看集群列表 | `tccli tke DescribeClusters` | 是 |
| 查看日志开关状态 | `tccli tke DescribeLogSwitches` | 是 |
| 查看 Prometheus 实例概览 | `tccli tke DescribePrometheusInstancesOverview` | 是 |
| 查看控制面组件日志 | `tccli tke DescribeControlPlaneLogs` | 是 |

## 操作步骤

### 查看集群基本信息

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

### 查看日志开关状态（审计、事件、业务日志）

```bash
tccli tke DescribeLogSwitches --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "2f9c3d4e-5b6f-7890-bcde-f01234567890",
  "SwitchSet": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "AuditLog": {
        "LogsetId": "logset-xxxxxxxx",
        "TopicId": "topic-xxxxxxxx",
        "Enable": false
      },
      "EventLog": {
        "LogsetId": "",
        "TopicId": "",
        "Enable": false,
        "TopicRegion": ""
      },
      "LogAgent": {
        "Enable": false
      }
    }
  ]
}
```

### 查看 Prometheus 实例概览

```bash
tccli tke DescribePrometheusInstancesOverview --region ap-guangzhou
```

```json
{
  "RequestId": "3f0d4e5f-6c7f-8901-cdef-123456789012",
  "Instances": [
    {
      "InstanceId": "prom-xxxxxxxx",
      "InstanceName": "tke-monitor",
      "InstanceStatus": "running",
      "Region": "ap-guangzhou",
      "ClusterId": "cls-xxxxxxxx"
    }
  ]
}
```

### 查看控制面组件日志（kube-apiserver / kube-scheduler / kube-controller-manager 等）

```bash
tccli tke DescribeControlPlaneLogs \
  --ClusterId cls-xxxxxxxx \
  --region ap-guangzhou
```

```json
{
  "RequestId": "4f1e5f6f-7d8f-9012-def0-234567890123",
  "ControlPlaneLogItems": [
    {
      "ComponentName": "kube-apiserver",
      "LogsetId": "logset-cp-xxxxxx",
      "TopicId": "topic-cp-xxxxxx",
      "Enable": false
    }
  ]
}
```

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

确认集群 Status 为 `Running`，可观测相关组件可正常启用。

## 清理

可观测体系为基础设施功能，无需清理。各子功能（日志采集、审计、事件持久化）分别开启和关闭。

## 排障

| 现象 | 处理 |
|------|------|
| 集群不可达 | 确认集群状态为 Running；托管集群公网端点可能受 CAM 策略限制 |
| 日志开关查询为空 | 集群可能尚未开启任何日志功能，SwitchSet 返回 Enable=false 为正常状态 |
| Prometheus 实例列表为空 | 未创建 Prometheus 监控实例；需通过控制台或 API 先创建 |
| 控制面日志未开启 | 通过 `tke:EnableControlPlaneLog` 开启对应组件的日志投递 |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。可观测配置全部通过 tccli API 操作，不依赖 kubectl |

## 下一步

- [日志采集概述](../日志管理/日志采集概述/tccli 操作.md) — 日志采集基本概念与开启
- [审计日志](../审计管理/审计日志/tccli 操作.md) — 集群审计日志开启/关闭
- [事件日志](../事件管理/事件日志/tccli 操作.md) — 集群事件持久化
- [告警管理](../告警管理/tccli 操作.md) — 配置告警策略

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 选择集群 > 在集群详情页左侧导航栏依次进入日志、监控、运维中心等模块，查看和配置各可观测组件。
