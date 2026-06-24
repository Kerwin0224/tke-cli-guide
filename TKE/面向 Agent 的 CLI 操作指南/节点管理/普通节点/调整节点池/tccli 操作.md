# 调整节点池

> 对照官方：[调整节点池](https://cloud.tencent.com/document/product/457/43737) · page_id `43737` · tccli ≥3.1.x · API 2018-05-25

## 概述

通过 `ModifyClusterNodePool` 调整节点池配置，包括节点规模、Label、Taint 和删除保护等参数。对于仅调整期望节点数（不修改其他配置）的场景，也可使用 `ModifyNodePoolDesiredCapacityAboutAsg`。

```bash
# 发现链路：获取集群 ID → 获取节点池 ID
tccli tke DescribeClusters --region ap-guangzhou
# expected: 从 TotalCount 和 Clusters[] 获取目标集群的 ClusterId

tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou
# expected: 从 NodePoolSet[].NodePoolId 获取目标节点池 ID
```

> **注意：** 调整节点数（`MaxNodesNum` 等）需要节点池已开启弹性伸缩（`AutoscalingGroupStatus: "enabled"`），否则仅参数修改不触发实际扩缩容操作。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，具备 TKE 读写权限
- 已存在运行中的集群和目标节点池（`LifeState: "normal"`）

### 环境检查

```bash
# 检查 tccli 版本
tccli --version
# expected: tccli version ≥3.1.x

# 确认 TKE 服务可用
tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表
```

> **CAM 权限要求**：当前账号需具备 `tke:DescribeClusterNodePools`（读）和 `tke:ModifyClusterNodePool`（写）权限。如遇 `UnauthorizedOperation`，联系 CAM 管理员授予 QcloudTKEFullAccess 或对应自定义策略。

### 获取当前节点池配置

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region ap-guangzhou
# expected: exit 0，LifeState 为 normal
```

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "<NodePoolId>",
            "Name": "<Name>",
            "ClusterInstanceId": "<ClusterId>",
            "LifeState": "normal",
            "MaxNodesNum": 1,
            "MinNodesNum": 1,
            "DesiredNodesNum": 1,
            "Labels": [
                {"Name": "env", "Value": "staging"}
            ],
            "Taints": [
                {"Key": "dedicated", "Value": "gpu", "Effect": "NoSchedule"}
            ],
            "DeletionProtection": false,
            "AutoscalingGroupStatus": "enabled",
            "LaunchConfigurationId": "<LaunchConfigurationId>",
            "AutoscalingGroupId": "<AutoscalingGroupId>"
        }
    ],
    "TotalCount": 1,
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

> **输出到输入映射**：`ClusterId` 来自 `$.Clusters[].ClusterId`；`NodePoolId` 来自 `$.NodePoolSet[0].NodePoolId`。将上述两个值记为后续命令的 `<ClusterId>` 和 `<NodePoolId>`。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看节点池配置 | `tccli tke DescribeClusterNodePools` | 是 |
| 调整节点数量 | `tccli tke ModifyClusterNodePool` | 否 |
| 仅调整期望节点数 | `tccli tke ModifyNodePoolDesiredCapacityAboutAsg` | 否 |
| 修改 Label/Taint | `tccli tke ModifyClusterNodePool` | 否 |
| 开启/关闭删除保护 | `tccli tke ModifyClusterNodePool --DeletionProtection` | 是 |
| 开启弹性伸缩 | 控制台 TKE → 节点池 → 弹性伸缩（无直接 CLI） | — |

## 操作步骤

### 1. 查看当前配置

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region ap-guangzhou
```

记录当前 `MaxNodesNum`、`MinNodesNum`、`DesiredNodesNum`、`Labels`、`Taints`、`DeletionProtection` 的值，作为调整基线。

### 2. 关键字段说明（`ModifyClusterNodePool`）

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|------|-----------|---------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在返回 `ResourceNotFound` |
| `NodePoolId` | String | 是 | 节点池 ID，格式 `np-xxxxxxxx` | 节点池不存在返回 `ResourceNotFound` |
| `MaxNodesNum` | Integer | 否 | 最大节点数，需 >= MinNodesNum | 小于 MinNodesNum 返回 `InvalidParameterValue` |
| `MinNodesNum` | Integer | 否 | 最小节点数，需 <= MaxNodesNum | 大于 MaxNodesNum 返回 `InvalidParameterValue` |
| `DesiredNodesNum` | Integer | 否 | 期望节点数，需在 MinNodesNum~MaxNodesNum 区间内 | 超出范围返回 `InvalidParameterValue.Size` |
| `Name` | String | 否 | 节点池名称，不传则不变 | 重名返回错误 |
| `Labels` | Array | 否 | 节点标签列表，传入则覆盖原有 Labels（不合并） | 格式错误返回 `InvalidParameterValue` |
| `Taints` | Array | 否 | 节点污点列表，传入则覆盖原有 Taints（不合并） | Effect 值无效返回 `InvalidParameterValue` |
| `Annotations` | Array | 否 | 节点注解列表，传入则覆盖原有 Annotations | — |
| `DeletionProtection` | Boolean | 否 | 是否开启删除保护。开启后需先关闭才能删除节点池 | — |
| `OsCustomizeType` | String | 否 | 操作系统定制类型 | 无效值返回错误 |

> **重要约束（`DesiredNodesNum`）**：API 先应用新 `MinNodesNum`/`MaxNodesNum`，再用**当前** `DesiredNodesNum`（而非请求值）校验是否在新区间内。若当前 `DesiredNodesNum` 不满足新阈值，需分两步操作：
> 1. 先将 `MinNodesNum` 设为 ≤ 当前 `DesiredNodesNum` 的值（如保持原值），仅扩大 `MaxNodesNum`
> 2. 再设置最终目标值（含 `DesiredNodesNum`）
>
> 示例：当前 `DesiredNodesNum=1`，目标 `MinNodesNum=2, MaxNodesNum=5, DesiredNodesNum=3`：
> ```bash
> # 第一步：扩大上限，保持 MinNodesNum ≤ 当前 DesiredNodesNum
> tccli tke ModifyClusterNodePool --cli-input-json file://modify-nodepool-scale-step1.json --region ap-guangzhou
> # 第二步：设置最终目标值（此时 MinNodesNum=2 内已含 DesiredNodesNum=3）
> tccli tke ModifyClusterNodePool --cli-input-json file://modify-nodepool-scale-step2.json --region ap-guangzhou
> ```

> **重要：** `Labels` 和 `Taints` 传入后会**覆盖**（非合并）原有配置。修改时需传入完整的期望列表。

### 3. 调整节点规模

调整节点池的节点数量范围。`modify-nodepool-scale.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "MaxNodesNum": 5,
    "MinNodesNum": 2,
    "DesiredNodesNum": 3
}
```

> **选择依据**：使用 `ModifyClusterNodePool` 而非 `ModifyNodePoolDesiredCapacityAboutAsg`，因为此场景需要同时修改节点数上下限和期望数。仅需调整期望节点数而不改阈值时，跳至 [步骤 3a](#3a-仅调整期望节点数modifynodepooldesiredcapacityaboutasg)。

执行修改：

```bash
tccli tke ModifyClusterNodePool \
    --cli-input-json file://modify-nodepool-scale.json \
    --region ap-guangzhou
```

参考输出：

```json
{
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

> **注意：** 只有在弹性伸缩已启用（`AutoscalingGroupStatus: "enabled"`）时，调整 `DesiredNodesNum` 才会触发实际的扩缩容操作。若弹性伸缩未启用，需先在控制台 [节点池 → 弹性伸缩](https://console.cloud.tencent.com/tke2/nodepool) 中开启。

### 3a. 仅调整期望节点数（ModifyNodePoolDesiredCapacityAboutAsg）

如果只需要调整期望节点数而不修改其他配置，可使用更轻量的 API：

```bash
tccli tke ModifyNodePoolDesiredCapacityAboutAsg \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --DesiredCapacity 3 \
    --region ap-guangzhou
```

> 此 API 直接操作弹性伸缩组的期望容量，不修改节点池的其他配置参数。

### 4. 调整 Labels 和 Taints

> **副作用警告：** 修改 Taints 可能导致已有 Pod 因不满足新的 Taint 而被调度器驱逐。若新增 `NoSchedule` 或 `NoExecute` Taint，已运行的 Pod 若无对应 Toleration 将无法继续在该节点运行。修改前请确认当前 Pod 的 Toleration 配置。

`modify-nodepool-labels.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "Labels": [
        {"Name": "env", "Value": "staging"},
        {"Name": "team", "Value": "platform"}
    ],
    "Taints": [
        {"Key": "dedicated", "Value": "gpu", "Effect": "NoSchedule"}
    ]
}
```

执行修改：

```bash
tccli tke ModifyClusterNodePool \
    --cli-input-json file://modify-nodepool-labels.json \
    --region ap-guangzhou
```

> **注意：** 此操作会**覆盖**节点池现有的所有 Labels 和 Taints。新 Labels/Taints 仅影响后续加入节点池的新节点，已有节点需手动更新（通过 `kubectl label` / `kubectl taint`）。

### 5. 开启删除保护

> **风险警告：** 开启删除保护后，无法通过控制台或 CLI 直接删除节点池。如需删除，必须先关闭删除保护。请确保管理员知晓此限制，避免误操作导致节点池无法清理。

`modify-nodepool-protection.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "DeletionProtection": true
}
```

```bash
tccli tke ModifyClusterNodePool \
    --cli-input-json file://modify-nodepool-protection.json \
    --region ap-guangzhou
