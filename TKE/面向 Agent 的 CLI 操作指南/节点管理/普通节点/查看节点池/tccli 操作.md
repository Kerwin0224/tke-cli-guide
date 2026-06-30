# 查看节点池

> 对照官方：[查看节点池](https://cloud.tencent.com/document/product/457/43736) · page_id `43736` · tccli ≥3.1.107.1 · API 2018-05-25

## 概述

通过 `DescribeClusterNodePools` 查询指定集群的节点池列表及摘要信息，通过 `DescribeClusterNodePoolDetail` 查看单个节点池的详细配置和节点计数。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置
- 已存在运行中的 TKE 集群（下文以 `<ClusterId>` 指代）
- 集群中已创建至少一个节点池
- CAM 权限：需具备以下只读 Action：
  - `tke:DescribeClusterStatus`
  - `tke:DescribeClusterNodePools`
  - `tke:DescribeClusterNodePoolDetail`
  - `tke:DescribeClusterInstances`

### 环境检查

```bash
# 检查 tccli 版本
tccli --version
# expected: tccli version 3.1.x（≥3.1.107）

# 确认集群存在且处于运行状态
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<ClusterId>"]'
# expected: "ClusterState": "Running"
```

```json
{
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-example",
            "ClusterState": "Running",
            "ClusterInstanceState": "AllNormal",
            "ClusterBMonitor": false,
            "ClusterInitNodeNum": 0,
            "ClusterRunningNodeNum": 1,
            "ClusterFailedNodeNum": 0,
            "ClusterClosedNodeNum": 0,
            "ClusterClosingNodeNum": 0,
            "ClusterDeletionProtection": false,
            "ClusterAuditEnabled": false
        }
    ],
    "TotalCount": 1,
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

```bash
# 确认集群中存在至少一个节点池
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou
# expected: "TotalCount" ≥ 1
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看节点池列表 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId>` | 是 |
| 按 ID 查看指定节点池 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --Filters '[{"Name":"NodePoolsId","Values":["<NodePoolId>"]}]'` | 是 |
| 按名称过滤 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --Filters '[{"Name":"NodePoolsName","Values":["<name>"]}]'` | 是 |
| 查看节点池详情（含节点计数） | `tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId>` | 是 |

## 操作步骤

### 1. 查看所有节点池（列表摘要）

查询集群下全部节点池的基本信息。

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region ap-guangzhou
```

> `<ClusterId>` 替换为实际的集群 ID（如 `cls-example`）。

参考输出：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "example-node-pool",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
            "LaunchConfigurationId": "asc-xxxxxxxx",
            "AutoscalingGroupId": "asg-xxxxxxxx",
            "Labels": [
                {
                    "Name": "env",
                    "Value": "staging"
                }
            ],
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
            "MaxNodesNum": 1,
            "MinNodesNum": 1,
            "DesiredNodesNum": 1,
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.6.9"
            },
            "NodePoolOs": "ubuntu22.04x86_64",
            "OsCustomizeType": "GENERAL",
            "DeletionProtection": false
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

### 2. 按节点池 ID 过滤查看

从上一步输出 `$.NodePoolSet[0].NodePoolId` 中提取目标节点池 ID，传入 `--Filters` 过滤。

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --Filters '[{"Name":"NodePoolsId","Values":["<NodePoolId>"]}]' \
    --region ap-guangzhou
