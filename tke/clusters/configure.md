---
doc_type: How-to
subtype: 6A
fused: true
---
# 配置集群属性与运行时

> 修改已创建集群的属性（名称/描述/等级/标签/镜像）、组件额外参数、容器运行时、Master 组件。写操作，有副作用。

> 本文档 Action 均属 **TKE 2018-05-25**（旧版独有，新版无对应 Action）。只读查询见 [查询集群](query.md)。注意 `DescribeClusterLevelAttribute`/`DescribeClusterLevelChangeRecords` 用 `--ClusterID`（大写 ID），与其他集群接口的 `--ClusterId`（小写 d）不同。

> 官方文档：[基本概念](https://cloud.tencent.com/document/product/457/45598) · [集群生命周期](https://cloud.tencent.com/document/product/457/32188)

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

> 配额：无额外配额限制，但需验证当前集群规格支持目标配置（如等级配额）。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- 集群节点/Pod/CRD 接近配额上限（`DescribeClusterLevelAttribute` 显示余量不足）— 用 `ModifyClusterAttribute` 升等级
- 需切换容器运行时（docker→containerd）或调整 Master 组件参数（ExtraArgs/启停组件）— 用本文对应接口
- GR 网络集群 Pod IP 将耗尽 — 用 `AddClusterCIDR` 扩容器网段（仅 GR 集群支持）

## 准备工作

- 已创建 TKE 集群 (见 [创建集群](create.md)) 且状态 Running
- 已配置 tccli 凭证 (见 [配置凭证](../../getting-started/credentials.md))


## 决策依据

### 改集群等级前先查价

等级变更（L5→L20）会提高配额上限（节点/Pod/CRD）但也提高计费。决策前先查可用等级与价格。真实等级枚举（`DescribeClusterLevelAttribute` 返回）：`L5`/`L20`/`L50`/`L100`/`L200`/`L500`/`L1000`/`L3000`/`L5000`——**无 L10**，`Enable=false` 的 L1000 及以上需工单开通：

```bash
# 查可用等级 (含节点/Pod/CRD 上限)
tccli tke DescribeClusterLevelAttribute --ClusterID "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, Items[] 含 Alias/NodeCount/PodCount/ConfigMapCount/RSCount/CRDCount/OtherCount/Enable
```
```json
{"TotalCount": 9, "Items": [{"Name": "5 nodes", "Alias": "L5", "NodeCount": 5, "PodCount": 150, "ConfigMapCount": 128, "RSCount": 900, "CRDCount": 150, "OtherCount": 150, "Enable": true}]}
```

> `--language en-US` 时 `Name` 为 `"5 nodes"`；`zh-CN` 时为 `"5节点"`。决策用 `Alias`（`L5`/`L20`/…）与 `Enable`。

> ⚠️ `DescribeClusterLevelAttribute`/`DescribeClusterLevelChangeRecords` 用 `--ClusterID`（大写 ID），与其他集群接口的 `--ClusterId`（小写 d）不同——大小写写错报 `Unknown options`。

```bash
# 查等级价格 (不绑集群, 按 ClusterLevel; 询价枚举 L5/L20/L50/L100/L200/L500/L1000/L3000/L5000——传不存在的等级如 L10 触发 FailedOperation.TradeCommon)
tccli tke GetClusterLevelPrice --ClusterLevel L20 --region <REGION>
# expected: exit 0, 返回 Cost/TotalCost/Policy（价目随计费策略与地域变，以实际返回为准）
```

#### 为什么选这个等级

- **L5 (5 节点)**: 测试/小项目，配额够用，费用低
- **L20+ (20 节点以上)**: 生产，需更多 Pod/CRD 配额（无 L10，最小生产档是 L20）
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
- **版本选择**: 优先 `DefaultVersion`（接口返回的默认推荐版本），仅在特定需求时选其他版本
- **能回退吗**: 不能直接回退，需重新变更（再次滚动重建）

### 改 ExtraArgs 前先备份当前值

`ModifyClusterExtraArgs` 是**覆盖式**更新——传空数组会清空原参数。决策前必须备份。ExtraArgs 属 `InstanceAdvancedSettings` 嵌套结构，见 [共享字段](../reference/shared-fields.md#instanceadvancedsettings-节点高级设置)：

```bash
# 备份当前组件参数 (Etcd/KubeAPIServer/KubeControllerManager/KubeScheduler)
tccli tke DescribeClusterExtraArgs --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, ClusterExtraArgs 四组件参数

# 查可用额外参数 (按版本+类型, 不绑集群, 确认目标参数存在)
tccli tke DescribeClusterAvailableExtraArgs --ClusterVersion "<VERSION>" --ClusterType MANAGED_CLUSTER --region <REGION>
# expected: exit 0, AvailableExtraArgs 按组件嵌套（KubeAPIServer/KubeControllerManager/…），项含 Name/Type/Usage/Default/Constraint；另回 ClusterVersion/ClusterType
```

| 字段 | 类型 | 必填 | 说明 |
|:-----|:-----|:----:|:-----|
| `ClusterVersion` | string | 是 | 要查询可用控制面参数的 Kubernetes 版本 |
| `ClusterType` | string | 是 | 集群类型 |

## 关键字段

> 下表"必填"列按业务必需标注（不传则操作无意义或失败）；API 层 `required` 可能不同（如 `ModifyClusterTags` 的 `Tags` 与 `ModifyClusterRuntimeConfig` 的 `ClusterRuntimeConfig` 在 API 层均为选填，但业务上必须传）。

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `ClusterId` | 多数 | 是 | 集群 ID（注意 Level 系列用 `ClusterID` 大写） |
| `ClusterLevel` | ModifyClusterAttribute | 否 | 真实枚举 L5/L20/L50/L100/L200/L500（无 L10），影响计费 |
| `Tags[]` | ModifyClusterTags | 是 | Key/Value 对（API 层选填，业务必需——覆盖式更新，不传会清空标签） |
| `SyncSubresource` | ModifyClusterTags | 否 | true 同步标签到子资源 |
| `SyncNodePoolTags` | ModifyClusterTags | 否 | true 同步标签到节点池，仅当 `SyncSubresource=true` 时生效 |
| `ImageId` | ModifyClusterImage | 是 | 目标镜像 ID |
| `ClusterExtraArgs` | ModifyClusterExtraArgs | 否 | 嵌套四组件参数（覆盖式；API 非必填，但传空会清空原参数，业务上必须传完整值） |
| `Operation` | ModifyClusterExtraArgsTaskState | 否 | 任务状态操作，枚举 `abort`（取消并回退任务） |
| `ClusterCIDRs[]` | AddClusterCIDR | 是 | 新增容器网段 CIDR |
| `DstK8SVersion` | ModifyClusterRuntimeConfig | 否 | 目标 K8s 版本 |
| `ClusterRuntimeConfig` | ModifyClusterRuntimeConfig | 否 | RuntimeType/RuntimeVersion；与 `NodePoolRuntimeConfig` 分别选择修改目标，两者均为 API 选填 |
| `Component` | ModifyMasterComponent | 是 | 组件名（kube-apiserver/kube-scheduler/kube-controller-manager） |
| `Operation` | ModifyMasterComponent | 是 | 停机或恢复（shutdown/restore） |
| `DryRun` | ModifyMasterComponent | 否 | `true` 仅验证不实际变更，生产操作前先用 DryRun 试运行 |
| `SubAccounts` | UpdateClusterKubeconfig | 否 | 子账户 Uin 列表，不传默认为调用者本人 |

## 操作步骤

> ⚠️ **高危操作**：`AddClusterCIDR` 扩容不可逆（扩后无法缩小）；IPVS 开启后不可关闭；`ModifyClusterImage` 滚动重建全集群节点影响全局。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

### 步骤 1：修改集群属性（名称/描述/等级）

```bash
tccli tke ModifyClusterAttribute --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --ClusterName "<NEW_NAME>" --ClusterDesc "<DESC>" --ClusterLevel L20
# expected: exit 0
```

> `ClusterLevel` 变更触发计费调整。`AutoUpgradeClusterLevel.IsAutoUpgrade` 控制自动升级。

### 步骤 2：修改集群标签

```bash
tccli tke ModifyClusterTags --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --Tags '[{"Key":"billing","Value":"<TAG_VALUE>"},{"Key":"env","Value":"prod"}]' --SyncSubresource true
# expected: exit 0；异步任务用 DescribeBatchModifyTagsStatus 等到 Status=done 后再做后续写操作
# expected: exit 0
```

> `SyncSubresource=true` 同步标签到集群下节点等子资源。
>
> ⚠️ **覆盖式更新——遗漏 billing 标签会导致后续写操作被拒**：`ModifyClusterTags` 是覆盖式，`--Tags` 只传新标签会**清空原有标签**（传 `env:prod` 后 `billing` 标签丢失）。若集群靠 `billing` 标签过 CAM 授权（见 [维护窗口](maintenance-window.md)/[Master 扩缩容](master-ops.md)），遗漏标签后所有写操作被 `UnauthorizedOperation.CamNoAuth` 拒。修改标签时**必须把要保留的标签（含 `billing`）同时传入**，改完用 `DescribeClusters` 复核。
>
> ⚠️ **标签策略约束**：账号的 CAM/标签策略可能限制允许的标签 Key/Value 对。传策略不允许的标签对报 `InvalidParameter.Param`（消息 `PARAM_ERROR(<key>:<value> is a pair of invalid tags)`）——策略可能允许某 Key 但限制其 Value（如同账号 `env:prod` 通过、`env:audit` 被拒）。修改前用 `DescribeClusters` 看集群现有合规标签，沿用策略允许的 Key/Value。
>
> ⚠️ **异步任务冲突**：`ModifyClusterTags` 是异步任务，连续调用（前一个任务仍在后台执行时）报 `FailedOperation.TradeCommon`（消息 `modify task ready running background, please wait until the task done`）。需等待前一次修改完成（约数十秒）后再调，或轮询 `DescribeClusters` 确认标签已稳定。

### 步骤 3：修改集群镜像

```bash
tccli tke ModifyClusterImage --ClusterId "<CLUSTER_ID>" --region <REGION> --ImageId "<IMAGE_ID>"
# expected: exit 0
```

> 镜像变更会滚动重建节点。`ImageId` 用 `DescribeOSImages` 查（无入参，返回全部可用 OS 镜像）：

```bash
# 查询可用 OS 镜像（无入参）
tccli tke DescribeOSImages --version 2018-05-25 --region <REGION>
# expected: exit 0, 返回 OSImageSeriesSet[]（每项含 SeriesName/Alias/OsName/OsCustomizeType/Status/ImageId）+ TotalCount
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<IMAGE_ID>` | OS 镜像 ID | `DescribeOSImages` → `OSImageSeriesSet[].ImageId` |

### 步骤 4：修改组件额外参数（覆盖式）

```bash
# 先备份 (见决策依据), 再覆盖
tccli tke ModifyClusterExtraArgs --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --ClusterExtraArgs '{"KubeAPIServer":["--feature-gates=XXX=true"]}'
# expected: exit 0

# 控制额外参数任务状态（Operation 枚举: abort 取消并回退任务）
tccli tke ModifyClusterExtraArgsTaskState --ClusterId "<CLUSTER_ID>" --region <REGION> --Operation abort
# expected: exit 0
```

> ⚠️ 覆盖式更新——必须含原有要保留的参数，否则丢失。修改后用 `DescribeClusterExtraArgs` 复核。
>
> ⚠️ **多层前置约束**（按拦截顺序）：
> 1. **CAM 标签授权**（首个拦截点）：与 [维护窗口](maintenance-window.md)/[Master 扩缩容](master-ops.md) 相同，要求目标集群带 `billing` 标签才放行。不带标签返回 `AuthFailure.UnauthorizedOperation`（消息含 `qcs:resource_tag` `billing` 条件），到不了业务校验层。
> 2. **仅托管集群可用**（CAM 放行后）：`ModifyClusterExtraArgs` 只支持托管集群（MANAGED_CLUSTER），独立集群调用被拒。
> 3. **参数校验**：`KubeAPIServer` 等组件参数必须是该 K8s 版本真实支持的 feature-gates，传不存在的参数报 `InvalidParameter.Param`（如 `Args not found: [1.34.1] is not in --feature-gates available args list`）。可用参数用 `DescribeClusterAvailableExtraArgs` 查。

### 步骤 5：扩容容器网段

```bash
tccli tke AddClusterCIDR --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --ClusterCIDRs '["10.244.0.0/16"]' --IgnoreClusterCIDRConflict false
# expected: exit 0
```

> `IgnoreClusterCIDRConflict=true` 强制添加即使与已有网段冲突。Pod IP 不足时扩容。
>
> ⚠️ **多层前置约束**（按拦截顺序）：
> 1. **CAM 标签授权**（首个拦截点）：与 [维护窗口](maintenance-window.md)/[Master 扩缩容](master-ops.md) 相同，要求目标集群带 `billing` 标签才放行。不带标签返回 `UnauthorizedOperation.CamNoAuth`（消息含 `qcs:resource_tag` `billing` 条件），到不了业务校验层。错误样本：`code:UnauthorizedOperation.CamNoAuth ... you are not authorized to perform operation (tke:AddClusterCIDR) ... has no permission`。
> 2. **GR 集群约束**（CAM 放行后）：`AddClusterCIDR` 仅给 **GR 集群**增加 ClusterCIDR。CiliumOverlay 实际调用 **exit≠0**：`UnsupportedOperation.ClusterNotSuitAddClusterCIDR`（消息含 `CLUSTER NOT SUIT ADD CLUSTER CIDR` / `failed to get tke-bridge-agent`），**不会**静默成功；见 [CiliumOverlay](../networking/cilium-overlay.md)。集群网络类型由创建时 `ClusterAdvancedSettings.NetworkType` 决定（见 [创建集群](create.md)）。
> 3. **集群状态约束**：集群 Master 处于升级/变更中时调用亦报 `UnsupportedOperation.ClusterNotSuitAddClusterCIDR`（消息含 `master of cluster <ID> is updating, can't add clusterCIDR now`），须等集群稳定后重试——与「非 GR」同码不同消息，用消息区分。

### 步骤 6：获取集群 admin 角色（RBAC 授权前置）

<!-- kubectl 管理 K8s 原生资源（TCCLI 无此能力） -->
```bash
tccli tke AcquireClusterAdminRole --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, RequestId
```

> 为当前账号获取集群 admin RBAC 角色，是 kubectl 操作集群的授权前置。

### 步骤 7：修改容器运行时

```bash
tccli tke ModifyClusterRuntimeConfig --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --DstK8SVersion "<VERSION>" \
  --ClusterRuntimeConfig '{"RuntimeType":"containerd","RuntimeVersion":"1.6.9"}'
# expected: exit 0（RuntimeVersion 须在 DescribeSupportedRuntime 的 RuntimeVersions 内；1.30 常见仅 1.6.9，1.32+/1.34+ 另有 1.7.28）
```

> `NodePoolRuntimeConfig[]` 可按节点池分别配置运行时。运行时变更滚动重建节点。`RuntimeVersion` 须取自 `DescribeSupportedRuntime --K8sVersion` 返回的 `DefaultVersion` 或 `RuntimeVersions[]`，勿传入列表外版本。

## 跨字段约束

| `DstK8SVersion` | `ClusterRuntimeConfig` | `NodePoolRuntimeConfig` | 关系 |
|:----------------|:-----------------------|:------------------------|:-----|
| 目标 Kubernetes 版本 | 传则修改集群默认运行时 | 不传 | 运行时版本必须受目标 Kubernetes 版本支持 |
| 目标 Kubernetes 版本 | 可选 | 传一个或多个节点池配置 | 各节点池运行时版本必须受目标 Kubernetes 版本支持 |
| 目标 Kubernetes 版本 | 传 | 传 | 同一次调用分别更新集群默认值与指定节点池，不互斥；两类配置都受同一目标版本约束 |

至少传 `ClusterRuntimeConfig` 或 `NodePoolRuntimeConfig` 中一个有实际修改的目标；二者同传是官方示例覆盖的合法组合，不应误判为互斥。

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
>
> ⚠️ **多层前置约束**（按拦截顺序，写操作 `ModifyMasterComponent`）：
> 1. **CAM 标签授权**（首个拦截点）：与 [维护窗口](maintenance-window.md)/[Master 扩缩容](master-ops.md) 相同，要求目标集群带 `billing` 标签（CAM 匹配 `qcs:resource_tag`）。不带标签的集群调用返回 `AuthFailure.UnauthorizedOperation`（消息含 `has no permission` + 要求的标签 key/value），到不了业务校验层。错误样本：`code:AuthFailure.UnauthorizedOperation ... resource (qcs::tke:<REGION>::cluster/<ID>) has no permission with or without condition:[{"condition":{"key":"qcs:resource_tag","value":["billing&<标签值>"],...}}]`。
> 2. **混沌演练标记**（CAM 放行后的业务约束）：`ModifyMasterComponent` 用于 Master 组件停机故障演练，**仅对标记为「混沌演练」（Chaos Experiment）的集群放行**，普通集群返回 `FailedOperation.OperationForbidden`（`this operation is only allowed for clusters marked with 'Chaos Experiment' or '混沌演练'`）。
>
> ⚠️ **`DescribeMasterComponent`（只读）约束不同**：它不要求混沌演练标记——普通托管集群（组件 workload 就绪）调用返回 `Status: Running`，exit 0 成功。仅在组件 workload 未就绪/不存在时返回 `FailedOperation.KubeCommon`（`get workload failed, please try again later`）——非 CAM 拒绝，是组件未就绪，稍后重试即可。注意写操作 `ModifyMasterComponent`（停机演练）仍受混沌演练标记约束，见上条。
>
> ⚠️ **`DescribeMasterComponent` 仅托管集群可用**：在独立集群（INDEPENDENT_CLUSTER）调用返回 `FailedOperation.OperationForbidden`（`this operation is only supported for managed clusters`）——独立集群的 Master 组件状态不由此接口暴露。独立集群 Master 运维见 [master-ops.md](master-ops.md)。

### 步骤 9：更新子账户 kubeconfig

为子账户更新/生成集群 kubeconfig（子账户首次访问集群或凭证失效时用）：

```bash
tccli tke UpdateClusterKubeconfig --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, 返回 UpdatedSubAccounts/RequestId（不传 SubAccounts 默认为调用者本人）
```

```bash
# 为指定子账户更新（SubAccounts 为子账户 Uin 列表）
tccli tke UpdateClusterKubeconfig --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --SubAccounts '["<SUB_UIN>"]'
# expected: exit 0, UpdatedSubAccounts 含该子账户
```

> `SubAccounts` 为子账户 Uin 列表，不传默认为调用此接口的子账户本人。更新后子账户可用 `DescribeClusterSecurity` 取回 kubeconfig，见 [查询集群](query.md#集群访问凭证)。

## 验证

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 等级已变 | `DescribeClusterLevelAttribute --ClusterID <CLUSTER_ID>` | 目标等级 `Enable=true` |
| 等级变更记录 | `DescribeClusterLevelChangeRecords --ClusterID <CLUSTER_ID>` | 含本次变更 |
| 标签已生效 | `DescribeClusters --ClusterIds '["<CLUSTER_ID>"]'` | Tags 含新标签 |
| 额外参数已变 | `DescribeClusterExtraArgs --ClusterId <CLUSTER_ID>` | 含新参数 |
| 运行时已变 | `DescribeClusters` → `ClusterVersion`/运行时字段 | 目标版本 |
| Master 组件状态 | `DescribeMasterComponent` | 目标 Operation 生效 |
| kubeconfig 已更新 | `UpdateClusterKubeconfig` 返回 `UpdatedSubAccounts` | 含目标子账户 |
| 集群仍 Running | `DescribeClusterStatus` | ClusterState=Running |

> 修改后必须确认集群仍 `Running`——配置变更可能触发滚动重建，期间集群短暂异常。

## 回滚

> 配置变更多为不可逆或需再次变更回退。回退方式：
> - 等级: 再次 `ModifyClusterAttribute --ClusterLevel <原等级>`（计费调整）
> - 标签: `ModifyClusterTags` 传空或原标签
> - 额外参数: 用备份的 `ClusterExtraArgs` 覆盖回原值
> - 运行时: 不能直接回退，需再次 `ModifyClusterRuntimeConfig`

## 副作用

- **ModifyClusterImage/ModifyClusterRuntimeConfig**: 滚动重建节点，业务 Pod 重调度，短暂中断
- **ModifyClusterAttribute ClusterLevel**: 计费等级调整，影响月费
- **ModifyClusterExtraArgs**: 覆盖式，误传空数组清空参数，可能导致控制面异常
- **AcquireClusterAdminRole**: 授予集群 admin 权限，安全敏感
- **ModifyMasterComponent**: 启停控制面组件，影响集群管理能力

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `Unknown options: --ClusterID` | 核对接口 | Level 系列用大写 `ClusterID`，其他用 `ClusterId` | 按接口要求用正确大小写 |
| `InvalidParameterValue` (ClusterLevel) | `DescribeClusterLevelAttribute` | 等级不存在或不可用 | 用 `Enable=true` 的等级 |
| `FailedOperation.TradeCommon` | GetClusterLevelPrice | `ClusterLevel` 值非真实枚举（传 `L10` 稳定触发，因 L10 不存在 → 计费中心 "all price code not match"） | 改用真实等级 `L5`/`L20`/`L50`/`L100`/`L200`/`L500`/`L1000`/`L3000`/`L5000`（`DescribeClusterLevelAttribute` 返回的 `Enable=true` 项，L5 可询价）；非临时错误，重试无效 |
| `ResourceNotFound` (ImageId) | `DescribeOSImages` | 镜像 ID 不存在 | 用真实镜像 ID |
| `AuthFailure.UnauthorizedOperation` (ModifyMasterComponent，消息含 `has no permission` + `qcs:resource_tag` `billing` 条件) | `tccli tke DescribeClusters --ClusterIds '["<ID>"]' --filter "Clusters[0].TagSpecification[*].Tags[*]"` | 目标集群未带 CAM 要求的 `billing` 标签（首个拦截点） | 给集群加 `billing` 标签，或申请 `tke:ModifyMasterComponent` 权限。环境限制，非命令错误 |
| `FailedOperation.OperationForbidden` (ModifyMasterComponent，消息含 `Chaos Experiment`/`混沌演练`) | 确认集群是否标记为混沌演练目标 | CAM 已放行，但 Master 组件停机演练仅对混沌演练集群放行（第二层业务约束） | 在控制台/CAM 标记集群为混沌演练目标；非演练场景勿用此接口 |
| `InvalidParameter.Param` (ModifyClusterExtraArgs，消息含 `Args not found ... not in --feature-gates available args list`) | `DescribeClusterAvailableExtraArgs` 查该版本支持的参数 | 传了目标 K8s 版本不支持的 feature-gates 参数 | 用 `DescribeClusterAvailableExtraArgs` 返回的真实参数名 |
| `InvalidParameter.Param` (ModifyClusterTags，消息含 `PARAM_ERROR(<key>:<value> is a pair of invalid tags)`) | `DescribeClusters` 看集群现有合规标签 | 账号 CAM/标签策略不允许该标签 Key/Value 对（策略可能允许 Key 但限制 Value） | 换用策略允许的 Key/Value 对（如 `env:audit` 被拒时改 `env:prod`） |
| `FailedOperation.TradeCommon` (ModifyClusterTags，消息含 `modify task ready running background`) | 等待数十秒后重试 | 前一次 `ModifyClusterTags` 异步任务仍在后台执行，连续调用冲突 | 轮询 `DescribeClusters` 确认标签稳定后再调 |
| `UnauthorizedOperation.CamNoAuth` (AddClusterCIDR，消息含 `qcs:resource_tag` `billing` 条件) | `tccli tke DescribeClusters --ClusterIds '["<ID>"]' --filter "Clusters[0].TagSpecification[*].Tags[*]"` | 目标集群未带 CAM 要求的 `billing` 标签（首个拦截点，到不了业务校验） | 给集群加 `billing` 标签，或申请 `tke:AddClusterCIDR` 权限。环境限制 |
| `UnsupportedOperation.ClusterNotSuitAddClusterCIDR` (AddClusterCIDR；消息含 `tke-bridge-agent` 或 `master ... is updating`) | `DescribeClusters` → `Property.NetworkType`；`DescribeClusterStatus` | 非 GR（如 CiliumOverlay）不适合扩网段；或 Master 升级中（同码不同消息） | Overlay/VPC-CNI：勿调；GR 且升级中：等 Running 后重试 |
| `InternalError.UnexpectedInternal` (AddClusterCIDR，消息含 `eniipamd ... UpgradFailed, can't add cluster cidr`) | `DescribeClusters` 看 `Property`/`ClusterNetworkSettings`；查集群组件/升级态 | GR 集群但 eniipamd 升级失败等内部组件异常，扩网段被拒 | 先恢复/重建异常组件或换健康 GR 集群；勿与 Overlay 的 `ClusterNotSuitAddClusterCIDR` 混淆 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 等级变更后配额未变 | `DescribeClusterLevelAttribute` 复核 | 异步未完成 | 等待后重查 |
| ExtraArgs 修改后参数丢失 | `DescribeClusterExtraArgs` 对比备份 | 覆盖式未保留原参数 | 用备份值重新覆盖 |
| 运行时变更后节点 NotReady | `DescribeClusterInstances` | 滚动重建未完成 | 等待滚动完成 |
| 集群长时间非 Running | `DescribeClusterStatus` → `ClusterInstanceState` | 配置变更触发异常 | 查节点状态，必要时回退 |
| `AddClusterCIDR` exit 0 但 `ClusterCIDR`/`MultiClusterCIDR` 未变 | `DescribeClusters` 复核网段字段 | 异步未生效，或账号/白名单边界（CiliumOverlay 实际调用 exit≠0，见上表） | 轮询复核；确认 `NetworkType=GR` |

## 收尾确认

```bash
# 按所改步骤核对配置值是否落到预期（先确认字段存在，再确认配置值生效）
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' --version 2018-05-25 \
  --filter "Clusters[0].{name:ClusterName,tags:TagSpecification[0].Tags,rt:ContainerRuntime,rtver:RuntimeVersion}"
# expected: name/标签/运行时按所改步骤落到预期值（如 Tags 含 billing 标签 + 新标签）

tccli tke DescribeClusterExtraArgs --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --filter "ClusterExtraArgs.KubeAPIServer"
# expected: 若步骤 4 改了 ExtraArgs，此处含新参数且原有保留参数未丢失（覆盖式副作用核查）
```

> 配置项落到预期值 + 集群 `Running`（步骤 8 已核）= 配置变更完成。**副作用核查**：① `ModifyClusterTags` 是覆盖式——核对 `billing` 标签仍在（丢失会被 CAM 拒后续写操作，见 [§步骤 2](#步骤-2修改集群标签) 警告）；② `ModifyClusterExtraArgs` 是覆盖式——核对原参数未丢失（传空数组会清空，见 [§步骤 4](#步骤-4修改组件额外参数覆盖式) 警告）。

## 下一步

- [查询集群](query.md) — 只读查询集群状态与配置
- [创建集群](create.md) — 新建集群（配置的前置）
- [升级集群版本](upgrade.md) — K8s 版本升级
- [认证配置](../security/auth.md) — AcquireClusterAdminRole 后的 RBAC 配置
- [故障排查](../troubleshooting.md) — 配置变更后集群异常诊断
