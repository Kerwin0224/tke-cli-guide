# 精简监控指标（tccli）

> 对照官方：[精简监控指标](https://cloud.tencent.com/document/product/457/71900) · page_id `71900`

## 概述

Prometheus 监控服务按数据点数量计费。通过精简监控指标——在 `metricRelabelings`（CRD 配置）或 `metric_relabel_configs`（原生 Job）中配置 `keep`/`drop` 规则——**过滤非必要指标、降低采集频率**，可大幅减少上报数据量和费用。

TMP 提供 100+ 基础免费指标（详见按量付费免费指标文档），超出免费额度的付费指标按数据点计费。免费指标存储时长调整为 15 天（自 2022-10-27 起）。

> kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971），以下 YAML 配置需通过控制台或脚本化 kubectl 写入集群。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已创建 Prometheus 监控实例（参见 [创建监控实例](../创建监控实例/tccli%20操作.md)）
- 目标集群已关联到监控实例（参见 [关联集群](../关联集群/tccli%20操作.md)）
- 已配置数据采集规则（参见 [数据采集配置](../数据采集配置/tccli%20操作.md)）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看当前采集配置 | `tccli monitor DescribePrometheusConfig --InstanceId <InstanceId> --ClusterId <ClusterId> --ClusterType <ClusterType>` | 是 |
| 修改采集配置（含 metricRelabelings） | `tccli monitor ModifyPrometheusConfig --cli-input-json file://modify.json` | 否 |
| 查看基础免费指标 | `tccli monitor DescribeBaseMetrics` | 是 |
| 按需勾选/取消基础指标 | 控制台操作 `指标详情` 页 | — |
| 排除命名空间/CRD | 添加 label `tps-skip-monitor: "true"` | 否 |

## 操作步骤

### 1. 查看当前采集配置

```bash
tccli monitor DescribePrometheusConfig --region ap-guangzhou --InstanceId prom-example01 --ClusterId cls-example --ClusterType tke
```

### 2. 精简 ServiceMonitor 指标（metricRelabelings）

在 ServiceMonitor YAML 的 `endpoints` 中添加 `metricRelabelings` 字段。只保留指定指标（`action: keep`）：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-svc-monitor
  namespace: default
spec:
  endpoints:
  - interval: 30s
    port: http-metrics
    path: /metrics
    metricRelabelings:
    - sourceLabels: ["__name__"]
      regex: "kube_node_info|kube_node_role|kube_pod_info"
      action: keep
    - sourceLabels: ["__name__"]
      regex: "kube_pod_status_phase|kube_pod_container_status_restarts_total"
      action: keep
```

**丢弃指定指标（`action: drop`）：**

```yaml
    metricRelabelings:
    - sourceLabels: ["__name__"]
      regex: "go_.*|process_.*"
      action: drop
```

### 3. 精简 PodMonitor 指标

PodMonitor 使用与 ServiceMonitor 相同的 `metricRelabelings` 配置：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app-pod-monitor
  namespace: default
spec:
  podMetricsEndpoints:
  - interval: 30s
    port: http-metrics
    path: /metrics
    metricRelabelings:
    - sourceLabels: ["__name__"]
      regex: "http_requests_total|http_request_duration_.*"
      action: keep
```

### 4. 精简原生 Job 指标（metric_relabel_configs）

原生 Prometheus Job 使用 `metric_relabel_configs`（注意：加 `s`，用下划线连接）：

```yaml
scrape_configs:
- job_name: custom-job
  scrape_interval: 60s
  static_configs:
  - targets:
    - 10.0.0.10:9090
  metric_relabel_configs:
  - source_labels: ["__name__"]
    regex: "app_important_.*"
    action: keep
```

### 5. 降低采集频率以减少数据量

```yaml
# 不重要的指标增大 interval（如 300s），
# 相比默认 15s 可降低约 20 倍数据量
endpoints:
- interval: 300s    # 5分钟采集一次
  port: http-metrics
  path: /metrics
```

注意：`scrapeTimeout` 必须满足 `scrapeTimeout <= interval`。

### 6. 通过 ModifyPrometheusConfig 提交精简后的配置

将包含精简规则的完整 YAML 填入 `Config` 字段：

```json
{
    "InstanceId": "prom-example01",
    "ClusterType": "tke",
    "ClusterId": "cls-example",
    "ServiceMonitors": [
        {
            "Name": "my-app-svc-monitor",
            "Config": "apiVersion: monitoring.coreos.com/v1\nkind: ServiceMonitor\nmetadata:\n  name: my-app-svc-monitor\n  namespace: default\nspec:\n  endpoints:\n  - interval: 60s\n    port: http-metrics\n    path: /metrics\n    metricRelabelings:\n    - sourceLabels: [\"__name__\"]\n      regex: \"http_requests_total|http_request_duration_.*\"\n      action: keep"
        }
    ]
}
```

```bash
tccli monitor ModifyPrometheusConfig --region ap-guangzhou --cli-input-json file://modify.json
```

### 7. 排除整个命名空间的监控

为命名空间添加 label `tps-skip-monitor: "true"`，TMP 将跳过该命名空间下的所有 ServiceMonitor 和 PodMonitor：

```bash
kubectl label namespace <NamespaceName> tps-skip-monitor="true"
```

### 8. 排除特定 ServiceMonitor/PodMonitor 对象

为指定 CRD 资源添加 label `tps-skip-monitor: "true"`：

```bash
kubectl label servicemonitor <MonitorName> -n <Namespace> tps-skip-monitor="true"
kubectl label podmonitor <MonitorName> -n <Namespace> tps-skip-monitor="true"
```

> kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971），以上 label 操作需通过控制台完成。

## 验证

```bash
# 确认精简后的配置已生效
tccli monitor DescribePrometheusConfig --region ap-guangzhou --InstanceId prom-example01 --ClusterId cls-example --ClusterType tke --filter "ServiceMonitors[].Config"
```

在返回的 `Config` 字段中搜索 `metricRelabelings` 或 `metric_relabel_configs` 确认规则已包含。

## 清理

如需恢复全量采集，通过 `ModifyPrometheusConfig` 提交不含 `metricRelabelings` / `metric_relabel_configs` 的原始配置即可。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 精简不生效 | `tccli monitor DescribePrometheusConfig --InstanceId <InstanceId> --ClusterId <ClusterId> --ClusterType <ClusterType>` 检查返回的 Config 内容 | 字段名错误：CRD 用 `metricRelabelings`（驼峰），Job 用 `metric_relabel_configs`（下划线）。误用了 `relabelings` 或 `relabel_configs` | 确认 YAML 中使用正确字段名；CRD ServiceMonitor/PodMonitor 用 `metricRelabelings`，RawJob 用 `metric_relabel_configs` |
| 指标全被丢弃 | `tccli monitor DescribePrometheusTargetsTMP --InstanceId <InstanceId> --ClusterId <ClusterId>` 检查采集目标 | `regex` 和 `action: keep` 未匹配到任何目标指标 | 先换用 `action: drop` 排除少量已知指标逐步验证，确认 regex 语法正确后改回 `action: keep` |
| `scrapeTimeout` 报错 | 检查 YAML 中 `scrapeTimeout` 与 `interval` 的值 | 配置约束：`scrapeTimeout` 必须小于等于 `interval` | 调整 `scrapeTimeout` 使其 `<= interval`，如 interval=60s 则 scrapeTimeout 设为 `30s` |

## 下一步

- [计费方式和资源使用](../计费方式和资源使用/tccli%20操作.md) 查看用量和费用估算
- [告警历史](../告警历史/tccli%20操作.md) 查看精简后指标的告警状态

## 控制台替代

控制台：Prometheus 监控 → 实例详情 → 数据采集配置 → 编辑 → 按控制台 UI 勾选/取消基础指标（免费/付费标记），或在 YAML 编辑器中添加 `metricRelabelings` 规则。
