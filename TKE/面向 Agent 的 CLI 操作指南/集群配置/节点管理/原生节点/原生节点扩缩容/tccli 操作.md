# 原生节点扩缩容（tccli）

> 对照官方：[原生节点扩缩容](https://cloud.tencent.com/document/product/457/78648) · page_id `78648`

## 概述

通过 `ModifyClusterNodePool` 实现原生节点池的**手动扩缩容**（修改 `DesiredNodesNum`）和**弹性伸缩配置**（修改 `MinNodesNum`/`MaxNodesNum` 范围）。扩缩容为**异步操作**，需轮询确认节点数达到预期。弹性伸缩开启后，CA（Cluster Autoscaler）会根据节点资源使用率自动在设定范围内伸缩。

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
#    tke:DescribeClusterInstances
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
# 4. 确认节点池当前状态和弹性伸缩配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: LifeState 为 "normal"，返回 DesiredNodesNum, MinNodesNum, MaxNodesNum

# 5. 确认当前节点数
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Filters '[{"Name":"node-pool-id","Values":["NODE_POOL_ID"]}]'
# expected: 返回当前节点列表及其状态
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI 命令 | 幂等 |
|-----------|---------|:--:|
| 调整节点池期望节点数 | `ModifyNodePoolDesiredCapacityAboutAsg` | 是 |

## 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-` 开头 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `NodePoolId` | String | 是 | 节点池 ID，格式 `np-` 开头 | 节点池不存在 → `ResourceNotFound.NodePoolNotFound` |
| `DesiredNodesNum` | Int | 否 | 期望节点数，必须在 `[MinNodesNum, MaxNodesNum]` 范围内 | 超范围 → `InvalidParameterValue.DesiredNodesNum` |
| `MinNodesNum` | Int | 否 | 伸缩下限，须 >= 0 | 大于 `DesiredNodesNum` → `InvalidParameterValue.MinNodesNum` |
| `MaxNodesNum` | Int | 否 | 伸缩上限，须 >= `DesiredNodesNum` | 小于期望数 → `InvalidParameterValue.MaxNodesNum` |

### 弹性伸缩策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| ZonePriority | 优先在首选可用区扩容，首选可用区资源不足时可回退到其他可用区 | 单可用区为主、多可用区为辅 |
| ZoneEquality | 多可用区均匀打散节点，需配置多个子网覆盖不同可用区 | 高可用、多可用区容灾 |

## 操作步骤

### 步骤1：手动扩缩容（修改期望节点数）

#### 选择依据

- **手动扩缩容**：直接修改 `DesiredNodesNum` 即可。节点池会将实际节点数调整到目标值——如期望数 > 当前数则扩容创建新节点，期望数 < 当前数则缩容驱逐并退还节点。
- **缩容行为**：缩容时优先驱逐低负载节点上的 Pod，Pod 会被调度到剩余节点。若 Pod 有 PDB 限制，缩容可能受阻。

#### 扩容到 5 个节点

`scale-nodepool-up.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "DesiredNodesNum": 5
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://scale-nodepool-up.json
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

#### 缩容到 2 个节点

`scale-nodepool-down.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "DesiredNodesNum": 2
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://scale-nodepool-down.json
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

### 步骤2：配置弹性伸缩范围

设置 `MinNodesNum` 和 `MaxNodesNum` 范围后，CA 在此范围内自动伸缩。

`scale-nodepool-autoscale.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "MinNodesNum": 2,
  "MaxNodesNum": 10,
  "DesiredNodesNum": 3
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://scale-nodepool-autoscale.json
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

### 步骤3：轮询确认扩容完成

扩缩容为异步操作，轮询确认节点数变化：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: DesiredNodesNum 已更新，NodeCountSummary 趋近期望值
```

**扩容后预期输出**：

```json
{
    "Response": {
        "NodePool": {
            "NodePoolId": "np-example",
            "Name": "native-pool-example",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "Type": "Native",
            "DesiredNodesNum": 5,
            "MinNodesNum": 2,
            "MaxNodesNum": 10,
            "NodeCountSummary": {
                "ManuallyAdded": {"Total": 4},
                "AutoscalingAdded": {"Total": 1}
            },
            "AutoscalingGroupStatus": "normal"
        },
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤4：确认节点状态

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Filters '[{"Name":"node-pool-id","Values":["NODE_POOL_ID"]}]'
# expected: InstanceSet 数量与期望节点数一致，InstanceState 为 "running"
```

**预期输出**：

```json
{
    "Response": {
        "TotalCount": 5,
        "InstanceSet": [
            {
                "InstanceId": "ins-example-01",
                "InstanceRole": "WORKER",
                "InstanceState": "running",
                "FailedReason": "",
                "NodePoolId": "np-example"
            }
        ],
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

## 验证

### 控制面（tccli）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 期望节点数 | `DescribeClusterNodePoolDetail` → `DesiredNodesNum` | 与修改值一致 |
| 节点总数 | `DescribeClusterNodePoolDetail` → `NodeCountSummary` 各字段之和 | 等于 `DesiredNodesNum` |
| 伸缩组状态 | `DescribeClusterNodePoolDetail` → `AutoscalingGroupStatus` | `normal` |
| 弹性伸缩范围 | `DescribeClusterNodePoolDetail` → `MinNodesNum` / `MaxNodesNum` | 与修改值一致 |
| 实例状态 | `DescribeClusterInstances` → `InstanceState` | `running` |

### 数据面（kubectl）

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID

# 确认节点数
kubectl --kubeconfig KUBECONFIG_PATH get nodes
# expected: 节点数量与 DesiredNodesNum 一致，STATUS 为 Ready

# 确认新节点已调度 Pod
kubectl --kubeconfig KUBECONFIG_PATH get pods -A -o wide
# expected: Pod 分布在新节点上
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 清理

本页为修改操作，无需额外清理。如需回退扩缩容，重新执行 `ModifyClusterNodePool` 传入原始值。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `InvalidParameterValue.DesiredNodesNum` | 检查当前 `MinNodesNum` 和 `MaxNodesNum` | `DesiredNodesNum` 不在 `[MinNodesNum, MaxNodesNum]` 范围内 | 先扩大 `MaxNodesNum` 或缩小 `MinNodesNum` |
| 返回 `ResourceNotFound.NodePoolNotFound` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 检查 NodePoolId | NodePoolId 不存在或已删除 | 核实正确的 NodePoolId |
| 返回 `QuotaExceeded.NodeCount` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 统计已有节点数 | 集群节点总数超过配额 | 申请提额或减少其他节点池节点数 |

### 伸缩成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 扩容后节点数不增长 | `tccli tke DescribeClusterNodePoolDetail` → `AutoscalingGroupStatus` | 伸缩组异常或 CVM/CXM 配额不足 | 检查伸缩组状态，确认 CXM 配额充足 |
| 扩容后新节点 `InstanceState` 为 `failed` | `tccli tke DescribeClusterInstances` → `FailedReason` | 子网 IP 耗尽、安全组配置错误或镜像拉取失败 | 更换子网或扩容子网 CIDR，检查安全组规则 |
| 缩容后 Pod 驱逐卡住 | `kubectl --kubeconfig KUBECONFIG_PATH get pods -A -o wide` 查看 Pending Pod | PDB 阻止驱逐或 Pod 有本地存储 | 调整 PDB 或手动迁移有状态 Pod |
| 弹性伸缩不触发 | `kubectl --kubeconfig KUBECONFIG_PATH describe configmap cluster-autoscaler-status -n kube-system` | CA 配置未生效或节点资源使用率未达到阈值 | 检查 CA 日志，手动扩容/缩容测试伸缩组连通性 |
| 缩容后节点仍出现在 `InstanceSet` | `tccli tke DescribeClusterInstances` 查看 `InstanceState` | 节点正在 `terminating` 状态，销毁未完成 | 等待 3-5 分钟，节点会自动消失 |

## 下一步

- [修改原生节点](../修改原生节点/tccli%20操作.md) — page_id `103599`
- [声明式操作实践](../声明式操作实践/tccli%20操作.md) — page_id `78649`
- [故障自愈规则](../故障自愈规则/tccli%20操作.md) — page_id `78650`
- [新建原生节点](../新建原生节点/tccli%20操作.md) — page_id `78198`

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点 → 弹性伸缩](https://console.cloud.tencent.com/tke2/cluster)
