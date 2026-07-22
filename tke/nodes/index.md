---
doc_type: Overview
---
# 节点管理

> 集群的工作节点管理——节点池创建、扩缩容、单节点运维。节点是实际运行 Pod 的算力。

## 是什么

Worker 节点是容器集群的基本元素（虚拟机或物理机），含运行 Pod 所需的 Kubelet、Kube-proxy 等。TKE 标准集群支持 4 种节点类型；节点通常经节点池批量创建与扩缩容。

## 何时阅读
- 你要给集群添加工作节点、选普通/原生/超级/注册节点类型 — [创建节点池](nodepool-create.md)（原生节点须 `--version 2022-05-01`）
- 你要调整节点池期望容量、安全缩容不丢 Pod — [扩缩容节点池](nodepool-scale.md)（`ModifyClusterNodePool`）
- 你要查询单节点状态、启停/驱逐/删除某台 CVM — [节点实例运维](instance-ops.md)（跨 TKE 双版本 Action）
- 你要在标准集群内免 CVM 容量运行 Pod，或接入自建 IDC 机器做混合云调度 — [虚拟节点](virtual-nodes.md) / [注册节点概述](registered-nodes/overview.md)
- 你遇到节点池 `LifeState`（creating/normal/updating/deleting）或单节点 `InstanceState` 枚举困惑，或要做健康检测与自动隔离 — 看 [节点池生命周期](#节点池生命周期) / [节点健康检查](health-check.md) / [状态机](../reference/states.md)

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Node | 运行 Pod 的计算单元（CVM / 原生 / 超级 / 注册） | 实际算力；类型决定计费与运维面 |
| NodePool | 统一机型、标签、Taint 与扩缩容策略的节点组 | 扩缩容与批量运维的基本单位 |
| NodeType | `Regular` / `Native` / `Super` / `External` | 决定计费模型与管理方式；控制台是四套向导 |
| LifeState | 节点池生命周期状态 | 决定能否修改/删除节点池 |
| InstanceState | 单节点状态 | 决定节点是否可调度 Pod |
| ASG | 底层弹性伸缩组（普通节点池） | 普通节点池加减节点由 ASG 执行 |

## 节点类型对比

> TKE 标准集群支持 4 种节点类型。控制台「创建节点池」是**四套向导**（字段差大），不要用一张参数表套四种类型——先选类型，再进对应文档。

| 类型 | API 枚举 | 特点 | 计费 | 适用场景 | 文档 |
|:-----|:--------:|:-----|:-----|:---------|:-----|
| 普通节点 | `Regular` | 适配多种 CVM 机型；基于弹性伸缩自动缩容；OS 可定制 | 按实例 | 对资源/运维管控强、OS 偏定制 | [创建节点池](nodepool-create.md) |
| 原生节点 | `Native` | TKE Insight 资源大盘；专有调度器；声明式 MachineSet | 按实例 | 降本、提升利用率、简化运维 | [创建节点池](nodepool-create.md)（`--version 2022-05-01`） |
| 超级节点 | `Super` | Serverless；单 Pod 独占轻量虚拟机；秒级扩缩 | 按 Pod | 弹性业务、强隔离、轻运维 | [虚拟节点](virtual-nodes.md) |
| 注册节点 | `External` | IDC/自建机器接入标准集群；云上云下混合调度 | 自费 | 本地资源利旧、混合云 | [注册节点概述](registered-nodes/overview.md) |

> 普通 vs 原生：原生基于 MachineSet，生命周期更细；普通基于 ASG。生产降本场景常用原生节点（须 API `2022-05-01`）。注册节点 ≠ 注册集群；边缘新建优先 [注册节点公网版](https://cloud.tencent.com/document/product/457/57916)。

## 节点池生命周期 {#节点池生命周期}

| 状态 | 含义 | 触发 | 用户可执行操作 |
|:-----|:-----|:-----|:--------------|
| creating | 创建中 | `CreateClusterNodePool` | 等待 |
| normal | 正常 | 创建/更新完成 | 扩缩容 / 修改 / 删除 |
| updating | 更新中 | `ModifyClusterNodePool` | 等待 |
| deleting | 删除中 | `DeleteClusterNodePool` | 等待 |

> 修改与删除节点池的命令（生命周期控制面，参数见各 Action 的 `help --detail`）：

```bash
# 修改节点池配置（Min/Max 区间、Labels、Taints 等，触发 updating → normal）
tccli tke ModifyClusterNodePool --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --NodePoolId "<NODE_POOL_ID>" \
  --MaxNodesNum 20 --MinNodesNum 2
# expected: exit 0; 节点池不存在报 FailedOperation.RecordNotFound

# 删除节点池（NodePoolIds[] 批量；KeepInstance=true 保留池内 CVM）
tccli tke DeleteClusterNodePool --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --NodePoolIds '["<NODE_POOL_ID>"]' --KeepInstance false
# expected: exit 0; 集群不存在报 ResourceNotFound
```

> `ModifyClusterNodePool` 用单数 `NodePoolId`，`DeleteClusterNodePool` 用复数 `NodePoolIds[]`——同域字段名不一致，切换接口前用 `--generate-cli-skeleton` 核对。完整扩缩容/创建见 [创建节点池](nodepool-create.md) 与 [扩缩容](nodepool-scale.md)。

> 完整状态机见 [状态机参考](../reference/states.md)。单节点状态（`InstanceState`：initializing/running/failed）见同页。

## 不适用场景

- 不需要扩缩容，只要单节点运维 → [节点实例运维](instance-ops.md)
- **新建**免 CVM 容量（要编排）→ 先有标准集群，再 [虚拟节点](virtual-nodes.md)（非一次 `CreateCluster` 完成）；无集群容器 → [容器实例](../specialized/eks-cluster.md#创建容器实例-部署-pod)；存量 EKS 集群 → [EKS](../specialized/eks-cluster.md)
- **新建**边缘/IDC 接入 → [注册节点](registered-nodes/overview.md) / [注册节点公网版](https://cloud.tencent.com/document/product/457/57916)；不要再开 [边缘集群](../specialized/edge-cluster.md)

## 快速检查

```bash
# 查看集群的节点池（旧版 ASG：NodePoolSet + LifeState=normal）
tccli tke DescribeClusterNodePools --region <REGION> --ClusterId "<CLUSTER_ID>" \
  --filter "NodePoolSet[].{id:NodePoolId,name:Name,state:LifeState,desired:DesiredNodesNum}"
# expected: 节点池列表，LifeState 含 normal
# 新版 Native：DescribeNodePools --version 2022-05-01 → NodePools[]，LifeState=Running，期望数 Native.Replicas
```

## 文档

- [创建节点池](nodepool-create.md) — 4 种 NodeType 选择、Native 强类型抽象
- [扩缩容节点池](nodepool-scale.md) — 调整 DesiredCapacity、安全缩容
- [节点实例运维](instance-ops.md) — 查询/启停/驱逐/删除单节点（跨 TKE 双版本）
- [虚拟节点 (超级节点)](virtual-nodes.md) — 标准集群内免 CVM 容量（先 CreateCluster；与 EKS 容器实例不同）
- [注册节点概述](registered-nodes/overview.md) — 自建 IDC/注册节点接入（不要与注册集群混淆）
- [节点健康检查](health-check.md) — 健康检测策略、自动隔离与修复
- [状态机](../reference/states.md) — LifeState / InstanceState 枚举
- [配额和限制](../reference/quotas.md) — 节点池数（≤20/集群）、节点数上限

## 删除节点池字段

| 字段 | 所属 Action | 必填 | 说明 |
|:---|:---|:---:|:---|
| `NodePoolIds` | `DeleteClusterNodePool` | 是 | 待删除节点池 ID 列表 |
