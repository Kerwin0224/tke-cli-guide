# 原生节点扩缩容

> 对照官方：[原生节点扩缩容](https://cloud.tencent.com/document/product/457/78648) · page_id `78648` · tccli ≥1.0.0 · API 2018-05-25

## 概述

通过 `ModifyNodePoolDesiredCapacityAboutAsg` 实现原生节点池的**手动扩缩容**（直接修改期望节点数）和配合 `ModifyClusterNodePool` 设置弹性伸缩范围（`MinNodesNum`/`MaxNodesNum`）。扩缩容为**异步操作**，需轮询确认节点数达到预期。

`ModifyNodePoolDesiredCapacityAboutAsg` 是针对原生节点池自动伸缩组（ASG）期望容量的专用 API，直接调整伸缩组内实例数量，比 `ModifyClusterNodePool` 修改 `DesiredNodesNum` 更精确地控制伸缩行为。

> ⚠️ **副作用警告**：扩容会创建新的 CXM 实例并产生计费（按量计费 CVM 实例 + 系统盘费用）。缩容会驱逐节点上的 Pod，可能影响正在运行的工作负载。详见[副作用声明](#副作用声明)。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 -- 需要以下 Action 名
#    tke:ModifyNodePoolDesiredCapacityAboutAsg, tke:ModifyClusterNodePool
#    tke:DescribeClusterNodePoolDetail, tke:DescribeClusterNodePools
#    tke:DescribeClusterInstances, tke:DescribeClusterLevelAttribute
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
                "Type": "Native",
                "DesiredNodesNum": 2,
                "MinNodesNum": 1,
                "MaxNodesNum": 5,
                "AutoscalingGroupId": "asg-example"
            }
        ],
        "TotalCount": 1,
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 资源检查

```bash
# 4. 检查集群层级（tier）及节点数上限
tccli tke DescribeClusterLevelAttribute --region <Region> \
    --ClusterID <ClusterId>
# expected: 返回当前 tier 的节点数上限（Items[0].NodeCount）

# 5. 确认节点池当前状态和弹性伸缩配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: LifeState 为 "normal"，返回 DesiredNodesNum, MinNodesNum, MaxNodesNum
```

