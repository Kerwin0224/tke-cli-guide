# 查看监控数据（tccli）

> 对照官方：[查看监控数据](https://cloud.tencent.com/document/product/457/34181) · page_id `34181`

## 概述

腾讯云容器服务默认为所有集群提供基础监控功能，支持查看集群、节点、节点内 Pod、工作负载、工作负载内 Pod、Pod 内 Container 六个维度的监控数据。本章介绍通过 CLI 查看监控数据的等价操作。

六个维度的监控数据采集方式：

| 维度 | 数据来源 | CLI 工具 |
|------|---------|---------|
| 集群 | TCOP QCE/TKE2 | `tccli tke DescribeClusters` |
| 节点 | TCOP QCE/NODE + Metrics Server | `kubectl top nodes` / `kubectl describe node` |
| 节点内 Pod | Metrics Server | `kubectl top pods -A --field-selector spec.nodeName=<NodeName>` |
| 工作负载 | Metrics Server | `kubectl top pods -n <NS> -l app=<Label>` |
| 工作负载内 Pod | Metrics Server | `kubectl top pods -n <NS> -l app=<Label>` |
| Pod 内 Container | Metrics Server | `kubectl top pods <PodName> -n <NS> --containers` |

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
| 查看集群指标 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看集群节点列表 | `tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看节点指标（实时） | `kubectl top nodes` / `kubectl describe node <NodeName>` | 是 |
| 查看节点指标（历史） | `monitor:GetMonitorData`（namespace=QCE/NODE） | 是 |
| 查看节点内 Pod 指标 | `kubectl top pods -A --field-selector spec.nodeName=<NodeName>` | 是 |
| 查看工作负载指标 | `kubectl top pods -n <NS> -l app=<Label>` | 是 |
| 查看工作负载内 Pod 指标 | `kubectl top pods -n <NS> --sort-by=cpu` | 是 |
| 查看 Pod 内 Container 指标 | `kubectl top pods <PodName> -n <NS> --containers` | 是 |
| 查看集群历史监控 | `monitor:GetMonitorData`（namespace=QCE/TKE2） | 是 |

## 操作步骤

### 1. 查看集群维度指标

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

### 2. 查看集群节点列表

```bash
tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-c3d4e5f6-7890-abcd-ef01-234567890abc",
  "TotalCount": 2,
  "InstanceSet": [
    {
      "InstanceId": "ins-abc12345",
      "InstanceRole": "WORKER",
      "InstanceState": "running",
      "LanIP": "10.0.1.5"
    },
    {
      "InstanceId": "ins-def67890",
      "InstanceRole": "WORKER",
      "InstanceState": "running",
      "LanIP": "10.0.1.6"
    }
  ]
}
```

### 3. 查看节点指标（kubectl — 文档参考）

节点资源使用情况：

```bash
kubectl top nodes
```

```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-10-0-1-5    1200m        30%    3200Mi          40%
node-10-0-1-6    800m         20%    2400Mi          30%
```

查看节点详细状态和资源分配：

```bash
kubectl describe node <NodeName> | grep -A 10 "Conditions:"
kubectl describe node <NodeName> | grep -A 5 "Allocated resources:"
```

```text
Conditions:
  Type                 Status
  NetworkUnavailable   False
  MemoryPressure       False
  DiskPressure         False
  PIDPressure          False
  Ready                True
---
Allocated resources:
  Resource           Requests      Limits
  --------           --------      ------
  cpu                2500m (62%)   4000m (100%)
  memory             4800Mi (60%)  6400Mi (80%)
```

### 4. 查看节点内 Pod 指标（kubectl — 文档参考）

```bash
kubectl top pods -A --field-selector spec.nodeName=<NodeName>
```

```text
NAMESPACE     NAME                              CPU(cores)   MEMORY(bytes)
default       nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
kube-system   coredns-6d8cf9b84-xm9lk           8m           45Mi
kube-system   kube-proxy-abc123                 5m           30Mi
```

### 5. 查看工作负载指标（kubectl — 文档参考）

按标签筛选工作负载下的所有 Pod：

```bash
kubectl top pods -n <Namespace> -l app=<LabelValue>
```

```text
NAME                              CPU(cores)   MEMORY(bytes)
nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
nginx-deploy-7d8f9b6c4-xyzab      12m          115Mi
nginx-deploy-7d8f9b6c4-defgh      18m          125Mi
```

### 6. 查看 Pod 内 Container 指标（kubectl — 文档参考）

```bash
kubectl top pods <PodName> -n <Namespace> --containers
```

```text
POD                               NAME      CPU(cores)   MEMORY(bytes)
nginx-deploy-7d8f9b6c4-abcde       nginx     10m          80Mi
nginx-deploy-7d8f9b6c4-abcde       sidecar    5m          40Mi
```

### 7. 通过云监控 API 获取历史监控数据

获取集群（QCE/TKE2 命名空间）的 CPU 使用率历史数据：

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
  "RequestId": "e.g.-d4e5f6a7-8901-bcde-f012-34567890abcd",
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
      "Timestamps": [1640000000, 1640000300],
      "Values": [2.5, 2.3]
    }
  ]
}
```

## 验证

### 控制面验证

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou && \
tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou
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

预期：集群状态 `Running`，节点数量正确。

### 数据面验证（kubectl 文档参考）

```bash
kubectl top nodes && echo "---" && kubectl top pods -A 2>/dev/null | head -6
```

```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-10-0-1-5    1200m        30%    3200Mi          40%
---
NAMESPACE     NAME                    CPU(cores)   MEMORY(bytes)
default       nginx-deploy-xxx        15m          120Mi
```

## 清理

只读操作，无清理。

## 排障

| 现象 | 处理 |
|------|------|
| `kubectl top` 无数据 | 确保 Metrics Server 已部署（TKE 默认安装于 kube-system）；新集群需等几分钟采集 |
| `monitor:GetMonitorData` 返回空 DataPoints | 检查 StartTime/EndTime 范围和 MetricName 是否正确；确认集群已运行足够时间产生数据 |
| `DescribeClusterInstances` 返回空 InstanceSet | 确认集群有可用节点；托管集群控制面节点不在返回列表中 |
| Pod 内 Container 指标刷新慢 | 监控数据采集有一定延迟，通常 1-2 分钟 |

## 下一步

- [监控及告警指标列表](../监控及告警指标列表/tccli 操作.md) — 五层监控告警指标参考
- [监控告警概述](../监控告警概述/tccli 操作.md) — 监控告警概念总览
- [查看 TKE 集群控制面组件监控](../../控制面组件监控/查看 TKE 集群控制面组件监控/tccli 操作.md) — 控制面组件监控数据

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 集群详情 > 各维度监控图表（集群/节点/工作负载/Pod/Container）。
