# 声明式操作实践

> 对照官方：[声明式操作实践](https://cloud.tencent.com/document/product/457/78649) · page_id `78649` · tccli ≥3.1.107 · API 2022-05-01

## 概述

原生节点池支持通过 **Annotations 声明式管理**节点配置，与 Kubernetes 声明式范式一致：你描述期望的终态，TKE 控制面将节点调整到目标状态。这与控制台手动操作形成对应——控制台是**命令式**（逐步点击完成单次操作），Annotations 是**声明式**（设置后，控制面持续驱动节点维持该状态）。

| 维度 | 控制台（命令式） | tccli + Annotations（声明式） |
|------|:---:|:---:|
| 操作方式 | 逐步点击表单 | 调用 `ModifyClusterNodePool` 设置 Annotations 键值 |
| 状态持久性 | 一次性操作 | 声明存在于节点池，持续生效 |
| 可复现性 | 手动重复 | JSON/命令可版本化，可批量执行 |
| 适用场景 | 临时调整个别节点 | 标准化配置、批量管理、IaC |

本页演示通过 `tccli tke ModifyClusterNodePool` 的 `Annotations` 参数声明式管理原生节点池配置。你也可以通过 kubectl 管理节点层面的 Annotations（需集群端点可达）。

## 前置条件

<!-- sync:relative-link -->
- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterNodePools
#    tke:DescribeClusterNodePoolDetail, tke:ModifyClusterNodePool
# 验证：执行 DescribeNodePools 确认权限
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表（可为空）
```

**预期输出**：

```json
{
    "NodePoolSet": [],
    "TotalCount": 0,
    "RequestId": "..."
}
```

### 资源检查

```bash
# 4. 确认集群状态为 Running 并获取关联 VPC
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus: "Running", VpcId 非空

# 5. 确认 VPC 和子网状态正常
tccli vpc DescribeVpcs --region <Region> \
    --VpcIds '["<VpcId>"]'
# expected: exit 0, VpcSet 含目标 VPC 且状态为可用
# <VpcId> 从上一步 DescribeClusters 输出的 VpcId 字段获取

# 6. 确认节点池配额充足
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: NodePoolSet 含目标节点池，LifeState 为 "normal"；检查 TotalCount 是否达到配额上限（默认每个集群最多 20 个节点池）

# 7. 确认 kubectl 连通性（如需数据面操作）
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> --IsExtranet false | jq -r '.Kubeconfig' > ~/.kube/config
kubectl cluster-info
# expected: Kubernetes control plane is running
# 如不可达，参见 [排障](#排障) 中的"kubectl 不可达"条目；tccli 命令不受影响
```

**预期输出**：

`DescribeClusters`（第 4 条）:

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "VpcId": "vpc-example"
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

`DescribeVpcs`（第 5 条）:

```json
{
    "VpcSet": [
        {
            "VpcId": "vpc-example",
            "VpcName": "<VpcName>",
            "CidrBlock": "10.0.0.0/12"
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

`DescribeClusterNodePools`（第 6 条）:

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "<NodePoolName>",
            "LifeState": "normal",
            "Labels": [
                {"Name": "pool-name", "Value": "example-pool"}
            ],
            "Annotations": [
                {"Name": "node.tke.cloud.tencent.com/in-place-upgrade", "Value": "true"}
            ],
            "NodeCountSummary": {
                "AutoscalingAdded": {"Normal": 1, "Total": 1}
            }
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

`DescribeClusterKubeconfig`（第 7 条）:

```yaml
# Kubeconfig 文件写入 ~/.kube/config 后，kubectl cluster-info 正常输出:
# Kubernetes control plane is running at https://172.x.x.x
# CoreDNS is running at https://172.x.x.x/api/v1/...
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 声明式配置节点池 Annotations | `ModifyClusterNodePool` | 是 |
| 查看节点池详情（含 Annotations） | `DescribeClusterNodePoolDetail` | 是 |
| 查看集群信息 | `DescribeClusters` | 是 |

## 关键字段说明

`ModifyClusterNodePool` 的 `Annotations` 是声明式操作的核心参数。以下为本次用到的关键字段说明。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-` 开头 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `NodePoolId` | String | 是 | 节点池 ID，格式 `np-` 开头。`tccli tke DescribeClusterNodePools` 获取 | 节点池不存在 → `ResourceNotFound.NodePoolNotFound` |
| `Annotations` | Object[] | 否 | 声明式注解数组，每项 `{"Name": "key", "Value": "val"}`。**重要：此参数是全量替换操作，非追加操作**——设 `[{"Name":"A","Value":"1"}]` 会**覆盖**节点池上已有的全部 Annotations | 覆盖已有 Annotations → 可能丢失其他 session 设置的注解；格式错误 → `InvalidParameterValue.Annotation` |
| `Labels` | Object[] | 否 | Kubernetes 标签，格式同 Annotations | 格式错误 → `InvalidParameterValue.Label` |
| `DeletionProtection` | Boolean | 否 | 节点池删除保护，默认 `false`。`true` 时需先关闭才能删节点池 | 忘关保护 → `OperationDenied.DeletionProtectionEnabled` |

### 常用原生节点 Annotations

以下为原生节点池常用的声明式 Annotations 键：

| Annotation 键 | 值示例 | 效果 |
|--------------|--------|------|
| `node.tke.cloud.tencent.com/in-place-upgrade` | `"true"` | 启用 Pod 原地升降配 |
| `node.tke.cloud.tencent.com/qos-enabled` | `"true"` | 启用内存压缩（QoS） |
| `tke.cloud.tencent.com/enable-healthcheck-policy` | `"true"` | 启用故障自愈规则 |

## 操作步骤

### 步骤 1：查看节点池列表

在执行声明式操作前，先获取集群中所有节点池，确认目标节点池 ID 和当前状态。

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0, NodePoolSet 包含节点池列表，每个池含 NodePoolId、LifeState、Annotations 等
```

**预期输出**：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "<NodePoolName>",
            "LifeState": "normal",
            "Annotations": [
                {
                    "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
                    "Value": "true"
                }
            ],
            "NodeCountSummary": {
                "AutoscalingAdded": {
                    "Normal": 1,
                    "Total": 1
                }
            },
            "DesiredNodesNum": 2,
            "MaxNodesNum": 3,
            "MinNodesNum": 0,
            "DeletionProtection": false
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

| 字段 | 说明 |
|------|------|
| `NodePoolId` | 节点池唯一 ID，后续操作需使用 |
| `LifeState` | 节点池状态。`normal` = 正常可操作；非 `normal` 状态可能拒绝写操作 |
| `Annotations` | 当前已设置的声明式注解列表 |

### 步骤 2：声明式设置节点池 Annotations

使用 `ModifyClusterNodePool` 的 `Annotations` 参数声明式配置节点池行为。修改 Annotations 后，TKE 控制面会根据键值驱动对应功能（如原地升级、QoS、故障自愈）。

#### 选择依据

- **目标节点池**：选择 `LifeState` 为 `normal` 的节点池。非 normal 状态（如 `updating`、`deleting`）的节点池执行 `ModifyClusterNodePool` 可能导致操作冲突。
- **声明式范式**：与 Kubernetes 风格一致，你描述期望的 Annotations 键值，TKE 控制面持续将节点调整到目标状态。这与控制台手动操作的单次行为不同——控制台是命令式，Annotations 是声明式。
- **关键 Annotations**：
  - `node.tke.cloud.tencent.com/in-place-upgrade=true`：启用 Pod 原地升降配，无需重建 Pod
  - `node.tke.cloud.tencent.com/qos-enabled=true`：启用内存压缩（QoS），提升资源利用率
  - `tke.cloud.tencent.com/enable-healthcheck-policy=true`：启用故障自愈规则，自动恢复异常节点
- **Annotations 是全量替换**：设置 `Annotations: [{"Name":"my-key","Value":"my-val"}]` 会覆盖已有的全部 Annotations——而非追加。必须先 `DescribeClusterNodePoolDetail` 获取当前 Annotations，合并后再提交。

#### 最小配置（仅添加一个测试注解）

`annotations-minimal.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "Annotations": [
        {
            "Name": "tke.cloud.tencent.com/test-declarative",
            "Value": "true"
        }
    ]
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://annotations-minimal.json
# expected: exit 0, 返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "..."
    }
}
```

#### 增强配置（合并已有 Annotations + 新增）

先获取当前 Annotations，合并后一次提交（避免覆盖已存在的注解）。`annotations-enhanced.json` 在保留已有 `in-place-upgrade` 注解的基础上新增 `qos-enabled` 和 `healthcheck-policy`：

```bash
# 先提取当前 Annotations
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> --NodePoolId <NodePoolId> | jq '.NodePool.Annotations'
# expected: 返回当前注解列表
```

**预期输出**：

```json
[
    {
        "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
        "Value": "true"
    }
]
```

`annotations-enhanced.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "Annotations": [
        {
            "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
            "Value": "true"
        },
        {
            "Name": "node.tke.cloud.tencent.com/qos-enabled",
            "Value": "true"
        },
        {
            "Name": "tke.cloud.tencent.com/enable-healthcheck-policy",
            "Value": "true"
        }
    ]
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://annotations-enhanced.json
# expected: exit 0, 返回 RequestId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-` 开头 | `tccli tke DescribeClusters --region <Region>` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-` 开头，LifeState 须为 `normal` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看已配置 region |

### 步骤 3：从集群维度确认集群信息

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0, ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterVersion": "1.32.2",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "VpcId": "vpc-example",
            "ClusterCIDR": "10.0.0.0/16",
            "ServiceCIDR": "10.1.0.0/20",
            "DeletionProtection": false
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

## 验证

### 控制面（tccli）

确认 Annotations 声明的配置已在节点池上生效：

```bash
# 验证完整详情
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> --NodePoolId <NodePoolId>
# expected: exit 0, Annotations 字段包含设置的键值
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "<NodePoolName>",
        "LifeState": "normal",
        "Annotations": [
            {
                "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
                "Value": "true"
            },
            {
                "Name": "node.tke.cloud.tencent.com/qos-enabled",
                "Value": "true"
            },
            {
                "Name": "tke.cloud.tencent.com/enable-healthcheck-policy",
                "Value": "true"
            }
        ],
        "Labels": [
            {
                "Name": "pool-name",
                "Value": "<PoolLabelValue>"
            }
        ],
        "DesiredNodesNum": 2,
        "MaxNodesNum": 3,
        "MinNodesNum": 0,
        "NodeCountSummary": {
            "AutoscalingAdded": {
                "Normal": 1,
                "Total": 1
            }
        },
        "DeletionProtection": false
    },
    "RequestId": "..."
}
```

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| 节点池状态 | `DescribeClusterNodePoolDetail` → `LifeState` | `"normal"` |
| Annotations 键存在 | 同上，检查 `Annotations[].Name` | 包含 `in-place-upgrade`、`qos-enabled`、`healthcheck-policy`（如以增强配置设置） |
| Annotations 值正确 | 同上，检查 `Annotations[].Value` | 各键对应的值为 `"true"` |
| 节点数正常 | 同上，检查 `NodeCountSummary` | 当前应有 ≥ 1 个节点运行（如节点池含节点） |

> 只有所有维度确认无误后才继续。Annotions 修改为同步操作（立即生效），无需轮询等待。

### 数据面（kubectl）

> **kubectl 可达性说明**：以下 kubectl 命令需在集群端点可达的环境下执行（通过 IOA/VPN/同 VPC CVM 等方式）。如不可达，不影响 tccli 命令的正常执行——`ModifyClusterNodePool` 通过腾讯云 API 网关直接操作控制面，不依赖 kubectl 连通性。

```bash
# 查看节点上的 annotations
kubectl get nodes -o custom-columns=NAME:.metadata.name,ANNOTATIONS:.metadata.annotations
# expected: 节点列表，含对应的 annotations
```

```bash
# 确认节点池节点均处于 Ready 状态
# <PoolLabelValue> 获取方式：
#   tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId> \
#     | jq -r '.NodePool.Labels[]? | select(.Name=="pool-name") | .Value'
# 如节点池未设置 pool-name 标签，可按其他标签筛选（如 env、team），或直接 kubectl get nodes 列出全部节点
kubectl get nodes -l pool-name=<PoolLabelValue>
# expected: 所有目标节点 STATUS 为 Ready
```

## 清理

> **警告**：Annotations 清理使用空数组 `[]` 会**移除节点池上的所有 Annotations**——包括其他 session 或途径设置的注解。生产环境务必先 `DescribeClusterNodePoolDetail` 获取当前 Annotations，仅移除自己设置的键。以下为完整替换演示，实际场景应只替换目标 Annotations 键。

### 控制面（tccli）

#### 1. 清理前状态检查

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> --NodePoolId <NodePoolId>
# 确认当前 Annotations 列表，记录需要保留的键，确认待移除的键
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "<NodePoolName>",
        "LifeState": "normal",
        "Annotations": [
            {"Name": "node.tke.cloud.tencent.com/in-place-upgrade", "Value": "true"},
            {"Name": "tke.cloud.tencent.com/test-declarative", "Value": "true"}
        ],
        "Labels": [
            {"Name": "pool-name", "Value": "example-pool"}
        ]
    },
    "RequestId": "..."
}
```

#### 2. 精确移除指定 Annotations（推荐）

仅移除测试注解，保留已存在的其他 Annotations：

```bash
# 提取当前 Annotations，用 jq 过滤掉待删除的键，写入清理 JSON
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> --NodePoolId <NodePoolId> \
    | jq '[.NodePool.Annotations[] | select(.Name != "tke.cloud.tencent.com/test-declarative")]' \
    > annotations-cleanup.json