```json
{
    "Response": {
        "NodePool": {
            "NodePoolId": "np-example",
            "Name": "native-pool-example",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "Type": "Native",
            "DesiredNodesNum": 2,
            "MinNodesNum": 1,
            "MaxNodesNum": 5,
            "NodeCountSummary": {
                "ManuallyAdded": {"Total": 0},
                "AutoscalingAdded": {"Total": 2}
            },
            "AutoscalingGroupStatus": "normal"
        },
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

```bash
# 6. 确认当前节点数和伸缩组状态
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --Filters '[{"Name":"node-pool-id","Values":["<NodePoolId>"]}]'
# expected: 返回当前节点列表及其状态
```

```json
{
    "Response": {
        "TotalCount": 2,
        "InstanceSet": [
            {
                "InstanceId": "ins-example-01",
                "InstanceRole": "WORKER",
                "InstanceState": "running",
                "FailedReason": "",
                "NodePoolId": "np-example"
            },
            {
                "InstanceId": "ins-example-02",
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

```bash
# 7. 确认 ASG 伸缩组信息
tccli tke DescribeClusterAsGroups --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回伸缩组列表，确认目标节点池关联的 ASG
```

## 控制台与 CLI 参数映射

### ModifyNodePoolDesiredCapacityAboutAsg 关键字段

| 控制台字段 | CLI 参数 | 类型 | 必填 | 幂等 | 取值与约束 | 错误后果 |
|-----------|---------|------|:--:|:--:|------|------|
| 集群 ID | `ClusterId` | String | 是 | 是 | 集群 ID，格式 `cls-` 开头 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| 节点池 ID | `NodePoolId` | String | 是 | 是 | 节点池 ID，格式 `np-` 开头，须为 Native 类型 | 节点池不存在 → `ResourceNotFound.NodePoolNotFound` |
| 期望节点数 | `DesiredCapacity` | Int | 否 | 否 | 须满足 `MinNodesNum ≤ DesiredCapacity ≤ MaxNodesNum` 且不超过集群 tier 节点数上限 | 超范围 → `InvalidParameterValue.Size`；超 tier 限制 → `InvalidParameter.Param`（集群层级限制） |
| 集群层级上限（只读） | — | Int | — | — | 由 `DescribeClusterLevelAttribute` 查询，当前 tier 的 `NodeCount` 为集群最大节点数 | 超出时 `DesiredCapacity` 设置被拒绝，需升级集群层级 |

### ModifyClusterNodePool 弹性伸缩范围字段

| 控制台字段 | CLI 参数 | 类型 | 必填 | 幂等 | 取值与约束 | 错误后果 |
|-----------|---------|------|:--:|:--:|------|------|
| 最小节点数 | `MinNodesNum` | Int | 否 | 否 | 伸缩下限，须 ≥ 0 | 大于 `DesiredNodesNum` → `InvalidParameterValue.Size` |
| 最大节点数 | `MaxNodesNum` | Int | 否 | 否 | 伸缩上限，须 ≥ `DesiredNodesNum`，且不超过集群 tier 节点数上限 | 小于期望数 → `InvalidParameterValue.Size` |

### 弹性伸缩策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| ZonePriority | 优先在首选可用区扩容，首选可用区资源不足时可回退到其他可用区 | 单可用区为主、多可用区为辅 |
| ZoneEquality | 多可用区均匀打散节点，需配置多个子网覆盖不同可用区 | 高可用、多可用区容灾 |

## 操作步骤

### 步骤 1：手动扩缩容（通过 ModifyNodePoolDesiredCapacityAboutAsg）

#### 选择依据

- **`ModifyNodePoolDesiredCapacityAboutAsg` vs `ModifyClusterNodePool`**：前者直接操作底层 ASG 伸缩组的期望容量，响应更快，适合精确控制节点数。后者通过节点池参数间接调整，适合配合其他节点池配置一起修改。
- **扩容行为**：期望数 > 当前数则 ASG 自动创建新 CXM 实例并加入集群。新节点加入后开始接受 Pod 调度。
- **缩容行为**：期望数 < 当前数则 ASG 缩容，优先驱逐低负载节点上的 Pod，Pod 被调度到剩余节点。若 Pod 有 PDB 限制，缩容可能受阻。
- **扩容取值依据**：扩容目标节点数 = 当前节点数 + 预期新增节点数。须确保扩容后的总数 ≤ `DescribeClusterLevelAttribute` 返回的 `NodeCount`（集群 tier 节点数上限），且 ≤ 当前节点池的 `MaxNodesNum`。若当前 `MaxNodesNum` 不足，需先用 `ModifyClusterNodePool` 扩大范围。
- **缩容取值依据**：缩容目标节点数 ≥ `MinNodesNum`，且不小于运行中关键工作负载所需的最小节点数。建议逐次少量缩容（每次 1~2 个节点），每次缩容后等待 Pod 重新调度完成再继续。
- **何时应先调 ModifyClusterNodePool 再调 ModifyNodePoolDesiredCapacityAboutAsg**：当当前的 `MaxNodesNum` < 目标 `DesiredCapacity` 时，需先扩大伸缩范围（步骤 2）；当当前的 `MinNodesNum` > 目标 `DesiredCapacity` 时，需先缩小伸缩下限。

#### 扩容到 3 个节点

> 示例假设当前节点池有 2 个节点（`DesiredNodesNum=2`），集群 tier 为 L5（最多 5 节点），`MinNodesNum=1, MaxNodesNum=5`。扩容到 3 在 tier 和节点池范围内均安全。

`scale-up.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "DesiredCapacity": 3
}
```

```bash
tccli tke ModifyNodePoolDesiredCapacityAboutAsg --region <Region> \
    --cli-input-json file://scale-up.json
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

> 从 3 个节点缩容到 2 个，目标值 ≥ `MinNodesNum=1`，安全。

`scale-down.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "DesiredCapacity": 2
}
```

```bash
tccli tke ModifyNodePoolDesiredCapacityAboutAsg --region <Region> \
    --cli-input-json file://scale-down.json
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

### 步骤 2：配置弹性伸缩范围（通过 ModifyClusterNodePool）

设置 `MinNodesNum` 和 `MaxNodesNum` 范围后，CA（Cluster Autoscaler）在此范围内根据节点资源使用率自动伸缩。

> ⚠️ **副作用警告**：修改伸缩范围后 CA 可能立即触发扩缩容。若新 `MaxNodesNum` > 当前 `DesiredNodesNum` 且节点资源紧张，CA 会自动扩容；若新 `MinNodesNum` > 当前 `DesiredNodesNum`，CA 会立即扩容至下限。请确保在业务低峰期执行此操作。

#### 选择依据

- **MinNodesNum 取值**：应设为可接受的最小节点数，低于此值集群可能无法承载全部工作负载。建议 ≥ 1 以保证集群基本可用性。
- **MaxNodesNum 取值**：应不超集群 tier 的 `NodeCount` 上限（通过 `DescribeClusterLevelAttribute` 查询），同时考虑成本预算。L5 tier 上限为 5 节点，L50 tier 上限为 50 节点。
- **避免 CA 意外扩缩容**：修改范围前先确认当前 `DesiredNodesNum` 在新范围内，否则 CA 会立即纠正偏差。

`set-autoscale-range.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "MinNodesNum": 1,
  "MaxNodesNum": 5
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://set-autoscale-range.json
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

### 步骤 3：轮询确认扩缩容完成

扩缩容为异步操作，轮询确认节点数变化：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
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
            "DesiredNodesNum": 3,
            "MinNodesNum": 1,
            "MaxNodesNum": 5,
            "NodeCountSummary": {
                "ManuallyAdded": {"Total": 0},
                "AutoscalingAdded": {"Total": 3}
            },
            "AutoscalingGroupStatus": "normal"
        },
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤 4：确认节点状态

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --Filters '[{"Name":"node-pool-id","Values":["<NodePoolId>"]}]'
# expected: InstanceSet 数量与期望节点数一致，InstanceState 为 "running"
```

**预期输出**：

```json
{
    "Response": {
        "TotalCount": 3,
        "InstanceSet": [
            {
                "InstanceId": "ins-example-01",
                "InstanceRole": "WORKER",
                "InstanceState": "running",
                "FailedReason": "",
                "NodePoolId": "np-example"
            },
            {
                "InstanceId": "ins-example-02",
                "InstanceRole": "WORKER",
                "InstanceState": "running",
                "FailedReason": "",
                "NodePoolId": "np-example"
            },
            {
                "InstanceId": "ins-example-03",
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

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行（内网端点需 IOA/VPN/专线，公网端点需 CAM 策略允许 `tke:clusterExtranetEndpoint`）。若端点不可达，请通过控制台或节点内 `crictl` 间接验证。

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 Kubeconfig 内容

# 确认节点数
kubectl --kubeconfig <KubeconfigPath> get nodes
# expected: 节点数量与 DesiredNodesNum 一致，STATUS 为 Ready

# 确认新节点已调度 Pod
kubectl --kubeconfig <KubeconfigPath> get pods -A -o wide
# expected: Pod 分布在新节点上
```

## 清理

本页的修改操作变更了以下状态，需逐一回退：

| 操作步骤 | 变更内容 | 回退方法 |
|---------|---------|---------|
| 步骤 1（扩缩容） | 修改了 `DesiredCapacity` | 重新执行 `ModifyNodePoolDesiredCapacityAboutAsg` 传入原始 `DesiredCapacity` 值 |
| 步骤 2（伸缩范围） | 修改了 `MinNodesNum` / `MaxNodesNum` | 重新执行 `ModifyClusterNodePool` 传入原始范围值 |

若仅执行了步骤 1（扩缩容），只需回退 `DesiredCapacity`。若同时执行了步骤 2（伸缩范围），需先回退 `DesiredCapacity` 再回退 `MinNodesNum`/`MaxNodesNum`。

## 副作用声明

| 操作 | 影响范围 | 不可逆？ | 说明 |
|------|---------|:------:|------|
| 扩容（`DesiredCapacity` 增大） | 计费 + 节点生命周期 | 否（可缩容回退） | 创建新 CXM 实例，产生按量计费（CVM + 系统盘）。新节点约 2~5 分钟就绪。 |
| 缩容（`DesiredCapacity` 减小） | Pod 调度 | 否（可扩容回退） | 优先驱逐低负载 Pod，Pod 被重新调度到剩余节点。若有 PDB 限制可能阻塞缩容。缩容后节点实例被销毁（不可恢复）。 |
| 扩大伸缩范围（`MaxNodesNum` 增大） | CA 行为 | 否 | CA 可能立即触发自动扩容（若节点资源紧张），产生额外计费。 |
| 缩小伸缩范围（`MinNodesNum` 增大） | CA 行为 | 否 | 若当前 `DesiredNodesNum` < 新 `MinNodesNum`，CA 立即扩容至下限，产生额外计费。 |
| 缩小伸缩范围（`MaxNodesNum` 减小） | CA 行为 | 否 | 若当前 `DesiredNodesNum` > 新 `MaxNodesNum`，无法执行（API 拒绝）。需先缩容再修改范围。 |

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyNodePoolDesiredCapacityAboutAsg` 返回 `InvalidParameterValue.Size` | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 检查当前 `MinNodesNum` 和 `MaxNodesNum` | `DesiredCapacity` 不在 `[MinNodesNum, MaxNodesNum]` 范围内 | 先通过 `ModifyClusterNodePool` 扩大 `MaxNodesNum` 或缩小 `MinNodesNum` |
| 返回 `InvalidParameter.Param`（含 "cluster level limit"） | `tccli tke DescribeClusterLevelAttribute --region <Region> --ClusterID <ClusterId>` 查当前 tier 的 `NodeCount` 上限 | `DesiredCapacity` 超过集群 tier 的节点数上限 | 升级集群 tier（控制台 → 集群 → 基本信息 → 升级集群规格），或降低 `DesiredCapacity` 值 |
| 返回 `ResourceNotFound.NodePoolNotFound` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 检查 NodePoolId | NodePoolId 不存在或已删除 | 核实正确的 NodePoolId |
| 返回 `QuotaExceeded.NodeCount` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 统计已有节点数 | 集群节点总数超过配额（此为环境限制，非命令错误） | 申请提额或减少其他节点池节点数 |
| 返回 `FailedOperation.AsgNotEnabled` | `tccli tke DescribeClusterAsGroups --region <Region> --ClusterId <ClusterId>` 检查 | ASG 伸缩组未启用或不关联该节点池 | 确认节点池已启用自动伸缩，关联正确的伸缩组 |
| `ModifyClusterNodePool` 返回 `OperationDenied` | `tccli tke DescribeClusterNodePoolDetail` 查看 `LifeState` | 节点池 `LifeState` 非 `normal`（如 `updating`） | 等待节点池操作完成，`LifeState` 恢复 `normal` 后重试。保留 RequestId 用于追踪 |

### 伸缩成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 扩容后节点数不增长 | `tccli tke DescribeClusterNodePoolDetail` → `AutoscalingGroupStatus` | 伸缩组异常或 CXM 配额不足 | 检查伸缩组状态，确认 CXM 配额充足 |
| 扩容后新节点 `InstanceState` 为 `failed` | `tccli tke DescribeClusterInstances` → `FailedReason` | 子网 IP 耗尽、安全组配置错误或镜像拉取失败 | 更换子网或扩容子网 CIDR，检查安全组规则放行必要端口 |
| 缩容后 Pod 驱逐卡住 | `kubectl --kubeconfig <KubeconfigPath> get pods -A -o wide` 查看 Pending Pod | PDB 阻止驱逐或 Pod 有本地存储 | 调整 PDB 或手动迁移有状态 Pod |
| 弹性伸缩不触发 | `kubectl --kubeconfig <KubeconfigPath> describe configmap cluster-autoscaler-status -n kube-system` | CA 配置未生效或节点资源使用率未达到阈值 | 检查 CA 日志，手动扩容/缩容测试伸缩组连通性 |
| 缩容后节点仍出现在 `InstanceSet` | `tccli tke DescribeClusterInstances` 查看 `InstanceState` | 节点正在 `terminating` 状态，销毁未完成 | 等待 3-5 分钟，节点会自动消失 |

## 下一步

- [修改原生节点](../修改原生节点/tccli%20操作.md) -- page_id `103599`
- [声明式操作实践](https://cloud.tencent.com/document/product/457/78649) -- page_id `78649`
- [故障自愈规则](https://cloud.tencent.com/document/product/457/78209) -- page_id `78650`
- [新建原生节点](../新建原生节点/tccli%20操作.md) -- page_id `78198`

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点 → 弹性伸缩](https://console.cloud.tencent.com/tke2/cluster)
