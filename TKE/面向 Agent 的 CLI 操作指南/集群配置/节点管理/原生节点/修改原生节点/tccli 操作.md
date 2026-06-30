# 修改原生节点池（tccli）

> 对照官方：[修改原生节点](https://cloud.tencent.com/document/product/457/103599) · page_id `103599`

## 概述

通过 `ModifyClusterNodePool` 修改原生节点池的配置：期望节点数、机型列表、系统盘/数据盘、安全组、Management 参数、Annotations 等。多数变更仅对**新增节点**生效，存量节点不受影响（安全组可选存量更新）。支持**差分更新**：只传需变更的字段，未传字段保持不变。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:ModifyClusterNodePool, tke:DescribeClusterNodePoolDetail
#    tke:DescribeClusterNodePools
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 资源检查

```bash
# 4. 确认节点池存在并查看当前配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，返回节点池详情，LifeState 为 "normal"
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI 命令 | 幂等 |
|-----------|---------|:--:|
| 修改节点池 | `ModifyClusterNodePool` | 是 |
| 查看节点池详情 | `DescribeClusterNodePoolDetail` | 是 |

## 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-` 开头 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `NodePoolId` | String | 是 | 节点池 ID，格式 `np-` 开头 | 节点池不存在 → `ResourceNotFound.NodePoolNotFound` |
| `Name` | String | 否 | 节点池新名称，2-64 字符 | 重名 → `InvalidParameterValue` |
| `DesiredNodesNum` | Int | 否 | 期望节点数，必须 >= `MinNodesNum` 且 <= `MaxNodesNum` | 超范围 → `InvalidParameterValue.DesiredNodesNum` |
| `MinNodesNum` | Int | 否 | 最小节点数 | 大于 `DesiredNodesNum` → `InvalidParameterValue.MinNodesNum` |
| `MaxNodesNum` | Int | 否 | 最大节点数 | 小于 `DesiredNodesNum` → `InvalidParameterValue.MaxNodesNum` |
| `InstanceTypes` | String[] | 否 | CXM 机型列表，最多 10 种 | 包含非 CXM 机型 → `InvalidParameterValue.InstanceType` |
| `SecurityGroupIds` | String[] | 否 | 安全组 ID 列表 | 安全组不存在 → `InvalidParameterValue.SecurityGroupId` |
| `Management` | Object | 否 | 管理参数（差分更新） | 配置错误 → 节点管理功能异常 |
| `Annotations` | Object[] | 否 | 声明式注解（差分更新） | 格式错误 → `InvalidParameterValue.Annotation` |

### 变更生效范围

| 变更项 | 生效范围 | 说明 |
|--------|---------|------|
| `Name` | 存量 + 新增 | 立即生效 |
| `DesiredNodesNum` | 存量 + 新增 | 触发扩缩容 |
| `MinNodesNum` / `MaxNodesNum` | 新增 | 弹性伸缩范围 |
| `InstanceTypes` | 仅新增节点 | 存量节点机型不变 |
| `SecurityGroupIds` | 新增（默认） | 可选存量更新 |
| `Management` | 仅新增节点 | 存量节点管理参数不变 |
| `Annotations` | 仅新增节点 | 存量节点注解不变 |

## 操作步骤

### 步骤1：修改期望节点数（手动扩缩容）

#### 选择依据

- **差分更新**：`ModifyClusterNodePool` 只需传入 `ClusterId`、`NodePoolId` 和要变更的字段，其他字段无需重复传入。
- **期望节点数**：直接修改 `DesiredNodesNum` 即可触发手动扩缩容，节点池会自动创建或退还 CXM 实例。

`modify-nodepool-desired.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "DesiredNodesNum": 3
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-desired.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤2：修改 Management 参数

`Management` 参数控制节点的 DNS、hosts、内核参数等。仅对新增节点生效。

`modify-nodepool-management.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Management": {
    "Nameservers": ["183.60.83.19", "183.60.82.98"],
    "Hosts": ["10.0.0.1 registry.internal"],
    "KernelArgs": [
      "net.core.somaxconn=65535",
      "net.ipv4.tcp_tw_reuse=1"
    ]
  }
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-management.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤3：修改 Annotations 声明式注解

`Annotations` 支持差分更新：传入新的 Annotation 列表会合并覆盖同名 Key，未传入的 Key 保持不变。

