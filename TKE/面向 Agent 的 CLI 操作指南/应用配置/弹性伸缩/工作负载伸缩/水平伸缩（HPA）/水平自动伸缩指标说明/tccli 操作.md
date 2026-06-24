# 水平自动伸缩指标说明（tccli）

> 对照官方：[水平自动伸缩指标说明](https://cloud.tencent.com/document/product/457/38929) · page_id `38929`

## 概述

水平自动伸缩（HPA）可根据 Pod 的 CPU、内存、网络、磁盘等指标自动扩缩容。TKE 控制台支持的触发策略指标类别包括 CPU、内存、硬盘、网络四大类，同时也支持通过 YAML 文件配置这些指标。

所有自定义指标均使用 `type: Pods`。`metricName` 变量已内置默认单位（如下表所示），编写 YAML 时可省略单位。指标中控制台显示单位与后端 metric 默认单位不一致，在 YAML 中直接使用数值，后端按默认单位解释。

TKE 同样兼容 Kubernetes 原生的 `type: Resource` 指标类型。

## 前置条件

- [环境准备](../../../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`
- 已部署 metrics-server 和 TKE 监控组件

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"

# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    | jq -r '.Kubeconfig' > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
```

> **注意**：kubectl 数据面当前因公网端点被组织级 CAM 策略拒绝（strategyId:240463971）而不可达。外网端点被 `tke:clusterExtranetEndpoint=true` 条件拦截，内网端点需通过 IOA/VPN 或同 VPC 内 CVM 访问。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 控制台与 CLI 参数映射

| 控制台字段 | YAML 字段 | 幂等 |
|-----------|----------|:--:|
| 触发策略指标选择 | `spec.metrics[].pods.metric.name` | 是 |
| 指标类型 | `spec.metrics[].type`（`Resource` / `Pods`） | 是 |
| 阈值 | `spec.metrics[].pods.target.averageValue` 或 `.resource.target.averageUtilization` | 是 |
| 目标类型 | `spec.metrics[].pods.target.type`（`AverageValue` / `Utilization`） | 是 |
| 工作负载引用 | `spec.scaleTargetRef` | 是 |

## 操作步骤

### 步骤 1：在 HPA YAML 中使用自定义指标

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: example
  namespace: default
  labels:
    qcloud-app: example
spec:
  minReplicas: 1
  maxReplicas: 2
  metrics:
  - type: Pods
    pods:
      metric:
        name: k8s_pod_cpu_core_used
      target:
        averageValue: "1"
        type: AverageValue
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
```

```bash
kubectl apply -f hpa-custom-metrics.yaml
# expected: horizontalpodautoscaler.autoscaling/example created
```

### 步骤 2：同时使用 Resource 和 Pods 指标

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: <HpaName>
  namespace: <Namespace>
spec:
  minReplicas: 1
  maxReplicas: 10
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <DeploymentName>
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Pods
    pods:
      metric:
        name: k8s_pod_network_receive_bytes_bw
      target:
        type: AverageValue
        averageValue: "104857600"
  - type: Pods
    pods:
      metric:
        name: k8s_pod_mem_no_cache_bytes
      target:
        type: AverageValue
        averageValue: "536870912"
```

> **说明**：多个指标取各自计算结果的最大值触发扩缩容。

### 步骤 3：CPU 指标

| 控制台名称 | 控制台单位 | 说明 | type | metricName | metric 默认单位 |
|-----------|-----------|------|------|------------|----------------|
| CPU 使用量 | 核 (cores) | Pod 的 CPU 使用量 | Pods | `k8s_pod_cpu_core_used` | 核 |
| CPU 利用率（占节点） | % | Pod CPU 使用量占节点总量比例 | Pods | `k8s_pod_rate_cpu_core_used_node` | % |
| CPU 利用率（占 Request） | % | Pod CPU 使用量占容器 Request 值比例 | Pods | `k8s_pod_rate_cpu_core_used_request` | % |
| CPU 利用率（占 Limit） | % | Pod CPU 使用量占容器 Limit 值总和比例 | Pods | `k8s_pod_rate_cpu_core_used_limit` | % |

YAML 示例 — CPU 使用量（Avg）：

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: k8s_pod_cpu_core_used
    target:
      type: AverageValue
      averageValue: "1"
```

YAML 示例 — CPU 利用率占 Request（Utilization）：

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 60
```

### 步骤 4：内存指标

| 控制台名称 | 控制台单位 | 说明 | type | metricName | metric 默认单位 |
|-----------|-----------|------|------|------------|----------------|
| 内存使用量 | MiB | Pod 内存使用量 | Pods | `k8s_pod_mem_usage_bytes` | B |
| 内存使用量（不包含 Cache） | MiB | Pod 内存使用量，不包含 Cache | Pods | `k8s_pod_mem_no_cache_bytes` | B |
| 内存利用率（占节点） | % | Pod 内存使用量占节点比例 | Pods | `k8s_pod_rate_mem_usage_node` | % |
| 内存利用率（占节点，不包含 Cache） | % | 同上，不含 Cache | Pods | `k8s_pod_rate_mem_no_cache_node` | % |
| 内存利用率（占 Request） | % | Pod 内存使用量占 Request 比例 | Pods | `k8s_pod_rate_mem_usage_request` | % |
| 内存利用率（占 Request，不包含 Cache） | % | 同上，不含 Cache | Pods | `k8s_pod_rate_mem_no_cache_request` | % |
| 内存利用率（占 Limit） | % | Pod 内存使用量占 Limit 比例 | Pods | `k8s_pod_rate_mem_usage_limit` | % |
| 内存利用率（占 Limit，不包含 Cache） | % | 同上，不含 Cache | Pods | `k8s_pod_rate_mem_no_cache_limit` | % |

YAML 示例 — 内存使用量（不包含 Cache）超过 512 MiB 时触发扩容：

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: k8s_pod_mem_no_cache_bytes
    target:
      type: AverageValue
      averageValue: "536870912"
```

> **重要**：metric 默认单位为 B（Bytes），控制台显示 MiB。设置阈值需换算：512 MiB = 512 × 1024 × 1024 = 536870912 B。

### 步骤 5：磁盘指标

| 控制台名称 | 控制台单位 | 说明 | type | metricName | metric 默认单位 |
|-----------|-----------|------|------|------------|----------------|
| 硬盘写流量 | KB/s | Pod 磁盘写速率 | Pods | `k8s_pod_fs_write_bytes` | B/s |
| 硬盘读流量 | KB/s | Pod 磁盘读速率 | Pods | `k8s_pod_fs_read_bytes` | B/s |
| 硬盘读 IOPS | 次/s | Pod 磁盘读 IO 次数 | Pods | `k8s_pod_fs_read_times` | 次/s |
| 硬盘写 IOPS | 次/s | Pod 磁盘写 IO 次数 | Pods | `k8s_pod_fs_write_times` | 次/s |

YAML 示例 — 硬盘读流量超过 10 MB/s 时扩容：

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: k8s_pod_fs_read_bytes
    target:
      type: AverageValue
      averageValue: "10485760"
```

> **重要**：默认单位为 B/s，控制台显示 KB/s。10 MB/s = 10 × 1024 × 1024 = 10485760 B/s。

### 步骤 6：网络指标

| 控制台名称 | 控制台单位 | 说明 | type | metricName | metric 默认单位 |
|-----------|-----------|------|------|------------|----------------|
| 网络入带宽 | Mbps | Pod 所有容器入带宽之和 | Pods | `k8s_pod_network_receive_bytes_bw` | bps |
| 网络出带宽 | Mbps | Pod 所有容器出带宽之和 | Pods | `k8s_pod_network_transmit_bytes_bw` | bps |
| 网络入流量 | KB/s | Pod 所有容器入流量之和 | Pods | `k8s_pod_network_receive_bytes` | B |
| 网络出流量 | KB/s | Pod 所有容器出流量之和 | Pods | `k8s_pod_network_transmit_bytes` | B |
| 网络入包量 | 个/s | Pod 所有容器入包量之和 | Pods | `k8s_pod_network_receive_packets` | 个/s |
| 网络出包量 | 个/s | Pod 所有容器出包量之和 | Pods | `k8s_pod_network_transmit_packets` | 个/s |

YAML 示例 — 网络入带宽超过 100 Mbps 时扩容：

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: k8s_pod_network_receive_bytes_bw
    target:
      type: AverageValue
      averageValue: "100000000"
```

> **重要**：默认单位为 bps，控制台显示 Mbps。100 Mbps = 100 × 1000 × 1000 = 100000000 bps。

### 步骤 7：单位换算速查表

| 控制台显示 | metric 默认单位 | 换算公式 |
|-----------|----------------|---------|
| 核 (cores) | 核 | 直接使用，无需换算 |
| MiB | B (Bytes) | × 1024 × 1024 |
| KB/s | B/s (Bytes/sec) | × 1024 |
| Mbps | bps (bits/sec) | × 1000 × 1000 |
| % | % | 直接使用，无需换算 |
| 次/s | 次/s | 直接使用，无需换算 |
| 个/s | 个/s | 直接使用，无需换算 |

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 数据面（kubectl）

```bash
kubectl get hpa <HpaName> -n <Namespace> -o yaml
# expected: 显示 HPA 完整配置和 currentMetrics
```

输出示例（节选）：

```yaml
status:
  currentMetrics:
  - pods:
      current:
        averageValue: 500m
      metric:
        name: k8s_pod_cpu_core_used
    type: Pods
  currentReplicas: 4
  desiredReplicas: 4
```

```bash
kubectl describe hpa <HpaName> -n <Namespace>
# expected: 显示各指标 current/target 值
```

输出示例（节选）：

```text
Metrics:                       ( current / target )
  "k8s_pod_cpu_core_used" on pods: 500m / 1
  "k8s_pod_network_receive_bytes_bw" on pods: 50M / 100M
```

## 清理

无需清理（指标说明为参考文档，不创建资源）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pods 指标在 TARGETS 列显示 `<unknown>` | `kubectl get pods -n kube-system \| grep tke-monitor` 确认组件状态 | TKE 监控组件未安装或异常 | 安装或重启 tke-monitor 组件 |
| Pods 指标有数据但值异常（过大/过小） | 对照"单位换算速查表"检查阈值 | metric 默认单位与控制台单位不一致，阈值设置未换算 | 重新计算 `averageValue` |
| `k8s_pod_rate_cpu_core_used_request` 无数据 | `kubectl describe deployment <name>` 检查 resources | 容器未设置 CPU Request | 设置 CPU Request：`kubectl set resources deployment <DeploymentName> -c <ContainerName> --requests=cpu=100m` |
| 基于 Pods 指标的 HPA 不生效 | `kubectl get hpa <name> -o yaml` 检查 apiVersion | API 版本不兼容，TKE 自定义指标需 `autoscaling/v2` | 确认 YAML 中 `apiVersion: autoscaling/v2` |

## 下一步

- [水平伸缩基本操作](../水平伸缩基本操作/tccli%20操作.md) — 创建和管理 HPA
- [定时水平伸缩（HPC）](../../定时水平伸缩（HPC）/tccli%20操作.md) — 按 Cron 表达式定时扩缩容
- [预测水平伸缩（EHPA）](../../预测水平伸缩（EHPA）/tccli%20操作.md) — DSP 算法预测性扩缩容
- [工作负载管理](../../../../工作负载管理/Deployment%20管理/tccli%20操作.md) — 创建和管理 Deployment

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 自动伸缩 → HorizontalPodAutoscaler → 新建，选择触发策略指标（CPU/内存/硬盘/网络）并设置阈值。