```

> `<NodePoolId>` 替换为步骤 1 输出中的 `NodePoolId`（如 `np-example`）。可用的 Filter Name 包括 `NodePoolsId`（按 ID 过滤）、`NodePoolsName`（按名称过滤）、`Tags`（按标签过滤）。多个 Filter 之间为 AND 关系，同一 Filter 内多个 Values 为 OR 关系。

参考输出：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "example-node-pool",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
            "LaunchConfigurationId": "asc-xxxxxxxx",
            "AutoscalingGroupId": "asg-xxxxxxxx",
            "Labels": [
                {
                    "Name": "env",
                    "Value": "staging"
                }
            ],
            "Taints": [
                {
                    "Key": "dedicated",
                    "Value": "gpu",
                    "Effect": "NoSchedule"
                }
            ],
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
            "MaxNodesNum": 1,
            "MinNodesNum": 1,
            "DesiredNodesNum": 1,
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.6.9"
            },
            "NodePoolOs": "ubuntu22.04x86_64",
            "OsCustomizeType": "GENERAL",
            "DeletionProtection": false,
            "Tags": [
                {
                    "Key": "billing",
                    "Value": "example"
                }
            ]
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

### 3. 查看节点池详细计数（DescribeClusterNodePoolDetail）

获取单个节点池的完整配置和节点计数明细。`<NodePoolId>` 可从步骤 1 输出 `$.NodePoolSet[0].NodePoolId` 中提取。

```bash
tccli tke DescribeClusterNodePoolDetail \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --region ap-guangzhou
```

参考输出：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "example-node-pool",
        "ClusterInstanceId": "cls-example",
        "LifeState": "normal",
        "LaunchConfigurationId": "asc-xxxxxxxx",
        "AutoscalingGroupId": "asg-xxxxxxxx",
        "Labels": [
            {
                "Name": "env",
                "Value": "staging"
            }
        ],
        "Taints": [
            {
                "Key": "dedicated",
                "Value": "gpu",
                "Effect": "NoSchedule"
            }
        ],
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
        "MaxNodesNum": 1,
        "MinNodesNum": 1,
        "DesiredNodesNum": 1,
        "RuntimeConfig": {
            "RuntimeType": "containerd",
            "RuntimeVersion": "1.6.9"
        },
        "NodePoolOs": "ubuntu22.04x86_64",
        "OsCustomizeType": "GENERAL",
        "DeletionProtection": false,
        "Tags": [
            {
                "Key": "billing",
                "Value": "example"
            }
        ]
    },
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

### 4. 关键字段说明

#### DescribeClusterNodePools 返回字段

| 字段名 | 类型 | 必填 | 取值与约束 | 错误后果 |
|--------|------|:----:|-----------|----------|
| `TotalCount` | Integer | 一定返回 | 符合条件的节点池总数，≥0 | — |
| `NodePoolId` | String | 一定返回 | 格式 `np-xxxxxxxx`，节点池唯一标识 | 传入错误的 NodePoolId 会导致 DescribeClusterNodePoolDetail 返回 ResourceNotFound |
| `Name` | String | 一定返回 | 节点池名称（创建时指定） | — |
| `LifeState` | String | 一定返回 | `normal`（正常）、`creating`（创建中）、`updating`（更新中）、`deleting`（删除中） | 将 `updating` 误判为异常状态可能导致错误的排障方向 |
| `MaxNodesNum` | Integer | 一定返回 | 弹性伸缩上限，≥ `MinNodesNum` | — |
| `MinNodesNum` | Integer | 一定返回 | 弹性伸缩下限，≤ `MaxNodesNum` | — |
| `DesiredNodesNum` | Integer | 一定返回 | 期望节点数，介于 `MinNodesNum` 和 `MaxNodesNum` 之间 | — |
| `NodePoolOs` | String | 一定返回 | 操作系统镜像标识符，如 `ubuntu22.04x86_64` | — |
| `OsCustomizeType` | String | 一定返回 | `GENERAL`（普通）、`DOCKER_CUSTOMIZE`、`GPU_CUSTOMIZE` | — |
| `RuntimeConfig.RuntimeType` | String | 一定返回 | `containerd`、`docker` | — |
| `RuntimeConfig.RuntimeVersion` | String | 一定返回 | 容器运行时版本号，如 `1.6.9` | — |
| `NodeCountSummary` | Object | 一定返回 | 节点计数汇总对象 | — |
| `NodeCountSummary.AutoscalingAdded` | Object | 有弹性伸缩节点时返回 | 弹性伸缩创建的节点分状态计数 | 字段缺失（非 null）表示该集群没有弹性伸缩节点，不是 API 错误 |
| `NodeCountSummary.AutoscalingAdded.Joining` | Integer | 有弹性伸缩节点时返回 | 正在加入集群的节点数 | — |
| `NodeCountSummary.AutoscalingAdded.Initializing` | Integer | 有弹性伸缩节点时返回 | 正在初始化的节点数 | — |
| `NodeCountSummary.AutoscalingAdded.Normal` | Integer | 有弹性伸缩节点时返回 | 正常运行中的节点数（弹性伸缩创建） | — |
| `NodeCountSummary.AutoscalingAdded.Total` | Integer | 有弹性伸缩节点时返回 | 弹性伸缩创建的节点总数 | — |
| `NodeCountSummary.ManuallyAdded` | Object | 有手动添加节点时返回 | 手动添加的节点分状态计数 | 字段缺失表示该集群没有手动添加的节点 |
| `NodeCountSummary.ManuallyAdded.Total` | Integer | 有手动添加节点时返回 | 手动添加的节点总数 | — |
| `AutoscalingGroupStatus` | String | 一定返回 | `enabled`（已启用）、`disabled`（已禁用） | 未启用弹性伸缩时节点数不会自动调整 |
| `DeletionProtection` | Boolean | 一定返回 | `true`（已开启删除保护）、`false`（未开启） | 开启删除保护后无法直接删除节点池 |

#### DescribeClusterNodePoolDetail 返回字段

| 字段名 | 类型 | 必填 | 取值与约束 | 错误后果 |
|--------|------|:----:|-----------|----------|
| `AutoscalingAdded.Joining` | Integer | 有弹性伸缩节点时返回 | 正在加入集群的节点数 | — |
| `AutoscalingAdded.Initializing` | Integer | 有弹性伸缩节点时返回 | 正在初始化的节点数 | — |
| `AutoscalingAdded.Normal` | Integer | 有弹性伸缩节点时返回 | 正常运行的节点数 | — |
| `AutoscalingAdded.Total` | Integer | 有弹性伸缩节点时返回 | 弹性伸缩节点总数 | — |
| `ManuallyAdded.Joining` | Integer | 有手动添加节点时返回 | 手动添加正在加入的节点数 | — |
| `ManuallyAdded.Initializing` | Integer | 有手动添加节点时返回 | 手动添加正在初始化的节点数 | — |
| `ManuallyAdded.Normal` | Integer | 有手动添加节点时返回 | 手动添加正常运行的节点数 | — |
| `ManuallyAdded.Total` | Integer | 有手动添加节点时返回 | 手动添加节点总数 | — |

### 5. 查看节点池内节点列表

如需查看节点池内具体节点信息，使用 `DescribeClusterInstances`：

```bash
tccli tke DescribeClusterInstances \
    --ClusterId <ClusterId> \
    --InstanceRole "WORKER" \
    --region ap-guangzhou
