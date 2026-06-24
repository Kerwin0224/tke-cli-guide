# Workload Map（tccli）

> 对照官方：[Workload Map](https://cloud.tencent.com/document/product/457/78329) · page_id `78329`

## 概述

Workload Map 是 TKE Insight 提供的工作负载可视化热力图，通过可视化方式展示集群中每个工作负载的 CPU/内存配置量、实际使用量和推荐量等指标。支持热力图和列表两种视图，可按类型、命名空间筛选和聚合。不支持提供超级节点上 Pod 推荐值的监控数据。

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
| 查看工作负载列表 | `kubectl get deploy,statefulset,daemonset -A` | 是 |
| 查看 CPU/内存用量 | `kubectl top pods -A` | 是 |
| 查看工作负载配置量 | `kubectl get deploy <Name> -o jsonpath='{.spec.template.spec.containers[*].resources.requests}'` | 是 |
| 查看推荐值 | `kubectl get deploy <Name> -o jsonpath='{.metadata.annotations.analysis\.crane\.io/resource-recommendation}'` | 是 |
| 删除工作负载 | `kubectl delete deploy <Name> -n <NS>` | 否 |
| 下载列表 | `kubectl get deploy,sts,ds -A -o json` | 是 |

## 操作步骤

### 查看所有工作负载

```bash
kubectl get deploy,statefulset,daemonset -A
```

```text
NAMESPACE     NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
default       deployment.apps/nginx-deploy      3/3     3            3           7d
kube-system   deployment.apps/coredns           2/2     2            2           30d
default       deployment.apps/app-server        1/1     1            1           3d
default       statefulset.apps/redis-cluster    3/3     3            3           5d
kube-system   daemonset.apps/kube-proxy         3       3            3           30d
```

### 查看工作负载 CPU/内存用量（kubectl top）

```bash
kubectl top pods -A --sort-by=cpu | head -15
```

```text
NAMESPACE     NAME                              CPU(cores)   MEMORY(bytes)
default       nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
default       nginx-deploy-7d8f9b6c4-xyzab      12m          115Mi
default       nginx-deploy-7d8f9b6c4-defgh      18m          125Mi
default       app-server-5c8d7f9b6-xyzab        35m          280Mi
kube-system   coredns-6d8cf9b84-xm9lk           8m           45Mi
```

### 查看工作负载资源配置量（Request/Limit）

```bash
kubectl get deploy <DeploymentName> -n <Namespace> -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}CPU-Request: {.resources.requests.cpu}{"\t"}Mem-Request: {.resources.requests.memory}{"\t"}CPU-Limit: {.resources.limits.cpu}{"\t"}Mem-Limit: {.resources.limits.memory}{"\n"}{end}'
```

```text
nginx   CPU-Request: 250m   Mem-Request: 256Mi   CPU-Limit: 500m   Mem-Limit: 512Mi
```

### 查看工作负载利用率

```bash
kubectl top pods -n <Namespace> -l app=<LabelValue>
```

```text
NAME                              CPU(cores)   MEMORY(bytes)
nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
nginx-deploy-7d8f9b6c4-xyzab      12m          115Mi
nginx-deploy-7d8f9b6c4-defgh      18m          125Mi
```

### 按命名空间聚合工作负载统计

```bash
kubectl get deploy -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.replicas}{"\t"}{.status.availableReplicas}{"\n"}{end}'
```

```text
default       nginx-deploy   3   3
default       app-server     1   1
kube-system   coredns        2   2
```

### 查看推荐值（需先开启 Request 推荐组件）

```bash
kubectl get deploy <DeploymentName> -n <Namespace> -o jsonpath='{.metadata.annotations.analysis\.crane\.io/resource-recommendation}'
```

```text
containers:
- containerName: nginx
  target:
    cpu: 125m
    memory: 125Mi
```

## 验证

### Data plane (kubectl)

```bash
kubectl get deploy,statefulset,daemonset -A --no-headers | wc -l && kubectl top pods -A 2>/dev/null | head -5
```

```text
8
NAMESPACE     NAME                              CPU(cores)   MEMORY(bytes)
default       nginx-deploy-7d8f9b6c4-abcde      15m          120Mi
default       app-server-5c8d7f9b6-xyzab        35m          280Mi
```

## 清理

Workload Map 为只读面板，无需清理。如需删除演示工作负载：

```bash
kubectl delete deploy nginx-deploy -n default
```

## 排障

| 现象 | 处理 |
|------|------|
| 推荐量缺失 | 超级节点上 Pod 不提供推荐值监控数据；需先开启 Request 推荐功能 |
| 副本数 0 或无推荐值 | 工作负载首次运行不超过 24 小时，资源推荐量可能为空 |
| 利用率数据不准 | 确认右上角时间范围（24h/7d/30d）选择 |
| 下载数据字段说明 | 24h/7d/30d CPU利用率和内存利用率的均值和峰值分别列出 |

## 下一步

- [Node Map](../Node Map/tccli 操作.md) — 节点可视化热力图
- [Request 智能推荐](../成本优化/Request 智能推荐/tccli 操作.md) — 智能推荐资源配置
- [成本洞察](../成本洞察/tccli 操作.md) — 集群成本可视化

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > TKE Insight > Workload Map，查看工作负载热力图/列表并进行推荐更新和删除操作。
