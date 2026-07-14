---
doc_type: How-to
subtype: 6A
fused: true
---
# 扩缩容节点池

> 控制台: [容器服务控制台 - 节点池](https://console.cloud.tencent.com/tke2/nodepool)
> 调整节点池的期望节点数（DesiredCapacity），触发 ASG 扩容（加节点）或缩容（驱逐并移除节点）。异步操作。
>
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [集群扩缩容](https://cloud.tencent.com/document/product/457/32190) · [超级节点资源规格](https://cloud.tencent.com/document/product/457/39808)
>
> 配额：扩缩不超节点池 Min/Max 区间限制、单集群节点 5000、ASG 冷却时间对账。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：缩容选错策略致关键 Pod 被误驱逐、扩容遇 ResourceInsufficient、ASG 冷却期不一致致缩容不生效。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 触发条件

- DescribeClusterNodePools 返回 LifeState=normal 但 DesiredNodesNum 不满足业务容量（需扩容或缩容）
- 集群节点数已达 MaxNodesNum 上限，或扩容报 `LimitExceeded`/`ResourceInsufficient.SpecifiedInstanceType`（机型售罄）需调区间或换机型
- 你遇到扩缩容节点池问题需查诊断路径 — 看 [故障恢复]段


## 概述

扩缩容改变节点池的 `DesiredCapacity`，由底层 ASG（弹性伸缩组）执行实际加减节点：

| 操作 | DesiredCapacity 变化 | ASG 行为 | 耗时 |
|:-----|:---------------------|:---------|:-----|
| 扩容 | 增大 | 按节点池配置新建 CVM 加入集群 | 2-5 分钟/节点 |
| 缩容 | 减小 | 按 ASG 缩容策略移出节点（驱逐 Pod 后释放） | 3-10 分钟/节点 |

> 缩容会驱逐节点上的 Pod。若有 PDB（PodDisruptionBudget）保护，驱逐可能阻塞。详见 [节点实例运维](instance-ops.md)。

操作是**异步**的。旧版 ASG 池：`ModifyNodePoolDesiredCapacityAboutAsg` → 轮询 `DescribeClusterNodePools` 的 `NodeCountSummary`。新版 Native 池：`ModifyNodePool --version 2022-05-01 --Native '{"Replicas":N}'` → 轮询 `DescribeNodePools` 的 `Native.Replicas` / `ReadyReplicas`（无 `NodeCountSummary`）。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterState"
# expected: "Running"
```

### 资源检查

```bash
# 1. 确认节点池存在且正常
tccli tke DescribeClusterNodePools --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "NodePoolSet[].{id:NodePoolId,name:Name,state:LifeState,max:MaxNodesNum,min:MinNodesNum,desired:DesiredNodesNum}"
# expected: 目标节点池 LifeState="normal"

# 2. 确认配额未满（扩容时）
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterRunningNodeNum"
# expected: 节点数 < 集群配额（见 [配额](../reference/quotas.md)）
```

## 关键字段

> 完整入参以 `tccli tke ModifyNodePoolDesiredCapacityAboutAsg help --detail` 为准。此 Action 仅 3 个参数——只改期望数，不改节点池配置。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound` |
| NodePoolId | string | 是 | `np-xxxxxxxx` | `ResourceNotFound` |
| DesiredCapacity | int | 是 | 须在 `[MinNodesNum, MaxNodesNum]` 区间 | `InvalidParameterValue` / `LimitExceeded` |

> `DesiredCapacity` 必须在节点池的 `MinNodesNum` ~ `MaxNodesNum` 区间内。超出区间报错；要突破区间需先 `ModifyClusterNodePool` 调整 Min/Max。`ModifyNodePoolDesiredCapacityAboutAsg` 与 `ModifyClusterNodePool` 区别：前者只改期望数（轻量），后者改全部配置（含 Min/Max/Labels/Taints）。

## 操作步骤

### 步骤 1：决策 — 扩容 vs 缩容

#### 为什么区分扩缩容策略

- **扩容**: 直接设 `DesiredCapacity` 为目标值，ASG 自动建 CVM。失败模式主要是机型售罄/配额不足
- **缩容**: ASG 按"移出最老节点"策略缩容，**TKE 不跟踪具体被缩容节点**，无提前驱逐/封锁。要安全缩容，先手动 `DrainClusterVirtualNode` 或 `kubectl drain`
- **默认推荐**: 缩容前先 drain 目标节点，避免 Pod 被强制终止
- **弹性伸缩**: 若节点池开了 `EnableAutoscale`，不建议手动改 `DesiredCapacity`——让 ASG 按负载自动调整

### 步骤 2：扩容

```bash
# 扩容到 5 个节点
tccli tke ModifyNodePoolDesiredCapacityAboutAsg --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --NodePoolId "<NODE_POOL_ID>" --DesiredCapacity 5
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<NODE_POOL_ID>` | 节点池 ID | `np-xxxxxxxx` | `tccli tke DescribeClusterNodePools` → `NodePoolSet[].NodePoolId` |

### 步骤 3：缩容（安全）

```bash
# 1. 先 drain 目标节点（避免 Pod 被强杀）
tccli tke DrainClusterVirtualNode --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --NodeName "<NODE_NAME>"
# expected: exit 0

# 2. 再缩容（DesiredCapacity 减小）
tccli tke ModifyNodePoolDesiredCapacityAboutAsg --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --NodePoolId "<NODE_POOL_ID>" --DesiredCapacity 2
# expected: exit 0
```

> 不 drain 直接缩容：ASG 随机选节点移出，Pod 被强制终止。有状态服务（数据库）可能丢数据。

### 步骤 4：验证

异步操作。**按节点池创建版本选查询 Action**（同名≠同契约）：

| 路径 | 查询 Action | 列表键 | 就绪 LifeState | 期望数 | 实际数 |
|:-----|:------------|:-------|:---------------|:-------|:-------|
| 旧版 ASG（上文 `ModifyNodePoolDesiredCapacityAboutAsg`） | `DescribeClusterNodePools`（默认 2018） | `NodePoolSet` | `normal` | `DesiredNodesNum` | `NodeCountSummary.Total` |
| 新版 Native（`CreateNodePool` 2022） | `DescribeNodePools --version 2022-05-01` | `NodePools` | `Running` | `Native.Replicas` | `Native.ReadyReplicas` |

```bash
# 旧版 ASG 节点池：状态与期望数
tccli tke DescribeClusterNodePools --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "NodePoolSet[?NodePoolId=='<NODE_POOL_ID>'].{state:LifeState,desired:DesiredNodesNum,summary:NodeCountSummary}"
# expected: LifeState="normal", desired=目标值

# 新版 Native 节点池（Filter Name 仅 NodePoolsName / NodePoolsId / tags / tag:tag-key）
tccli tke DescribeNodePools --version 2022-05-01 --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Filters '[{"Name":"NodePoolsId","Values":["<NODE_POOL_ID>"]}]' \
  --filter "NodePools[0].{state:LifeState,desired:Native.Replicas,ready:Native.ReadyReplicas}"
# expected: LifeState="Running", desired=目标值；无 NodeCountSummary / DesiredNodesNum
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 节点池状态（旧） | `DescribeClusterNodePools` → `LifeState` | `normal` |
| 节点池状态（新） | `DescribeNodePools` → `LifeState` | `Running` |
| 期望数（旧） | `DesiredNodesNum` | 等于设定的 `DesiredCapacity` |
| 期望数（新） | `Native.Replicas` | 等于 `ModifyNodePool --Native.Replicas` |
| 实际节点数（旧） | `NodeCountSummary.Total` | 逐渐趋近 DesiredNodesNum |
| 实际节点数（新） | `Native.ReadyReplicas` | 逐渐趋近 Replicas |
| 节点 Ready | `DescribeClusterInstances` → `InstanceState` | 扩容节点 `running` |

## Master 扩缩容与 ASG 选项

> 独立集群（INDEPENDENT_CLUSTER）的 Master 扩缩容，与集群级 autoscaler 选项管理。节点池扩缩容见上文。

### Master 扩缩容（独立集群）

#### 为什么扩容 Master / 扩容 vs 不扩容

- **扩容 Master**: 独立集群 Master 负载高（API 请求多/etcd 数据多），扩容提升控制面可用性与性能
- **不扩容（托管集群）**: MANAGED_CLUSTER 的 Master 由 TKE 托管，无需也不能手动扩容——本段仅适用 INDEPENDENT_CLUSTER
- **决策依据**: `DescribeClusterStatus` 看 `ClusterInstanceState` + Master 负载；仅当 Master 成为瓶颈时扩容
- **InstanceDeleteMode 选择**: 缩容时 `terminate`（销毁 CVM，停止计费）/ `retain`（保留 CVM 作他用，继续计费）——默认 `terminate` 避免孤儿 CVM

```bash
# 扩容 Master (RunInstancesForNode 透传 CVM RunInstances JSON, NodeRole=master)
tccli tke ScaleOutClusterMaster --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --RunInstancesForNode '[{"NodeRole":"master","RunInstancesPara":["<CVM_JSON>"]}]'
# expected: exit 0

# 缩容 Master (按 InstanceId, InstanceDeleteMode 销毁方式)
tccli tke ScaleInClusterMaster --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --ScaleInMasters '[{"InstanceId":"<INSTANCE_ID>","NodeRole":"master","InstanceDeleteMode":"terminate"}]'
# expected: exit 0
```

> Master 扩缩容仅独立集群（INDEPENDENT_CLUSTER）需要——托管集群（MANAGED_CLUSTER）的 Master 由 TKE 管理。`InstanceDeleteMode` 如 `terminate`（销毁）/ `retain`（保留）。

### 集群级 ASG 选项

#### Expander 扩容策略选择

- **random（默认）**: 随机选节点池扩容，不按资源利用率或可调度 Pod 数择优
- **most-pods**: 选能调度最多 Pod 的节点池，优先填满大机型
- **least-waste**: 选扩容后剩余资源最少的，最小化浪费（推荐生产）
- **IsScaleDownEnabled**: true 开启自动缩容（空闲节点回收）；false 仅扩不缩
- **ScaleDownUtilizationThreshold**: 节点利用率低于此值（默认 50%）触发缩容
- **SkipNodesWithLocalStorage/SystemPods**: true 跳过有本地存储/系统 Pod 的节点，避免缩容误删

```bash
# 查询集群 autoscaler 选项 (缩容策略/阈值/扩容策略)
tccli tke DescribeClusterAsGroupOption --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, ClusterAsGroupOption 含 IsScaleDownEnabled/Expander/阈值
```
```json
{
    "ClusterAsGroupOption": {
        "IsScaleDownEnabled": false, "Expander": "random", "MaxEmptyBulkDelete": 10,
        "ScaleDownDelay": 10, "ScaleDownUtilizationThreshold": 50,
        "SkipNodesWithLocalStorage": true, "SkipNodesWithSystemPods": true
    }
}
```

```bash
# 修改集群级 autoscaler 选项 (IsScaleDownEnabled/Expander/阈值等)
tccli tke ModifyClusterAsGroupOptionAttribute --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --ClusterAsGroupOption '{"IsScaleDownEnabled":true,"Expander":"most-pods","ScaleDownUtilizationThreshold":60}'
# expected: exit 0

# 修改节点池 ASG 属性 (AutoScalingGroupId/Enabled/MinSize/MaxSize)
tccli tke ModifyClusterAsGroupAttribute --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --ClusterAsGroupAttribute '{"AutoScalingGroupId":"<ASG_ID>","AutoScalingGroupEnabled":true,"AutoScalingGroupRange":{"MinSize":2,"MaxSize":10}}'
# expected: exit 0

# 删除集群 ASG (KeepInstance=true 保留节点)
tccli tke DeleteClusterAsGroups --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --AutoScalingGroupIds '["<ASG_ID>"]' --KeepInstance true
# expected: exit 0
```

> `DescribeClusterAsGroupOption` 返回集群级 autoscaler 配置（`Expander` 扩容策略如 `random`/`most-pods`/`least-waste`，`ScaleDownUtilizationThreshold` 缩容阈值）。`ModifyClusterAsGroupOptionAttribute` 改集群级选项，`ModifyClusterAsGroupAttribute` 改单 ASG 属性（MinSize/MaxSize）。`DeleteClusterAsGroups` 的 `KeepInstance=true` 保留节点仅删 ASG。

### 查询 ASG 活动与调整节点池区间

> 扩容不增长时查 ASG 活动历史定位根因（机型售罄/子网 IP 不足）；`DesiredCapacity` 超区间时先 `ModifyClusterNodePool` 调 Min/Max。两者参数见各 Action 的 `help --detail`。

```bash
# 查询集群 ASG 活动与实例数（ClusterId + AutoScalingGroupIds[] + 分页）
tccli tke DescribeClusterAsGroups --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --AutoScalingGroupIds '["<ASG_ID>"]' \
  --Offset 0 --Limit 20
# expected: exit 0，返回 TotalCount+ClusterAsGroupSet[]（含 MinSize/MaxSize/CurrentSize/Activity）

# 调整节点池 Min/Max 区间（突破 DesiredCapacity 区间限制前先调此）
tccli tke ModifyClusterNodePool --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --NodePoolId "<NODE_POOL_ID>" \
  --MaxNodesNum 20 --MinNodesNum 2
# expected: exit 0; 节点池不存在报 FailedOperation.RecordNotFound
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<ASG_ID>` | 弹性伸缩组 ID | `DescribeClusterNodePools` → `NodePoolSet[].AutoScalingGroupId` |
| `<NODE_POOL_ID>` | 节点池 ID | `tccli tke DescribeClusterNodePools` → `NodePoolSet[].NodePoolId` |

> `ModifyClusterNodePool` 改全部配置（Min/Max/Labels/Taints/OsName 等），与 `ModifyNodePoolDesiredCapacityAboutAsg`（只改期望数，轻量）分工：扩缩容走后者，调区间/改配置走前者。

## 清理

> **副作用警告**：缩容会释放 CVM 实例，节点上数据不可恢复。扩容产生的新 CVM 按量计费，缩容后停止计费。
> 若想保留节点但移出节点池，用 `RemoveNodeFromNodePool`（不销毁 CVM），见 [节点实例运维](instance-ops.md)。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `InvalidParameterValue.DesiredCapacity` | `DescribeClusterNodePools` 看 Min/Max | DesiredCapacity 超出 `[MinNodesNum, MaxNodesNum]` | 先 `ModifyClusterNodePool` 调大 MaxNodesNum，或设较小值 |
| `LimitExceeded` | `DescribeClusterStatus` 看节点数 | 集群节点数达上限 | 删除闲置节点或提工单提额 |
| `ResourceInsufficient.SpecifiedInstanceType` | `tccli cvm DescribeZoneInstanceConfigInfos` | 机型库存不足（售罄） | 换机型或换可用区，见 [创建节点池](nodepool-create.md) |
| `ResourceNotFound` | `DescribeClusterNodePools` 核对 ID | ClusterId 或 NodePoolId 错 | 确认 ID 格式 |
| `UnsupportedOperation` | `DescribeClusterNodePools` → `LifeState` | 节点池非 `normal`（updating/deleting） | 等节点池 `normal` 后重试 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 扩容后节点数不增长 | `DescribeClusterAsGroups` 看 ASG 活动 | 机型售罄或子网 IP 不足 | 查 ASG 活动历史，换机型/加子网 |
| 缩容后 Pod 处于 Pending | `kubectl get pods -A` | 缩容过度，剩余节点资源不足 | 扩容回原值，或优化 Pod 资源请求 |
| 节点池卡在 `updating` | `DescribeClusterNodePools` → `LifeState` | ASG 操作未完成 | 等待；超 30 分钟查 `DescribeClusterAsGroups` |
| 缩容节点上 Pod 被强杀 | `kubectl get events` | 未先 drain | 缩容前必须 `DrainClusterVirtualNode`；有状态服务用 PDB 保护 |

> 扩容失败常是机型/子网问题（环境限制，非命令错误）；缩容失败常是 PDB 阻塞驱逐。两者诊断路径不同。

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- kubectl验证tccli扩缩容操作结果，kubectl get nodes观察K8s层节点Ready与Pod调度状态，非tccli边界 -->
```bash
# ②业务可用性端到端: 扩容新节点不仅 InstanceState=running 须 K8s Ready（缩容后无 Pod Pending）
# 扩容后 kubectl get nodes 看新节点真 Ready（InstanceState=running 不等于 K8s Ready，kubelet 注册需时间）
kubectl get nodes --show-labels | grep -v NotReady | wc -l
# expected: Ready 节点数 == 扩容后 DesiredCapacity（缩容后无 Pod Pending: kubectl get pods -A 看无 Pending）

# ③跨步骤多资源汇总（新版 Native）: LifeState=Running + Replicas == ReadyReplicas == Ready 节点数
tccli tke DescribeNodePools --version 2022-05-01 --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --Filters '[{"Name":"NodePoolsId","Values":["<NODE_POOL_ID>"]}]' \
  --filter "NodePools[0].{state:LifeState,desired:Native.Replicas,ready:Native.ReadyReplicas}"
# expected: state="Running"; desired=扩缩后目标值；ReadyReplicas 逐渐趋近 desired
# 旧版 ASG 对照: DescribeClusterNodePools → LifeState=normal + DesiredNodesNum + NodeCountSummary.Total
```

> 新版：`LifeState=Running` + `Native.Replicas`==`ReadyReplicas`==Ready 节点数；旧版：`LifeState=normal` + `DesiredNodesNum`==`NodeCountSummary.Total`==Ready。缩容前若用 `DrainClusterVirtualNode`/`kubectl drain` 驱逐节点，三者一致即链路终态。

---

## 下一步

- [创建节点池](nodepool-create.md) — 节点池生命周期
- [节点实例运维](instance-ops.md) — drain/移出/删除单个节点
- [集群状态机](../reference/states.md) — 节点池 `LifeState` 含义
- [配额和限制](../reference/quotas.md) — 节点数上限
- [故障排查](../troubleshooting.md) — 节点 NotReady 的诊断