```

> **选择依据**：使用 `--cli-input-json` 而非 `--DeletionProtection` CLI flag。实跑验证发现 `--DeletionProtection` 布尔 flag 在此 API 上行为不稳定（时而成功，时而 `OperationDenied`），而 `--cli-input-json` 方式始终可靠。
>
> **并发更新注意**：节点池在 `LifeState: "updating"` 期间，任何 `ModifyClusterNodePool` 调用（包括删除保护切换）都将返回 `OperationDenied`。请先确认节点池处于 `LifeState: "normal"` 再执行修改操作。

## 验证

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region ap-guangzhou
```

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "<NodePoolId>",
            "Name": "<Name>",
            "ClusterInstanceId": "<ClusterId>",
            "LifeState": "normal",
            "MaxNodesNum": 5,
            "MinNodesNum": 2,
            "DesiredNodesNum": 3,
            "Labels": [
                {"Name": "env", "Value": "staging"},
                {"Name": "team", "Value": "platform"}
            ],
            "Taints": [
                {"Key": "dedicated", "Value": "gpu", "Effect": "NoSchedule"}
            ],
            "DeletionProtection": true,
            "AutoscalingGroupStatus": "enabled",
            "LaunchConfigurationId": "<LaunchConfigurationId>"
        }
    ],
    "TotalCount": 1,
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点数 | `MaxNodesNum` / `MinNodesNum` / `DesiredNodesNum` | 与修改后的值一致 |
| Labels | `Labels` | 与修改后的列表一致（覆盖） |
| Taints | `Taints` | 与修改后的列表一致（覆盖） |
| 删除保护 | `DeletionProtection` | `true` 或 `false`（与操作一致） |
| 弹性伸缩 | `AutoscalingGroupStatus` | 确认伸缩组状态正常 |