```

参考输出：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example",
            "InstanceRole": "WORKER",
            "FailedReason": "=Ready:True",
            "InstanceState": "running",
            "DrainStatus": "",
            "InstanceAdvancedSettings": {
                "DesiredPodNumber": 64,
                "GPUArgs": {
                    "CUDA": {},
                    "CUDNN": {},
                    "CustomDriver": {},
                    "Driver": {},
                    "MIGEnable": false
                },
                "PreStartUserScript": null,
                "Taints": null,
                "MountTarget": "",
                "DockerGraphPath": "",
                "UserScript": "",
                "Unschedulable": 0,
                "Labels": [],
                "DataDisks": null,
                "ExtraArgs": {
                    "Kubelet": []
                }
            },
            "CreatedTime": "2026-06-23T08:00:00Z",
            "LanIP": "10.0.0.1",
            "NodePoolId": "np-example",
            "AutoscalingGroupId": "asg-xxxxxxxx"
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

> 当前 `DescribeClusterInstances` API 不支持直接按 `NodePoolId` 过滤节点。可结合输出的 `InstanceId` 在控制台或 `tccli cvm DescribeInstances` 中查看关联的节点池信息。输出中的 `NodePoolId` 字段可用于确认节点所属的节点池。

## 验证

```bash
# 确认节点池状态正常
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region ap-guangzhou
```

参考输出：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "example-node-pool",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
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
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.6.9"
            },
            "DeletionProtection": false
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 列表非空 | `TotalCount` | >= 1 |
| 状态正常 | `NodePoolSet[].LifeState` | `normal` |
| 节点数正常 | `NodePoolSet[].NodeCountSummary` | 节点数与创建时预期一致 |
| 弹性伸缩状态 | `AutoscalingGroupStatus` | `enabled` 或 `disabled`（取决于是否开启） |
| 容器运行时 | `RuntimeConfig.RuntimeType` | `containerd` |

## 清理

<!-- 查看操作无需清理 -->

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePools` 返回空列表 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou` 确认 `TotalCount` 值 | 集群尚未创建节点池，或 ClusterId 错误 | 使用 `tccli tke DescribeClusters --region ap-guangzhou` 列出所有集群，确认 ClusterId 正确；参考 [创建节点池](https://cloud.tencent.com/document/product/457/43735) 创建节点池 |
| `LifeState` 为 `creating` 且超过 5 分钟未变为 `normal` | 持续轮询观察：`tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region ap-guangzhou` + 检查底层 CVM 状态：`tccli tke DescribeClusterInstances --ClusterId <ClusterId> --InstanceRole WORKER --region ap-guangzhou | jq '.InstanceSet[] | {InstanceId, InstanceState, NodePoolId}'` | 节点池底层 CVM 创建缓慢（镜像拉取、初始化等）或 CVM 配额不足 | 超过 10 分钟仍为 `creating` → 检查 CVM 配额（`tccli cvm DescribeAccountQuota`）和子网 IP 余量（`tccli vpc DescribeSubnets --SubnetIds '["<SubnetId>"]'` 查看 `AvailableIpAddressCount`）；记下 RequestId 联系售后 |
| `AutoscalingGroupStatus` 为 `enabled` 但节点数不达期望 | `tccli tke DescribeClusterAsGroups --ClusterId <ClusterId> --region ap-guangzhou` 查看伸缩活动记录 | 弹性伸缩因资源不足或配置错误无法创建节点 | 检查机型可用性（`tccli cvm DescribeZoneInstanceConfigInfos`）、子网 IP 余量、CVM 配额 |
| 返回 `InvalidParameterValue` — ClusterId 格式错误 | `tccli tke DescribeClusters --region ap-guangzhou | jq '.Clusters[] | {ClusterId, ClusterName, ClusterStatus}'` 列出所有集群以确认正确的 ClusterId | ClusterId 格式错误（不是 `cls-xxxxxxxx`） | 从 DescribeClusters 输出中复制正确的 ClusterId |
| `DescribeClusterNodePoolDetail` 返回 `ResourceNotFound` | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou | jq '.NodePoolSet[] | {NodePoolId, Name, LifeState}'` 列出所有节点池确认 NodePoolId | NodePoolId 错误或节点池已被删除 | 从 DescribeClusterNodePools 输出中复制正确的 `NodePoolId`（格式 `np-xxxxxxxx`） |

## 下一步

- [调整节点池](https://cloud.tencent.com/document/product/457/43737) — 修改节点池配置（规模、Label、Taint 等）
- [删除节点池](https://cloud.tencent.com/document/product/457/43738) — 删除不再需要的节点池
- [创建节点池](https://cloud.tencent.com/document/product/457/43735) — 创建新的节点池

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 节点池](https://console.cloud.tencent.com/tke2/nodepool) — 查看节点池列表和详情。
