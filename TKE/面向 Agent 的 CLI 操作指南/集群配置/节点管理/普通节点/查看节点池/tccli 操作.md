# 查看节点池

> 对照官方：[查看节点池](https://cloud.tencent.com/document/product/457/43736) · page_id `43736`

## 概述

查看集群下所有节点池的列表摘要（名称、状态、节点数量），以及单个节点池的详细配置（启动配置、伸缩组、Label/Taint、运行时、系统镜像等）。所有操作均为只读，走 TKE 控制面 API，不依赖 kubectl 或集群数据面连通性。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail, tke:DescribeClusterInstances
# 验证：执行 DescribeClusters 确认 TKE 只读权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

### 资源检查

```bash
# 4. 确认目标集群存在且可访问
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0，返回目标集群，ClusterStatus 为 Running 或正常状态

# 5. 确认集群下节点池可查询（首次查看，可返回空列表）
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>
# expected: exit 0，返回 NodePoolSet（可为空数组，TotalCount 可为 0）
```

> 空节点池列表是正常状态，不是错误。集群刚创建或未配置节点池时，`DescribeClusterNodePools` 返回 `TotalCount: 0`、`NodePoolSet: []`，exit 0。如需创建节点池，参见 [创建节点池](../创建节点池/tccli%20操作.md)。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli API | 幂等 |
|------------|-----------|:--:|
| 节点池列表页 | `DescribeClusterNodePools` | 是 |
| 节点池详情页 | `DescribeClusterNodePoolDetail` | 是 |
| 按状态/名称筛选池 | `DescribeClusterNodePools` + `--filter`（客户端 JMESPath） | 是 |
| 查看池内节点列表 | `DescribeClusterInstances` + `--filter`（按 `NodePoolId`） | 是 |

> `--filter` 是 tccli 客户端侧 JMESPath 过滤，不是 API 参数。每次仍拉取全量数据后在本地过滤。API 级筛选用 `--Filters` 数组（本页示例均为客户端侧 `--filter`）。

## 操作步骤

---

### 查看节点池列表

获取集群下全部节点池的摘要信息。适合批量查看、确认池是否存在、获取 `NodePoolId`。

```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>
# expected: exit 0，返回 NodePoolSet 数组与 TotalCount
```

```json
{
  "NodePoolSet": [
    {
      "NodePoolId": "np-example",
      "Name": "example-node-pool",
      "ClusterInstanceId": "cls-example",
      "LifeState": "normal",
      "LaunchConfigurationId": "asc-example",
      "AutoscalingGroupId": "asg-example",
      "Labels": [],
      "Taints": [],
      "Annotations": [],
      "NodeCountSummary": {
        "ManuallyAdded": {
          "Joining": 0,
          "Initializing": 0,
          "Normal": 0,
          "Total": 0
        },
        "AutoscalingAdded": {
          "Joining": 0,
          "Initializing": 1,
          "Normal": 0,
          "Total": 1
        }
      },
      "AutoscalingGroupStatus": "disabled",
      "MaxNodesNum": 1,
      "MinNodesNum": 1,
      "DesiredNodesNum": 1,
      "RuntimeConfig": {
        "RuntimeType": "containerd",
        "RuntimeVersion": "1.6.9"
      },
      "NodePoolOs": "ubuntu22.04x86_64",
      "OsCustomizeType": "GENERAL",
      "DesiredPodNum": 64
    }
  ],
  "TotalCount": 1,
  "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

#### 空节点池列表（无节点池时的正常输出）

集群无节点池时，`DescribeClusterNodePools` 正常返回空列表，exit 0。这不是错误，无需排障。

```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>
# expected: exit 0，TotalCount: 0，NodePoolSet: []
```

```json
{
  "NodePoolSet": [],
  "TotalCount": 0,
  "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

---

### 按 LifeState 筛选节点池

节点池数量较多时，用 `--filter` 在客户端按 `LifeState` 过滤。`--filter` 后返回的是过滤后的数组，不再是外层 `NodePoolSet` 包装。

```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region> \
    --filter "NodePoolSet[?LifeState=='normal']"
# expected: exit 0，返回 LifeState 为 normal 的节点池数组
```

```json
[
  {
    "NodePoolId": "np-example",
    "Name": "example-node-pool",
    "ClusterInstanceId": "cls-example",
    "LifeState": "normal",
    "LaunchConfigurationId": "asc-example",
    "AutoscalingGroupId": "asg-example",
    "Labels": [],
    "Taints": [],
    "Annotations": [],
    "NodeCountSummary": {
      "ManuallyAdded": {
        "Joining": 0,
        "Initializing": 0,
        "Normal": 0,
        "Total": 0
      },
      "AutoscalingAdded": {
        "Joining": 0,
        "Initializing": 1,
        "Normal": 0,
        "Total": 1
      }
    },
    "AutoscalingGroupStatus": "disabled",
    "MaxNodesNum": 1,
    "MinNodesNum": 1,
    "DesiredNodesNum": 1
  }
]
```

---

### 按名称关键词筛选

用 JMESPath 的 `contains()` 按名称关键词过滤，并用投影 `.{}` 只返回需要的字段，减少输出噪声：

```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region> \
    --filter "NodePoolSet[?contains(Name,'example')].{NodePoolId:NodePoolId,Name:Name,LifeState:LifeState}"
# expected: exit 0，返回名称含 example 的节点池，仅含三个字段
```

```json
[
  {
    "NodePoolId": "np-example",
    "Name": "example-node-pool",
    "LifeState": "normal"
  }
]
```

---

### 查看单个节点池详情

传入 `NodePoolId` 获取完整配置，包括启动配置、伸缩组参数、Label/Taint、运行时、系统镜像、删除保护等。`NodePoolId` 需先从 `DescribeClusterNodePools` 获取。

```bash
tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region <Region>
# expected: exit 0，返回 NodePool 对象，含完整配置
```

```json
{
  "NodePool": {
    "NodePoolId": "np-example",
    "Name": "example-node-pool",
    "ClusterInstanceId": "cls-example",
    "LifeState": "normal",
    "LaunchConfigurationId": "asc-example",
    "AutoscalingGroupId": "asg-example",
    "Labels": [],
    "Taints": [],
    "Annotations": [],
    "NodeCountSummary": {
      "ManuallyAdded": {
        "Joining": 0,
        "Initializing": 0,
        "Normal": 0,
        "Total": 0
      },
      "AutoscalingAdded": {
        "Joining": 0,
        "Initializing": 1,
        "Normal": 0,
        "Total": 1
      }
    },
    "AutoscalingGroupStatus": "disabled",
    "MaxNodesNum": 1,
    "MinNodesNum": 1,
    "DesiredNodesNum": 1,
    "RuntimeConfig": {
      "RuntimeType": "containerd",
      "RuntimeVersion": "1.6.9"
    },
    "NodePoolOs": "ubuntu22.04x86_64",
    "OsCustomizeType": "GENERAL",
    "ImageId": "",
    "DesiredPodNum": 64,
    "DeletionProtection": false,
    "Unschedulable": 0,
    "DataDisks": null,
    "Tags": [
      {
        "Key": "env",
        "Value": "production"
      }
    ],
    "GPUArgs": {
      "CUDA": { "Name": "", "Version": "" },
      "CUDNN": { "Name": "", "Version": "", "DevName": "", "DocName": "" },
      "CustomDriver": { "Address": "" },
      "Driver": { "Name": "", "Version": "" },
      "MIGEnable": false
    }
  },
  "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

---

### 查看节点池关联节点

通过 `DescribeClusterInstances` 配合 `--filter` 按 `NodePoolId` 筛选池内 CVM 实例，查看各节点的实时状态：

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region <Region> \
    --filter "InstanceSet[?NodePoolId=='<NodePoolId>'].{InstanceId:InstanceId,InstanceState:InstanceState,InstanceRole:InstanceRole}"
# expected: exit 0，返回池内节点列表
```

```json
[
  {
    "InstanceId": "ins-example",
    "InstanceState": "initializing",
    "InstanceRole": "WORKER"
  }
]
```

> `NodeCountSummary` 反映伸缩组的实时状态。当 `AutoscalingGroupStatus` 为 `disabled` 时，节点数为伸缩组配置数。若 `AutoscalingAdded.Initializing` 持续不归零，用 `DescribeClusterInstances` 查 CVM 的 `InstanceState` 确认节点是否创建成功。

#### LifeState 与 NodeCountSummary 字段含义

| 字段 | 取值 | 含义 |
|------|------|------|
| `LifeState` | `normal` | 节点池正常 |
| `LifeState` | `creating` | 节点池创建中 |
| `LifeState` | `deleting` | 节点池删除中 |
| `LifeState` | `updating` | 节点池更新中 |
| `LifeState` | `abnormal` | 节点池异常 |
| `NodeCountSummary.ManuallyAdded` | `Joining`/`Initializing`/`Normal`/`Total` | 手动加入的节点统计 |
| `NodeCountSummary.AutoscalingAdded` | `Joining`/`Initializing`/`Normal`/`Total` | 伸缩组创建的节点统计 |

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 列表可达 | `DescribeClusterNodePools --ClusterId <ClusterId>` | exit 0，`TotalCount >= 0` |
| 池存在 | 同上，检查 `NodePoolSet` | 包含目标 `NodePoolId`（空列表则 `TotalCount: 0`） |
| 详情完整 | `DescribeClusterNodePoolDetail --NodePoolId <NodePoolId>` | `NodePool.LifeState` 为预期状态 |
| 节点状态 | `DescribeClusterInstances --filter "InstanceSet[?NodePoolId=='<NodePoolId>']"` | `InstanceState` 与 `NodeCountSummary` 一致 |

```bash
# 确认 NodePoolId 在列表中
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region> \
    --filter "NodePoolSet[*].NodePoolId"
# expected: exit 0，返回数组，包含目标 NodePoolId
```

## 清理

只读操作，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePoolDetail` 返回 `FailedOperation.NodePoolQueryFailed`，错误信息含 `[E501001 DBRecordNotFound] record not found` | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region> --filter "NodePoolSet[*].NodePoolId"` 确认有效 ID 列表 | `NodePoolId` 不存在或拼写错误（常见于从控制台 URL 或截图复制），API 返回 `DBRecordNotFound` | 从 `DescribeClusterNodePools` 的 `NodePoolSet[*].NodePoolId` 获取有效 ID 后重新查询 |
| `DescribeClusterNodePools` 返回 `InvalidParameter.Param`，提示 invalid filter name | 检查命令中用的是 `--Filters`（API 级参数）还是 `--filter`（客户端 JMESPath） | 混淆 `--Filters`（API 级，需传数组）与 `--filter`（客户端 JMESPath 表达式） | 客户端结果过滤用 `--filter` + JMESPath 表达式，如 `--filter "NodePoolSet[?LifeState=='normal']"`；API 级筛选用 `--Filters` 数组 |
| `kubectl get nodes` 返回 `http: server gave HTTP response to HTTPS client` | 检查 kubeconfig 中 server 地址，确认是否为内网端点（如 `172.x.x.x:443`） | 集群内网端点不支持 HTTPS，kubectl 需在端点可达环境下执行 | `CreateClusterEndpoint --IsExtranet true` 开启公网端点，或通过 IOA/VPN/专线连通内网。注意：查看节点池信息本身不受影响，`DescribeClusterNodePools` 和 `DescribeClusterNodePoolDetail` 走控制面 API，不依赖 kubectl |

### 查询成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `NodeCountSummary.AutoscalingAdded` 中 `Joining`/`Initializing` 持续不归零 | `tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region <Region> --filter "InstanceSet[?NodePoolId=='<NodePoolId>'].{InstanceId:InstanceId,InstanceState:InstanceState}"` 查各节点状态 | CVM 创建中或卡住；`AutoscalingGroupStatus` 为 `disabled` 时节点数为伸缩组配置数，非实时伸缩结果 | 确认 CVM `InstanceState`；若 CVM 创建失败，检查伸缩组活动、安全组、子网可用 IP |
| `DescribeClusterNodePools` 返回空 `NodePoolSet`（`TotalCount: 0`） | 确认 `ClusterId` 正确：`tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 集群无节点池（正常状态）或 `ClusterId` 错误 | 若 `ClusterId` 正确且集群存在，空列表是正常状态，exit 0，无需排障。如需节点池，参见 [创建节点池](../创建节点池/tccli%20操作.md) |
| `LifeState` 长期为 `creating` | `tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region <Region>` 查看 `AutoscalingGroupStatus` 与 `NodeCountSummary` | 节点池创建卡住，可能是伸缩组活动失败、安全组或子网 IP 不足 | 检查伸缩组活动日志、安全组规则、子网 `AvailableIpCount`；保留 `NodePoolId`、`RequestId` 以备工单查询 |

> 涉及 RequestId 的排查路径，请保留 RequestId 以备工单或日志查询。

## 下一步

- [创建节点池](../创建节点池/tccli%20操作.md)：新建节点池
- [调整节点池](../调整节点池/tccli%20操作.md)：修改期望节点数、Labels、Taints 等配置
- [删除节点池](../删除节点池/tccli%20操作.md)：删除不再使用的节点池
- [查看节点池伸缩记录](../查看节点池伸缩记录/tccli%20操作.md)：查看伸缩组活动历史
- [查看节点池 - 官方文档](https://cloud.tencent.com/document/product/457/43736)

## 控制台替代

[查看节点池 - TKE 控制台](https://console.cloud.tencent.com/tke2/cluster)
