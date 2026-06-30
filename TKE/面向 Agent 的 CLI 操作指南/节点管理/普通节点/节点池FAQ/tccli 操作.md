# 节点池 FAQ

> 对照官方：[节点池 FAQ](https://cloud.tencent.com/document/product/457/90886) · page_id `90886`

## 概述

节点池 FAQ 涵盖节点池与伸缩组的关系、可修改/不可修改参数、节点命名规则等常见问题。本页将 FAQ 转为可操作的 CLI 查询与排障指南。

节点池底层依赖 Auto Scaling（AS）的伸缩组（ASG）和启动配置（ASC）。每个节点池对应一个伸缩组，每个伸缩组对应一个启动配置。通过 CLI 可直接查看这些底层映射关系。

以本集群 `cls-xxxxxxxx`（`ap-guangzhou`，Kubernetes 1.30.0，containerd 1.6.9）的节点池 `np-lmvsqkuu`（`kerwinwjyan-rewrite-s3-np`，`LifeState: normal`，`AutoscalingGroupStatus: disabled`）为例。

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

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail, as:DescribeAutoScalingGroups
#    验证：查询节点池列表确认权限
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 NodePoolSet（可为空）
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点池列表 | `tke DescribeClusterNodePools` | 是 |
| 查看节点池详情（含伸缩组） | `tke DescribeClusterNodePoolDetail` | 是 |
| 修改节点池基本配置 | `tke ModifyClusterNodePool` | 否 |
| 查看伸缩组详情 | `as DescribeAutoScalingGroups` | 是 |
| 查看启动配置 | `as DescribeLaunchConfigurations` | 是 |

## 关键字段说明

节点池详情返回的关键字段：

| 字段 | 类型 | 说明 | 可修改 |
|------|------|------|:--:|
| `Name` | String | 节点池名称 | 是 |
| `AutoscalingEnabled` | Boolean | 是否开启弹性伸缩 | 是 |
| `MaxNodesNum` / `MinNodesNum` | Integer | 伸缩范围 | 是 |
| `DesiredNodesNum` | Integer | 期望节点数 | 是 |
| `Labels` | Array | 节点 Label | 是（对全部节点生效） |
| `Taints` | Array | 节点 Taint | 是（对全部节点生效，可能引起重调度） |
| `LaunchConfiguration.InstanceType` | String | 机型 | 是（仅新节点生效） |
| `LaunchConfiguration.SystemDisk` | Object | 系统盘配置 | 是（仅新节点生效） |
| `LaunchConfiguration.DataDisks` | Array | 数据盘配置 | 是（仅新节点生效） |
| `LaunchConfiguration.InstanceChargeType` | String | 计费模式 | **不可修改** |

## 操作步骤

### FAQ 1：查看节点池与伸缩组的对应关系

```bash
# 获取节点池详情，找到 LaunchConfigureId（即伸缩组启动配置 ID）
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，返回 LaunchConfigureId

# 通过启动配置 ID 查找伸缩组
tccli as DescribeAutoScalingGroups --region <Region> \
    --Filters '[{"Name":"launch-configuration-id","Values":["<LaunchConfigureId>"]}]'
# expected: 返回 AutoScalingGroupSet，包含伸缩组 ID、期望/最大/最小实例数
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

以 `np-lmvsqkuu` 为例：

```bash
tccli tke DescribeClusterNodePoolDetail \
    --ClusterId cls-xxxxxxxx \
    --NodePoolId np-lmvsqkuu \
    --region ap-guangzhou
```

**预期输出**：

```json
{
    "AutoScalingGroupSet": [
        {
            "AutoScalingGroupId": "asg-example",
            "AutoScalingGroupName": "tke-np-example",
            "LaunchConfigurationId": "asc-example",
            "DesiredCapacity": 3,
            "MaxSize": 10,
            "MinSize": 1,
            "InstanceCount": 3,
            "InServiceInstanceCount": 3
        }
    ]
}
```

### FAQ 2：查看哪些参数可以修改

```bash
# 查看当前节点池完整配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 返回 NodePool 完整对象

# 可修改参数使用 ModifyClusterNodePool（以修改名称和伸缩范围为例）
cat > modify-nodepool.json <<'EOF'
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "Name": "new-pool-name",
    "MaxNodesNum": 20,
    "MinNodesNum": 2
}
EOF
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool.json
# expected: exit 0
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

