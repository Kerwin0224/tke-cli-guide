---
doc_type: How-to
subtype: 6A
fused: true
---
# 配置集群属性与运行时

> 修改已创建集群的属性（名称/描述/等级/标签/镜像）、组件额外参数、容器运行时、Master 组件。写操作，有副作用。

> 本文档 Action 均属 **TKE 2018-05-25**。只读查询见 [查询集群](query.md)。

## 概述

集群创建后，常需调整配置：升等级扩容配额、改运行时版本、调组件参数、绑 admin 角色。这些是**写操作**，改集群行为或计费，与 [查询集群](query.md)（只读）严格分离。

| 任务 | 接口 | 副作用 |
|:-----|:-----|:-----|
| 改名称/描述/等级 | `ModifyClusterAttribute` | 等级变更影响计费 |
| 改标签 | `ModifyClusterTags` | 可同步子资源 |
| 改镜像 | `ModifyClusterImage` | 滚动重建节点 |
| 改组件参数 | `ModifyClusterExtraArgs` | 覆盖式，影响控制面 |
| 扩容器网段 | `AddClusterCIDR` | 增加可用 Pod IP |
| 绑 admin 角色 | `AcquireClusterAdminRole` | RBAC 授权 |
| 改运行时 | `ModifyClusterRuntimeConfig` | 滚动重建节点 |
| 改 Master 组件 | `ModifyMasterComponent` | 启停控制面组件 |

## 决策依据

### 改集群等级前先查价

等级变更（L5→L20）会提高配额上限（节点/Pod/CRD）但也提高计费。决策前先查可用等级与价格。真实等级枚举（实测 `DescribeClusterLevelAttribute`）：`L5`/`L20`/`L50`/`L100`/`L200`/`L500`/`L1000`/`L3000`/`L5000`——**无 L10**，`Enable=false` 的 L1000 及以上需工单开通：

```bash
# 查可用等级 (含节点/Pod/CRD 上限)
tccli tke DescribeClusterLevelAttribute --ClusterID "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, Items[] 含 Alias/NodeCount/PodCount/Enable
```
```json
{"TotalCount": 9, "Items": [{"Alias": "L5", "NodeCount": 5, "PodCount": 150, "Enable": true}]}
```

> ⚠️ `DescribeClusterLevelAttribute`/`DescribeClusterLevelChangeRecords` 用 `--ClusterID`（大写 ID），与其他集群接口的 `--ClusterId`（小写 d）不同——大小写写错报 `Unknown options`。

```bash
# 查等级价格 (不绑集群, 按 ClusterLevel; 用真实枚举 L5/L20/L50/L100/L200/L500)
tccli tke GetClusterLevelPrice --ClusterLevel L20 --region <REGION>
# expected: exit 0, 返回 Cost/TotalCost/Policy（实测 L5→13/L20→37/L50→47/L100→83）
```

#### 为什么选这个等级

- **L5 (5 节点)**: 测试/小项目，配额够用，费用低
- **L20+ (20 节点以上)**: 生产，需更多 Pod/CRD 配额（实测无 L10，最小生产档是 L20）
- **自动升级**: `AutoUpgradeClusterLevel.IsAutoUpgrade=true` 让系统在配额将满时自动升级（谨慎，有计费风险）
- **决策依据**: 当前节点数 + Pod 数接近上限才升级；不接近则保持，避免无谓计费

### 改运行时前先查支持版本

运行时变更（docker→containerd）会滚动重建节点，业务短暂中断。决策前查目标版本是否支持：

```bash
# 查支持的运行时 (按 K8s 版本, 不绑集群；注意入参是 K8sVersion 非 ClusterVersion)
tccli tke DescribeSupportedRuntime --K8sVersion "<VERSION>" --region <REGION>
# expected: exit 0, OptionalRuntimes[] 含 RuntimeType/RuntimeVersions/DefaultVersion（containerd 默认 1.6.9，docker 默认 20.10）
```
```json
{"OptionalRuntimes": [{"RuntimeType": "containerd", "RuntimeVersions": ["1.6.9", "1.7.28"], "DefaultVersion": "1.6.9"}]}
```

