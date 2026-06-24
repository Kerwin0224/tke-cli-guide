# 监控及告警指标列表（tccli）

> 对照官方：[监控及告警指标列表](https://cloud.tencent.com/document/product/457/34183) · page_id `34183`

## 概述

容器服务 TKE 提供集群、Master&Etcd&普通节点、工作负载、Pod、Container 五个维度的监控告警指标，所有指标均为统计周期内的平均值。本文档将控制台各维度指标映射到 CLI 可用的查询方式。

**注意**：TKE 基础监控所有指标均为统计周期内的平均值，不提供瞬时值。如需更细粒度指标（如 P99 延迟、分位数），请使用 Prometheus 监控服务。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。

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
| 查看集群监控指标 | `monitor:GetMonitorData`（namespace=QCE/TKE2） | 是 |
| 查看节点监控指标 | `kubectl top nodes` / `kubectl describe node` / `monitor:GetMonitorData`（namespace=QCE/NODE） | 是 |
| 查看工作负载监控指标 | `kubectl top pods -n <NS> -l app=<Label>` | 是 |
| 查看 Pod 监控指标 | `kubectl top pods <PodName> -n <NS>` | 是 |
| 查看 Container 监控指标 | `kubectl top pods <PodName> -n <NS> --containers` | 是 |
| 查看节点状态 | `kubectl get nodes` / `kubectl describe node` | 是 |
| 查看节点重启次数 | `kubectl get pods -A -o wide --field-selector spec.nodeName=<NodeName>` | 是 |

## 操作步骤

### 1. 集群维度指标

**QCE/TKE2 命名空间**，通过云监控 API 获取。

集群级概览（kubectl 文档参考）：

```bash
kubectl top nodes
kubectl get nodes --no-headers | wc -l
```

```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-10-0-1-5    1200m        30%    3200Mi          40%
node-10-0-1-6    800m         20%    2400Mi          30%
2
```

获取集群 CPU 总核数：

```bash
tccli monitor GetMonitorData \
  --Namespace "QCE/TKE2" \
  --MetricName "K8sClusterCpuCoreTotal" \
  --Instances '[{"Dimensions":[{"Name":"tke_cluster_instance_id","Value":"cls-xxxxxxxx"}]}]' \
  --StartTime "$(date -u -v-1H '+%Y-%m-%dT%H:%M:%S+00:00')" \
  --EndTime "$(date -u '+%Y-%m-%dT%H:%M:%S+00:00')" \
  --Period 300 \
  --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-e5f6a7b8-9012-cdef-0123-4567890abcde",
  "MetricName": "K8sClusterCpuCoreTotal",
  "StartTime": "2026-06-18T10:00:00+00:00",
  "EndTime": "2026-06-18T11:00:00+00:00",
  "Period": 300,
  "DataPoints": [
    {
      "Dimensions": [
        {
          "Name": "tke_cluster_instance_id",
          "Value": "cls-xxxxxxxx"
        }
      ],
      "Timestamps": [1753257600, 1753257900],
      "Values": [4.0, 3.8]
    }
  ]
}
```

### 2. 节点维度指标

**QCE/NODE 命名空间**，覆盖 CPU/内存/网络/磁盘等指标。

查看节点资源使用率和状态（kubectl 文档参考）：

```bash
kubectl top nodes
kubectl describe node <NodeName> | grep -A 5 "Allocated resources:"
```

```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-10-0-1-5    1200m        30%    3200Mi          40%
---
Allocated resources:
  Resource           Requests      Limits
  --------           --------      ------
  cpu                2500m (62%)   4000m (100%)
  memory             4800Mi (60%)  6400Mi (80%)
```

查看节点上 Pod 的重启次数：

```bash
kubectl get pods -A -o wide --field-selector spec.nodeName=<NodeName> \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[*].restartCount}{"\n"}{end}'
```

```text
NAME  STATUS  AGE
...
```

### 3. 工作负载维度指标

通过 Kubernetes 标签选择器聚合 Pod 指标：

```bash
kubectl top pods -n <Namespace> -l app=<LabelValue>
```

```text
NAME                              CPU(cores)   MEMORY(bytes)
nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
nginx-deploy-7d8f9b6c4-xyzab      12m          115Mi
```

工作负载级聚合（kubectl 文档参考）：

```bash
kubectl top pods -n <Namespace> --sort-by=cpu
```

```text
NAME                              CPU(cores)   MEMORY(bytes)
app-server-5c8d7f9b6-xyzab        35m          280Mi
nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
redis-cluster-0                    10m          90Mi
```

### 4. Pod 维度指标

查看单个 Pod 的 CPU 和内存使用：

```bash
kubectl top pods <PodName> -n <Namespace>
```

```text
NAME                              CPU(cores)   MEMORY(bytes)
nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
```

查看 Pod 重启次数：

```bash
kubectl get pods <PodName> -n <Namespace> -o jsonpath='{.status.containerStatuses[*].restartCount}'
```

```text
0
```

### 5. Container 维度指标

查看 Pod 内各容器的资源使用：

```bash
kubectl top pods <PodName> -n <Namespace> --containers
```

```text
POD                               NAME      CPU(cores)   MEMORY(bytes)
nginx-deploy-7d8f9b6c4-abcde       nginx     10m          80Mi
nginx-deploy-7d8f9b6c4-abcde       sidecar    5m          40Mi
```

## 验证

### 控制面验证

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

预期：集群状态 `Running`，节点数 >= 1。

### 数据面验证（kubectl 文档参考）

```bash
kubectl top nodes && kubectl top pods -A 2>/dev/null | head -5
```

```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-10-0-1-5    1200m        30%    3200Mi          40%
NAMESPACE     NAME                    CPU(cores)   MEMORY(bytes)
kube-system   coredns-6d8cf9b84-xm9lk  8m          45Mi
```

## 清理

只读操作，无清理。

## 排障

| 现象 | 处理 |
|------|------|
| 指标名与控制台不一致 | 控制台指标名可能包含中文描述，云监控 API 使用英文 metricName；参见 [GetMonitorData 指标名](https://cloud.tencent.com/document/product/248/51283) 文档 |
| 指标均为平均值 | TKE 基础监控所有指标均为统计周期内（60s/300s）平均值，不提供 P50/P99 等分位数 |
| 需要更细粒度指标 | 使用 Prometheus 监控服务采集自定义指标；节点指标参见 CVM 监控，数据盘指标参见云硬盘监控 |
| 网络指标缺失 | 工作负载外部服务绑定的 Service 网络指标参见负载均衡（CLB）监控 |
| `kubectl top` 报错 `Metrics API not available` | Metrics Server 未安装或未就绪；TKE 默认安装于 kube-system 命名空间 |

## 下一步

- [查看监控数据](../查看监控数据/tccli 操作.md) — 控制台与 CLI 查看各维度监控数据
- [监控告警概述](../监控告警概述/tccli 操作.md) — 监控告警概念总览
- [kube-apiserver 组件指标说明](../../控制面组件监控/kube-apiserver 组件指标说明/tccli 操作.md) — 控制面自定义 Prometheus 指标
- [事件日志](../../../事件管理/事件日志/tccli 操作.md) — 集群事件持久化到 CLS

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 集群详情 > 各维度监控图表，查看对应指标。
