# 成本洞察（tccli）

> 对照官方：[成本洞察](https://cloud.tencent.com/document/product/457/126969) · page_id `126969`

## 概述

TKE Insight 成本洞察通过可视化面板帮助用户理解、监控和控制容器环境中的资源使用和成本情况，覆盖普通节点（CVM）、原生节点和超级节点三类计费项。成本数值为参考值，准确费用请登录腾讯云费用中心查看。仅适用于 TKE 标准集群。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 集群已开启 TCOP（腾讯云可观测平台）数据上报

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
| 选择地域/集群 | `tccli tke DescribeClusters --region ap-guangzhou` | 是 |
| 查看当月成本 | TCOP 监控 API（`monitor:GetMonitorData`） | 是 |
| 查看节点详情 | `kubectl get nodes -o json` | 是 |
| 查看节点机型 | `kubectl get node <NodeName> -o jsonpath='{.metadata.annotations.node\.kubernetes\.io/instance-type}'` | 是 |
| 查看超级节点 Pod 规格 | `kubectl get pod <PodName> -o jsonpath='{.metadata.annotations.eks\.tke\.cloud\.tencent\.com/node-resource}'` | 是 |

## 操作步骤

### 查看集群节点列表

```bash
kubectl get nodes -o wide
```

```text
NAME             STATUS   ROLES    AGE   VERSION          INTERNAL-IP   EXTERNAL-IP   OS-IMAGE
node-10-0-1-5    Ready    <none>   30d   v1.30.0-tke.5   10.0.1.5      <none>        TencentOS Server 3.1
node-10-0-1-6    Ready    <none>   30d   v1.30.0-tke.5   10.0.1.6      <none>        TencentOS Server 3.1
node-10-0-1-7    Ready    <none>   15d   v1.30.0-tke.5   10.0.1.7      <none>        TencentOS Server 3.1
```

### 查看节点机型（普通节点/原生节点）

节点机型从 annotation `node.kubernetes.io/instance-type` 获取：

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.node\.kubernetes\.io/instance-type}{"\n"}{end}'
```

```text
node-10-0-1-5    S5.MEDIUM4
node-10-0-1-6    SA2.MEDIUM4
node-10-0-1-7    S5.LARGE8
```

### 查看超级节点 Pod 规格

超级节点上 Pod 的规格从 annotation `eks.tke.cloud.tencent.com/node-resource` 获取：

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\t"}{.metadata.annotations.eks\.tke\.cloud\.tencent\.com/node-resource}{"\n"}{end}' 2>/dev/null
```

```text
default/nginx-super-7d8f9b6c4-abcde  {"cpu":"2","memory":"4Gi"}
```

### 查看节点 CPU/内存 Allocatable

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}CPU: {.status.allocatable.cpu}{"\t"}Memory: {.status.allocatable.memory}{"\n"}{end}'
```

```text
node-10-0-1-5    CPU: 4    Memory: 8192Mi
node-10-0-1-6    CPU: 4    Memory: 8192Mi
node-10-0-1-7    CPU: 8    Memory: 16384Mi
```

### 查看 Pod 资源请求与用量（命名空间级别成本分布基础）

```bash
kubectl top pods -A --sort-by=cpu
```

```text
NAMESPACE     NAME                              CPU(cores)   MEMORY(bytes)
default       nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
kube-system   coredns-6d8cf9b84-xm9lk           8m           45Mi
default       app-server-5c8d7f9b6-xyzab        5m           80Mi
```

## 验证

### Data plane (kubectl)

```bash
kubectl get nodes && kubectl top pods -A --sort-by=cpu 2>/dev/null | head -10
```

```text
NAME             STATUS   ROLES    AGE   VERSION
node-10-0-1-5    Ready    <none>   30d   v1.30.0-tke.5
node-10-0-1-6    Ready    <none>   30d   v1.30.0-tke.5
node-10-0-1-7    Ready    <none>   15d   v1.30.0-tke.5
```

## 清理

成本洞察为只读面板，无清理操作。

## 排障

| 现象 | 处理 |
|------|------|
| 成本数据为空 | 确认集群为标准集群；超级节点不支持使用预留券抵消对应规格 Pod 的成本 |
| 上月无成本数据 | 当月成本相较于上月成本的百分比不展示数据 |
| 超级节点 Pod 成本异常 | GPU 价格分摊在 CPU 和内存上，即 GPU 节点上的 CPU/内存单价更高 |
| 计费模式不确定 | 原生节点和普通节点统一按按量计费模式单价上报 |

## 下一步

- [Request 智能推荐](../成本优化/Request 智能推荐/tccli 操作.md) — 为 Workload 智能推荐资源 Request/Limit
- [Node Map](../Node Map/tccli 操作.md) — 节点可视化热力图
- [Workload Map](../Workload Map/tccli 操作.md) — 工作负载可视化热力图

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > TKE Insight > 成本洞察，查看成本可视化面板。