#### 为什么选 containerd 不选 docker

- **containerd（推荐）**: K8s 官方默认，资源占用低，1.24+ 版本 docker 已废弃
- **docker**: 老集群兼容，但 1.24+ 不再支持
- **版本选择**: 优先 `DefaultVersion`（经过验证），仅在特定需求时选其他版本
- **能回退吗**: 不能直接回退，需重新变更（再次滚动重建）

### 改 ExtraArgs 前先备份当前值

`ModifyClusterExtraArgs` 是**覆盖式**更新——传空数组会清空原参数。决策前必须备份：

```bash
# 备份当前组件参数 (Etcd/KubeAPIServer/KubeControllerManager/KubeScheduler)
tccli tke DescribeClusterExtraArgs --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, ClusterExtraArgs 四组件参数

# 查可用额外参数 (按版本+类型, 不绑集群, 确认目标参数存在)
tccli tke DescribeClusterAvailableExtraArgs --ClusterVersion "<VERSION>" --ClusterType MANAGED_CLUSTER --region <REGION>
# expected: exit 0, AvailableExtraArgs 含 Name/Type/Usage/Constraint
```

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `ClusterId` | 多数 | 是 | 集群 ID（注意 Level 系列用 `ClusterID` 大写） |
| `ClusterLevel` | ModifyClusterAttribute | 否 | 真实枚举 L5/L20/L50/L100/L200/L500（无 L10），影响计费 |
| `Tags[]` | ModifyClusterTags | 是 | Key/Value 对 |
| `SyncSubresource` | ModifyClusterTags | 否 | true 同步标签到子资源 |
| `ImageId` | ModifyClusterImage | 是 | 目标镜像 ID |
| `ClusterExtraArgs` | ModifyClusterExtraArgs | 是 | 嵌套四组件参数（覆盖式） |
| `Operation` | ModifyClusterExtraArgsTaskState | 是 | 任务状态操作 |
| `ClusterCIDRs[]` | AddClusterCIDR | 是 | 新增容器网段 CIDR |
| `DstK8SVersion` | ModifyClusterRuntimeConfig | 否 | 目标 K8s 版本 |
| `ClusterRuntimeConfig` | ModifyClusterRuntimeConfig | 是 | RuntimeType/RuntimeVersion |
| `Component`/`Operation`/`DryRun` | ModifyMasterComponent | 是 | 组件名(kube-apiserver/kube-scheduler/kube-controller-manager)/停机或恢复(shutdown/restore)/试运行 |

> 参数名实测自各 Action `--generate-cli-skeleton`。

## 操作步骤

### 步骤 1：修改集群属性（名称/描述/等级）

```bash
tccli tke ModifyClusterAttribute --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --ClusterName "<NEW_NAME>" --ClusterDesc "<DESC>" --ClusterLevel L20
# expected: exit 0
```

> `ClusterLevel` 变更触发计费调整（P8）。`AutoUpgradeClusterLevel.IsAutoUpgrade` 控制自动升级。

### 步骤 2：修改集群标签

```bash
tccli tke ModifyClusterTags --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --Tags '[{"Key":"env","Value":"prod"}]' --SyncSubresource true
# expected: exit 0
```

> `SyncSubresource=true` 同步标签到集群下节点等子资源。

### 步骤 3：修改集群镜像

```bash
tccli tke ModifyClusterImage --ClusterId "<CLUSTER_ID>" --region <REGION> --ImageId "<IMAGE_ID>"
# expected: exit 0
```

> 镜像变更会滚动重建节点。`ImageId` 用 `DescribeOSImages` 查（无入参，返回全部可用 OS 镜像）：

```bash
# 查询可用 OS 镜像（无入参，返回 ImageId/OSName 列表）
tccli tke DescribeOSImages --version 2018-05-25 --region <REGION>
# expected: exit 0, ImageSet[] 含 ImageId/OSName/Status
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<IMAGE_ID>` | OS 镜像 ID | `DescribeOSImages` → `ImageSet[].ImageId` |

