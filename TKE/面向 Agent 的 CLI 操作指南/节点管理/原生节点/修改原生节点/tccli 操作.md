# 修改原生节点

> 对照官方：[修改原生节点](https://cloud.tencent.com/document/product/457/103599) · page_id `103599` · tccli ≥3.1.107 · API 2018-05-25

## 概述

通过 `ModifyClusterNodePool` 修改原生节点池的配置：名称、伸缩范围、Labels、Taints、Annotations 等。多数变更仅对**新增节点**生效，存量节点不受影响。支持**差分更新**：只传需变更的字段，未传字段保持不变。

> **API 参数白名单**：`ModifyClusterNodePool` 支持以下参数（通过 `--generate-cli-skeleton` 获取）：`ClusterId`、`NodePoolId`、`Name`、`MaxNodesNum`、`MinNodesNum`、`Labels`、`Taints`、`Annotations`、`EnableAutoscale`、`OsName`、`OsCustomizeType`、`GPUArgs`、`UserScript`、`IgnoreExistedNode`、`ExtraArgs`、`Tags`、`Unschedulable`、`DeletionProtection`、`DockerGraphPath`、`PreStartUserScript`。**不在白名单中的参数无法通过此 API 修改**。例如：期望节点数（`DesiredNodesNum`）需使用 `ModifyNodePoolDesiredCapacityAboutAsg` 修改，Management 仅在 `CreateClusterNodePool` 时设置。

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

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:ModifyClusterNodePool, tke:DescribeClusterNodePoolDetail
#    tke:DescribeClusterNodePools, tke:DescribeClusters
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表
```

预期输出：

```json
{
    "TotalCount": 2,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example01",
            "Name": "native-pool-1",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Native"
        },
        {
            "NodePoolId": "np-example02",
            "Name": "regular-pool-1",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Regular"
        }
    ],
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

> 从上述输出获取目标节点池的 `<NodePoolId>`：`$.NodePoolSet[?(@.NodePoolType=="Native")].NodePoolId`。

### 资源检查

```bash
# 4. 确认节点池存在并查看当前配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，返回节点池详情，LifeState 为 "normal"
```

预期输出：

```json
{
    "NodePool": {
        "NodePoolId": "np-example01",
        "Name": "native-pool-1",
        "ClusterInstanceId": "cls-example",
        "LifeState": "normal",
        "DesiredNodesNum": 2,
        "MinNodesNum": 1,
        "MaxNodesNum": 5,
        "Labels": [
            {"Name": "env", "Value": "dev"}
        ],
        "Taints": [],
        "Annotations": [],
        "AutoscalingGroupStatus": "normal"
    },
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

> **操作前快照**：请保存上述 `DescribeClusterNodePoolDetail` 输出，以便修改后回退。记录 `Name`、`MaxNodesNum`、`MinNodesNum`、`Labels`、`Taints`、`Annotations` 等字段的当前值。

## 控制台与 CLI 参数映射

以下为 `ModifyClusterNodePool` API 的主要参数。所有参数均可通过差分更新（只传需变更的字段）。执行前请使用 `tccli tke ModifyClusterNodePool --generate-cli-skeleton` 获取最新完整参数签名。

| 控制台参数 | CLI 参数 | 类型 | 必填 | 取值与约束 | 幂等 | 错误后果 |
|-----------|---------|------|:--:|------|:--:|------|
| 集群 ID | `ClusterId` | String | 是 | 集群 ID，格式 `cls-` 开头 | 是 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| 节点池 ID | `NodePoolId` | String | 是 | 节点池 ID，格式 `np-` 开头 | 是 | 节点池不存在 → `ResourceNotFound.NodePoolNotFound` |
| 节点池名称 | `Name` | String | 否 | 节点池新名称，2-64 字符 | 否（重名冲突） | 重名 → `InvalidParameterValue` |
| 最小节点数 | `MinNodesNum` | Int | 否 | 最小节点数，<= `DesiredNodesNum` | 是（相同值） | 大于期望数 → `InvalidParameter.Param` |
| 最大节点数 | `MaxNodesNum` | Int | 否 | 最大节点数，>= `DesiredNodesNum` | 是（相同值） | 小于期望数 → `InvalidParameter.Param` |
| Kubernetes 标签 | `Labels` | Object[] | 否 | 差分更新（合并同名 Key） | 是（相同值） | 格式错误 → `InvalidParameterValue.Label` |
| Kubernetes 污点 | `Taints` | Object[] | 否 | 差分更新（合并同名 Key） | 是（相同值） | 格式错误 → `InvalidParameterValue.Taint` |
| 声明式注解 | `Annotations` | Object[] | 否 | PUT 整体替换，仅对新增节点生效 | 否（整体替换） | 格式错误 → `InvalidParameterValue.Annotation` |
| 启用自动伸缩 | `EnableAutoscale` | Boolean | 否 | `true` 或 `false` | 是（相同值） | 默认值导致非预期行为 |

### 变更生效范围

| 变更项 | 生效范围 | 说明 |
|--------|---------|------|
| `Name` | 存量 + 新增 | 立即生效 |
| `MinNodesNum` / `MaxNodesNum` | 存量 + 新增 | 立即生效，影响弹性伸缩边界 |
| `EnableAutoscale` | 存量 + 新增 | 立即生效 |
| `Labels` | 存量 + 新增 | 差分更新（仅更新传入的 Key） |
| `Taints` | 存量 + 新增 | 差分更新（仅更新传入的 Key） |
| `Annotations` | **仅新增节点** | 存量节点注解不变 |
| `OsCustomizeType` / `GPUArgs` / `UserScript` | **仅新增节点** | 存量节点不受影响 |
| `ExtraArgs` | **仅新增节点** | 存量节点 Kubelet 参数不变 |

> **不在本 API 的常见参数**：
> - `DesiredNodesNum`（期望节点数）：需使用 `ModifyNodePoolDesiredCapacityAboutAsg` API，详见 [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md)
> - `Management`（DNS/Hosts/内核参数）：仅在 `CreateClusterNodePool` 时设置，无法通过 `ModifyClusterNodePool` 修改
> - `SecurityGroupIds`/`InstanceTypes`：不在 `ModifyClusterNodePool` API 参数中

## 操作步骤

### 步骤 1：修改节点池名称和伸缩范围

#### 选择依据

- **差分更新**：`ModifyClusterNodePool` 只需传入 `ClusterId`、`NodePoolId` 和要变更的字段，未传入的字段保持不变。无需每次都传全量配置。
- **名称变更**：修改 `Name` 用于更好地区分节点池，立即生效。
- **伸缩范围**：调整 `MinNodesNum` 和 `MaxNodesNum` 仅改变弹性伸缩边界，不直接触发节点创建或销毁。实际扩缩容由 `DesiredNodesNum` 驱动（见扩缩容页 `ModifyNodePoolDesiredCapacityAboutAsg`）。

`modify-nodepool-name-range.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Name": "native-pool-production",
  "MinNodesNum": 2,
  "MaxNodesNum": 10
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-name-range.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

