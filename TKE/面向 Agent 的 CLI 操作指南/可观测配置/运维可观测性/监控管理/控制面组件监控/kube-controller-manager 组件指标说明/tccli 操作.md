# kube-controller-manager 组件指标说明（tccli）

> 对照官方：[kube-controller-manager 组件指标说明](https://cloud.tencent.com/document/product/457/116447) · page_id `116447`

## 概述

kube-controller-manager 是 Kubernetes 控制面中运行多种控制器的组件，负责将集群从当前状态向期望状态进行调谐（Reconcile）。关键 Prometheus 指标包括工作队列深度、调谐速率、与 API Server 的 REST 调用延迟等。本文档列出关键指标及其 PromQL 查询方式。

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
| 查看工作队列深度 | `sum(workqueue_depth{cluster="cls-xxxxxxxx"}) by (name)` | — |
| 查看工作队列新增任务速率 | `sum(rate(workqueue_adds_total{cluster="cls-xxxxxxxx"}[5m])) by (name)` | — |
| 查看队列中项目保留时间 | `histogram_quantile(0.99, sum(rate(workqueue_queue_duration_seconds_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le, name))` | — |
| 查看 REST 客户端请求速率 | `sum(rate(rest_client_requests_total{cluster="cls-xxxxxxxx"}[5m])) by (method, host)` | — |
| 查看列表操作数量 | `sum(rate(apiserver_request_total{cluster="cls-xxxxxxxx",verb="LIST"}[5m])) by (resource)` | — |

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

### 2. 工作队列深度

工作队列深度反映各控制器待处理的调谐任务数量，持续高位可能表示控制器跟不上变化速率：

```promql
sum(workqueue_depth{cluster="cls-xxxxxxxx"}) by (name)
```

| 指标 | 说明 |
|------|------|
| `workqueue_depth` | 工作队列中待处理项目数（Gauge），按 controller name 分组 |

### 3. 工作队列新增任务速率

控制面每发生变化（如 Pod 创建、Deployment 更新），对应控制器会将调谐任务入队：

```promql
sum(rate(workqueue_adds_total{cluster="cls-xxxxxxxx"}[5m])) by (name)
```

| 指标 | 说明 |
|------|------|
| `workqueue_adds_total` | 工作队列累计新增任务数（Counter） |

### 4. 工作队列处理延迟 (P99)

任务在队列中排队等待处理的时间：

```promql
histogram_quantile(0.99, sum(rate(workqueue_queue_duration_seconds_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le, name))
```

| 指标 | 说明 |
|------|------|
| `workqueue_queue_duration_seconds_bucket` | 任务在队列中的排队持续时间（Histogram） |

### 5. REST 客户端请求速率

Controller Manager 通过 REST Client 与 API Server 通信：

```promql
sum(rate(rest_client_requests_total{cluster="cls-xxxxxxxx"}[5m])) by (method, host)
```

| 指标 | 说明 |
|------|------|
| `rest_client_requests_total` | controller-manager 发起的 HTTP 请求总数（Counter），按 method/host/code 分组 |

### 6. 调谐操作速率

查看各控制器的调谐结果：

```promql
sum(rate(workqueue_retries_total{cluster="cls-xxxxxxxx"}[5m])) by (name)
```

| 指标 | 说明 |
|------|------|
| `workqueue_retries_total` | 重试次数（Counter），高重试率可能表示调谐逻辑有错误 |

```promql
sum(rate(workqueue_unfinished_work_seconds{cluster="cls-xxxxxxxx"}[5m])) by (name)
```

| 指标 | 说明 |
|------|------|
| `workqueue_unfinished_work_seconds` | 未完成工作的持续时长（Gauge），检测控制器是否卡住 |

### 7. 查看集群内各控制器状态（kubectl 文档参考）

```bash
kubectl get pods -n kube-system -l component=kube-controller-manager
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

在 Prometheus 控制台或 Grafana 中执行以下基线查询：

```promql
sum(workqueue_depth{cluster="cls-xxxxxxxx"})
```

预期：返回非空数据；正常情况下各队列深度应保持在较低水平。

## 清理

PromQL 查询为只读操作，无清理。

## 排障

| 现象 | 处理 |
|------|------|
| `workqueue_depth` 持续高位 | 控制器处理速率低于变更速率；检查控制器日志和调谐逻辑是否有错误 |
| `workqueue_retries_total` 高 | 调谐失败率过高；检查近期的资源变更是否正确，是否存在 RBAC 权限不足 |
| `rest_client_requests_total` 出现大量 409 Conflict | 资源的乐观并发冲突；检查是否有多个控制器/客户端同时修改同一资源 |
| `workqueue_unfinished_work_seconds` 保持非零 | 控制器可能卡住；检查 Pod 日志 `kubectl logs -n kube-system -l component=kube-controller-manager` |
| PromQL 无数据 | 确认 Prometheus 代理状态为 running；等待 2-3 分钟初始化 |

## 下一步

- [kube-apiserver 组件指标说明](../kube-apiserver 组件指标说明/tccli 操作.md) — API Server Prometheus 指标
- [kube-scheduler 组件指标说明](../kube-scheduler 组件指标说明/tccli 操作.md) — 调度器 Prometheus 指标
- [查看 TKE 集群控制面组件监控](../查看 TKE 集群控制面组件监控/tccli 操作.md) — 开通控制面监控
- [用户自建 Prometheus 采集控制面监控](../用户自建 Prometheus 采集控制面监控/tccli 操作.md) — 自建 Prometheus 方案

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 集群详情 > 控制面组件监控 > kube-controller-manager，查看预置 Grafana 仪表盘。
