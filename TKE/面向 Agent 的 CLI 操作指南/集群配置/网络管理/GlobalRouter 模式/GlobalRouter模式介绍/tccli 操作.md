# GlobalRouter模式介绍

> 对照官方：[GlobalRouter模式介绍](https://cloud.tencent.com/document/product/457/50354) · page_id `50354`

## 概述

GlobalRouter 是 TKE 基于 VPC 全局路由能力实现的容器网络模式。在该模式下，容器的 IP 地址从独立于 VPC 的容器 CIDR 中分配，通过 VPC 路由表实现容器间及容器与 VPC 内其他资源的互通。

核心特点：

- 容器网段与 VPC 网段相互独立，容器 IP 不占用 VPC 子网
- 依赖 VPC 路由表转发容器流量，路由条目数受 VPC 路由表上限约束
- 每个节点从容器 CIDR 中分配一个子网段，用于该节点上 Pod 的 IP 分配
- Service CIDR 从容器 CIDR 的最后一个网段中划分
- 不支持固定 Pod IP

与 VPC-CNI 模式的关键差异：

| 对比维度 | GlobalRouter | VPC-CNI |
|---------|-------------|---------|
| Pod IP 来源 | 容器 CIDR（独立于 VPC） | VPC 子网 |
| 路由方式 | VPC 路由表 | VPC 弹性网卡 |
| 固定 Pod IP | 不支持 | 支持 |
| Pod 与 VPC 资源互通 | 通过路由转发 | 同一网络平面 |
| 适用场景 | 通用场景、传统容器化应用 | 需要固定 IP、低延迟场景 |

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `<Region>` 与凭证已配置。

  ```bash
  tccli --version
  # expected: tccli 版本号输出
  ```

- 已有一个 TKE 集群（GlobalRouter 或 VPC-CNI 均可，本节以查询为主）。

  ```bash
  tccli tke DescribeClusters --region <Region> --Limit 1 --output json
  # expected: {"TotalCount": ..., "Clusters": [...], "RequestId": "..."}
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

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群网络模式 | `tccli tke DescribeClusters --filter` 筛选 `Property` 中的 `NetworkType` | 是 |
| 查看容器网段/Service 网段/VPC | `tccli tke DescribeClusters` 读取 `ClusterNetworkSettings` | 是 |
| 查看单节点 Pod 上限 | `tccli tke DescribeClusters` 读取 `MaxNodePodNum` | 是 |
| 创建 GlobalRouter 集群 | `tccli tke CreateCluster` 设 `ClusterCIDR` 且不设 `Cni`/`Subnets` | 否 |

## 操作步骤

### 1. 识别集群是否为 GlobalRouter 模式

`DescribeClusters` 响应的 `Property` 字段是一个 JSON 字符串，其中的 `NetworkType` 取值为 `GR` 表示 GlobalRouter 模式（`VPC-CNI` 表示 VPC-CNI 模式）。

```bash
tccli tke DescribeClusters --region <Region> --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,NetworkType:Property.NetworkType}"
```

```json
{
  "ClusterId": "cls-example",
  "NetworkType": "GR"
}
```

### 2. 查看集群网络设置详情

查询完整的 `ClusterNetworkSettings`，包含容器 CIDR、Service CIDR、VPC、单节点 Pod 上限、最大 Service 数量等。

```bash
tccli tke DescribeClusters --region <Region> --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,ClusterNetworkSettings:ClusterNetworkSettings}"
```

```json
{
  "ClusterId": "cls-example",
  "ClusterNetworkSettings": {
    "ClusterCIDR": "10.12.0.0/14",
    "ServiceCIDR": "10.15.252.0/22",
    "VpcId": "<VpcId>",
    "Cni": false,
    "MaxNodePodNum": 64,
    "MaxClusterServiceNum": 1024,
    "Ipvs": false,
    "IgnoreClusterCIDRConflict": false,
    "IgnoreServiceCIDRConflict": false
  }
}
```

字段说明：

| 字段 | 含义 |
|------|------|
| `ClusterCIDR` | 容器网络 CIDR，集群内所有 Pod IP 从此范围分配 |
| `ServiceCIDR` | Service 网络 CIDR，ClusterIP 从此范围分配 |
| `VpcId` | 集群所属 VPC |
| `Cni` | `false` 表示 GlobalRouter 模式，`true` 表示 VPC-CNI 模式 |
| `MaxNodePodNum` | 单节点可运行的最大 Pod 数 |
| `MaxClusterServiceNum` | 集群最大 Service 数量 |
| `Ipvs` | 是否启用 IPVS 转发 |

### 3. 查看节点 Pod 子网分配

GlobalRouter 模式下，每个节点从容器 CIDR 中分配一个子网段。可通过 `kubectl` 查看节点上分配的 `podCIDR`。

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,POD-CIDR:.spec.podCIDR
```

