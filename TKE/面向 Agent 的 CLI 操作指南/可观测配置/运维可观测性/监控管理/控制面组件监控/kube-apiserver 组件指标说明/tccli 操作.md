# kube-apiserver 组件指标说明（tccli）

> 对照官方：[kube-apiserver 组件指标说明](https://cloud.tencent.com/document/product/457/116445) · page_id `116445`

## 概述

kube-apiserver 是 Kubernetes 控制面的核心组件，暴露了丰富的 Prometheus 指标用于监控 API 请求延迟、请求量、当前处理中的请求数、etcd 存储对象数量等。本文档列出关键指标及其 PromQL 查询方式，适用于已关联腾讯云 Prometheus 监控服务或自建 Prometheus 的 TKE 集群。

**重要**：以下 PromQL 查询需在 Prometheus 控制台或 Grafana 中执行，不直接通过 tccli。tccli 仅用于获取集群基本信息和 Prometheus 实例关联状态。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已为集群 cls-xxxxxxxx 开通控制面组件监控（参见 [查看 TKE 集群控制面组件监控](../查看 TKE 集群控制面组件监控/tccli 操作.md)）
- 已关联 Prometheus 实例（`CreatePrometheusClusterAgent`）

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
| 查看 API 请求延迟 | `histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, verb))` | — |
| 查看 API 请求总量 | `sum(rate(apiserver_request_total[5m])) by (verb, resource)` | — |
| 查看当前处理请求数 | `sum(apiserver_current_inflight_requests)` | — |
| 查看 etcd 存储对象数 | `apiserver_storage_objects` | — |
| 查看 etcd 请求延迟 | `histogram_quantile(0.99, sum(rate(etcd_request_duration_seconds_bucket[5m])) by (le, type))` | — |

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

### 2. API 请求延迟 (P99)

查看各 HTTP 动词的 P99 请求延迟：

```promql
histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le, verb))
```

| 指标 | 说明 |
|------|------|
| `apiserver_request_duration_seconds_bucket` | API 请求处理时间的直方图桶（Histogram） |

### 3. API 请求总量 (QPS)

按 HTTP 动词和资源类型分组查看请求速率：

```promql
sum(rate(apiserver_request_total{cluster="cls-xxxxxxxx"}[5m])) by (verb, resource)
```

| 指标 | 说明 |
|------|------|
| `apiserver_request_total` | API 请求总数（Counter） |

查看请求成功率：

```promql
sum(rate(apiserver_request_total{cluster="cls-xxxxxxxx",code!~"5.."}[5m])) by (verb)
/
sum(rate(apiserver_request_total{cluster="cls-xxxxxxxx"}[5m])) by (verb)
```

### 4. 当前处理中的请求数

```promql
sum(apiserver_current_inflight_requests{cluster="cls-xxxxxxxx"})
```

| 指标 | 说明 |
|------|------|
| `apiserver_current_inflight_requests` | 当前处理中的请求数（Gauge），按 requestKind（mutating/readOnly）分组 |

### 5. etcd 存储对象数

```promql
apiserver_storage_objects{cluster="cls-xxxxxxxx"}
```

| 指标 | 说明 |
|------|------|
| `apiserver_storage_objects` | etcd 中各资源类型存储的对象数量（Gauge） |

### 6. etcd 请求延迟

```promql
histogram_quantile(0.99, sum(rate(etcd_request_duration_seconds_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le, type))
```

| 指标 | 说明 |
|------|------|
| `etcd_request_duration_seconds_bucket` | apiserver 到 etcd 请求的延迟直方图（Histogram） |

### 7. API 请求响应大小

```promql
histogram_quantile(0.99, sum(rate(apiserver_response_sizes_bucket{cluster="cls-xxxxxxxx"}[5m])) by (le, verb))
```

### 8. 查看当前集群版本（数据面参考）

```bash
kubectl version --short 2>/dev/null || kubectl version
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

在 Prometheus 控制台或 Grafana 中执行上述任一 PromQL，确认返回非空数据。

## 清理

PromQL 查询为只读操作，无清理。

如需停止采集控制面指标，请参见 [查看 TKE 集群控制面组件监控](../查看 TKE 集群控制面组件监控/tccli 操作.md) 中的 Cleanup 部分。

## 排障

| 现象 | 处理 |
|------|------|
| PromQL 查询无数据 | 确认 Prometheus 代理状态为 running；新关联集群需等 2-3 分钟数据上报 |
| `apiserver_request_duration_seconds_bucket` 高延迟 | 检查 etcd 性能（`etcd_request_duration_seconds_bucket`）；减少 LIST 操作使用 FieldSelector |
| `apiserver_current_inflight_requests` 持续高位 | 可能存在过载或慢请求；增加 kube-apiserver `--max-requests-inflight` 参数 |
| `apiserver_storage_objects` 持续增长 | 检查是否有资源泄漏（未清理的 ConfigMap/Secret/CRD 等） |

## 下一步

- [kube-controller-manager 组件指标说明](../kube-controller-manager 组件指标说明/tccli 操作.md) — 控制器管理器 Prometheus 指标
- [kube-scheduler 组件指标说明](../kube-scheduler 组件指标说明/tccli 操作.md) — 调度器 Prometheus 指标
- [查看 TKE 集群控制面组件监控](../查看 TKE 集群控制面组件监控/tccli 操作.md) — 开通控制面监控
- [用户自建 Prometheus 采集控制面监控](../用户自建 Prometheus 采集控制面监控/tccli 操作.md) — 自建 Prometheus 方案

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 集群详情 > 控制面组件监控 > kube-apiserver，查看预置 Grafana 仪表盘。
