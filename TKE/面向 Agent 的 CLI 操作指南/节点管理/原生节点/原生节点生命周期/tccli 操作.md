# 原生节点生命周期

> 对照官方：[原生节点生命周期](https://cloud.tencent.com/document/product/457/78200) · page_id `78200`

## 概述

原生节点从创建到销毁经历多个状态阶段：创建中 -> 运行中 -> 异常/维护中 -> 移除中 -> 已移除。通过 `DescribeClusterNodePoolDetail` 查看节点状态（`NodeCountSummary` 中各子状态计数）和节点池状态（`LifeState`），各状态对应不同的 API 操作。

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
#    tke:DescribeClusters, tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail
#    tke:DeleteClusterNodePool, tke:ModifyClusterNodePool
# 验证：执行 DescribeClusterNodePools 确认权限
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表（可为空）
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
# 4. 确认目标集群和原生节点池存在
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 "Running"

tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，NodePoolSet 中包含 Type="Native" 的节点池
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点池状态 | `DescribeClusterNodePools` | 是 |
| 查看节点池详情（含节点状态分布） | `DescribeClusterNodePoolDetail` | 是 |
| 删除节点池 | `DeleteClusterNodePool` | 是 |
| 修改节点池 | `ModifyClusterNodePool` | 否 |

## 操作步骤

### 生命周期阶段

原生节点池和节点的生命周期分为以下几个阶段：

```
创建中（creating）-> 运行中（normal）-> 异常/维护（abnormal/updating）-> 移除中（deleting）-> 已移除
```

#### 节点池生命周期（LifeState）

| LifeState | 说明 | 允许的操作 |
|-----------|------|-----------|
| `creating` | 节点池创建中，节点正在初始化 | 等待，不可修改或删除 |
| `normal` | 节点池正常运行 | 可修改、扩缩容、删除 |
| `updating` | 节点池更新中（如修改参数、升级版本） | 等待，不可再次修改 |
| `deleting` | 节点池删除中 | 等待，不可操作 |
| `abnormal` | 节点池异常 | 排查异常后修复或删除 |

#### 节点状态（NodeCountSummary 子状态）

| 节点状态 | 说明 | 参与调度 |
|---------|------|:--:|
| `Joining` | 节点正在加入集群 | 否 |
| `Initializing` | 节点正在初始化（安装组件、配置网络） | 否 |
| `Normal` | 节点正常运行 | 是 |
| `Abnormal` | 节点异常（如 NotReady） | 否 |

### 查询节点池生命周期状态

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，NodePoolSet 中每项含 LifeState 字段
```

**预期输出**：

```json
{
    "TotalCount": 2,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-native",
            "Name": "native-pool-prod",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Native",
            "DesiredNodesNum": 3
        },
        {
            "NodePoolId": "np-example-creating",
            "Name": "native-pool-new",
            "ClusterId": "cls-example",
            "LifeState": "creating",
            "NodePoolType": "Native",
            "DesiredNodesNum": 5
        }
    ]
}
```

### 查看节点池内各节点状态分布

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，NodeCountSummary 中展示各状态的节点计数
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-native",
        "Name": "native-pool-prod",
        "ClusterId": "cls-example",
        "LifeState": "normal",
        "Type": "Native",
        "NodeCountSummary": {
            "ManuallyAdded": {
                "Joining": 0,
                "Initializing": 1,
                "Normal": 4,
                "Abnormal": 1
            },
            "AutoscalingAdded": {
                "Joining": 1,
                "Initializing": 0,
                "Normal": 2,
                "Abnormal": 0
            }
        }
    }
}
```

字段说明：`ManuallyAdded` 为手动创建的节点，`AutoscalingAdded` 为弹性伸缩创建的节点。四种子状态 `Joining` / `Initializing` / `Normal` / `Abnormal` 为计数，总和等于该来源的当前节点数。

### 控制台概念与 API 枚举映射

| 控制台状态名称 | API 枚举值 | 所属层级 |
|--------------|-----------|---------|
| 创建中 | `creating` | 节点池 LifeState |
| 运行中 | `normal` | 节点池 LifeState / 节点 InstanceState |
| 更新中 | `updating` | 节点池 LifeState |
| 异常 | `abnormal` | 节点池 LifeState |
| 删除中 | `deleting` | 节点池 LifeState |
| 加入中 | `Joining` | 节点 InstanceState |
| 初始化中 | `Initializing` | 节点 InstanceState |
| 正常 | `Normal` | 节点 InstanceState |
| 移除中 | `Removing` | 节点 InstanceState |

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `NODE_POOL_ID` | 原生节点池 ID | 格式 `np-xxxxxxxx`，Type 须为 Native | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

## 验证

### 控制面（tccli）

```bash
# 验证节点池生命周期状态
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，每项 LifeState 为 creating / normal / updating / deleting / abnormal 之一

# 验证节点详情（含各状态计数）
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，NodeCountSummary 含 ManuallyAdded 和 AutoscalingAdded 的状态分布
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

## 清理

本页为概念说明页，无资源需清理。如需删除节点池，参考 [删除原生节点](../删除原生节点/tccli%20操作.md)。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePoolDetail` 返回 `ResourceNotFound.NodePoolNotFound` | 检查 NODE_POOL_ID 是否正确 | 节点池 ID 不存在或不属于该集群 | 通过 `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 确认有效的节点池 ID |
| `DescribeClusterNodePools` 返回空列表 | 执行 `DescribeClusters` 确认集群存在且可操作 | 集群中尚未创建节点池 | 前往 [新建原生节点](../新建原生节点/tccli%20操作.md) 创建原生节点池 |

### 查询成功但节点状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| LifeState 长时间为 `creating` | `DescribeClusterNodePoolDetail` 检查 NodeCountSummary 各计数 | 节点初始化可能卡住：镜像拉取慢、网络未就绪、资源不足 | 等待。超过 15 分钟仍为 `creating` 则保留 CLUSTER_ID、NODE_POOL_ID、RequestId -- 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看节点事件 -- 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |
| LifeState 为 `abnormal` | `DescribeClusterNodePoolDetail` 查看具体节点的 InstanceState | 部分节点异常（NotReady），原因可能是磁盘满、网络不通、kubelet 异常 | 如 kubectl 可达：`kubectl describe node NODE_NAME` 查看 Conditions；登录节点检查 kubelet 日志 |
| `NodeCountSummary.ManuallyAdded.Abnormal` > 0 | 统计异常节点数量 | 节点可能 NotReady，原因多样（资源耗尽、网络分区、内核 OOM） | `kubectl describe node NODE_NAME` 查看具体原因 -- 根据 Condition 修复 -- 若无法自愈，节点会被故障自愈规则处理后重建 |
| `NodeCountSummary.ManuallyAdded.Joining` 持续有计数 | 检查集群资源配额和子网 IP 余量 | 可能原因：子网 IP 不足、CVM 配额耗尽、机型库存不足 | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 查看 AvailableIpAddressCount；`tccli cvm DescribeZoneInstanceConfigInfos --region <Region>` 检查机型可用性 |

## 下一步

- [新建原生节点](https://cloud.tencent.com/document/product/457/78198) -- 创建原生节点池
- [删除原生节点](https://cloud.tencent.com/document/product/457/78199) -- 删除节点池
- [故障自愈规则](https://cloud.tencent.com/document/product/457/78209) -- 配置故障自愈策略
- [原生节点扩缩容](https://cloud.tencent.com/document/product/457/78648) -- 调整节点数量

## 控制台替代

[容器服务控制台 - 节点管理](https://console.cloud.tencent.com/tke2/cluster)