### 步骤 4：修改组件额外参数（覆盖式）

```bash
# 先备份 (见决策依据), 再覆盖
tccli tke ModifyClusterExtraArgs --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --ClusterExtraArgs '{"KubeAPIServer":["--feature-gates=XXX=true"]}'
# expected: exit 0

# 控制额外参数任务状态
tccli tke ModifyClusterExtraArgsTaskState --ClusterId "<CLUSTER_ID>" --region <REGION> --Operation "<OPERATION>"
# expected: exit 0
```

> ⚠️ 覆盖式更新——必须含原有要保留的参数，否则丢失。修改后用 `DescribeClusterExtraArgs` 复核。

### 步骤 5：扩容容器网段

```bash
tccli tke AddClusterCIDR --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --ClusterCIDRs '["10.244.0.0/16"]' --IgnoreClusterCIDRConflict false
# expected: exit 0
```

> `IgnoreClusterCIDRConflict=true` 强制添加即使与已有网段冲突。Pod IP 不足时扩容。

### 步骤 6：获取集群 admin 角色（RBAC 授权前置）

```bash
tccli tke AcquireClusterAdminRole --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, RequestId
```

> 为当前账号获取集群 admin RBAC 角色，是 kubectl 操作集群的授权前置。

### 步骤 7：修改容器运行时

```bash
tccli tke ModifyClusterRuntimeConfig --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --DstK8SVersion "<VERSION>" \
  --ClusterRuntimeConfig '{"RuntimeType":"containerd","RuntimeVersion":"1.7.28"}'
# expected: exit 0
```

> `NodePoolRuntimeConfig[]` 可按节点池分别配置运行时。运行时变更滚动重建节点。

### 步骤 8：修改 Master 组件

```bash
# 先查组件状态（Component 枚举: kube-apiserver / kube-scheduler / kube-controller-manager）
tccli tke DescribeMasterComponent --ClusterId "<CLUSTER_ID>" --Component "kube-apiserver" --region <REGION>
# expected: exit 0, Status=Running（返回 Component/Status/RequestId）

# 修改 (DryRun=true 先试运行, false 实际变更)
tccli tke ModifyMasterComponent --ClusterId "<CLUSTER_ID>" --Component "kube-apiserver" --Operation "shutdown" --DryRun true --region <REGION>
# expected: exit 0（Operation 枚举: shutdown 停机 / restore 恢复）
```

> `Operation` 枚举为 `shutdown`（停机）/`restore`（恢复），非 enable/disable。`DryRun=true` 仅验证不实际变更，生产操作前先用 DryRun。

## 验证

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 等级已变 | `DescribeClusterLevelAttribute --ClusterID <CLUSTER_ID>` | 目标等级 `Enable=true` |
| 等级变更记录 | `DescribeClusterLevelChangeRecords --ClusterID <CLUSTER_ID>` | 含本次变更 |
| 标签已生效 | `DescribeClusters --ClusterIds '["<CLUSTER_ID>"]'` | Tags 含新标签 |
| 额外参数已变 | `DescribeClusterExtraArgs --ClusterId <CLUSTER_ID>` | 含新参数 |
| 运行时已变 | `DescribeClusters` → `ClusterVersion`/运行时字段 | 目标版本 |
| Master 组件状态 | `DescribeMasterComponent` | 目标 Operation 生效 |
| 集群仍 Running | `DescribeClusterStatus` | ClusterState=Running |

> 修改后必须确认集群仍 `Running`——配置变更可能触发滚动重建，期间集群短暂异常。

## 清理

> 配置变更多为不可逆或需再次变更回退。无独立清理段，回退方式：
> - 等级: 再次 `ModifyClusterAttribute --ClusterLevel <原等级>`（计费调整）
> - 标签: `ModifyClusterTags` 传空或原标签
> - 额外参数: 用备份的 `ClusterExtraArgs` 覆盖回原值
> - 运行时: 不能直接回退，需再次 `ModifyClusterRuntimeConfig`

## 副作用

