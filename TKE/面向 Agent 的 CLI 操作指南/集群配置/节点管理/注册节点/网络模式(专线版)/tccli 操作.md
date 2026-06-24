# 网络模式(专线版)
> 对照官方：[网络模式（专线版）](https://cloud.tencent.com/document/product/457/79748) · page_id `79748`

## 概述

在专线场景下，包含注册节点的集群可能同时拥有来自不同网络环境的计算节点（如 IDC 机房和公有云 VPC）。根据 TKE 集群网络类型（GlobalRouter、VPC-CNI、Cilium-Overlay），IDC 注册节点上 Pod 的网络使用存在不同限制。

本页说明三种网络模式在注册节点场景下的行为差异，并提供通过 tccli 查询集群网络配置的命令。

三种模式对比如下：

| 网络模式 | IDC 节点 Pod 网络 | 封装协议 | 性能损耗 | 固定 Pod IP | 适用场景 |
|----------|------------------|---------|---------|------------|---------|
| Cilium-Overlay | Overlay 网络，Pod 与云上节点共享容器网段 | Cilium VXLan | 约 10% 以内 | 不支持 | 新建集群，IDC 节点入集群 |
| GlobalRouter | Hostnetwork 模式，复用宿主机网络 | 无 | 无 | 不适用 | 存量 GR 集群接入 IDC 节点 |
| VPC-CNI | Hostnetwork 模式，复用宿主机网络 | 无 | 无 | 不适用 | 存量 VPC-CNI 集群接入 IDC 节点 |

> **注意：** Cilium-Overlay 模式下，请勿同时创建超级节点池，否则会导致普通节点/注册节点上的 Pod 与超级节点池 Pod 无法直接互通。

## 前置条件

每条均可 COPY-PASTE 验证。

- [环境准备](../../../../环境准备.md)：`tccli`、地域 `ap-guangzhou` 与凭证已配置。

  ```bash
  tccli tke help > /dev/null && echo "OK" # expected: OK
  ```

- 集群已存在，且已开启注册节点支持。

  ```bash
  tccli tke DescribeClusters --region <Region> --output json \
    --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,EnableExternalNode:EnableExternalNode}"
  # expected: {"ClusterId":"<ClusterId>","EnableExternalNode":true}
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

- 已创建注册节点池（可选，用于查看节点池级别配置）。

  ```bash
  tccli tke DescribeExternalNodePools --region <Region> --ClusterId <ClusterId>
  # expected: TotalCount >= 0，NodePoolSet 列表
  ```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "LifeState": "<LifeState>",
  "ClusterCIDR": "<ClusterCIDR>"
}
```

## 控制台与 CLI 参数映射

| 官方 / 控制台 | 含义 | tccli / 说明 | 幂等 |
|---------------|------|-------------|------|
| 网络模式（Cilium-Overlay / GlobalRouter / VPC-CNI） | 集群创建时选择的容器网络类型 | `DescribeClusters` 返回 `ClusterNetworkSettings.CiliumMode` 与 `ClusterNetworkSettings.Cni` | 只读 |
| 注册节点网络类型（CiliumBGP / HostNetwork） | 开启注册节点支持时指定的网络类型 | `DescribeExternalNodeSupportConfig` 返回 `NetworkType`；`EnableExternalNodeSupport` 入参 `NetworkType` 可选 `HostNetwork` 或 `CiliumBGP` | 只读查 / 开启时指定 |
| 注册节点开关 | 集群是否允许注册 IDC 节点 | `DescribeClusters` 返回 `EnableExternalNode`（Boolean） | 只读 |
| 集群 CIDR | 容器网络网段（Cilium-Overlay 下为 Overlay 网段） | `DescribeExternalNodeSupportConfig` 返回 `ClusterCIDR` | 只读 |

## 操作步骤

### Cilium-Overlay 网络模式

Cilium-Overlay 是 TKE 基于 Cilium VXLan 实现的容器网络插件，为容器层提供统一的网络平面，Pod 无需感知自身运行在 IDC 节点还是公有云节点上。

**特点：**
- 云上节点与注册节点共享指定的容器网段。
- 容器网段分配灵活，容器 IP 范围不占用其他 VPC 网段。
- 使用 Cilium VXLan 隧道封装协议构建 Overlay 网络。

**查询集群是否使用 Cilium-Overlay：**

```bash
tccli tke DescribeClusters --region <Region> --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,CiliumMode:ClusterNetworkSettings.CiliumMode,Cni:ClusterNetworkSettings.Cni,VpcId:ClusterNetworkSettings.VpcId}"
```

```json
{
  "ClusterId": "<ClusterId>",
  "CiliumMode": "Overlay",
  "Cni": true,
  "VpcId": "<VpcId>"
}
```

**查询注册节点支持配置（确认 NetworkType 与 ClusterCIDR）：**

```bash
tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId>
```

```json
{
  "ClusterCIDR": "10.0.0.0/16",
  "NetworkType": "CiliumBGP",
  "SubnetId": "<SubnetId>",
  "Enabled": true,
  "Status": "Running",
  "RequestId": "..."
}
```

> **关键字段说明：** `NetworkType` 为 `CiliumBGP` 表示注册节点使用 Cilium-Overlay 网络；`ClusterCIDR` 为容器 Overlay 网段。

**使用须知：**
- Pod IP 无法从集群外直接访问。
- Pod 可访问集群外节点，但 Pod IP 会被 SNAT 为 Pod 所在节点的 IP。
- 不支持固定 Pod IP。
- Cilium VXLan 隧道封装存在约 10% 以内的性能损耗。

### GlobalRouter 和 VPC-CNI 模式

对于已有的 GlobalRouter 或 VPC-CNI 类型 TKE 集群，当接入 IDC 节点以利用存量资源时，IDC 上的 Pod 仅支持 **Hostnetwork 模式**，即复用 IDC 节点的宿主机网络。

**查询集群网络类型：**

```bash
tccli tke DescribeClusters --region <Region> --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,Cni:ClusterNetworkSettings.Cni,ClusterCIDR:ClusterNetworkSettings.ClusterCIDR,Subnets:ClusterNetworkSettings.Subnets}"
```

```json
{
  "ClusterId": "<ClusterId>",
  "Cni": false,
  "ClusterCIDR": "10.168.0.0/16",
  "Subnets": null
}
```

> `Cni` 为 `false` 且 `ClusterCIDR` 非空 → GlobalRouter 模式；`Cni` 为 `true` 且 `Subnets` 非空 → VPC-CNI 模式。

**查询注册节点支持配置：**

```bash
tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId>
```

```json
{
  "ClusterCIDR": "",
  "NetworkType": "HostNetwork",
  "SubnetId": "<SubnetId>",
  "Enabled": true,
  "Status": "Running",
  "RequestId": "..."
}
```

> **关键字段说明：** `NetworkType` 为 `HostNetwork` 表示注册节点使用宿主机网络模式（Hostnetwork）。此时 `ClusterCIDR` 为空，因为 Pod 直接复用宿主机 IP。

**使用须知：**
- Hostnetwork 复用宿主机网络，监听同一端口的 Pod 应避免调度到同一台宿主机（通过污点和亲和性等机制），否则 Pod 将无法进入 Running 状态。

### 查询注册节点池网络类型

```bash
tccli tke DescribeExternalNodePools --region <Region> --ClusterId <ClusterId> --output json \
  --filter "NodePoolSet[*].{NodePoolId:NodePoolId,Name:Name,NetworkType:NetworkType,LifeState:LifeState}"
```

```json
[
  {
    "NodePoolId": "np-example",
    "Name": "idc-pool",
    "NetworkType": "CiliumBGP",
    "LifeState": "normal"
  }
]
```

## 验证

### 控制面（tccli）

确认集群网络配置与注册节点支持状态一致：

```bash
# 1. 确认集群维度的网络设置
tccli tke DescribeClusters --region <Region> --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,EnableExternalNode:EnableExternalNode,CiliumMode:ClusterNetworkSettings.CiliumMode}"
# expected: EnableExternalNode 为 true，CiliumMode 反映集群网络类型
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

```bash
# 2. 确认注册节点支持的网络类型
tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId> --output json \
  --filter "{NetworkType:NetworkType,Enabled:Enabled,Status:Status}"
# expected: Enabled 为 true，Status 为 Running
```

```json
{
  "ClusterCIDR": "<ClusterCIDR>",
  "NetworkType": "<NetworkType>",
  "SubnetId": "<SubnetId>",
  "Enabled": "<Enabled>",
  "AS": "<AS>",
  "SwitchIP": "<SwitchIP>"
}
```

```bash
# 3. 确认节点池网络配置
tccli tke DescribeExternalNodePools --region <Region> --ClusterId <ClusterId> --output json \
  --filter "NodePoolSet[*].{NodePoolId:NodePoolId,NetworkType:NetworkType}"
# expected: 返回节点池列表，NetworkType 与 DescribeExternalNodeSupportConfig 一致
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "LifeState": "<LifeState>",
  "ClusterCIDR": "<ClusterCIDR>"
}
```

### 数据面（kubectl）

验证注册节点上 Pod 的实际网络模式（需先[连接集群](../../../集群管理/连接集群/tccli%20操作.md)）：

```bash
# 查看注册节点
kubectl get nodes -l tke.cloud.tencent.com/external-node=true
# expected: 返回 IDC 注册节点列表