```
NAME          POD-CIDR
10.0.0.2      10.12.0.0/24
10.0.0.3      10.12.1.0/24
10.0.0.4      10.12.2.0/24
```

### 4. 查看 VPC 路由表（容器网段相关条目）

GlobalRouter 模式下，每个节点的容器子网会在 VPC 路由表中生成一条路由条目，指向该节点。

```bash
tccli vpc DescribeRouteTables --region <Region> --output json \
  --filter "RouteTableSet[?VpcId=='<VpcId>'] | [0].{RouteTableId:RouteTableId,RouteSet:RouteSet}"
```

```json
{
  "RouteTableId": "rtb-example",
  "RouteSet": [
    {
      "DestinationCidrBlock": "10.12.0.0/24",
      "GatewayType": "CVM",
      "GatewayId": "ins-node1",
      "RouteDescription": "tke-cluster-cls-example"
    },
    {
      "DestinationCidrBlock": "10.12.1.0/24",
      "GatewayType": "CVM",
      "GatewayId": "ins-node2",
      "RouteDescription": "tke-cluster-cls-example"
    }
  ]
}
```

### 5. 使用限制说明

- 同一 VPC 内，不同集群的容器 CIDR 不可重叠
- 容器网段与 VPC 网段不可重叠；若重叠，流量优先在容器网络内转发
- 固定 Pod IP 不支持
- VPC 路由表条目数上限影响集群最大节点数（每条路由对应一个节点）

## 验证

### 控制面（tccli）

```bash
# 确认 NetworkType 为 GR
tccli tke DescribeClusters --region <Region> --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{NetworkType:Property.NetworkType}"

# 确认 Cni 为 false
tccli tke DescribeClusters --region <Region> --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{Cni:ClusterNetworkSettings.Cni}"

# 确认 ClusterCIDR 非空
tccli tke DescribeClusters --region <Region> --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterCIDR:ClusterNetworkSettings.ClusterCIDR}"
```

```json
{
  "NetworkType": "GR",
  "Cni": false,
  "ClusterCIDR": "10.12.0.0/14"
}
```

### 数据面（kubectl）

```bash
# 验证节点已分配 podCIDR
kubectl get nodes -o custom-columns=NAME:.metadata.name,POD-CIDR:.spec.podCIDR
# expected: 每个节点均显示非空的 POD-CIDR

# 验证跨节点 Pod 互通（可选）
kubectl run test-pod --image=busybox --restart=Never -- sleep 3600
kubectl exec test-pod -- ping -c 2 <另一节点上的Pod IP>
kubectl delete pod test-pod
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页为概念说明与只读查询，无额外资源需清理。若执行了 `kubectl run test-pod` 验证，请执行：

```bash
kubectl delete pod test-pod
```

**无计费影响**：GlobalRouter 模式本身不产生额外费用。节点实例、VPC 路由表等资源按各自计费规则收费。

## 排障

### 命令返回错误

| 现象 | 原因 | 处理 |
|------|------|------|
| `DescribeClusters` 返回空 `Clusters` 数组 | `<ClusterId>` 不存在或 `<Region>` 不匹配 | 检查 ClusterId 和 Region 是否正确 |
| `Property.NetworkType` 为 `null` | 旧版集群可能无此字段 | 通过 `ClusterNetworkSettings.Cni` 是否为 `false` 判断 |
| `kubectl get nodes` 报 `Unable to connect` | kubeconfig 未配置或过期 | 重新执行 [连接集群](../../../集群管理/连接集群/tccli%20操作.md) 下载 kubeconfig |

### 查询成功但状态异常

| 现象 | 可能原因 | 处理 |
|------|---------|------|
| `ClusterNetworkSettings.ClusterCIDR` 为空 | 集群信息不完整 | 核对集群状态是否为 `Running`；非 Running 状态集群先排查集群状态 |
| `podCIDR` 未分配（kubectl 显示 `<none>`） | 节点尚未就绪或 IP 池耗尽 | `kubectl describe node <NodeName>` 查看节点状态和事件 |
| 新增节点无 VPC 路由条目 | VPC 路由表条目已达上限 | 检查 VPC 路由表配额，考虑使用云联网收敛路由或切换网络模式 |

## 下一步

- [GlobalRouter 模式集群与 IDC 互通](../GlobalRouter%20模式集群与%20IDC%20互通/tccli%20操作.md)
- [同地域及跨地域 GlobalRouter 模式集群间互通](../同地域及跨地域%20GlobalRouter%20模式集群间互通/tccli%20操作.md)
- [注册 GlobalRouter 模式集群到云联网](../注册%20GlobalRouter%20模式集群到云联网/tccli%20操作.md)
- [容器集群网络方案选型](../../容器集群网络方案选型/tccli%20操作.md)
- [容器集群网络规划](../../容器集群网络规划/tccli%20操作.md)

## 控制台替代

[控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → 基本信息 → 查看「集群网络」字段，网络模式显示为「Global Router」。
