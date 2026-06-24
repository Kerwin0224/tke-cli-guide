# 腾讯云 Prometheus 一键关联监控容器服务（tccli）

> 对照官方：[腾讯云 Prometheus 一键关联监控容器服务](https://cloud.tencent.com/document/product/457/90923) · page_id `90923`

## 概述

使用腾讯云托管 Prometheus 监控 TKE 集群，相比自建 Prometheus 运维成本更低。在创建集群或已有集群上绑定 Prometheus 实例后，自动采集集群指标数据，通过 Grafana 查看预设面板，并配置告警策略。

## 前置条件

- 已创建 [腾讯云 Prometheus 监控实例](https://console.cloud.tencent.com/tke2/prometheus2)。
- 已有 TKE 标准集群或 Serverless 集群。
- 已 [连接集群](../../../集群配置/集群管理/连接集群/tccli%20操作.md)。

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,ClusterStatus:ClusterStatus}"
```

```
{
  "ClusterId": "cls-example",
  "ClusterStatus": "Running"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看 Prometheus 实例 | `tccli monitor DescribePrometheusInstances` | 是 |
| 关联集群 | `tccli tke CreatePrometheusClusterAgent` | 是（已关联则跳过） |
| 查看绑定状态 | `tccli tke DescribePrometheusClusterAgents` | 是 |
| 查看采集配置 | `tccli tke DescribePrometheusConfig` | 是 |
| 创建告警策略 | `tccli tke CreatePrometheusAlertPolicy` | 否 |

## 操作步骤

### 1. 查看 Prometheus 实例

```bash
tccli monitor DescribePrometheusInstances --region ap-guangzhou --output json \
  --filter "InstanceSet[0].{InstanceId:InstanceId,InstanceName:InstanceName}"
```

```
{
  "InstanceId": "prom-xxxxxxxx",
  "InstanceName": "tke-monitor"
}
```

### 2. 绑定已有集群到 Prometheus

```bash
tccli tke CreatePrometheusClusterAgent \
  --InstanceId prom-xxxxxxxx \
  --Agents '[{"Region":"ap-guangzhou","ClusterId":"cls-example","ClusterType":"tke"}]' \
  --region ap-guangzhou
```

```
{
  "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

### 3. 查看关联状态

```bash
tccli tke DescribePrometheusClusterAgents \
  --InstanceId prom-xxxxxxxx \
  --region ap-guangzhou --output json \
  --filter "Agents[?ClusterId=='cls-example'].{ClusterId:ClusterId,Status:Status}"
```

```
[
  {
    "ClusterId": "cls-example",
    "Status": "running"
  }
]
```

### 4. 配置数据采集

在 Prometheus 实例详情页的**数据采集 → 集成容器服务**页面，单击**数据采集配置**，勾选需采集的指标。

通过 API 查看/修改采集配置：

```bash
tccli tke DescribePrometheusConfig --InstanceId prom-xxxxxxxx --ClusterId cls-example --ClusterType tke --region ap-guangzhou --output json
```

```json
{
  "Config": "<Config>",
  "ServiceMonitors": [],
  "Name": "<Name>",
  "TemplateId": "<TemplateId>",
  "PodMonitors": []
}
```

### 5. 配置告警策略

```bash
tccli tke CreatePrometheusAlertPolicy \
  --InstanceId prom-xxxxxxxx \
  --AlertRule '{"Name":"HighCPU","Rules":[{"Alert":"NodeCPUHigh","Expr":"100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100) > 80","For":"5m","Labels":{"severity":"warning"},"Annotations":{"summary":"Node CPU usage high"}}]}' \
  --region ap-guangzhou
```

```
{
  "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

### 6. 通过 Grafana 查看监控

从 Prometheus 控制台进入 Grafana，搜索预设的容器相关 Dashboard（如 `Kubernetes / Compute Resources / Cluster`）：

![](预设仪表板展示集群 CPU、内存、网络、磁盘等监控图表)

## 验证

### Control plane (tccli)

```bash
tccli tke DescribePrometheusCluster

```json
{
  "Agents": [],
  "ClusterType": "<ClusterType>",
  "ClusterId": "<ClusterId>",
  "Status": "<Status>",
  "ClusterName": "<ClusterName>",
  "ExternalLabels": [],
  "Name": "<Name>",
  "Value": "<Value>"
}
```Agents --InstanceId prom-xxxxxxxx --region ap-guangzhou --output json \
  --filter "Agents[?ClusterId=='cls-example'].{Status:Status}"
```

```
[
  { "Status": "running" }
]
```

Grafana Dashboard 中可看到集群监控数据。

## 清理

### Control plane (tccli)

```bash
tccli tke DeletePrometheusClusterAgent \
  --InstanceId prom-xxxxxxxx \
  --Agents '[{"Region":"ap-guangzhou","ClusterId":"cls-example","ClusterType":"tke"}]' \
  --region ap-guangzhou
```

```
{
  "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

## 排障

| 现象 | 处理 |
|------|------|
| `CreatePrometheusClusterAgent` 返回 `FailedOperation` | 检查 ClusterId 是否正确，集群状态是否为 Running |
| Grafana 无数据 | 确认数据采集配置已勾选指标；等待 1-2 分钟数据上报 |
| 告警策略未触发 | 确认告警规则表达式正确；检查通知模板配置 |
| Prometheus 实例不在同一地域 | 绑定时 Agent 的 Region 需与集群 Region 一致 |

## 下一步

- [自建 Prometheus 监控 TKE 集群](../自建 Prometheus 监控 TKE 集群/tccli 操作.md)（page_id `82640`）
- [一键接入腾讯云应用性能监控 APM](https://cloud.tencent.com/document/product/457/107997)

## 控制台替代

控制台 **云原生服务 → Prometheus → 关联集群**；或创建集群时在"组件配置"步骤中绑定 Prometheus 实例。
