# kube-scheduler 组件指标说明（tccli）

> 对照官方：[kube-scheduler 组件指标说明](https://cloud.tencent.com/document/product/457/116446) · page_id `116446`

## 概述

kube-scheduler 是 Kubernetes 控制面中负责将等待调度的 Pod 分配至合适节点的组件。关键 Prometheus 指标包括待调度 Pod 数量、调度尝试次数、调度延迟等。本文档列出关键指标及其 PromQL 查询方式。

**重要**：以下 PromQL 查询需在 Prometheus 控制台或 Grafana 中执行。tccli 用于获取集群基本信息和 Prometheus 关联状态。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已为集群 cls-xxxxxxxx 开通控制面组件监控（参见 [查看 TKE 集群控制面组件监控](../查看 TKE 集群控制面组件监控/tccli 操作.md)）
- 已关联 Prometheus 实例

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

| 控制台操作 | CLI / PromQL | 幂等 |
|-----------|-----|:--:|
| 查看 Prometheus 关联状态 | `tccli monitor DescribePrometheusClusterAgents --InstanceId "prom-xxx" --region ap-guangzhou` | 是 |
| 查看待调度 Pod 数量 | `sum(scheduler_pending_pods{cluster="cls-xxxxxxxx"}) by (queue)` | — |
| 查看调度尝试次数 | `sum(rate(scheduler_pod_scheduling_attempts_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le)` | — |
| 查看调度延迟 (P99) | `histogram_quantile(0.99, sum(rate(scheduler_scheduling_duration_seconds_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le))` | — |
| 查看调度阶段延迟 | `histogram_quantile(0.99, sum(rate(scheduler_framework_extension_point_duration_seconds_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le, extension_point))` | — |
| 查看 REST 客户端请求速率 | `sum(rate(rest_client_requests_total{cluster="cls-xxxxxxxx"}[5m])) by (method, host)` | — |

## 操作步骤

### 1. 确认 Prometheus 监控已关联

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

### 2. 待调度 Pod 数量

查看调度队列中等待调度的 Pod 数量（按队列分组）：

```promql
sum(scheduler_pending_pods{cluster="cls-xxxxxxxx"}) by (queue)
```

| 指标 | 说明 |
|------|------|
| `scheduler_pending_pods` | 待调度 Pod 数量（Gauge），queue 标签值：`active`（正在调度）、`backoff`（退避中）、`unschedulable`（不可调度） |

正常情况：`active` 应保持低值；`backoff` 和 `unschedulable` 持续增长表示调度失败。

### 3. 调度尝试次数

查看调度器每 5 分钟内的调度尝试次数分布：

```promql
sum(rate(scheduler_pod_scheduling_attempts_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le)
```

```promql
# 统计每个 Pod 的调度尝试次数和最终结果
sum(rate(scheduler_pod_scheduling_attempts_total{cluster="cls-xxxxxxxx"}[5m])) by (result)
```

| 指标 | 说明 |
|------|------|
| `scheduler_pod_scheduling_attempts_bucket` | 调度尝试次数直方图（Histogram），按 `result`（scheduled/unschedulable/error）分组 |

### 4. 调度延迟 (P99)

从 Pod 进入调度队列到完成绑定的端到端延迟：

```promql
histogram_quantile(0.99, sum(rate(scheduler_scheduling_duration_seconds_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le))
```

| 指标 | 说明 |
|------|------|
| `scheduler_scheduling_duration_seconds_bucket` | 调度全流程耗时直方图（Histogram） |

### 5. 调度框架各阶段延迟

kube-scheduler 使用调度框架（Scheduling Framework），分为多个扩展点（PreFilter、Filter、Score、Reserve、Bind 等）。查看各阶段耗时：

```promql
histogram_quantile(0.99, sum(rate(scheduler_framework_extension_point_duration_seconds_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le, extension_point))
```

| 扩展点 | 说明 |
|--------|------|
| `PreFilter` | 预过滤：预处理 Pod 信息 |
| `Filter` | 过滤：排除不满足条件的节点 |
| `PostFilter` | 后过滤：无可调度节点时的处理 |
| `PreScore` | 预打分 |
| `Score` | 打分：为剩余节点打分 |
| `NormalizeScore` | 标准化打分 |
| `Reserve` | 预留：在绑定前预留资源 |
| `Permit` | 许可：等待 permit 批准 |
| `Bind` | 绑定：将 Pod 绑定到节点 |
| `WaitOnPermit` | 等待许可批准 |

### 6. REST 客户端请求速率

Scheduler 通过 REST Client 与 API Server 通信：

```promql
sum(rate(rest_client_requests_total{cluster="cls-xxxxxxxx"}[5m])) by (method, host)
```

### 7. 查看待调度 Pod（kubectl 文档参考）

```bash
kubectl get pods -A --field-selector=status.phase=Pending | grep -v NAME
# 查看 Pod 调度事件
kubectl describe pod <PendingPodName> -n <Namespace> | grep -A 20 "Events:"
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面验证

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou && \
tccli monitor DescribePrometheusClusterAgents \
  --InstanceId "prom-abc123" \
  --region ap-guangzhou
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

预期：集群状态 `Running`，代理状态 `running`。

### Prometheus 查询验证

在 Prometheus 控制台或 Grafana 中执行：

```promql
sum(scheduler_pending_pods{cluster="cls-xxxxxxxx"})
```

预期：返回非空数据；正常情况下总量应较低（接近 0）。

## 清理

PromQL 查询为只读操作，无清理。

## 排障

| 现象 | 处理 |
|------|------|
| `scheduler_pending_pods` 持续高位 | 节点资源不足或节点亲和性/污点不匹配；检查 `unschedulable` 队列中的 Pod |
| `scheduler_pod_scheduling_attempts` 中 `unschedulable` 高 | 集群可能无可用节点满足 Pod 的调度需求；检查节点资源、标签和污点 |
| `scheduler_scheduling_duration_seconds` P99 > 5s | 调度器可能因大量 Pod 或复杂调度约束而过载；检查 Scheduling Framework 各阶段耗时定位瓶颈 |
| `Bind` 阶段延迟高 | 可能因 API Server 高负载或网络延迟导致；检查 `apiserver_request_duration_seconds` |
| `scheduler_framework_extension_point_duration_seconds` Filter 阶段高 | 大量节点或复杂节点标签/亲和性规则；考虑使用 Pod 亲和性简化调度约束 |
| PromQL 无数据 | 确认 Prometheus 代理状态为 running；等待 2-3 分钟初始化 |

## 下一步

- [kube-apiserver 组件指标说明](../kube-apiserver 组件指标说明/tccli 操作.md) — API Server Prometheus 指标
- [kube-controller-manager 组件指标说明](../kube-controller-manager 组件指标说明/tccli 操作.md) — Controller Manager Prometheus 指标
- [查看 TKE 集群控制面组件监控](../查看 TKE 集群控制面组件监控/tccli 操作.md) — 开通控制面监控
- [用户自建 Prometheus 采集控制面监控](../用户自建 Prometheus 采集控制面监控/tccli 操作.md) — 自建 Prometheus 方案

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 集群详情 > 控制面组件监控 > kube-scheduler，查看预置 Grafana 仪表盘。
