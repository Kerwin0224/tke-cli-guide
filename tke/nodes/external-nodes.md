---
doc_type: How-to
subtype: 6A
fused: true
---
# 扩展节点接入（混合云）

> 将自建 IDC 或第三方（非腾讯云）节点接入 TKE 集群，统一管理混合云工作负载。
> 控制台: [容器服务 - 节点池](https://console.cloud.tencent.com/tke2/nodepool)

> 本文档 Action 均属 **TKE 2018-05-25**（扩展节点功能仅旧版有，2022-05-01 无对应抽象）。

## 概述

扩展节点（External Node）让非腾讯云机器（自建机房、边缘服务器、其他云）作为节点加入 TKE 集群。接入流程：开启集群外部节点支持 → 创建外部节点池 → 获取注册脚本 → 在目标机器执行脚本 → 节点注册上线。

> ⚠️ 扩展节点接入流程与腾讯云 CVM 节点池**完全不同**：CVM 节点由 TKE 创建 CVM 实例，外部节点需用户自行在目标机器跑注册脚本。本文档独立成篇，勿与 [创建节点池](nodepool-create.md)（CVM 节点池）混用。

## 决策依据

### 网络模式（NetworkType）

外部节点支持依赖集群网络模式（实测 `DescribeExternalNodeSupportConfig`）：

| NetworkType | 含义 | 外部节点适用性 |
|:------------|:-----|:-------------|
| `GR`（Global Router） | 容器网段独立，节点走 VPC 路由 | 外部节点需能路由到集群容器网段 |
| `VPC-CNI` | Pod 直接占用 VPC 子网 IP | 外部节点需访问 VPC 子网 |

接入前先查集群网络模式，据此配置外部节点的网络可达性。

### 是否开启公网接入

`EnableExternalNodeSupport` 的 `ClusterExternalConfig.Enabled` 控制开启；若外部节点跨公网接入，需配合 `EnabledPublicConnect`/`PublicConnectUrl`/`PublicCustomDomain`（实测 SupportConfig 含这些字段）。跨公网有安全风险，优先用内网/VPN。

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `ClusterExternalConfig.NetworkType` | EnableExternalNodeSupport | 是 | 网络模式 GR/VPC-CNI |
| `ClusterExternalConfig.SubnetId` | EnableExternalNodeSupport | 是 | 子网 ID |
| `ClusterExternalConfig.ClusterCIDR` | EnableExternalNodeSupport | 是 | 集群容器网段 |
| `ClusterExternalConfig.Enabled` | EnableExternalNodeSupport | 是 | 是否开启 |
| `Name` | CreateExternalNodePool | 是 | 节点池名 |
| `ContainerRuntime` | CreateExternalNodePool | 是 | 容器运行时（如 `containerd`） |
| `RuntimeVersion` | CreateExternalNodePool | 是 | 运行时版本（如 `1.6.9`） |
| `NodePoolId` | DescribeExternalNodeScript | 是 | 节点池 ID（生成注册脚本必填） |
| `Interface` | DescribeExternalNodeScript | 否 | 节点网卡名 |
| `Internal` | DescribeExternalNodeScript | 否 | 是否内网脚本 |

> 参数名实测自各 Action `--generate-cli-skeleton`（P7）。`DeleteExternalNode` 用 `Names[]`（复数），`DrainExternalNode` 用单数 `Name`，`DeleteExternalNodePool` 用 `NodePoolIds[]`——三者字段名不同，勿混。

## 操作步骤

### 步骤 1：查询集群外部节点支持现状

```bash
tccli tke DescribeExternalNodeSupportConfig --ClusterId <CLUSTER_ID> --region <REGION>
# expected: exit 0, Status 字段反映当前支持状态
```
```json
{
    "NetworkType": "GR",
    "Enabled": false,
    "Status": "Disabled",
    "FailedReason": "",
    "EnabledPublicConnect": false,
    "PublicConnectUrl": ""
}
```

> `Status` 状态机：`Disabled`（未开启）→ `enabling`（开启中）→ `Enabled`（已开启）。`FailedReason` 非空表示开启失败的原因。

### 步骤 2：开启外部节点支持（若 Status=Disabled）

```bash
tccli tke EnableExternalNodeSupport --region <REGION> \
  --ClusterId <CLUSTER_ID> \
  --ClusterExternalConfig '{"NetworkType":"GR","SubnetId":"<SUBNET_ID>","ClusterCIDR":"<CLUSTER_CIDR>","Enabled":true}'
# expected: exit 0
```

### 步骤 3：创建外部节点池

```bash
tccli tke CreateExternalNodePool --region <REGION> \
  --ClusterId <CLUSTER_ID> \
  --Name <NODEPOOL_NAME> \
  --ContainerRuntime containerd \
  --RuntimeVersion 1.6.9
# expected: exit 0, 返回 NodePoolId
```

### 步骤 4：获取节点注册脚本

```bash
tccli tke DescribeExternalNodeScript --ClusterId <CLUSTER_ID> --NodePoolId <NODEPOOL_ID> --region <REGION>
# expected: exit 0, 返回可在目标机器执行的注册脚本
```

> ⚠️ `--NodePoolId` 必填，缺失报 `the following arguments are required: --NodePoolId`（exit 252）。必须先完成步骤 3 拿到 NodePoolId。

### 步骤 5：在目标机器执行注册脚本

将步骤 4 返回的脚本在自建 IDC 机器执行，节点注册上线后出现在节点池中。

## 验证

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 支持已开启 | `DescribeExternalNodeSupportConfig` | `Status=Enabled` |
| 节点池存在 | `DescribeExternalNodePools --ClusterId <CLUSTER_ID>` | `TotalCount >= 1` |
| 节点已注册 | `DescribeExternalNode --ClusterId <CLUSTER_ID> --NodePoolId <NODEPOOL_ID>` | 返回节点对象 |
| 注册脚本可用 | `DescribeExternalNodeScript` | 返回非空脚本 |

```bash
tccli tke DescribeExternalNodePools --ClusterId <CLUSTER_ID> --region <REGION>
# expected: exit 0, TotalCount >= 1, NodePoolSet 含新建池
```
```json
{
    "TotalCount": 0,
    "NodePoolSet": [],
    "RequestId": "..."
}
```

> 上图为空结果示例（未创建池时）。创建成功后 `TotalCount` ≥ 1。

```bash
# 查询外部节点（ClusterId + NodePoolId 定位池内节点）
tccli tke DescribeExternalNode --ClusterId <CLUSTER_ID> \
  --NodePoolId <NODEPOOL_ID> --region <REGION>
# expected: exit 0, 返回池内已注册的外部节点列表
```

```bash
# 修改外部节点池（Labels/Taints/DeletionProtection 等，触发 updating → normal）
tccli tke ModifyExternalNodePool --region <REGION> \
  --ClusterId <CLUSTER_ID> --NodePoolId <NODEPOOL_ID> \
  --Labels '[{"Name":"env","Value":"hybrid"}]'
# expected: exit 0
```

> `DescribeExternalNode` 用 `NodePoolId` 定位池后返回池内节点；`ModifyExternalNodePool` 改池级配置（Labels/Taints/DeletionProtection），单数 `NodePoolId`。两者参数以 `--generate-cli-skeleton` 实测为准。

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<CLUSTER_ID>` | 集群 ID | 已存在 | `tccli tke DescribeClusters --region <REGION>` |
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<SUBNET_ID>` | 子网 ID | VPC 子网 | `tccli vpc DescribeSubnets` |
| `<CLUSTER_CIDR>` | 集群容器网段 | CIDR 格式 | `DescribeClusterStatus` 或自定义 |
| `<NODEPOOL_NAME>` | 节点池名 | 集群内唯一 | 自定义 |
| `<NODEPOOL_ID>` | 节点池 ID | 步骤 3 返回 | `DescribeExternalNodePools` |

## 清理

### 驱逐并删除外部节点

```bash
# 1. 优雅驱逐节点上的 Pod
tccli tke DrainExternalNode --ClusterId <CLUSTER_ID> --Name <NODE_NAME> --region <REGION>
# expected: exit 0

# 2. 删除节点（Force=true 强制，false 优雅）
tccli tke DeleteExternalNode --ClusterId <CLUSTER_ID> --Names '["<NODE_NAME>"]' --Force false --region <REGION>
# expected: exit 0
```

### 删除外部节点池

```bash
tccli tke DeleteExternalNodePool --ClusterId <CLUSTER_ID> --NodePoolIds '["<NODEPOOL_ID>"]' --Force false --region <REGION>
# expected: exit 0
```

> `DeleteExternalNode` 用 `Names[]`（节点名数组），`DeleteExternalNodePool` 用 `NodePoolIds[]`（节点池 ID 数组），两者都支持 `Force` 强制删除。

## 副作用

- **开启外部节点支持**会修改集群网络配置，影响集群路由。生产集群操作前确认网络模式匹配。
- **DrainExternalNode** 会驱逐节点上所有 Pod，业务会重调度到其他节点。
- **DeleteExternalNode Force=true** 不等 Pod 驱逐直接删除，可能导致 Pod 丢失。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `the following arguments are required: --NodePoolId`（exit 252） | `DescribeExternalNodeScript` 缺 NodePoolId | 先 `CreateExternalNodePool` 拿 NodePoolId |
| `AuthFailure.UnauthorizedOperation` (tke:CreateExternalNodePool) | CAM 策略要求集群带特定标签（实测要求 `billing&kerwinwjyan`） | 给集群加授权标签或申请权限 |
| `Status=Disabled` 始终不变 | `EnableExternalNodeSupport` 未成功 | 查 `FailedReason` 字段定位开启失败原因 |
| 节点注册脚本执行后节点未上线 | 外部机器网络不可达集群 / 运行时版本不兼容 | 检查机器到集群网络、`ContainerRuntime`/`RuntimeVersion` 匹配 |
| `DeleteExternalNode` 报节点不存在 | `Names[]` 用了节点池 ID 而非节点名 | 用节点名（`DescribeExternalNode` 返回的 `Names`） |

> 实测 CAM 拒绝样本（ap-guangzhou）：
> `code:AuthFailure.UnauthorizedOperation ... you are not authorized to perform operation (tke:CreateExternalNodePool)`

## 下一步

- 腾讯云 CVM 节点池（区别于外部节点）：[创建节点池](nodepool-create.md)
- 节点健康检查：[节点健康检查策略](health-check.md)
- 节点实例操作（驱逐/移出/升级）：[节点实例操作](instance-ops.md)

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `DescribeExternalNodeSupportConfig` | 验证 | 2018-05-25 | 查询支持状态（含 NetworkType/Status/FailedReason） |
| `EnableExternalNodeSupport` | 主操作 | 2018-05-25 | 开启外部节点支持（ClusterExternalConfig） |
| `CreateExternalNodePool` | 主操作 | 2018-05-25 | 创建外部节点池（含 Runtime/Labels/Taints） |
| `DescribeExternalNodePools` | 验证 | 2018-05-25 | 查询节点池列表 |
| `DescribeExternalNodeScript` | 验证 | 2018-05-25 | 获取注册脚本（NodePoolId 必填） |
| `DescribeExternalNode` | 验证 | 2018-05-25 | 查询外部节点（NodePoolId + Names） |
| `ModifyExternalNodePool` | 主操作 | 2018-05-25 | 修改节点池（Labels/Taints/DeletionProtection） |
| `DrainExternalNode` | 主操作 | 2018-05-25 | 驱逐节点（单数 Name） |
| `DeleteExternalNode` | 清理 | 2018-05-25 | 删除节点（Names[] + Force） |
| `DeleteExternalNodePool` | 清理 | 2018-05-25 | 删除节点池（NodePoolIds[] + Force） |
