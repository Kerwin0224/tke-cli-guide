# 查看 TKE 集群控制面组件监控（tccli）

> 对照官方：[查看 TKE 集群控制面组件监控](https://cloud.tencent.com/document/product/457/116444) · page_id `116444`

## 概述

TKE 支持通过控制面日志和 Prometheus 监控两种方式查看控制面组件（kube-apiserver、kube-controller-manager、kube-scheduler、etcd）的运行状态。

**两种观测方式**：

| 方式 | 数据内容 | CLI 工具 |
|------|---------|---------|
| 控制面日志 | 组件审计日志、事件日志 | `tccli tke DescribeControlPlaneLogs` / `EnableControlPlaneLogs` / `DisableControlPlaneLogs` |
| Prometheus 监控 | 组件内部 metrics 指标 | `tccli monitor DescribePrometheusInstancesOverview` / `CreatePrometheusClusterAgent` / `DescribePrometheusClusterAgents` |

本文档介绍如何通过 CLI 开通、查看和管理控制面组件的监控能力。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 控制面日志功能需集群为托管集群（MANAGED_CLUSTER），cls-xxxxxxxx 满足此条件
- Prometheus 监控需开通腾讯云 Prometheus 监控服务

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-a1b2c3d4-5678-90ab-cdef-0123456789ab",
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
|-----------|-----|:--:|
| 开通控制面日志 | `tccli tke EnableControlPlaneLogs --ClusterId cls-xxxxxxxx` | 否（重复执行无副作用） |
| 查看控制面日志配置 | `tccli tke DescribeControlPlaneLogs --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 关闭控制面日志 | `tccli tke DisableControlPlaneLogs --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 否 |
| 查看 Prometheus 实例列表 | `tccli monitor DescribePrometheusInstancesOverview --region ap-guangzhou` | 是 |
| 关联 Prometheus 采集集群 | `tccli monitor CreatePrometheusClusterAgent --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 否（幂等接口，重复返回 success） |
| 查看 Prometheus 采集代理 | `tccli monitor DescribePrometheusClusterAgents --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |

## 操作步骤

### 1. 开通控制面日志

控制面日志将 kube-apiserver、kube-scheduler、kube-controller-manager 等组件的日志投递到 CLS（日志服务）。

```bash
tccli tke EnableControlPlaneLogs \
  --ClusterId cls-xxxxxxxx \
  --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-f6a7b8c9-0123-def4-5678-90abcdef0123"
}
```

### 2. 查看控制面日志配置

查看已开通的控制面日志组件和 CLS 投递配置：

```bash
tccli tke DescribeControlPlaneLogs --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-a7b8c9d0-1234-ef56-7890-abcdef012345",
  "ClusterId": "cls-xxxxxxxx",
  "ControlPlaneLogItems": [
    {
      "Component": "kube-apiserver",
      "LogsetId": "logset-abc123xxx",
      "TopicId": "topic-abc123xxx",
      "Enable": true
    },
    {
      "Component": "kube-scheduler",
      "LogsetId": "logset-abc123xxx",
      "TopicId": "topic-abc123xxx",
      "Enable": true
    }
  ]
}
```

### 3. 查看 Prometheus 实例列表

```bash
tccli monitor DescribePrometheusInstancesOverview --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-b8c9d0e1-2345-f678-9012-bcdef0123456",
  "TotalCount": 1,
  "InstanceSet": [
    {
      "InstanceId": "prom-abc123",
      "InstanceName": "tke-prom-instance",
      "Region": "ap-guangzhou",
      "Status": "running"
    }
  ]
}
```

### 4. 为集群关联 Prometheus 采集代理

将 TKE 集群关联到 Prometheus 监控实例，自动采集控制面组件指标：

```bash
tccli monitor CreatePrometheusClusterAgent \
  --InstanceId "prom-abc123" \
  --Agents '[{"ClusterId":"cls-xxxxxxxx","Region":"ap-guangzhou"}]' \
  --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-c9d0e1f2-3456-7890-1234-cdef01234567"
}
```

### 5. 查看集群关联的 Prometheus 代理状态

```bash
tccli monitor DescribePrometheusClusterAgents \
  --InstanceId "prom-abc123" \
  --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-d0e1f2a3-4567-8901-2345-def012345678",
  "Agents": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterType": "Managed cluster",
      "Region": "ap-guangzhou",
      "Status": "running"
    }
  ]
}
```

### 6. 关闭控制面日志（按需）

```bash
tccli tke DisableControlPlaneLogs --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-e1f2a3b4-5678-9012-3456-ef0123456789"
}
```

## 验证

### 验证控制面日志配置

```bash
tccli tke DescribeControlPlaneLogs --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "Details": [],
  "Name": "<Name>",
  "LogLevel": 0,
  "LogSetId": "<LogSetId>",
  "TopicId": "<TopicId>",
  "TopicRegion": "<TopicRegion>",
  "RequestId": "<RequestId>"
}
```

预期：`ControlPlaneLogItems` 中目标组件的 `Enable` 为 `true`。

### 验证 Prometheus 代理状态

```bash
tccli monitor DescribePrometheusClusterAgents \
  --InstanceId "prom-abc123" \
  --region ap-guangzhou
```

预期：集群 cls-xxxxxxxx 的 `Status` 为 `running`。

## 清理

### 关闭控制面日志

```bash
tccli tke DisableControlPlaneLogs --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

### 移除 Prometheus 采集代理（如有需要）

在 Prometheus 控制台或通过对应接口解绑集群。

## 排障

| 现象 | 处理 |
|------|------|
| EnableControlPlaneLogs 报错 `UnsupportedOperation` | 仅托管集群（MANAGED_CLUSTER）支持控制面日志；独立集群控制面由用户自行管理 |
| DescribeControlPlaneLogs 返回空 | 尚未开通控制面日志，先执行 `EnableControlPlaneLogs` |
| CreatePrometheusClusterAgent 报错 | 确认已开通腾讯云 Prometheus 监控服务且实例状态正常 |
| Prometheus 代理状态非 running | 等待 2-3 分钟初始化；检查集群网络与 Prometheus 实例 VPC 连通性 |
| 控制面日志未在 CLS 搜索到 | 开通后数据投递有 1-2 分钟延迟；确认 CLS 日志集和主题已创建 |

## 下一步

- [kube-apiserver 组件指标说明](../kube-apiserver 组件指标说明/tccli 操作.md) — kube-apiserver Prometheus 指标详解
- [kube-controller-manager 组件指标说明](../kube-controller-manager 组件指标说明/tccli 操作.md) — kube-controller-manager Prometheus 指标详解
- [kube-scheduler 组件指标说明](../kube-scheduler 组件指标说明/tccli 操作.md) — kube-scheduler Prometheus 指标详解
- [用户自建 Prometheus 采集控制面监控](../用户自建 Prometheus 采集控制面监控/tccli 操作.md) — 自建 Prometheus 方案
- [监控告警概述](../../基础监控与告警/监控告警概述/tccli 操作.md) — TKE 基础监控概念

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 集群详情 > 控制面组件监控，查看控制面日志和 Prometheus 监控数据。
