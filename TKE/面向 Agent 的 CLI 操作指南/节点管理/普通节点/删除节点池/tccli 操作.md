# 删除节点池

> 对照官方：[删除节点池](https://cloud.tencent.com/document/product/457/43738) · page_id `43738`

## 概述

通过 `DeleteClusterNodePool` 删除指定集群中的节点池。支持单次删除一个或多个节点池。

> **危险警告：** `--KeepInstance false` 会级联终止池内所有 CVM 实例，本地盘数据随实例销毁而永久丢失，不可恢复。CBS 云硬盘是否随实例销毁取决于创建时的设置——不一定同步删除，需单独检查。关联的 Auto Scaling Group 和 Launch Configuration 会自动清理。生产环境执行前务必确认数据已备份或迁移。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DeleteClusterNodePool, tke:DescribeClusterNodePools
#    tke:DescribeClusterNodePoolDetail, tke:DescribeClusterInstances
# 验证：执行 DescribeClusterNodePools 确认权限
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>
# expected: exit 0，返回节点池列表（可为空）
```

### 资源检查

```bash
# 4. 确认目标节点池存在
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>
# expected: 返回的 NodePoolSet 中包含目标节点池，记录其 NodePoolId、Name、LifeState、DesiredNodesNum

# 5. 查看节点池详情（含删除保护状态）
tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --region <Region>
# expected: exit 0，确认 DeletionProtection 为 false（CLI 不强制检查此标志，但建议确认）

# 6. 确认池内节点状态
tccli tke DescribeClusterInstances --ClusterId <ClusterId> \
    --InstanceRole WORKER \
    --region <Region>
# expected: exit 0，确认池内节点已排空工作负载
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 删除节点池 | `DeleteClusterNodePool` | 否（已删除节点池的查询返回 `FailedOperation.NodePoolQueryFailed`，含 `DBRecordNotFound`） |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看节点池详情 | `DescribeClusterNodePoolDetail` | 是 |
| 查看集群节点 | `DescribeClusterInstances` | 是 |

> **参数差异：** 控制台删除节点池时有"是否保留节点"勾选框；CLI 对应 `--KeepInstance` 参数。**该参数是 Required 而非 Optional——每次调用必须显式指定 `true` 或 `false`，无默认值。**

## 操作步骤

### 1. 删除前确认

#### 1.1 列出所有节点池

`DescribeClusterNodePools` 仅接受 `--ClusterId` 和 `--Filters` 参数，**不接受 `--NodePoolIds`**。如需定位特定节点池，使用 `jq` 过滤：

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region <Region>
# expected: exit 0，返回集群下所有节点池
```

**预期输出**：

```json
{
  "NodePoolSet": [
    {
      "NodePoolId": "np-example2",
      "Name": "another-nodepool",
      "LifeState": "normal",
      "DesiredNodesNum": 1
    },
    {
      "NodePoolId": "np-example",
      "Name": "target-nodepool",
      "LifeState": "normal",
      "DesiredNodesNum": 0
    }
  ],
  "TotalCount": 2,
  "RequestId": "eb3fe2fe-ad62-40fc-a4ad-cda245437c8b"
}
```

使用 `jq` 过滤出目标节点池：

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region <Region> \
    | jq '.NodePoolSet[] | select(.NodePoolId == "<NodePoolId>")'
```

#### 1.2 查看目标节点池详情

查询单个节点池详情使用 `DescribeClusterNodePoolDetail`（`--NodePoolId` 单数）：

```bash
tccli tke DescribeClusterNodePoolDetail \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --region <Region>
# expected: exit 0，返回节点池完整配置
```

**预期输出**：

```json
{
  "NodePool": {
    "NodePoolId": "np-example",
    "Name": "target-nodepool",
    "ClusterInstanceId": "cls-example",
    "LifeState": "normal",
    "LaunchConfigurationId": "asc-example",
    "AutoscalingGroupId": "asg-example",
    "Labels": [],
    "Taints": [],
    "Annotations": [],
    "NodeCountSummary": {
      "ManuallyAdded": { "Joining": 0, "Initializing": 0, "Normal": 0, "Total": 0 },
      "AutoscalingAdded": { "Joining": 0, "Initializing": 0, "Normal": 0, "Total": 0 }
    },
    "AutoscalingGroupStatus": "disabled",
    "MaxNodesNum": 1,
    "MinNodesNum": 0,
    "DesiredNodesNum": 0,
    "NodePoolOs": "tlinux3.1x86_64",
    "OsCustomizeType": "GENERAL",
    "DeletionProtection": false
  },
  "RequestId": "6e0fd52e-3e62-4863-9323-cbe5cae4f1b3"
}
```

#### 1.3 确认池内节点状态

```bash
tccli tke DescribeClusterInstances \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER \
    --region <Region>
# expected: exit 0，列出所有 WORKER 节点
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "InstanceSet": [
    {
      "InstanceId": "ins-example",
      "InstanceRole": "WORKER",
      "FailedReason": "",
      "InstanceState": "running",
      "DrainStatus": "",
      "LanIP": "172.24.0.34",
      "NodePoolId": "np-example2",
      "CreatedTime": "2026-06-23T08:13:42Z"
    }
  ],
  "RequestId": "02f01412-a2e9-487e-9ac3-5dc3888cd3e4"
}
```

> 若目标节点池的 `DesiredNodesNum` 为 `0`，则池内无节点，`InstanceSet` 中不包含该池的节点——这是预期行为。

### 2. 执行删除

#### 选择依据

- **是否保留 CVM 实例**：若节点池内无实际节点（`DesiredNodesNum=0`），`--KeepInstance false` 可确保关联资源（ASG、Launch Configuration）被完全清理。若池内有需要保留的 CVM 实例，应设为 `true` 以防止节点被误删，但需注意 CVM 会脱离节点池管理并继续计费。
- **参数为 Required**：`--KeepInstance` 是 Required 参数，每次调用必须显式指定 `true` 或 `false`，无默认值。
- **批量删除**：单次调用可删除多个节点池，`--NodePoolIds` 接受 JSON 数组格式。

#### 最小删除（单个节点池）

```bash
tccli tke DeleteClusterNodePool \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId>"]' \
    --KeepInstance false \
    --region <Region>
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
  "RequestId": "650229e7-b06a-467c-815e-b46fdfab94ab"
}
```

**参数说明**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|:--:|------|
| `--ClusterId` | String | 是 | 集群 ID，如 `<ClusterId>` |
| `--NodePoolIds` | Array | 是 | 节点池 ID 数组，格式 `'["<NodePoolId1>"]'`。即使只删一个也必须是数组格式 |
| `--KeepInstance` | Boolean | **是**（Required） | `true`=保留 CVM 实例继续计费；`false`=同步销毁 CVM 实例停止计费。**无默认值，每次必须显式指定** |
| `--region` | String | 是 | 地域，如 `ap-guangzhou` |

#### 批量删除（多个节点池）

```bash
tccli tke DeleteClusterNodePool \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId1>", "<NodePoolId2>"]' \
    --KeepInstance false \
    --region <Region>
```

**预期输出**：

```json
{
  "RequestId": "650229e7-b06a-467c-815e-b46fdfab94ab"
}
```

> 批量删除与单节点池删除返回格式相同，仅 `RequestId` 不同。因为 DeleteClusterNodePool 是一个原子操作，`--NodePoolIds` 中的所有节点池会被一起提交删除。

### 3. 确认删除（异步）

删除是异步操作。节点池先进入 `LifeState=deleting` 状态，数秒后完全消失。

#### 3.1 第一次确认（异步进行中）

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region <Region>
# expected: 目标节点池 LifeState 变为 "deleting"
```

**预期输出**：

```json
{
  "NodePoolSet": [
    {
      "NodePoolId": "np-example2",
      "Name": "another-nodepool",
      "LifeState": "normal",
      "DesiredNodesNum": 1
    },
    {
      "NodePoolId": "np-example",
      "Name": "target-nodepool",
      "LifeState": "deleting",
      "DesiredNodesNum": 0
    }
  ],
  "TotalCount": 2,
  "RequestId": "7dfb4a3e-bcc9-4d5b-988e-6d83733341cf"
}
```

#### 3.2 第二次确认（已完全删除）

等待数秒后重新查询：

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region <Region>
# expected: NodePoolSet 中不包含目标节点池，TotalCount 减少
```

**预期输出**：

```json
{
  "NodePoolSet": [
    {
      "NodePoolId": "np-example2",
      "Name": "another-nodepool",
      "LifeState": "normal",
      "DesiredNodesNum": 1
    }
  ],
  "TotalCount": 1,
  "RequestId": "8bf05b66-0121-4d2f-8c0a-50bbb6e08bdf"
}
```

#### 3.3 使用 jq 精确确认

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region <Region> \
    | jq '[.NodePoolSet[] | select(.NodePoolId == "<NodePoolId>")] | length'
# expected: 0
```

## 验证

| 维度 | 检查命令 | 预期 |
|------|---------|------|
| 节点池已删除 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>` | `NodePoolSet` 中不包含目标 `NodePoolId`，`TotalCount` 减少 |
| 节点已销毁 | `tccli tke DescribeClusterInstances --ClusterId <ClusterId> --InstanceRole WORKER --region <Region>` | 不包含属于目标节点池的 WORKER 节点 |
| 节点池详情不可查 | `tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region <Region>` | 返回 `FailedOperation.NodePoolQueryFailed`（`DBRecordNotFound`） |
| 关联资源已清理 | 查看 ASG（Auto Scaling Group）和 Launch Configuration | 关联的弹性伸缩组和启动配置随节点池自动删除 |

## 清理

此操作本身即为清理操作。但以下副作用资源可能需要额外关注：

> **⚠️ 清理警告：**
> - `DeleteClusterNodePool --KeepInstance false` 会级联终止池内所有 CVM 实例
> - CVM 本地盘数据随实例销毁而永久丢失，不可恢复
> - CBS 云硬盘是否随实例销毁取决于创建时的设置——不一定同步删除，需单独检查
> - 关联的 Auto Scaling Group 和 Launch Configuration 自动清理
> - 空节点池（`DesiredNodesNum=0`）删除无 CVM 影响，但仍会清理 ASG/LC 资源

| 资源 | 是否自动清理 | 说明 |
|------|:--:|------|
| 节点池记录 | 是 | 从 `NodePoolSet` 中移除 |
| 池内 CVM 实例 | 取决于 `--KeepInstance` | `false` 时销毁 CVM 并停止计费；`true` 时保留 CVM 继续计费 |
| CVM 本地盘 | 否 | 随 CVM 销毁而丢失，不可恢复 |
| CBS 云硬盘 | 取决于购买设置 | 购买时勾选"随实例销毁"则同步销毁；否则需手动清理 |
| 弹性伸缩组（ASG） | 是 | 自动删除 |
| 启动配置（Launch Configuration） | 是 | 自动删除 |

### 手动清理 CBS 云硬盘

若 `--KeepInstance false` 后 CBS 云硬盘未随实例销毁（取决于创建实例时是否勾选"随实例销毁"），需手动释放。

> **上下文映射**：`<InstanceId>` 取自步骤 [1.3 确认池内节点状态](#13-确认池内节点状态) 的 `DescribeClusterInstances` 输出，使用 `jq` 过滤：
> ```bash
> tccli tke DescribeClusterInstances --ClusterId <ClusterId> --InstanceRole WORKER --region <Region> \
>     | jq -r '.InstanceSet[] | select(.NodePoolId == "<NodePoolId>") | .InstanceId'
> ```

```bash
# 查看是否存在未被释放的云硬盘（<InstanceId> 从上一步 jq 过滤得到）
tccli cbs DescribeDisks --region <Region> \
    --Filters '[{"Name":"instance-id","Values":["<InstanceId>"]}]'
# expected: exit 0，列出关联的云硬盘
```

**预期输出**：

```json
{
  "DiskSet": [
    {
      "DiskId": "disk-example",
      "DiskName": "data-disk",
      "DiskSize": 50,
      "DiskState": "ATTACHED",
      "DiskType": "CLOUD_PREMIUM"
    }
  ],
  "TotalCount": 1,
  "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

> `<DiskId>` 取自上述输出 `$.DiskSet[].DiskId`，使用 `jq -r '.DiskSet[].DiskId'` 提取：

```bash
# 释放不用的云硬盘（<DiskId> 从上一步输出提取）
tccli cbs TerminateDisks --region <Region> \
    --DiskIds '["<DiskId>"]'
```

**预期输出**：

```json
{
  "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DeleteClusterNodePool` 返回 `MissingParameter.KeepInstance` | 检查命令是否包含 `--KeepInstance` 参数 | `--KeepInstance` 是 Required 参数，无默认值，未指定 | 显式指定 `--KeepInstance true` 或 `--KeepInstance false` |
| `DeleteClusterNodePool` 返回 `InvalidParameter.Param`（NodePoolIds 格式错误） | `echo '["<NodePoolId>"]'` 检查 JSON 格式 | 传入了单数字符串（如 `"np-xxx"`）而非 JSON 数组 | 使用 `'["<NodePoolId>"]'` 格式。即使只删一个节点池，也必须用数组 |
| `DeleteClusterNodePool` 返回 `ResourceNotFound.NodePoolNotFound` | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>` 确认节点池是否存在 | 目标节点池已被删除或 NodePoolId 错误 | 用 `DescribeClusterNodePools` 列出现有节点池，确认正确的 NodePoolId |
| `DescribeClusterNodePools` 返回 `InvalidParameter.Param`（invalid filter name NodePoolIds） | 检查是否在 `--Filters` 中使用了 `Name=NodePoolIds` | `DescribeClusterNodePools` 不接受 `--NodePoolIds` 参数，也不支持按 NodePoolIds 过滤 | 查询单个节点池详情用 `DescribeClusterNodePoolDetail --NodePoolId <NodePoolId>`（单数）；批量查询用 `DescribeClusterNodePools` + `jq` 过滤 |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`（autoscaling group name can not be set） | 检查 `AutoScalingGroupPara` 中是否设置了 `AutoScalingGroupName` | `AutoScalingGroupName` 由系统自动生成，不可手动设置 | 从 `AutoScalingGroupPara` 中移除 `AutoScalingGroupName` 字段 |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`（image id can not be set） | 检查 `LaunchConfigurePara` 中是否设置了 `ImageId` | `ImageId` 由系统自动配置，不可手动设置 | 从 `LaunchConfigurePara` 中移除 `ImageId` 字段 |
| `CreateClusterNodePool` 返回 `InvalidParameter.Param`（security group ids is not set） | 检查 `LaunchConfigurePara` 中是否包含 `SecurityGroupIds` | 安全组 ID 必须显式指定 | 在 `LaunchConfigurePara` 中添加 `"SecurityGroupIds": ["<SecurityGroupId>"]` |

### 删除已提交但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池长时间处于 `deleting` 状态 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>` 持续查询 `LifeState` | 节点池内 CVM 实例退还缓慢，或伸缩活动未完成 | 继续等待（通常数分钟内完成）。若超过 10 分钟仍未消失，保留 `RequestId` → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/nodepool) 查看详细状态 |
| `--KeepInstance false` 后 CVM 实例仍在计费 | `tccli cvm DescribeInstances --region <Region>` 检查实例状态 | CVM 退还延迟 | 等待数分钟；如持续未释放，保留 `RequestId` 在控制台手动退还 |
| `--KeepInstance false` 后 CBS 云硬盘仍在计费 | `tccli cbs DescribeDisks --region <Region>` 检查云硬盘 | 云硬盘创建时未勾选"随实例销毁" | 手动执行 `tccli cbs TerminateDisks --region <Region> --DiskIds '["<DiskId>"]'` |
| 删除操作被控制台拒绝 | `tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region <Region>` 检查 `DeletionProtection` | 控制台开启了删除保护 | CLI 命令不受控制台删除保护限制，可直接执行；或先执行 `tccli tke ModifyClusterNodePool --ClusterId <ClusterId> --NodePoolId <NodePoolId> --DeletionProtection false --region <Region>` |
| 伸缩组锁定导致删除失败 | `tccli tke DescribeClusterAsGroups --ClusterId <ClusterId> --region <Region>` 确认伸缩组状态 | 伸缩组处于活跃状态（扩容/缩容进行中） | 等待伸缩活动完成后重试删除 |

## 下一步

- [创建节点池](https://cloud.tencent.com/document/product/457/43735) — 重新创建节点池
- [查看节点池](https://cloud.tencent.com/document/product/457/43736) — 查看其他节点池状态
- [调整节点池](https://cloud.tencent.com/document/product/457/43737) — 修改节点池配置

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 节点池 → 删除](https://console.cloud.tencent.com/tke2/nodepool)
