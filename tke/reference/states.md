---
doc_type: Reference
subtype: 8B
---
# TKE 状态机

> 集群、节点池、节点三类资源的完整状态机。状态值来自 `DescribeClusterStatus` / `DescribeClusterNodePools` / `DescribeClusterInstances` 的响应字段，以官方文档为准。

## 查询命令

```bash
# 集群状态 (ClusterState + ClusterInstanceState + 节点计数 + 删除保护 + 审计)
tccli tke DescribeClusterStatus --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'
# expected: ClusterStatusSet[0].ClusterState = "Running"

# 节点池状态 (LifeState)
tccli tke DescribeClusterNodePools --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: NodePoolSet[].LifeState = "normal"

# 节点状态 (InstanceState)
tccli tke DescribeClusterInstances --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: InstanceSet[].InstanceState = "running"
```

## 集群状态 (ClusterState)

> **字段分源**：`DescribeClusterStatus` 返回的 `ClusterState` 与 `DescribeClusters` 返回的 `ClusterStatus` **不是同一字段**。本表以 `DescribeClusterStatus` → `ClusterStatusSet[].ClusterState` 的契约枚举为准（与 `help --detail` / api.json `ClusterStatus.ClusterState` 一致）。`DescribeClusters` 的 `ClusterStatus` 文档串另含 `Abnormal` / `Idling` / `Recovering` / `Scaling` / `WaittingForConnect` / `Trading` 等取值——轮询状态机请用 `DescribeClusterStatus`，列表页状态看 `DescribeClusters`。

| 状态 | 含义 | 触发条件 | 用户可执行操作 | 终态 |
|:-----|:-----|:---------|:--------------|:----:|
| `Creating` | 集群创建中 | `CreateCluster` | 等待 | 否 |
| `Running` | 集群正常运行 | 创建完成 / 升级完成 | 全部操作 | 否 |
| `Upgrading` | 集群升级中 | `UpdateClusterVersion` | 等待 / `CancelUpgradePlan` | 否 |
| `NodeUpgrading` | 节点升级中 | 节点版本升级 | 等待 | 否 |
| `RuntimeUpgrading` | 托管集群修改参数中 | 运行时 / 参数变更 | 等待 | 否 |
| `MasterScaling` | 控制面扩容中 | 托管 Master 自动扩缩容 | 等待 | 否 |
| `ClusterLevelUpgrading` | 集群等级调整中 | `ModifyClusterLevel` | 等待 | 否 |
| `ClusterLevelTrading` | 集群变配交易中 | 等级变配计费处理 | 等待 | 否 |
| `Pause` | 集群升级暂停 | 升级暂停 | 恢复升级 | 否 |
| `Deleting` | 集群删除中 | `DeleteCluster` | 等待 | 是 |
| `Isolated` | 集群已隔离 | 欠费隔离 | 充值恢复 | 否 |
| `ResourceIsolate` | 执行隔离中 | 欠费隔离流程 | 等待 | 否 |
| `ResourceIsolated` | 已隔离 | 隔离完成 | 充值恢复 | 否 |
| `ResourceReverse` | 执行冲正中 | 隔离冲正流程 | 等待 | 否 |
| `ResourceReversal` | 冲正中 | 冲正流程 | 等待 | 否 |
| `ResourceDestroy` | 执行销毁中 | 销毁流程 | 等待 | 否 |
| `ResourceDestroyed` | 已销毁 | 销毁完成 | — | 是 |

> 常驻工作状态：`Running`。隔离分支：`Isolated` / `ResourceIsolate*`。终态：`Deleting` / `ResourceDestroyed`。
>
> **`DescribeClusters.ClusterStatus` 补充取值**（列表字段，勿与上表 `ClusterState` 混用）：`Trading`（开通中）、`Idling`（闲置）、`Recovering`（唤醒中）、`Scaling`（规模调整）、`WaittingForConnect`（独立集群待注册，拼写以 API 为准）、`Abnormal`（异常）。

## 集群节点健康 (ClusterInstanceState)

> `ClusterInstanceState` 字段取自 `DescribeClusterStatus` 响应，汇总集群下所有**工作节点**的健康度。有工作节点时为 `AllNormal`。**空集群（无工作节点，如托管空集群或仅 Master 的独立集群）该字段为空**（返回 `-`/空字符串），非异常——有工作节点后才返回 `AllNormal`/`PartialAbnormal`/`AllAbnormal`。

| 状态 | 含义 | 用户可执行操作 |
|:-----|:-----|:--------------|
| `AllNormal` | 节点全部正常 | — |
| `PartialAbnormal` | 节点部分异常 | 查 `DescribeClusterInstances` 定位异常节点 |
| `AllAbnormal` | 节点全部异常 | 排查网络 / 控制面 |

## 节点池状态 (LifeState)

> `LifeState` 字段取自 `DescribeClusterNodePools` 响应。枚举见腾讯云官方数据结构文档（小写）。

| 状态 | 含义 | 触发条件 | 用户可执行操作 | 终态 |
|:-----|:-----|:---------|:--------------|:----:|
| `creating` | 节点池创建中 | `CreateClusterNodePool` | 等待 | 否 |
| `normal` | 节点池正常 | 创建完成 / 更新完成 | 扩缩容 / 修改 / 删除 | 否 |
| `updating` | 节点池更新中 | `ModifyClusterNodePool` | 等待 | 否 |
| `deleting` | 节点池删除中 | `DeleteClusterNodePool` | 等待 | 否 |
| `deleted` | 节点池已删除 | 删除完成 | — | 是 |

> 注：2022-05-01 新版 API 的节点池（`CreateNodePool`）`LifeState` 用 `Running`；2018-05-25 旧版（`CreateClusterNodePool`）用小写 `normal`。两版抽象不同，切换前用 `--generate-cli-skeleton` 核契约。

## 节点状态 (InstanceState)

> `InstanceState` 字段取自 `DescribeClusterInstances` 响应。正常运行时为 `running`。

| 状态 | 含义 | 触发条件 | 用户可执行操作 |
|:-----|:-----|:---------|:--------------|
| `initializing` | 初始化中 | 节点刚加入，kubelet 未就绪 | 等待 |
| `running` | 运行中 | 初始化完成，可调度 Pod | 驱逐 / 移出 / 删除 / 升级 |
| `failed` | 异常 | 初始化失败 / 组件故障 | 查 `FailedReason` 字段，删除重建 |

> `FailedReason` 字段给出异常原因（如 `=Ready:False`）。节点启停（2022-05-01 的 `StartMachines`/`StopMachines`）不改变 `InstanceState`，改变的是底层 CVM 的开关机状态。

## 相关文档

- [创建集群](../clusters/create.md) — 触发 `Creating → Running`
- [删除集群](../clusters/delete.md) — 触发 `Deleting`
- [升级集群](../clusters/upgrade.md) — 触发 `Upgrading`
- [创建节点池](../nodes/nodepool-create.md) — 触发 `creating → normal`
- [故障排查](../troubleshooting.md) — 状态异常的诊断路径
- [错误码](error-codes.md) — 状态查询失败时的错误码