```

`annotations-cleanup.json` 示例内容（过滤掉测试注解后）：

```json
[
    {
        "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
        "Value": "true"
    }
]
```

```bash
# 将过滤后的 Annotations 与 ClusterId、NodePoolId 合并为完整请求 JSON
jq -n --arg cluster "<ClusterId>" --arg pool "<NodePoolId>" \
  --argjson anns "$(cat annotations-cleanup.json)" \
  '{ClusterId: $cluster, NodePoolId: $pool, Annotations: $anns}' > cleanup-input.json

tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://cleanup-input.json
# expected: exit 0, 返回 RequestId
```

#### 3. 验证清理结果

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> --NodePoolId <NodePoolId> \
    | jq '.NodePool.Annotations'
# expected: 不包含被移除的测试注解键
```

**预期输出**：

```json
[
    {
        "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
        "Value": "true"
    }
]
```

### 数据面（kubectl）

> 以下 kubectl 命令需在集群端点可达的环境下执行。`<NodeName>` 可通过 `kubectl get nodes -o jsonpath='{.items[0].metadata.name}'` 或根据节点池标签筛选获取。

```bash
# 获取节点名（如节点池有 pool-name 标签）
kubectl get nodes -l pool-name=<PoolLabelValue> -o jsonpath='{.items[0].metadata.name}'
# expected: 返回节点名，如 "eklet-subnet-xxx"

# 移除节点上的特定 annotation
kubectl annotate node <NodeName> tke.cloud.tencent.com/test-declarative-
# expected: node/<NodeName> annotated（键名后加 - 表示删除该 annotation）

# 确认已移除
kubectl get node <NodeName> -o json | jq '.metadata.annotations["tke.cloud.tencent.com/test-declarative"]'
# expected: null
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `MissingParameter`，提示 `NodePoolId is required` | 检查命令中是否包含 `--NodePoolId` 参数 | 未填写 NodePoolId 参数，这是必填字段 | 先 `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 获取 NodePoolId |
| `ModifyClusterNodePool` 返回 `InvalidParameter.ClusterNotFound` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | ClusterId 格式错误或不属于当前账号/地域 | 确认 ClusterId 格式为 `cls-` 开头，且属于当前地域 |
| `ModifyClusterNodePool` 返回 `InvalidParameterValue.Annotation` | 检查 JSON 中 `Annotations` 数组格式 | Annotations 每项结构不符合 `{"Name":"key","Value":"val"}` 格式 | 确保每项含 `Name` 和 `Value` 两个字段，类型为 String |
| Annotations 设置后原有注解消失 | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId> \| jq '.NodePool.Annotations'` 对比设置前后 | `Annotations` 参数是**全量替换操作**，非追加。设 `[{"Name":"A","Value":"1"}]` 会覆盖已有的全部 Annotations | 先 `DescribeClusterNodePoolDetail` 获取当前 Annotations，合并后再 `ModifyClusterNodePool`（参考 [步骤 2：增强配置](#增强配置合并已有-annotations--新增)） |
| `ModifyClusterNodePool` 返回 `ResourceNotFound.NodePoolNotFound` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 确认 NodePoolId | NodePoolId 不存在或不属于该集群 | 从 `DescribeClusterNodePools` 输出中获取正确的 NodePoolId |
| `ModifyClusterNodePool` 返回参数名相关错误 | 检查 JSON 中参数名大小写：`grep -E '"(ClusterId|NodePoolId|Annotations)"' input.json` | 参数名大小写与 API 不匹配（如 `annotations` 而非 `Annotations`） | 参数名必须为：`Annotations`（首字母大写）、`ClusterId`、`NodePoolId`，JSON key 区分大小写 |
| `DescribeClusterNodePoolDetail` 返回的 Annotations 未更新 | 检查 `ModifyClusterNodePool` 的 RequestId，用其查询操作审计 | 可能存在并发修改覆盖，或 API 调用返回成功但后端处理中 | 等待数秒后重新 `DescribeClusterNodePoolDetail`，如仍未生效则记录 RequestId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/node-pool) 查看节点池详情 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |

### 操作成功但效果异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Annotations 设置成功但节点行为未变化 | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 检查 `LifeState` | Annotations 生效依赖控制面组件；某些功能（如故障自愈）可能需要额外的组件启用 | <!-- sync:relative-link -->检查对应功能的[原生节点功能支持说明](../原生节点功能支持说明/tccli%20操作.md) 确认前提条件是否满足 |
| kubectl 不可达（公网端点被 CAM 拒绝） | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 查看端点状态 | 公网端点被组织级 CAM 策略 `strategyId:240463971` 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝。自建安全组无法绕过。（此为环境限制，非命令错误） | 使用内网端点（`--IsExtranet false`）作为替代，需通过 IOA/VPN/专线 或同 VPC CVM 才能 kubectl 可达。**tccli 命令不受此限制**——`ModifyClusterNodePool` 通过腾讯云 API 网关直接操作控制面 |
| kubectl 不可达（内网 DNS 不可解析） | `kubectl cluster-info` 返回 `dial tcp: lookup cls-xxx.ccs.tencent-cloud.com: no such host` | 内网端点域名 `cls-xxx.ccs.tencent-cloud.com` 仅可在腾讯云内网解析。本地开发机不在 VPC 内，无法通过内网域名访问（此为环境限制，非命令错误） | 通过以下任一方式建立连接：1) 连接 IOA/VPN 获取内网访问能力，2) 在同 VPC 的 CVM 上执行 kubectl 命令，3) 使用专线连接。tccli 命令不依赖 kubectl 可达性，可直接通过 API 网关操作 |

## 下一步

<!-- sync:relative-link -->
- [原生节点概述](../原生节点概述/tccli%20操作.md) — 了解原生节点的概念和能力边界
- [Pod 原地升降配](../Pod%20原地升降配/tccli%20操作.md) — 使用 `in-place-upgrade` 注解后的实操步骤
- [修改原生节点](../修改原生节点/tccli%20操作.md) — 通过 CLI 修改原生节点池的完整配置（含 Annotations、Labels、Taints 等）
- [弹性健康度](../弹性健康度/tccli%20操作.md) — 使用 `enable-healthcheck-policy` 注解后的实操步骤
- [原生节点功能支持说明](../原生节点功能支持说明/tccli%20操作.md) — 各功能对原生节点类型的支持矩阵

## 控制台替代

[TKE 控制台 → 节点池](https://console.cloud.tencent.com/tke2/node-pool)：节点池详情中查看和编辑 Annotations 配置。
