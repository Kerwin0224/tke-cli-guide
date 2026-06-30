# 删除原生节点

> 对照官方：[删除原生节点](https://cloud.tencent.com/document/product/457/78199) · page_id `78199` · tccli ≥3.1.107 · API 2018-05-25

## 概述

通过 `DeleteClusterNodePool` 删除原生节点池及其下的 CXM 实例。删除节点池会销毁或退还其下所有 CXM 实例（取决于 `InstanceDeleteMode`），请确保工作负载已迁移至其他节点或节点池。删除为**异步操作**，需轮询确认完成。

与普通节点池不同，原生节点池删除时：
- CXM 实例默认被**销毁**（`InstanceDeleteMode=terminate`），不可恢复
- 节点上的本地存储（emptyDir 等）数据会丢失
- 节点池级别的 Annotations、Management 配置一并删除

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 3.1.107

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 获取集群 ID -- 从集群列表中确认目标集群
tccli tke DescribeClusters --region <Region>
# expected: exit 0，从输出中提取 $.Clusters[0].ClusterId

# 4. 检查 CAM 权限 -- 需要以下 Action 名
#    tke:DeleteClusterNodePool, tke:DescribeClusterNodePools
#    tke:DescribeClusterInstances, tke:ModifyNodePoolDesiredCapacityAboutAsg
#    tke:DescribeClusterKubeconfig
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表
```

```json
{
    "Response": {
        "NodePoolSet": [
            {
                "NodePoolId": "np-example",
                "Name": "native-pool-example",
                "ClusterInstanceId": "cls-example",
                "LifeState": "normal",
                "LaunchConfigurationId": "asc-example",
                "AutoscalingGroupId": "asg-example",
                "NodeCountSummary": {
                    "ManuallyAdded": {
                        "Joining": 0,
                        "Initializing": 0,
                        "Normal": 0,
                        "Total": 0
                    },
                    "AutoscalingAdded": {
                        "Joining": 0,
                        "Initializing": 0,
                        "Normal": 1,
                        "Total": 1
                    }
                },
                "AutoscalingGroupStatus": "disabled",
                "MaxNodesNum": 5,
                "MinNodesNum": 1,
                "DesiredNodesNum": 1,
                "Tags": [
                    {
                        "Key": "billing",
                        "Value": "<your-name>"
                    }
                ]
            }
        ],
        "TotalCount": 1,
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

> 从上述输出中获取目标节点池的 `<NodePoolId>`：取 `$.Response.NodePoolSet[n].NodePoolId`。

### 资源检查

```bash
# 5. 确认节点池存在并查看当前节点
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，返回节点池详情，包含 NodeCountSummary

# 6. 查看节点池内节点列表
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --Filters '[{"Name":"nodepool-id","Values":["<NodePoolId>"]}]'
# expected: 返回节点实例列表，确认将要被删除的节点

# 7. 确认集群状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus 为 "Running"
```

```json
{
    "Response": {
        "NodePool": {
            "NodePoolId": "np-example",
            "Name": "native-pool-example",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "LaunchConfigurationId": "asc-example",
            "AutoscalingGroupId": "asg-example",
            "NodeCountSummary": {
                "ManuallyAdded": {
                    "Joining": 0,
                    "Initializing": 0,
                    "Normal": 0,
                    "Total": 0
                },
                "AutoscalingAdded": {
                    "Joining": 0,
                    "Initializing": 0,
                    "Normal": 1,
                    "Total": 1
                }
            },
            "AutoscalingGroupStatus": "disabled",
            "MaxNodesNum": 5,
            "MinNodesNum": 1,
            "DesiredNodesNum": 1,
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.6.9"
            },
            "Tags": [
                {
                    "Key": "billing",
                    "Value": "<your-name>"
                }
            ],
            "DeletionProtection": false
        },
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

## 控制台与 CLI 参数映射

| 字段 | 类型 | 必填 | 幂等 | 取值与约束 | 错误后果 |
|------|------|:--:|:--:|------|------|
| `ClusterId` | String | 是 | 是 | 集群 ID，格式 `cls-` 开头 | 集群不存在 -> `ResourceNotFound.ClusterNotFound` |
| `NodePoolIds` | String[] | 是 | 否 | 节点池 ID 列表，一次可删除多个 | 节点池不存在 -> `ResourceNotFound.NodePoolNotFound` |
| `KeepInstance` | Bool | 否 | 否 | `false`（销毁实例，停止计费）或 `true`（保留实例，继续计费）。默认 `false` | 误设 `true` -> 实例残留并持续计费 |
| `ForceDelete` | Bool | 否 | 否 | `true` 跳过 Pod 驱逐直接删除；`false`（默认）先驱逐再删除 | 设为 `true` -> 运行中 Pod 被强制终止，可能导致数据丢失 |

### 关键字段说明

**`KeepInstance`**：设为 `true` 时仅删除节点池资源记录，保留其下所有 CXM 实例。该操作**不可逆**——保留的 CXM 实例将继续按量计费，且不再受节点池管理。如需重新管理，需通过 `AddExistedInstances` 将实例添加到其他节点池。

**`ForceDelete`**：设为 `true` 时跳过 Pod 驱逐和优雅终止，直接销毁节点。仅在节点已不可用（如网络隔离、kubelet 僵死）无法正常驱逐时使用。正常删除流程应使用 `kubectl drain` 安全驱逐 Pod 后再删除节点池，避免工作负载服务中断。

## 操作步骤

### 步骤 1：确认工作负载已迁移

> **警告**：删除节点池会销毁或退还其下的 CXM 实例。请先确认节点上的 Pod 已通过 `kubectl drain` 驱逐并调度至其他节点。

#### 选择依据

- **先驱逐再删除**：使用 `kubectl drain` 安全迁移 Pod 后再删除节点池，避免 Pod 被强制终止导致请求丢失。
- **先缩容到 0**：通过 `ModifyNodePoolDesiredCapacityAboutAsg --DesiredCapacity 0` 将节点池关联的伸缩组期望实例数设为 0，比直接删节点池更可控——可在缩容过程中观察 Pod 迁移情况。
- **`ForceDelete` 仅限紧急场景**：仅在节点已不可用（如网络隔离、kubelet 僵死）无法正常驱逐时使用。

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 Kubeconfig 内容（base64 编码）
```

**预期输出**：

```json
{
    "Response": {
        "Kubeconfig": "YXBpVm...（base64 编码的 kubeconfig 内容）",
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

> **将 Kubeconfig 写入文件**：使用以下命令解码并保存 Kubeconfig：
> ```bash
> # 方法一：从 JSON 输出中提取并解码
> tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
>     | python3 -c "import sys,json; print(json.load(sys.stdin)['Response']['Kubeconfig'])" \
>     | base64 -d > <KubeconfigPath>
> 
> # 方法二：设置 KUBECONFIG 环境变量指向该文件
> export KUBECONFIG=<KubeconfigPath>
> ```

```bash
# 查看节点池内节点列表（需先将 Kubeconfig 写入文件，见上方说明）
kubectl --kubeconfig <KubeconfigPath> get nodes
# expected: 确认哪些节点属于要删除的节点池

# 查看节点上运行的 Pod
kubectl --kubeconfig <KubeconfigPath> get pods -A -o wide \
    --field-selector spec.nodeName=<NodeName>
# expected: 列出该节点上所有 Pod，确认可迁移

# 安全驱逐节点（逐出 Pod 并标记不可调度）
kubectl --kubeconfig <KubeconfigPath> drain <NodeName> \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --force
# expected: 节点被 cordon，Pod 被调度到其他节点

# 确认 Pod 已迁移
kubectl --kubeconfig <KubeconfigPath> get pods -A -o wide \
    --field-selector spec.nodeName=<NodeName>
# expected: 节点上无残留 Pod（DaemonSet 除外）
```

### 步骤 2（推荐）：先缩容到 0 再删除

为减少风险，建议先将节点池缩容到 0，确认无残留节点后再删除节点池：

```bash
# 步骤 2a：将伸缩组期望实例数设为 0
tccli tke ModifyNodePoolDesiredCapacityAboutAsg --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --DesiredCapacity 0
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
# 步骤 2b：轮询确认节点数为 0
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: NodeCountSummary 各字段 Total 为 0，DesiredNodesNum 为 0
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
                "ManuallyAdded": {
                    "Joining": 0,
                    "Initializing": 0,
                    "Normal": 0,
                    "Total": 0
                },
                "AutoscalingAdded": {
                    "Joining": 0,
                    "Initializing": 0,
                    "Normal": 0,
                    "Total": 0
                }
            }
        },
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

```bash
# 步骤 2c：从 Kubernetes 侧确认节点已移除
kubectl --kubeconfig <KubeconfigPath> get nodes
# expected: 被缩容的节点不再出现在列表中
```

### 步骤 3：删除节点池

缩容到 0 并确认后，执行节点池删除：

```bash
# 删除节点池（销毁 CXM 实例）
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId>"]' \
    --KeepInstance false
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

### 步骤 4：确认删除完成

删除为异步操作，轮询确认节点池不再存在：

```bash
# 轮询节点池列表
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
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
    --ClusterId <ClusterId>
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
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 返回 ResourceNotFound.NodePoolNotFound 错误
```

### 步骤 5（可选）：清理关联资源

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
| 节点池存在性 | `DescribeClusterNodePools` -> `TotalCount` | 减 1，被删除的 NodePoolId 不再出现 |
| 节点池详情 | `DescribeClusterNodePoolDetail` -> 用原 `<NodePoolId>` 查询 | 返回 `ResourceNotFound` 错误 |
| 实例存在性 | `DescribeClusterInstances` -> `InstanceSet` | 不包含被删除节点池的实例 |
| 集群状态 | `DescribeClusters` -> `ClusterStatus` | `Running`（不受影响） |

### 数据面（kubectl）

```bash
kubectl --kubeconfig <KubeconfigPath> get nodes
# expected: 被删除节点池的节点不再出现

kubectl --kubeconfig <KubeconfigPath> get pods -A -o wide
# expected: 所有 Pod 在剩余节点上正常运行，无 Pending 或 Error 状态
```

## 清理

> **副作用警告**：此操作本身即为清理操作。以下副作用需注意：

| 资源 | 是否自动清理 | 说明 |
|------|:--:|------|
| 节点池记录 | 是 | `NodePoolSet` 中移除 |
| 池内 CXM 实例 | 取决于 `--KeepInstance` | `false` 时销毁 CXM，计费停止；`true` 时保留 CXM 继续计费 |
| CXM 数据盘（本地） | 是 | 本地数据随 CXM 销毁而丢失，不可恢复 |
| CBS 云硬盘（随实例创建） | 取决于购买方式 | 购买时勾选"随实例销毁"则同步销毁；否则需手动清理 |
| 弹性伸缩组 | 是 | 关联的 Auto Scaling Group 自动删除 |
| 安全组规则 | 否 | 如果安全组被其他资源引用，规则会残留；确认安全组不再被引用后手动清理 |

> **手动清理建议（当 CBS 云硬盘未随实例销毁时）：**
>
> ```bash
> # 查看是否存在未被释放的云硬盘
> tccli cbs DescribeDisks --region <Region> \
>     --Filters '[{"Name":"instance-id","Values":["<InstanceId>"]}]'
> # 如需释放不用的云硬盘
> tccli cbs TerminateDisks --region <Region> \
>     --DiskIds '["<DiskId>"]'
> ```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DeleteClusterNodePool` 返回 `ResourceNotFound.NodePoolNotFound` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 确认节点池是否存在 | 节点池已不存在（可能已被删除） | 确认 `<NodePoolId>` 正确 |
| 删除失败 -- 节点池内有残留节点 | `tccli tke DescribeClusterNodePoolDetail` 查看 `NodeCountSummary` | `DesiredNodesNum` > 0，节点未被驱逐 | 先用 `ModifyNodePoolDesiredCapacityAboutAsg --DesiredCapacity 0` 缩容到 0，或使用 `ForceDelete` 跳过驱逐 |
| 删除失败 -- `InvalidParameterValue.KeepInstance` | 检查 `KeepInstance` 参数值 | 参数值不在允许的布尔范围内 | `KeepInstance` 设为 `false` 或 `true` |
| 删除超时 -- `RequestId` 返回但节点池长时间处于 `deleting` | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 轮询 | CXM 实例销毁缓慢或 API 服务繁忙 | 等待 3-5 分钟后重查，超过 10 分钟则保留 region、`<ClusterId>`、`<NodePoolId>`、RequestId -- [提交工单](https://console.cloud.tencent.com/workorder) |
| 缩容到 0 失败 -- `InvalidParameterValue.DesiredCapacity` | `tccli tke DescribeClusterNodePoolDetail` 查看 `MinNodesNum` | `MinNodesNum` > 0，不允许缩容到 0 | 先通过 `ModifyClusterNodePool --MinNodesNum 0` 将最小节点数改为 0，再缩容 |

### 删除成功但存在残留

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池删除后 EIP 未释放 | `tccli vpc DescribeAddresses --region <Region>` 查看 EIP 列表 | EIP 绑定的 CXM 实例已销毁但 EIP 独立计费 | 在 VPC 控制台或 `tccli vpc DisassociateAddress` + `tccli vpc ReleaseAddresses` 手动释放 |
| 节点池删除后磁盘未释放 | `tccli cbs DescribeDisks --region <Region>` 过滤检查 | 数据盘开启"随实例释放"保护或 CBS 独立计费 | 在 CBS 控制台手动销毁磁盘 |
| 节点池删除后安全组仍有规则残留 | `tccli vpc DescribeSecurityGroupPolicies --region <Region> --SecurityGroupId <SecurityGroupId>` | 安全组在节点池之外被其他资源引用 | 确认安全组不再被引用后，在 VPC 控制台清理规则 |
| 节点池删除后 kubectl 仍显示节点（短时间） | `kubectl --kubeconfig <KubeconfigPath> get nodes` | 节点 NotReady 状态有宽限期（默认 5 分钟），kubelet 心跳超时后才会移除 | 等待 5 分钟后节点自动从 Kubernetes 中移除 |

## 下一步

- [新建原生节点](../新建原生节点/tccli%20操作.md) -- page_id `78198`
- [修改原生节点](../修改原生节点/tccli%20操作.md) -- page_id `103599`
- [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md) -- page_id `78648`
- [原生节点概述](../原生节点概述/tccli%20操作.md)

## 控制台替代

[容器服务控制台 -> 集群 -> 节点管理 -> 原生节点 -> 删除](https://console.cloud.tencent.com/tke2/cluster)
