# 数据采集配置（tccli）

> 对照官方：[数据采集配置](https://cloud.tencent.com/document/product/457/71899) · page_id `71899`

## 概述

通过 `tccli monitor CreatePrometheusConfig` 配置 Prometheus 监控实例的数据采集规则，支持三种采集类型：**ServiceMonitor**（监控 Service）、**PodMonitor**（监控工作负载）、**RawJobs**（原生 Prometheus Job 配置）。配置后可通过 `DescribePrometheusConfig` 查看已有配置，通过 `ModifyPrometheusConfig` 修改配置。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已创建 Prometheus 监控实例（参见 [创建监控实例](../创建监控实例/tccli%20操作.md)）
- 目标集群已关联到监控实例（参见 [关联集群](../关联集群/tccli%20操作.md)）
- 目标 Service/工作负载 已在集群中运行并暴露 metrics 端口

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 新增 ServiceMonitor | `tccli monitor CreatePrometheusConfig --cli-input-json file://config.json` | 否 |
| 新增 PodMonitor | `tccli monitor CreatePrometheusConfig --cli-input-json file://config.json` | 否 |
| 新增 RawJob | `tccli monitor CreatePrometheusConfig --cli-input-json file://config.json` | 否 |
| 查看配置列表 | `tccli monitor DescribePrometheusConfig --InstanceId <InstanceId> --ClusterId <ClusterId> --ClusterType <ClusterType>` | 是 |
| 修改配置 | `tccli monitor ModifyPrometheusConfig --cli-input-json file://modify.json` | 否 |
| 查看采集目标 | `tccli monitor DescribePrometheusTargetsTMP --InstanceId <InstanceId> --ClusterId <ClusterId>` | 是 |

## 操作步骤

### 1. 查看当前采集配置

```bash
tccli monitor DescribePrometheusConfig --region ap-guangzhou --InstanceId prom-example01 --ClusterId cls-example --ClusterType tke
```

```json
{
    "Response": {
        "Config": "...",
        "ServiceMonitors": [
            {
                "Name": "default-kubelet",
                "Config": "apiVersion: monitoring.coreos.com/v1\nkind: ServiceMonitor\n..."
            }
        ],
        "PodMonitors": [],
        "RawJobs": [],
        "RequestId": "abc123-..."
    }
}
```

### 2. 新增 ServiceMonitor 配置

编辑 `config.json`，设置采集类型为 Service 监控：

```json
{
    "InstanceId": "prom-example01",
    "ClusterType": "tke",
    "ClusterId": "cls-example",
    "ServiceMonitors": [
        {
            "Name": "my-app-svc-monitor",
            "Config": "apiVersion: monitoring.coreos.com/v1\nkind: ServiceMonitor\nmetadata:\n  name: my-app-svc-monitor\n  namespace: default\nspec:\n  endpoints:\n  - interval: 30s\n    port: http-metrics\n    path: /metrics\n  selector:\n    matchLabels:\n      app: my-app"
        }
    ]
}
```

```bash
tccli monitor CreatePrometheusConfig --region ap-guangzhou --cli-input-json file://config.json
```

```json
{
    "Response": {
        "RequestId": "def456-..."
    }
}
```

**ServiceMonitor 关键字段：**

| 字段 | 说明 |
|------|------|
| `namespace` | Service 所在命名空间 |
| `endpoints[].port` | Service 端口名（非端口号） |
| `endpoints[].path` | 采集路径，默认 `/metrics` |
| `endpoints[].interval` | 采集频率，如 `30s`、`60s` |
| `selector.matchLabels` | 匹配 Service 的标签选择器 |

### 3. 新增 PodMonitor 配置

```json
{
    "InstanceId": "prom-example01",
    "ClusterType": "tke",
    "ClusterId": "cls-example",
    "PodMonitors": [
        {
            "Name": "my-app-pod-monitor",
            "Config": "apiVersion: monitoring.coreos.com/v1\nkind: PodMonitor\nmetadata:\n  name: my-app-pod-monitor\n  namespace: default\nspec:\n  podMetricsEndpoints:\n  - interval: 30s\n    port: http-metrics\n    path: /metrics\n  selector:\n    matchLabels:\n      app: my-app"
        }
    ]
}
```

### 4. 新增 RawJob 配置（原生 Prometheus Job）

```json
{
    "InstanceId": "prom-example01",
    "ClusterType": "tke",
    "ClusterId": "cls-example",
    "RawJobs": [
        {
            "Name": "custom-scrape-job",
            "Config": "scrape_configs:\n- job_name: custom-job\n  scrape_interval: 30s\n  static_configs:\n  - targets:\n    - 10.0.0.10:9090\n    - 10.0.0.11:9090"
        }
    ]
}
```

### 5. 修改已有配置

```bash
tccli monitor ModifyPrometheusConfig --region ap-guangzhou --cli-input-json file://modify.json
```

`modify.json` 结构与 `CreatePrometheusConfig` 相同，会覆盖目标实例-集群组合下的全部采集配置。

### 6. 挂载文件到采集器

当采集需要证书等文件时，可在 `prom-xxx` 命名空间中创建带指定 label 的 ConfigMap 或 Secret：

```yaml
# 在 prom-<InstanceId> 命名空间中创建
apiVersion: v1
kind: ConfigMap
metadata:
  name: scrape-certs
  namespace: prom-example01
  labels:
    prometheus.tke.tencent.cloud.com/scrape-mount: "true"
data:
  ca.crt: |
    -----BEGIN CERTIFICATE-----
    ...
```

文件会自动挂载到采集器路径：
- ConfigMap：`/etc/prometheus/configmaps/<configmap-name>/`
- Secret：`/etc/prometheus/secrets/<secret-name>/`

> kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971），以上挂载操作需通过控制台或脚本化 kubectl 完成。

## 验证

```bash
# 确认配置已生效
tccli monitor DescribePrometheusConfig --region ap-guangzhou --InstanceId prom-example01 --ClusterId cls-example --ClusterType tke --filter "ServiceMonitors[].Name, PodMonitors[].Name, RawJobs[].Name"

# 检查采集目标状态
tccli monitor DescribePrometheusTargetsTMP --region ap-guangzhou --InstanceId prom-example01 --ClusterId cls-example --filter "Targets[?State=='up']"
```

Targets 中 `State: up` 表示采集正常，`State: down` 表示采集失败，需检查端点可达性。

## 清理

删除配置需调用 `ModifyPrometheusConfig` 提交不含目标配置项的 JSON（空数组），或将不需要的配置从数组中移除后提交。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePrometheusConfig` 返回 `InvalidParameter` | 检查 Config 字段中的 YAML 格式 | ServiceMonitor/PodMonitor/RawJob 的 Config 字段 YAML 语法错误 | 用 `echo '...' \| yq` 验证 YAML 语法；确保 `\n` 换行符正确 |
| 配置提交后不生效 | `tccli monitor DescribePrometheusConfig --InstanceId <InstanceId> --ClusterId <ClusterId> --ClusterType <ClusterType>` | 配置有语法错误时静默忽略，或 `InstanceId`/`ClusterId` 不匹配 | 检查 YAML 缩进和字段名（区分大小写），确认 InstanceId/ClusterId 正确 |

### 配置成功但未采集到数据

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 采集目标状态 `down` | `tccli monitor DescribePrometheusTargetsTMP --InstanceId <InstanceId> --ClusterId <ClusterId>` | Service/Pod 端口名不正确或 `/metrics` 路径不可达 | 执行 `kubectl exec` 到采集器 Pod 内 `curl http://<service>:<port>/metrics` 验证端点可达性 |
| ServiceMonitor 未发现目标 | `kubectl get servicemonitor -n <Namespace>` 检查配置 | `selector.matchLabels` 与目标 Service 标签不匹配 | 修正 `matchLabels` 使其匹配目标 Service 的 labels |
| 跨命名空间采集失败 | 检查 ServiceMonitor 的 `namespaceSelector` 配置 | ServiceMonitor 默认只监控自身所在命名空间 | 在 Config 中添加 `namespaceSelector: any: true` 或 `matchNames: [...]` |

## 下一步

- [精简监控指标](../精简监控指标/tccli%20操作.md) 通过 `metricRelabelings` 过滤非必要指标，控制上报量
- [告警历史](../告警历史/tccli%20操作.md) 查看基于采集数据的告警记录

## 控制台替代

控制台：Prometheus 监控 → 实例详情 → 数据采集 → 集成容器服务 → 新建自定义监控（Service 监控 / 工作负载监控 / YAML 新增）。
