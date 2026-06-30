---
doc_type: Overview
---
# 节点管理

> 集群的工作节点管理——节点池创建、扩缩容、单节点运维。节点是实际运行 Pod 的算力。

## 这是什么

节点是集群里运行 Pod 的机器（CVM 或虚拟节点）。节点通过节点池管理：节点池是同配置节点的分组，是扩缩容的基本单位。

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| NodePool | 同配置节点的分组 | 扩缩容的基本单位，不同池可设不同策略 |
| NodeType | 节点类型（Regular/Native/Super/External） | 决定计费模型与管理方式 |
| LifeState | 节点池生命周期状态 | 决定能否修改/删除节点池 |
| InstanceState | 单节点状态 | 决定节点是否可调度 Pod |
| ASG | 底层弹性伸缩组 | 节点池的实际加减节点由 ASG 执行 |

## 节点类型对比

| 类型 | 含义 | 节点来源 | 计费 | 适用场景 |
|:-----|:-----|:---------|:-----|:---------|
| Native（原生） | 腾讯云优化的 CVM | CVM | 按实例 | 生产环境推荐，MachineSet 管理 |
| Regular（普通） | 标准 CVM | CVM | 按实例 | 兼容旧版，ASG 管理 |
| Super（超级） | Serverless 虚拟节点 | 虚拟 | 按 Pod | 弹性补充、批处理 |
| External（第三方） | 非腾讯云机器 | 自建机房 | 自费 | 混合云 |

> Native 与 Regular 区别：Native 基于 MachineSet，支持原地升级与更细粒度生命周期管理；Regular 基于 ASG，功能较基础。生产环境首选 Native。

## 节点池生命周期

| 状态 | 含义 | 触发 | 用户可执行操作 |
|:-----|:-----|:-----|:--------------|
| creating | 创建中 | `CreateClusterNodePool` | 等待 |
| normal | 正常 | 创建/更新完成 | 扩缩容 / 修改 / 删除 |
| updating | 更新中 | `ModifyClusterNodePool` | 等待 |
| deleting | 删除中 | `DeleteClusterNodePool` | 等待 |

> 修改与删除节点池的命令（生命周期控制面，参数以 `--generate-cli-skeleton` 实测为准）：

```bash
# 修改节点池配置（Min/Max 区间、Labels、Taints 等，触发 updating → normal）
tccli tke ModifyClusterNodePool --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --NodePoolId "<NODE_POOL_ID>" \
  --MaxNodesNum 20 --MinNodesNum 2
# expected: exit 0

# 删除节点池（NodePoolIds[] 批量；KeepInstance=true 保留池内 CVM）
tccli tke DeleteClusterNodePool --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --NodePoolIds '["<NODE_POOL_ID>"]' --KeepInstance false
# expected: exit 0
```

> `ModifyClusterNodePool` 用单数 `NodePoolId`，`DeleteClusterNodePool` 用复数 `NodePoolIds[]`——同域字段名不一致，切换接口前用 `--generate-cli-skeleton` 核对。完整扩缩容/创建见 [创建节点池](nodepool-create.md) 与 [扩缩容](nodepool-scale.md)。

> 完整状态机见 [状态机参考](../reference/states.md)。单节点状态（`InstanceState`：initializing/running/failed）见同页。

## 不适用场景

- 不需要扩缩容，只要单节点 → 看 [节点实例运维](instance-ops.md) 的单节点操作
- 需要 Serverless，不想管节点 → [EKS 弹性集群](../specialized/eks-cluster.md)
- 边缘节点（IDC 机器）→ [边缘集群](../specialized/edge-cluster.md)

## 快速检查

```bash
# 查看集群的节点池
tccli tke DescribeClusterNodePools --region <REGION> --ClusterId "<CLUSTER_ID>" \
  --filter "NodePoolSet[].{id:NodePoolId,name:Name,state:LifeState,desired:DesiredNodesNum}"
# expected: 节点池列表，LifeState 含 normal
```

## 文档

- [创建节点池](nodepool-create.md) — 4 种 NodeType 选择、Native 强类型抽象
- [扩缩容节点池](nodepool-scale.md) — 调整 DesiredCapacity、安全缩容
- [节点实例运维](instance-ops.md) — 查询/启停/驱逐/删除单节点（跨 TKE 双版本）
- [状态机](../reference/states.md) — LifeState / InstanceState 枚举
- [配额和限制](../reference/quotas.md) — 节点池数（≤20/集群）、节点数上限
