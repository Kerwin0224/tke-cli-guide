# Node Map（tccli）

> 对照官方：[Node Map](https://cloud.tencent.com/document/product/457/64169) · page_id `64169`

## 概述

Node Map 是 TKE Insight 提供的节点可视化热力图，通过可视化的方式展示集群中每个节点的 CPU/内存利用率、装箱率等指标，支持筛选、聚合、状态归类。不支持统计超级节点的各项指标。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
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
|-----------|-----|------|
| 查看集群节点列表 | `kubectl get nodes` | 是 |
| 查看节点类型 | `kubectl get nodes -o jsonpath='{.items[*].metadata.labels.node\.kubernetes\.io/instance-type}'` | 是 |
| 查看 CPU 装箱率 | `kubectl top nodes` / `kubectl describe node` | 是 |
| 查看节点规格 | `kubectl get node <NodeName> -o jsonpath='{.status.capacity}'` | 是 |
| 查看集群节点（控制面） | `tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 驱逐节点 | `kubectl drain <NodeName>` | 否 |
| 取消封锁 | `kubectl uncordon <NodeName>` | 是 |
| 开启 Request 推荐 | `tccli tke InstallAddon --AddonName craned --region ap-guangzhou` | 是 |

## 操作步骤

### 查看集群节点概览

```bash
kubectl get nodes -o wide
```

```text
NAME             STATUS   ROLES    AGE   VERSION          INTERNAL-IP   EXTERNAL-IP   OS-IMAGE
node-10-0-1-5    Ready    <none>   30d   v1.30.0-tke.5   10.0.1.5      <none>        TencentOS Server 3.1
node-10-0-1-6    Ready    <none>   30d   v1.30.0-tke.5   10.0.1.6      <none>        TencentOS Server 3.1
node-10-0-1-7    Ready    <none>   15d   v1.30.0-tke.5   10.0.1.7      <none>        TencentOS Server 3.1
```

### 查看节点类型（原生节点/普通节点/注册节点）

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.cloud\.tencent\.com/node-instance-type}{"\t"}{.metadata.labels.node\.kubernetes\.io/instance-type}{"\n"}{end}'
```

```text
node-10-0-1-5    S5.MEDIUM4
node-10-0-1-6    SA2.MEDIUM4
node-10-0-1-7    S5.LARGE8
```

### 查看节点 CPU 和内存资源

```bash
kubectl top nodes
```

```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-10-0-1-5    1200m        30%    3200Mi          40%
node-10-0-1-6    800m         20%    2400Mi          30%
node-10-0-1-7    2200m        27%    6400Mi          39%
```

### 查看节点详细 Capacity 和 Allocatable

```bash
kubectl describe node <NodeName> | grep -A 10 "Capacity:"
```

```text
Capacity:
  cpu:                4
  ephemeral-storage:  102401Mi
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             8192Mi
  pods:               64
Allocatable:
  cpu:                3.8
  ephemeral-storage:  94378Mi
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7300Mi
  pods:               64
```

### 查看节点装箱率（Pod Request 之和 / 节点规格）

```bash
kubectl describe node <NodeName> | grep -A 8 "Allocated resources:"
```

```text
Allocated resources:
  Resource           Requests      Limits
  --------           --------      ------
  cpu                2500m (62%)   4000m (100%)
  memory             4800Mi (60%)  6400Mi (80%)
  ephemeral-storage  0 (0%)        0 (0%)
```

### 驱逐节点 Pod

```bash
kubectl drain <NodeName> --ignore-daemonsets --delete-emptydir-data
```

```text
node/node-10-0-1-7 cordoned
evicting pod default/nginx-deploy-7d8f9b6c4-abcde
pod/nginx-deploy-7d8f9b6c4-abcde evicted
node/node-10-0-1-7 drained
```

### 取消封锁节点

```bash
kubectl uncordon <NodeName>
```

```text
node/node-10-0-1-7 uncordoned
```

### 控制面：查看节点列表（tccli）

```bash
tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "TotalCount": 2,
  "InstanceSet": [
    {
      "InstanceId": "ins-example-001",
      "InstanceRole": "WORKER",
      "InstanceState": "running",
      "LanIP": "10.0.1.5",
      "DrainStatus": "none"
    },
    {
      "InstanceId": "ins-example-002",
      "InstanceRole": "WORKER",
      "InstanceState": "running",
      "LanIP": "10.0.1.6",
      "DrainStatus": "none"
    }
  ]
}
```

## 验证

### Data plane (kubectl)

```bash
kubectl get nodes && kubectl top nodes
```

```text
NAME             STATUS   ROLES    AGE   VERSION
node-10-0-1-5    Ready    <none>   30d   v1.30.0-tke.5
node-10-0-1-6    Ready    <none>   30d   v1.30.0-tke.5
node-10-0-1-7    Ready    <none>   15d   v1.30.0-tke.5

NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-10-0-1-5    1200m        30%    3200Mi          40%
node-10-0-1-6    800m         20%    2400Mi          30%
node-10-0-1-7    2200m        27%    6400Mi          39%
```

## 清理

### Data plane (kubectl)

```bash
kubectl uncordon <NodeName>
```

驱逐后的节点需解除封锁以恢复调度。

## 排障

| 现象 | 处理 |
|------|------|
| 节点不在统计范围 | 超级节点没有节点的概念，不在统计范围之内 |
| 节点移出需去节点池 | 若节点归属于节点池，需前往节点池页面处理；若不在节点池，可从集群中删除 |
| 装箱率/利用率数据不准 | 确认右上角时间范围（24h/7d/30d）和统计模式（均值/峰值） |
| 驱逐后 Pod 无法重新调度 | 确认集群有其他可用节点，节点驱逐后会自动封锁 |

## 下一步

- [Workload Map](../Workload Map/tccli 操作.md) — 工作负载可视化热力图
- [成本洞察](../成本洞察/tccli 操作.md) — 集群成本可视化
- [Request 智能推荐](../成本优化/Request 智能推荐/tccli 操作.md) — 智能推荐资源配置

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > TKE Insight > Node Map，查看节点热力图并进行驱逐/移出/取消封锁操作。