## 清理

若不再需要本次调整的配置，可通过以下命令恢复：

### 恢复 Labels/Taints 为空

```bash
# 恢复 Labels 和 Taints 为空（modify-nodepool-clear.json）
# {
#     "ClusterId": "<ClusterId>",
#     "NodePoolId": "<NodePoolId>",
#     "Labels": [],
#     "Taints": []
# }
tccli tke ModifyClusterNodePool \
    --cli-input-json file://modify-nodepool-clear.json \
    --region ap-guangzhou
```

### 恢复节点数阈值

```bash
# 恢复为最小值（如单节点配置），modify-nodepool-scale-minimal.json
# {
#     "ClusterId": "<ClusterId>",
#     "NodePoolId": "<NodePoolId>",
#     "MaxNodesNum": 1,
#     "MinNodesNum": 1
# }
tccli tke ModifyClusterNodePool \
    --cli-input-json file://modify-nodepool-scale-minimal.json \
    --region ap-guangzhou
```

### 关闭删除保护

若开启了删除保护，后续删除节点池前需先关闭。`modify-nodepool-protection-off.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "DeletionProtection": false
}
```

```bash
tccli tke ModifyClusterNodePool \
    --cli-input-json file://modify-nodepool-protection-off.json \
    --region ap-guangzhou
```

