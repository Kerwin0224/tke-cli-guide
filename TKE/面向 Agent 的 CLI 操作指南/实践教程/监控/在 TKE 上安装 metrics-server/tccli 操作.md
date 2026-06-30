# 在 TKE 上安装 metrics-server（tccli）

> 对照官方：[在 TKE 上安装 metrics-server](https://cloud.tencent.com/document/product/457/50074) · page_id `50074`
## 概述

在 TKE 集群中安装 metrics-server 组件，为 HPA、VPA、`kubectl top` 等提供核心指标数据（CPU/内存使用率）。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已获取集群 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: 生成 kubeconfig.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 相关 kubectl 操作 | 见 Procedure | -- |

## 操作步骤

### 1. 控制面：安装 metrics-server 组件

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName metrics-server
# expected: RequestId 返回，组件安装中
```

### 2. 控制面：验证安装

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> \
    --filter "AddonSet[?AddonName=='metrics-server'].Status"
# expected: "Running"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 3. 数据面：验证指标数据（需 VPN/IOA）

```bash
# 查看节点资源
kubectl top nodes
# expected: CPU/MEM 列显示节点用量
```

预期输出：

```text
NAME              CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
10.0.0.1          150m         3%     2048Mi          25%
10.0.0.2          200m         5%     3072Mi          38%
```

```bash
# 查看 Pod 资源
kubectl top pods -A --sort-by=cpu
# expected: CPU/MEM 列显示 Pod 用量
```

预期输出：

```text
NAMESPACE     NAME                     CPU(cores)   MEMORY(bytes)
default       nginx-7d8c4f9b6c-abc12   2m           4Mi
kube-system   coredns-5d8c4f9b6c-def   5m           16Mi
```

### 4. 查看 metrics API

```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
# expected: 返回节点指标 JSON
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> \
    --filter "AddonSet[?AddonName=='metrics-server'].Status"
# expected: "Running"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl top nodes
# expected: CPU/MEM 用量数据显示

kubectl top pods -A
# expected: Pod CPU/MEM 数据显示
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令报错/无响应（本环境实测） | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' --filter "Clusters[0].ClusterStatus"` | CAM 拒绝公网端点（strategyId:240463971），数据面不可达 | 接入 VPN/IOA 内网后执行 kubectl；控制面 tccli 不受影响 |
| `kubectl top` 无数据 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId <ClusterId> --AddonName metrics-server` | metrics-server 未就绪或未安装 | 等待组件 Running（2-5 分钟初始化） |
| `metrics not available` | `kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes` | metrics API 未注册 | 等待 2-5 分钟初始化 |
| 指标延迟大 | 对比 `kubectl top` 与实际用量 | metrics-server 采集间隔长 | 默认 60s，不可修改 |

## 清理

如需卸载：

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName metrics-server
# expected: RequestId 返回
```

## 下一步

- [在 TKE 上利用 HPA 实现业务的弹性伸缩](../../弹性伸缩/在%20TKE%20上利用%20HPA%20实现业务的弹性伸缩/tccli%20操作.md) -- page_id `50084`
- [HPC 说明](../../../应用配置/组件和应用管理/组件管理/HPC%20说明/tccli%20操作.md) -- page_id `56753`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
