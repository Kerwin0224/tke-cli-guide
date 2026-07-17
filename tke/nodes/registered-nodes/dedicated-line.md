---
doc_type: How-to
subtype: 6A
fused: true
---
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [容器服务安全组设置](https://cloud.tencent.com/document/product/457/9084) · [注册节点价格和配额说明](https://cloud.tencent.com/document/product/457/79747)
>
> 配额：专线带宽配额、节点规格受账号限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：CIDR 与 VPC 冲突致路由不可达；节点脚本泄露致未授权加入；专线中断致节点失联。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

# 创建注册节点（专线版）

> 通过专线 / 内网 / VPN 将自备机器接入已有标准集群，支持 CLB 流量接入与 Prometheus 监控。
> 控制台: [容器服务 - 节点池 - 注册节点](https://console.cloud.tencent.com/tke2/nodepool)

专线版接入要求目标机器与集群 VPC 网络互通（专线、VPN 或对等连接）。目标机器操作系统须为 TencentOS Server 3.1 或 2.4（TK4）。

## 触发条件

- 集群内已有云上节点（注册节点特性要求系统组件运行在云上节点）。
- `DescribeExternalNodeSupportConfig` 返回 `Status=Disabled`，需先开启外部节点支持。
- 你要在已开启支持的集群中创建注册节点池并注册机器。

## 准备工作

- 已创建 TKE 标准集群（见[创建集群](../../clusters/create.md)）。
- 已配置 tccli 凭证（见[配置凭证](../../../getting-started/credentials.md)）。
- 目标机器与集群 VPC 网络互通，且操作系统为 TencentOS Server 3.1 / 2.4（TK4）。

## 决策依据

### 网络模式

`DescribeExternalNodeSupportConfig` 返回集群外部节点支持的网络模式（**入参**与**出参**可能不同）：

| NetworkType（Enable 入参） | 含义 | 适用 |
|:---------------------------|:-----|:-----|
| `HostNetwork` | 主机网络模式 | 目标机器与集群同 VPC / 可达；**无需** `ClusterCIDR` |
| `CiliumBGP` | Cilium BGP 模式 | 需 BGP 互通；可配 `ClusterCIDR` |

> ⚠️ `EnableExternalNodeSupport` 的 `ClusterExternalConfig.NetworkType` **仅** `HostNetwork` / `CiliumBGP`。**不是** `GR` / `VPC-CNI`（那是集群容器网络模型，见 `CreateCluster` 的 `NetworkType`）。

接入前先查当前配置，再按目标网络可达性选型。

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `ClusterExternalConfig.NetworkType` | EnableExternalNodeSupport | 是 | **仅** `HostNetwork` / `CiliumBGP` |
| `ClusterExternalConfig.SubnetId` | EnableExternalNodeSupport | 是 | 子网 ID |
| `ClusterExternalConfig.ClusterCIDR` | EnableExternalNodeSupport | 否 | 集群容器网段；`HostNetwork` 时无需填 |
| `ClusterExternalConfig.Enabled` | EnableExternalNodeSupport | 否 | **已废弃**；是否开启专线连接能力 |
| `Name` | CreateExternalNodePool | 是 | 节点池名 |
| `ContainerRuntime` | CreateExternalNodePool | 是 | 容器运行时，如 `containerd` |
| `RuntimeVersion` | CreateExternalNodePool | 是 | 运行时版本，如 `1.6.9` |
| `NodePoolId` | DescribeExternalNodeScript | 是 | 节点池 ID（生成脚本必填） |
| `Interface` | DescribeExternalNodeScript | 否 | 节点网卡名 |
| `Internal` | DescribeExternalNodeScript | 否 | 是否内网脚本 |

> `DeleteExternalNode` 用 `Names[]`（节点名），`DrainExternalNode` 用单数 `Name`，`DeleteExternalNodePool` 用 `NodePoolIds[]`——三者字段名不同。

## 操作步骤

### 步骤 1：查询集群外部节点支持现状

```bash
tccli tke DescribeExternalNodeSupportConfig --ClusterId <CLUSTER_ID> --region <REGION>
# expected: exit 0, Status 字段反映当前支持状态
```
```json
{
    "NetworkType": "HostNetwork",
    "Enabled": false,
    "Status": "Disabled",
    "FailedReason": "",
    "EnabledPublicConnect": false,
    "PublicConnectUrl": ""
}
```

`Status` 合法枚举：`Disabled`（未开启）→ `Initializing`（开启中）→ `Enabled`（已开启）/ `InitFailed`（开启失败）。`FailedReason` 非空表示开启失败原因。`Enabled` 布尔可能与 `Status` 不同步，**以 `Status` 为准**。

### 步骤 2：开启外部节点支持（若 Status=Disabled）

```bash
# NetworkType 仅 HostNetwork / CiliumBGP（非 GR/VPC-CNI）
tccli tke EnableExternalNodeSupport --region <REGION> \
  --ClusterId <CLUSTER_ID> \
  --ClusterExternalConfig '{"NetworkType":"HostNetwork","SubnetId":"<SUBNET_ID>"}'
# expected: exit 0
```

### 步骤 3：创建注册节点池

```bash
tccli tke CreateExternalNodePool --region <REGION> \
  --ClusterId <CLUSTER_ID> \
  --Name <NODEPOOL_NAME> \
  --ContainerRuntime containerd \
  --RuntimeVersion 1.6.9
# expected: exit 0, 返回 NodePoolId
```

### 步骤 4：获取节点注册脚本 {#步骤-4获取节点注册脚本}

```bash
tccli tke DescribeExternalNodeScript --ClusterId <CLUSTER_ID> --NodePoolId <NODEPOOL_ID> --region <REGION>
# expected: exit 0, 顶层含 Command（下载/执行命令）、Link（COS 链接）、Token（临时密钥）
```

> `--NodePoolId` 必填，缺失报 `the following arguments are required: --NodePoolId`（exit 252）。必须先完成步骤 3 拿到 NodePoolId。在目标机执行的是响应里的 `Command`。

### 步骤 5：在目标机器执行注册脚本

将步骤 4 返回的 `Command` 在目标机器执行，节点注册上线后出现在节点池中。

## 验证

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 支持已开启 | `DescribeExternalNodeSupportConfig` | `Status=Enabled` |
| 节点池存在 | `DescribeExternalNodePools --ClusterId <CLUSTER_ID>` | `TotalCount >= 1` |
| 节点已注册 | `DescribeExternalNode --ClusterId <CLUSTER_ID> --NodePoolId <NODEPOOL_ID>` | 返回节点对象 |
| 注册脚本可用 | `DescribeExternalNodeScript` | `Command`/`Link` 非空 |

```bash
tccli tke DescribeExternalNodePools --ClusterId <CLUSTER_ID> --region <REGION>
# expected: exit 0, TotalCount >= 1, NodePoolSet 含新建池
```

```bash
# 查询节点池内节点
tccli tke DescribeExternalNode --ClusterId <CLUSTER_ID> \
  --NodePoolId <NODEPOOL_ID> --region <REGION>
# expected: exit 0，返回 Nodes[]+TotalCount（NodePoolId 可选，传则限定单池）
```

| 占位符 | 含义 | 获取方式 |
|--------|------|---------|
| `<CLUSTER_ID>` | 集群 ID | `tccli tke DescribeClusters --region <REGION>` |
| `<REGION>` | 地域，如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<SUBNET_ID>` | 子网 ID | `tccli vpc DescribeSubnets`；或 `DescribeClusters --version 2018-05-25` → `Clusters[].ClusterNetworkSettings.SubnetId` |
| `<CLUSTER_CIDR>` | 集群容器网段（CIDR；CiliumBGP 等场景） | `DescribeClusters --version 2018-05-25` → `Clusters[].ClusterNetworkSettings.ClusterCIDR`（**非** `DescribeClusterStatus`） |
| `<NODEPOOL_NAME>` | 节点池名（集群内唯一） | 自定义 |
| `<NODEPOOL_ID>` | 节点池 ID（步骤 3 返回） | `DescribeExternalNodePools` |

## 清理

下线节点与节点池见[移除注册节点](remove.md)。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `the following arguments are required: --NodePoolId`（exit 252） | `DescribeExternalNodeScript` 缺 NodePoolId | 先 `CreateExternalNodePool` 拿 NodePoolId |
| `InvalidParameter.Param`（`invalid NetworkType`） | `ClusterExternalConfig.NetworkType` 传了 `GR`/`VPC-CNI` 等集群网络模型值 | 改用 `HostNetwork`（默认）或 `CiliumBGP` |
| `InvalidParameter.Param`（`clusterCIDR should not be empty`） | `CiliumBGP` 未传 `ClusterCIDR` | 从 `DescribeClusters --version 2018-05-25` → `ClusterNetworkSettings.ClusterCIDR` 取值后重试 |
| `AuthFailure.UnauthorizedOperation` (tke:CreateExternalNodePool) | CAM 策略要求集群带特定标签 | 给集群加授权标签或申请权限 |
| `Status=Disabled`/`InitFailed` | `EnableExternalNodeSupport` 未成功或初始化失败 | 查 `FailedReason`/`Progress`；修正子网/网络后重试 Enable |
| 脚本执行后节点未上线 | 目标机器网络不可达集群 / 运行时版本不兼容 | 检查网络连通性、`ContainerRuntime` / `RuntimeVersion` 匹配 |
| `DeleteExternalNode` 报节点不存在 | `Names[]` 用了节点池 ID 而非节点名 | 用 `DescribeExternalNode` 返回的节点名 |

## 收尾确认

```bash
# 端到端核对：节点已注册上线（核心交付物）
tccli tke DescribeExternalNode --ClusterId "<CLUSTER_ID>" \
  --NodePoolId "<NODEPOOL_ID>" --region ap-guangzhou
# expected: 返回 Nodes[]+TotalCount>=1，节点对象非空

# 汇总核对：支持开启 → 池存在 → 节点注册 → 脚本可用
tccli tke DescribeExternalNodeSupportConfig --ClusterId "<CLUSTER_ID>" --region ap-guangzhou \
  --filter "{status:Status,network:NetworkType}" \
  && tccli tke DescribeExternalNodePools --ClusterId "<CLUSTER_ID>" --region ap-guangzhou \
  --filter "NodePoolSet[0].{state:LifeState,name:Name}" \
  && tccli tke DescribeExternalNode --ClusterId "<CLUSTER_ID>" \
  --NodePoolId "<NODEPOOL_ID>" --region ap-guangzhou \
  --filter "{total:TotalCount,nodes:Nodes}" \
  && tccli tke DescribeExternalNodeScript --ClusterId "<CLUSTER_ID>" \
  --NodePoolId "<NODEPOOL_ID>" --region ap-guangzhou \
  --filter "{command:Command,link:Link}"
# expected: Status=Enabled；节点池 LifeState=normal；节点 total>=1 且 nodes 非空；Command 或 Link 非空
```

## 下一步

- 修改节点池配置：[编辑注册节点池](edit-pool.md)
- 节点健康检查：[节点健康检查策略](../health-check.md)
- 对外暴露服务：[流量接入](traffic-access.md)
- 公网接入：[创建注册节点（公网版）](public-network.md)