`modify-nodepool-annotations.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Annotations": [
    {
      "Name": "tke.cloud.tencent.com/enable-inplace-update",
      "Value": "true"
    },
    {
      "Name": "tke.cloud.tencent.com/example",
      "Value": "updated-value"
    }
  ]
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-annotations.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤4：修改安全组（含存量节点更新）

安全组变更默认仅对新增节点生效。如需存量节点也更新安全组，需在控制台勾选"存量更新"选项，或通过数据面 MachineSet 配置。

### 步骤5：确认变更生效

修改为异步操作，需轮询确认：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: 返回节点池详情，修改的字段已更新
```

**预期输出**：

```json
{
    "Response": {
        "NodePool": {
            "NodePoolId": "np-example",
            "Name": "native-pool-example",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "Type": "Native",
            "DesiredNodesNum": 3,
            "MinNodesNum": 1,
            "MaxNodesNum": 5,
            "NodeCountSummary": {
                "ManuallyAdded": {"Total": 2},
                "AutoscalingAdded": {"Total": 1}
            },
            "AutoscalingGroupStatus": "normal"
        },
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

## 验证

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 期望节点数 | `DescribeClusterNodePoolDetail` → `DesiredNodesNum` | 与修改值一致 |
| 节点总数 | `DescribeClusterNodePoolDetail` → `NodeCountSummary` 各字段 `Total` 之和 | 趋近于新 `DesiredNodesNum` |
| Management | `DescribeClusterNodePoolDetail` → `Management` | 新增节点应用新参数 |
| Annotations | `DescribeClusterNodePoolDetail` → `Annotations` | 列表包含更新的 Key-Value |
| 伸缩组状态 | `DescribeClusterNodePoolDetail` → `AutoscalingGroupStatus` | `normal` |
| 实例状态 | `DescribeClusterInstances` → `InstanceState` | `running` |

### 确认节点已注册

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID

# 确认节点数
kubectl --kubeconfig KUBECONFIG_PATH get nodes
# expected: 节点数量与 DesiredNodesNum 一致，STATUS 为 Ready
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 清理

本页为修改操作，无需额外清理。如需回退修改，重新执行 `ModifyClusterNodePool` 传入修改前的值。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `ResourceNotFound.NodePoolNotFound` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 检查 NodePoolId | NodePoolId 不存在或拼写错误 | 核实正确的 NodePoolId |
| 返回 `InvalidParameterValue.DesiredNodesNum` | 检查当前 `MinNodesNum` 和 `MaxNodesNum` | 修改值超出弹性伸缩范围 | 先调整 `MinNodesNum`/`MaxNodesNum` 范围 |
| 返回 `InvalidParameterValue.InstanceType` | `tccli cvm DescribeZoneInstanceConfigInfos --region <Region>` 确认 CXM 机型可用 | 机型列表包含非 CXM 机型或在指定可用区不可用 | 移除不支持的机型，确保全部为 CXM 前缀 |
| 修改 `Management` 后新增节点不生效 | `tccli tke DescribeClusterNodePoolDetail` 查看 `Management` | `Management` 仅对新增节点生效，存量节点不变 | 扩容触发新节点创建以应用新参数 |

### 修改成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 修改 `DesiredNodesNum` 后节点数不变 | `tccli tke DescribeClusterNodePoolDetail` → `AutoscalingGroupStatus` | 伸缩组异常或 CVM 配额不足 | 检查 `AutoscalingGroupStatus`，确认 CVM 配额充足 |
| 修改 `Annotations` 后存量节点未变化 | `tccli tke DescribeClusterInstances` 查看节点注解 | Annotations 仅对新增节点生效（控制面 API 修改节点池级注解） | 重新创建节点以应用新注解，或通过数据面 `kubectl annotate node` 单独处理存量节点 |
| 安全组修改后存量节点仍被旧安全组限制 | `tccli tke DescribeClusterInstances` 查看 SecurityGroupIds | 安全组变更默认仅对新增节点生效 | 通过控制台勾选「存量更新」，或登录 CXM 控制台手动修改存量节点安全组 |

## 下一步

- [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md) — page_id `78648`
- [声明式操作实践](../声明式操作实践/tccli%20操作.md) — page_id `78649`
- [删除原生节点](../删除原生节点/tccli%20操作.md) — page_id `78199`
- [原生节点开启公网访问](../原生节点开启公网访问/tccli%20操作.md) — page_id `82334`

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点 → 详情 → 编辑](https://console.cloud.tencent.com/tke2/cluster)
