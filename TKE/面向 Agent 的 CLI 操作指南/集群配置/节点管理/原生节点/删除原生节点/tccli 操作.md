# 删除原生节点池（tccli）

> 对照官方：[删除原生节点](https://cloud.tencent.com/document/product/457/78199) · page_id `78199`

## 概述

通过 `DeleteClusterNodePool` 删除原生节点池及其下的 CXM 实例。删除节点池会销毁或退还其下所有 CXM 实例（取决于 `InstanceDeleteMode`），请确保工作负载已迁移至其他节点或节点池。删除为**异步操作**，需轮询确认完成。

与普通节点池不同，原生节点池删除时：
- CXM 实例默认被**销毁**（`InstanceDeleteMode=terminate`），不可恢复
- 节点上的本地存储（emptyDir 等）数据会丢失
- 节点池级别的 Annotations、Management 配置一并删除

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
#    tke:DeleteClusterNodePool, tke:DescribeClusterNodePools
#    tke:DescribeClusterInstances, tke:ModifyClusterNodePool
#    tke:DescribeClusterKubeconfig
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
# 4. 确认节点池存在并查看当前节点
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，返回节点池详情，包含 NodeCountSummary

# 5. 查看节点池内节点列表
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Filters '[{"Name":"node-pool-id","Values":["NODE_POOL_ID"]}]'
# expected: 返回节点实例列表，确认将要被删除的节点

# 6. 确认集群状态
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterState "Running"
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI 命令 | 幂等 |
|-----------|---------|:--:|
| 删除节点池 | `DeleteClusterNodePool` | 是 |

## 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-` 开头 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `NodePoolId` | String | 是 | 节点池 ID，格式 `np-` 开头 | 节点池不存在 → `ResourceNotFound.NodePoolNotFound` |
| `InstanceDeleteMode` | String | 否 | `terminate`（销毁实例）或 `retain`（退还实例，保留计费） | 误设 `retain` → 实例残留并持续计费 |
| `ForceDelete` | Bool | 否 | `true` 跳过 Pod 驱逐直接删除；`false`（默认）先驱逐再删除 | 设为 `true` → 运行中 Pod 被强制终止，可能导致数据丢失 |

## 操作步骤

### 步骤1：确认工作负载已迁移

> **警告**：删除节点池会销毁或退还其下的 CXM 实例。请先确认节点上的 Pod 已通过 `kubectl drain` 驱逐并调度至其他节点。

#### 选择依据

- **先驱逐再删除**：使用 `kubectl drain` 安全迁移 Pod 后再删除节点池，避免 Pod 被强制终止导致请求丢失。
- **先缩容到 0**：通过修改 `DesiredNodesNum=0` 间接删除节点，比直接删节点池更可控——可在缩容过程中观察 Pod 迁移情况。
- **`ForceDelete` 仅限紧急场景**：仅在节点已不可用（如网络隔离、kubelet 僵死）无法正常驱逐时使用。

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID

# 查看节点池内节点列表（将 Kubeconfig 写入 KUBECONFIG_PATH 后执行）
kubectl --kubeconfig KUBECONFIG_PATH get nodes
# expected: 确认哪些节点属于要删除的节点池

# 查看节点上运行的 Pod
kubectl --kubeconfig KUBECONFIG_PATH get pods -A -o wide --field-selector spec.nodeName=NODE_NAME
# expected: 列出该节点上所有 Pod，确认可迁移

# 安全驱逐节点（逐出 Pod 并标记不可调度）
kubectl --kubeconfig KUBECONFIG_PATH drain NODE_NAME \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --force
# expected: 节点被 cordon，Pod 被调度到其他节点

# 确认 Pod 已迁移
kubectl --kubeconfig KUBECONFIG_PATH get pods -A -o wide --field-selector spec.nodeName=NODE_NAME
# expected: 节点上无残留 Pod（DaemonSet 除外）
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### 步骤2（推荐）：先缩容到 0 再删除

为减少风险，建议先将节点池缩容到 0，确认无残留节点后再删除节点池。这种方式可以在缩容过程中逐节点观察 Pod 迁移情况，出现问题时可以回滚。

```bash
# 步骤2a：缩容到 0
tccli tke ModifyClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    --DesiredNodesNum 0
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

```bash
# 步骤2b：轮询确认节点数为 0
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: NodeCountSummary 各字段 Total 为 0
```

**缩容完成预期输出**：

```json
{
    "Response": {
        "NodePool": {
            "NodePoolId": "np-example",
            "Name": "native-pool-example",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "DesiredNodesNum": 0,
            "NodeCountSummary": {
                "ManuallyAdded": {"Total": 0},
                "AutoscalingAdded": {"Total": 0}
            }
        },
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

```bash
# 步骤2c：从 Kubernetes 侧确认节点已移除
kubectl --kubeconfig KUBECONFIG_PATH get nodes
# expected: 被缩容的节点不再出现在列表中
```

### 步骤3：删除节点池

缩容到 0 并确认后，执行节点池删除：

```bash
# 删除节点池（销毁 CXM 实例）
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
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

### 步骤4：确认删除完成

删除为异步操作，轮询确认节点池不再存在：

```bash
# 轮询节点池列表
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: 返回的 NodePoolSet 中不包含已删除的 NodePoolId
```

**预期输出**：

```json
{
    "Response": {
        "TotalCount": 0,
        "NodePoolSet": [],
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

```bash
# 确认集群中不再有该节点池的实例
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 返回的 InstanceSet 中不包含被删除节点的 InstanceId
```

**预期输出**：

```json
{
    "Response": {
        "TotalCount": 0,
        "InstanceSet": [],
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

```bash
# 尝试查询已删除的节点池详情（应返回错误）
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: 返回 ResourceNotFound.NodePoolNotFound 错误
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 步骤5（可选）：清理关联资源

节点池删除后，以下资源可能需要手动清理：

```bash
# 检查是否有残留的弹性公网 IP
tccli vpc DescribeAddresses --region <Region>
# expected: 如果 EIP 未随实例释放，需手动解绑和释放

# 检查是否有残留的云硬盘
tccli cbs DescribeDisks --region <Region>
# expected: 如果数据盘开启"随实例释放"保护，需手动销毁
```

## 验证

### 控制面（tccli）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点池存在性 | `DescribeClusterNodePools` → `TotalCount` | 减 1，被删除的 NodePoolId 不再出现 |
| 节点池详情 | `DescribeClusterNodePoolDetail` → 用原 NODE_POOL_ID 查询 | 返回 `ResourceNotFound` 错误 |
| 实例存在性 | `DescribeClusterInstances` → `InstanceSet` | 不包含被删除节点池的实例 |
| 集群状态 | `DescribeClusterStatus` → `ClusterState` | `Running`（不受影响） |

### 数据面（kubectl）

```bash
kubectl --kubeconfig KUBECONFIG_PATH get nodes
# expected: 被删除节点池的节点不再出现

kubectl --kubeconfig KUBECONFIG_PATH get pods -A -o wide
# expected: 所有 Pod 在剩余节点上正常运行，无 Pending 或 Error 状态
```

## 清理

本页即删除操作，无额外清理步骤。若有其他关联资源（如独立计费的 EIP、未随实例释放的数据盘、安全组规则等），需手动在对应产品控制台或 API 中清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DeleteClusterNodePool` 返回 `ResourceNotFound.NodePoolNotFound` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 确认 NodePoolId | NodePoolId 不存在或已被删除 | 核实节点池是否已被他人删除 |
| 删除失败 — 节点池内有残留节点 | `tccli tke DescribeClusterNodePoolDetail` 查看 `NodeCountSummary` | `DesiredNodesNum` > 0，节点未被驱逐 | 先缩容到 0，或使用 `ForceDelete` 跳过驱逐 |
| 删除失败 — `InvalidParameterValue` | 检查 `InstanceDeleteMode` 和 `ForceDelete` | 参数值不在允许的枚举范围内 | `InstanceDeleteMode` 设为 `terminate` 或 `retain` |
| 删除超时 — `RequestId` 返回但节点池长时间处于 `deleting` | `tccli tke DescribeClusterNodePoolDetail` 轮询 | CXM 实例销毁缓慢或 API 服务繁忙 | 等待 3-5 分钟后重查，超过 10 分钟提交工单 |
| 缩容到 0 时返回 `InvalidParameterValue.DesiredNodesNum` | 检查 `MinNodesNum` 值 | `MinNodesNum` > 0，不允许缩容到 0 | 先将 `MinNodesNum` 改为 0，再修改 `DesiredNodesNum` |

### 删除成功但存在残留

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池删除后 EIP 未释放 | `tccli vpc DescribeAddresses --region <Region>` 查看 EIP 列表 | EIP 绑定的 CXM 实例已销毁但 EIP 独立计费 | 在 VPC 控制台或 `tccli vpc DisassociateAddress` + `ReleaseAddresses` 手动释放 |
| 节点池删除后磁盘未释放 | `tccli cbs DescribeDisks --region <Region>` 过滤检查 | 数据盘开启"随实例释放"保护或 CBS 独立计费 | 在 CBS 控制台手动销毁磁盘 |
| 节点池删除后安全组仍有规则残留 | `tccli vpc DescribeSecurityGroupPolicies --region <Region> --SecurityGroupId SECURITY_GROUP_ID` | 安全组在节点池之外被其他资源引用 | 确认安全组不再被引用后，在 VPC 控制台清理规则 |
| 节点池删除后 kubectl 仍显示节点（短时间） | `kubectl get nodes` | 节点 NotReady 状态有宽限期（默认 5 分钟），kubelet 心跳超时后才会移除 | 等待 5 分钟后节点自动从 Kubernetes 中移除 |

## 下一步

- [新建原生节点](../新建原生节点/tccli%20操作.md) — page_id `78198`
- [修改原生节点](../修改原生节点/tccli%20操作.md) — page_id `103599`
- [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md) — page_id `78648`
- [原生节点概述](../原生节点概述/tccli%20操作.md)

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点 → 删除](https://console.cloud.tencent.com/tke2/cluster)