### 步骤 2：修改 Labels 和 Taints

#### 选择依据

- **Labels**：为节点添加 Kubernetes 标签，用于 Pod 调度亲和性（nodeSelector/nodeAffinity）。差分更新意味着传入的 Key 会合并到已有 Labels 中。
- **Taints**：为节点添加污点，阻止不匹配的 Pod 调度到节点上。支持 `NoSchedule`、`PreferNoSchedule`、`NoExecute` 三种效果。
- **使用场景**：标签用于区分环境（dev/staging/prod）、GPU 类型、专用节点池等；污点用于实现节点独占（dedicated node）。

`modify-nodepool-labels-taints.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Labels": [
    {
      "Name": "env",
      "Value": "production"
    },
    {
      "Name": "tier",
      "Value": "backend"
    }
  ],
  "Taints": [
    {
      "Key": "dedicated",
      "Value": "backend",
      "Effect": "NoSchedule"
    }
  ]
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-labels-taints.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

### 步骤 3：修改 Annotations 声明式注解

#### 选择依据

- **Annotations**：`ModifyClusterNodePool` 的 `Annotations` 参数是 **PUT（整体替换）**而非 PATCH（追加）。每次调用需传入完整的 `Annotations` 列表。
- **仅对新增节点生效**：Annotations 修改后，已有节点的注解不变。只有新扩容的节点会携带新的 Annotations。
- **建议流程**：先 `DescribeClusterNodePoolDetail` 获取当前 Annotations → 合并新旧 → 一次性完整写入，避免覆盖已有注解。

`modify-nodepool-annotations.json`：

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
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

### 步骤 4：确认变更生效

修改为异步操作，需轮询确认：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 返回节点池详情，修改的字段已更新
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example01",
        "Name": "native-pool-production",
        "ClusterInstanceId": "cls-example",
        "LifeState": "normal",
        "DesiredNodesNum": 2,
        "MinNodesNum": 2,
        "MaxNodesNum": 10,
        "Labels": [
            {"Name": "env", "Value": "production"},
            {"Name": "tier", "Value": "backend"}
        ],
        "Taints": [
            {"Key": "dedicated", "Value": "backend", "Effect": "NoSchedule"}
        ],
        "Annotations": [
            {"Name": "node.tke.cloud.tencent.com/in-place-upgrade", "Value": "true"},
            {"Name": "tke.cloud.tencent.com/example", "Value": "updated-value"}
        ],
        "AutoscalingGroupStatus": "normal"
    },
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

## 验证

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 名称 | `DescribeClusterNodePoolDetail` -> `Name` | 与修改值一致 |
| 伸缩范围 | `DescribeClusterNodePoolDetail` -> `MinNodesNum`/`MaxNodesNum` | 与修改值一致 |
| Labels | `DescribeClusterNodePoolDetail` -> `Labels` | 包含更新后的 Key-Value |
| Taints | `DescribeClusterNodePoolDetail` -> `Taints` | 包含更新后的 Key-Value-Effect |
| Annotations | `DescribeClusterNodePoolDetail` -> `Annotations` | 与传入列表一致（PUT 整体替换） |
| 节点池状态 | `DescribeClusterNodePoolDetail` -> `LifeState` | `normal` |
| 伸缩组状态 | `DescribeClusterNodePoolDetail` -> `AutoscalingGroupStatus` | `normal` |

### 确认节点已注册

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> | jq -r '.Kubeconfig' | base64 -d > <KubeconfigPath>
# expected: exit 0，返回 Kubeconfig 内容

# 确认节点数
kubectl --kubeconfig <KubeconfigPath> get nodes
# expected: 节点数量与 DesiredNodesNum 一致，STATUS 为 Ready
```