# 在 Cilium-Overlay 模式下查看 Pod IP 与宿主机 IP 关系
kubectl get pods -n default -o wide
# expected: Pod IP 来自 ClusterCIDR 网段，与宿主机 IP 不在同一网段

# 在 Hostnetwork 模式下查看 Pod 是否使用 hostNetwork
kubectl get pods -n default -o json --filter "items[?spec.hostNetwork=='true'] | [0].{Name:metadata.name,HostIP:status.hostIP,PodIP:status.podIP}"
# expected: Pod IP 与 HostIP 相同
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页为概念说明与只读查询，无额外资源需要清理。

> **无副作用警告：** 所有命令均为只读查询，不会修改集群配置或创建资源。无需清理步骤。

## 排障

### 命令返回错误

| 现象 | 处理 |
|------|------|
| `DescribeExternalNodeSupportConfig` 返回 `Enabled: false` | 集群未开启注册节点支持，需先执行 `EnableExternalNodeSupport` 或通过控制台开启 |
| `DescribeClusters` 返回 `EnableExternalNode: false` | 同上，集群注册节点功能未开启 |
| `ClusterId` 不存在或鉴权失败 | 检查 `--region` 与账号权限，确认集群 ID 正确 |
| `DescribeExternalNodePools` 返回空 `NodePoolSet` | 未创建注册节点池，属于正常状态（无节点池时仅影响节点池级别查询） |

### 配置查询成功但状态异常

| 现象 | 处理 |
|------|------|
| `DescribeExternalNodeSupportConfig.Status` 非 Running（如 `Failed`） | 查看 `FailedReason` 字段，可能是开启过程中出错，建议重新开启或提工单 |
| `CiliumMode` 为空但预期为 Cilium-Overlay | 集群可能为 GlobalRouter 或 VPC-CNI 模式，检查 `Cni` 与 `ClusterCIDR` 字段确认 |
| 注册节点 Pod 无法与超级节点池 Pod 互通 | Cilium-Overlay 与超级节点池冲突，二者不可共存。考虑移除超级节点池或将注册节点迁移至其他集群 |

## 下一步

- [创建注册节点（专线版）](../创建注册节点（专线版）/tccli%20操作.md) · [注册节点概述](../注册节点概述/tccli%20操作.md)
- [容器集群网络方案选型](../../../网络管理/容器集群网络方案选型/tccli%20操作.md) · [容器集群网络规划](../../../网络管理/容器集群网络规划/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 基本信息页](https://console.cloud.tencent.com/tke2/cluster) 可查看集群网络模式与注册节点开关；[官方文档](https://cloud.tencent.com/document/product/457/79748) 含模式对比说明。