> **注意**：计费模式、VPC 不可修改。不要直接在伸缩组控制台或 AS API 修改启动配置和伸缩组，否则可能导致节点池删除失败或集群数据不一致。

### FAQ 3：查看节点命名规则

```bash
# 查看节点池内节点的实例名称
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --Filters '[{"Name":"node-pool-id","Values":["<NodePoolId>"]}]' \
    | jq '.InstanceSet[] | {InstanceId, InstanceName: .InstanceName}'
# expected: 列出节点池内所有节点 ID 和名称

# 节点名称格式：as-tke-np-<random> 或 as-tke-np-<random><序号>（多节点扩容时）
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` |
| `<LaunchConfigureId>` | 启动配置 ID | 格式 `asc-xxxxxxxx` | `DescribeClusterNodePoolDetail` 返回的 `LaunchConfigureId` |

## 验证

### 控制面（tccli）

```bash
# 验证 1：节点池与伸缩组数据一致
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    | jq '.NodePool | {DesiredNodesNum, AutoscalingEnabled, LaunchConfigureId}'
# expected: DesiredNodesNum 与伸缩组的 DesiredCapacity 一致

# 验证 2：不可修改参数（计费模式）未被意外修改
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    | jq '.NodePool.LaunchConfiguration.InstanceChargeType'
# expected: 返回原始计费模式（如 POSTPAID_BY_HOUR）
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

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池与 ASG 实例数一致 | 对比 `DescribeClusterNodePoolDetail` 和 `as DescribeAutoScalingGroups` | `DesiredNodesNum` = `DesiredCapacity` |
| 节点池可修改参数生效 | `DescribeClusterNodePoolDetail` 查看修改后的字段 | 值与修改时一致 |
| 节点池命名规则 | `DescribeClusterInstances` 查看节点名称 | 名称遵循 `as-tke-np-<random>` 格式 |

## 清理

无需清理。本页为只读查询，不创建任何资源。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeAutoScalingGroups` 返回空 | 检查 `LaunchConfigureId` 是否正确：`tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId> \| jq '.NodePool.LaunchConfigureId'` | 启动配置 ID 不正确，或伸缩组已被意外删除 | 确认节点池状态为 `normal`；如果伸缩组被删除，节点池将无法正常工作，需重新创建节点池 |
| `ModifyClusterNodePool` 失败 | 检查修改的参数是否包含不可修改字段（计费模式、VPC） | 尝试修改了不可修改的参数 | 只修改允许修改的字段：名称、伸缩范围、Label/Taint、删除保护 |
| 修改配置后已有节点未更新 | `DescribeClusterInstances` 检查已有节点配置 | OS、数据盘、自定义脚本等配置仅对新节点生效 | 如需对已有节点生效，需移出节点后重新加入 |

### 概念理解类问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 多次扩容后节点名重复 | `DescribeClusterInstances` 检查节点名称 | 每次扩容独立生成随机标识，跨批次可能有碰撞 | 不影响集群功能，节点有唯一 InstanceId。可在 AS 启动配置高级设置中自定义命名规则 |
| 在 AS 控制台改了伸缩组期望实例数 | 对比 `DescribeClusterNodePoolDetail` 的 `DesiredNodesNum` 和 `as DescribeAutoScalingGroups` 的 `DesiredCapacity` | TKE 无法感知 AS 控制台的直接变更 | 通过 `ModifyClusterNodePool` 调整 `DesiredNodesNum`，让 TKE 同步到 AS |
| 启动配置被绑定到其他伸缩组 | `tccli as DescribeLaunchConfigurations --region <Region> --LaunchConfigurationIds '["<LaunchConfigureId>"]'` | 启动配置只能在创建节点池时绑定，不能复用 | 重新创建节点池，使用新的启动配置 |

## 下一步

- [创建节点池](../创建节点池/tccli%20操作.md) —— 新建节点池
- [调整节点池](../调整节点池/tccli%20操作.md) —— 修改节点池参数
- [删除节点池](../删除节点池/tccli%20操作.md) —— 删除节点池
- [节点初始化流程 FAQ](../节点初始化流程FAQ/tccli%20操作.md) —— 节点加入失败排查

## 控制台替代

在 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 进入目标集群 → **节点管理** → **节点池** → 点击目标节点池查看详情和伸缩组关联。