预期输出：

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci0gY2x1c3Rlcjo...",
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

## 清理

> **⚠️ 副作用声明**：
> - 修改 `MinNodesNum`/`MaxNodesNum` 仅改变伸缩边界，不立即触发扩缩容。但后续弹性伸缩行为会受新边界约束。
> - `Annotations` 修改为 PUT（整体替换），可能覆盖已有注解。操作前请先 `DescribeClusterNodePoolDetail` 保存快照。
> - Labels/Taints 修改立即对所有存量节点生效，可能影响正在运行的 Pod 调度。

本页为修改操作，需手动回退：

```bash
# 1. 获取操作前保存的快照值（操作前已通过 DescribeClusterNodePoolDetail 记录）
# 2. 传入修改前的值恢复原始配置
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-restore.json
# expected: exit 0，返回 RequestId
```

`modify-nodepool-restore.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Name": "native-pool-1",
  "MinNodesNum": 1,
  "MaxNodesNum": 5,
  "Labels": [
    {"Name": "env", "Value": "dev"}
  ],
  "Taints": [],
  "Annotations": []
}
```

```bash
# 验证回退生效
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 各字段恢复为修改前的值
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `ResourceNotFound.NodePoolNotFound` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 检查 NodePoolId | NodePoolId 不存在或拼写错误 | 核实正确的 `<NodePoolId>` |
| 返回 `InvalidParameter.Param: nothing is updated` | 检查请求 JSON 中变更的字段 | API 无视不认识的字段，传入的字段值与原值相同（无实际变更） | 确认至少有一个字段的值与原值不同 |
| 返回 `OperationDenied` | 检查节点池 `LifeState` | 节点池当前处于 updating/creating/deleting 状态，不接受新的修改请求 | 等待节点池 `LifeState` 恢复为 `normal` 后重试 |
| 修改 `Annotations` 后旧注解丢失 | `tccli tke DescribeClusterNodePoolDetail` 查看 Annotations | `Annotations` 参数是 PUT（整体替换），未传入的 Key 被清除 | 操作前先 `DescribeClusterNodePoolDetail` 获取完整 Annotations → 合并新旧 → 一次性传入完整数组 |
| `kubectl get nodes` 显示节点状态 `NotReady` | `kubectl --kubeconfig <KubeconfigPath> describe node <NodeName>` | Labels/Taints 变更可能触发了节点的短暂状态波动（通常 30 秒内恢复） | 等待 1-2 分钟，节点应自动恢复；如未恢复，登录节点排查 kubelet 日志 |

### 修改成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Labels/Taints 在控制台已更新但 `kubectl get nodes` 不显示 | `kubectl --kubeconfig <KubeconfigPath> describe node <NodeName>` | 节点控制器同步延迟（通常 1-2 分钟） | 等待后重查 |
| 修改 `Annotations` 后扩容的新节点未携带注解 | `tccli tke DescribeClusterNodePoolDetail` -> `Annotations` | 确认 `Annotations` 确实已写入节点池配置（PUT 可能意外清除了注解） | 重新执行 Annotations 修改，确保传入完整列表 |
| 修改 `MaxNodesNum` 后弹性伸缩不生效 | `tccli tke DescribeClusterNodePoolDetail` -> `AutoscalingGroupStatus` | `EnableAutoscale` 可能为 `false` | 确认 `EnableAutoscale` 已设为 `true` |

## 下一步

- [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md) -- page_id `78648`
- [声明式操作实践](https://cloud.tencent.com/document/product/457/78649) -- page_id `78649`
- [删除原生节点](../删除原生节点/tccli%20操作.md) -- page_id `78199`
- [原生节点开启公网访问](https://cloud.tencent.com/document/product/457/82334) -- page_id `82334`

## 控制台替代

[容器服务控制台 -> 集群 -> 节点管理 -> 原生节点 -> 详情 -> 编辑](https://console.cloud.tencent.com/tke2/cluster)