> 调整操作本身无计费资源残留；仅当 `DesiredNodesNum` 增加且弹性伸缩触发实际节点创建时，会产生 CVM 计费。此部分节点为节点池管理的持久资源，不属于本文档的临时验证资源，需在节点管理流程中统一清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `--NodePoolIds is not a valid parameter` | `tccli tke DescribeClusterNodePools --generate-cli-skeleton` 确认参数列表 | `DescribeClusterNodePools` API 不支持 `--NodePoolIds` 参数，仅支持 `ClusterId` + `Filters` | 使用 `--ClusterId` 查询全部节点池，通过 `jq '.NodePoolSet[] \| select(.NodePoolId=="<target>")'` 客户端侧过滤 |
| 修改节点数返回 `InvalidParameterValue.Size` | 检查 `当前 DesiredNodesNum` vs `新 MinNodesNum` | API 先用新 MinNodesNum 校验当前 DesiredNodesNum，当前值小于新最小值时报错 | 采用两步法：①先设 MinNodesNum ≤ 当前 DesiredNodesNum + 扩大 MaxNodesNum；②再设最终目标值（参见关键字段说明） |
| 修改节点数不生效 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId>` 检查 `AutoscalingGroupStatus` | 弹性伸缩未启用 | 在控制台开启弹性伸缩后重试 |
| `Labels` 修改后原有标签丢失 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId>` 检查 Labels 列表 | `ModifyClusterNodePool` 的 Labels 参数为覆盖语义 | 修改时传入完整的 Labels 列表（含要保留的旧标签） |
| 返回 `InvalidParameterValue` — `MaxNodesNum` | 检查 MaxNodesNum >= MinNodesNum | MaxNodesNum 小于 MinNodesNum | 确保 MaxNodesNum >= MinNodesNum |
| 返回 `ResourceNotFound` — NodePool | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId>` 检查节点池是否存在 | NodePoolId 错误或节点池已被删除 | 确认 NodePoolId 正确 |
| `--DeletionProtection` CLI flag 返回 `OperationDenied` | 使用 `--cli-input-json file://...` 方式重试 | tccli 对 `ModifyClusterNodePool` 的布尔参数 CLI flag 行为不稳定（时而成功时而失败）；同时确认节点池 `LifeState` 非 `updating` | 改用 `--cli-input-json file://...json` 方式传入 `DeletionProtection` 值 |
| `ModifyClusterNodePool` 返回 `OperationDenied`（非 DeletionProtection 参数） | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId>` 检查 `LifeState` | 节点池正处于 `updating` 状态（上一步修改尚未完成），拒绝并发修改 | 等待节点池 `LifeState` 恢复 `normal` 后重试 |
| 删除保护开启后仍可 CLI 删除 | — | CLI 删除命令（`DeleteClusterNodePool`）不检查 `DeletionProtection` 标志 | 删除前通过 `DescribeClusterNodePools` 确认 DeletionProtection 状态 |
| `ModifyNodePoolDesiredCapacityAboutAsg` 返回错误 | `tccli tke DescribeClusterAsGroups --ClusterId <ClusterId>` 确认伸缩组状态 | 伸缩组状态异常或未启用 | 确保弹性伸缩已启用，DesiredCapacity 在 MinSize-MaxSize 范围内 |
| Taint 修改后 Pod 被驱逐 | `kubectl describe node <node-name> \| grep Taints` + `kubectl get pods -o wide` 确认受影响 Pod | 新增 NoSchedule/NoExecute Taint，Pod 无对应 Toleration | 为受影响 Pod 添加 Toleration，或先撤 Taint 再逐步迁移 |

## 下一步

- [查看节点池](https://cloud.tencent.com/document/product/457/43736) — 查看调整后的节点池配置
- [删除节点池](https://cloud.tencent.com/document/product/457/43738) — 删除不再需要的节点池
- [创建节点池](https://cloud.tencent.com/document/product/457/43735) — 创建新的节点池

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 节点池 → 编辑](https://console.cloud.tencent.com/tke2/nodepool) — 通过控制台调整节点池配置。