- **ModifyClusterImage/ModifyClusterRuntimeConfig**: 滚动重建节点，业务 Pod 重调度，短暂中断（P8）
- **ModifyClusterAttribute ClusterLevel**: 计费等级调整，影响月费（P8）
- **ModifyClusterExtraArgs**: 覆盖式，误传空数组清空参数，可能导致控制面异常
- **AcquireClusterAdminRole**: 授予集群 admin 权限，安全敏感（P8）
- **ModifyMasterComponent**: 启停控制面组件，影响集群管理能力

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `Unknown options: --ClusterID` | 核对接口 | Level 系列用大写 `ClusterID`，其他用 `ClusterId` | 按接口要求用正确大小写 |
| `InvalidParameterValue` (ClusterLevel) | `DescribeClusterLevelAttribute` | 等级不存在或不可用 | 用 `Enable=true` 的等级 |
| `FailedOperation.TradeCommon` | GetClusterLevelPrice | `ClusterLevel` 值非真实枚举（实测传 `L10` 稳定触发，因 L10 不存在 → 计费中心 "all price code not match"） | 改用真实等级 `L5`/`L20`/`L50`/`L100`/`L200`/`L500`（`DescribeClusterLevelAttribute` 返回的 `Enable=true` 项）；非临时错误，重试无效 |
| `ResourceNotFound` (ImageId) | `DescribeOSImages` | 镜像 ID 不存在 | 用真实镜像 ID |
| `UnsupportedOperation` | `DescribeClusterStatus` | 集群非 Running | 等集群 Running 后重试 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 等级变更后配额未变 | `DescribeClusterLevelAttribute` 复核 | 异步未完成 | 等待后重查 |
| ExtraArgs 修改后参数丢失 | `DescribeClusterExtraArgs` 对比备份 | 覆盖式未保留原参数 | 用备份值重新覆盖 |
| 运行时变更后节点 NotReady | `DescribeClusterInstances` | 滚动重建未完成 | 等待滚动完成 |
| 集群长时间非 Running | `DescribeClusterStatus` → `ClusterInstanceState` | 配置变更触发异常 | 查节点状态，必要时回退 |

## 下一步

- [查询集群](query.md) — 只读查询集群状态与配置
- [创建集群](create.md) — 新建集群（配置的前置）
- [升级集群版本](upgrade.md) — K8s 版本升级
- [认证配置](../security/auth.md) — AcquireClusterAdminRole 后的 RBAC 配置
- [故障排查](../troubleshooting.md) — 配置变更后集群异常诊断

## Action 清单

| Action | 类型 | 说明 |
|:-------|:-----|:-----|
| `DescribeClusterLevelAttribute` | 验证 | 查可用等级（`ClusterID` 大写） |
| `DescribeClusterLevelChangeRecords` | 验证 | 查等级变更记录（`ClusterID` 大写） |
| `GetClusterLevelPrice` | 验证 | 查等级价格（不绑集群） |
| `DescribeClusterExtraArgs` | 验证 | 查当前组件参数（备份用） |
| `DescribeClusterAvailableExtraArgs` | 验证 | 查可用额外参数（不绑集群） |
| `DescribeSupportedRuntime` | 验证 | 查支持运行时（不绑集群） |
| `DescribeMasterComponent` | 验证 | 查 Master 组件状态 |
| `ModifyClusterAttribute` | 主操作 | 改名称/描述/等级/自动升级 |
| `ModifyClusterTags` | 主操作 | 改标签（SyncSubresource） |
| `ModifyClusterImage` | 主操作 | 改镜像（滚动重建） |
| `ModifyClusterExtraArgs` | 主操作 | 改组件参数（覆盖式） |
| `ModifyClusterExtraArgsTaskState` | 主操作 | 控制参数任务状态 |
| `AddClusterCIDR` | 主操作 | 扩容器网段 |
| `AcquireClusterAdminRole` | 主操作 | 获取 admin RBAC 角色 |
| `ModifyClusterRuntimeConfig` | 主操作 | 改运行时/K8s 版本（滚动重建） |
| `ModifyMasterComponent` | 主操作 | 启停 Master 组件（DryRun 可试运行） |
