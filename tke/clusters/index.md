---
doc_type: Overview
---
# 集群管理

> TKE 集群（控制面）的创建、查询、配置、升级、维护、备份与销毁。集群是 TKE 的顶级资源，所有操作的基础。
>
> **集群是控制面，跑 Pod 还需节点池**——"能跑 Pod 的集群"完整闭环见下方 [集群生命周期故事线](#集群生命周期故事线)。

## 是什么

集群是腾讯云上运行的 Kubernetes 实例。创建集群后，腾讯云管理 Master 节点（托管模式），你管理工作节点。一个集群包含若干节点池、网络配置、安全策略与工作负载。

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Cluster | K8s 集群（托管或独立） | 所有资源的顶级容器，决定网络和版本 |
| ClusterType | `MANAGED_CLUSTER`（托管）/ `INDEPENDENT_CLUSTER`（独立） | 决定你负责哪些运维（Master 归谁管） |
| ClusterLevel | L5 / L20 / L50 / L100 / L200 / L500（无 L10） | 决定集群管理费与可管理节点规模 |
| ClusterState | 集群生命周期状态 | 决定能否执行操作（非 Running 多被拒绝） |
| 删除保护 | `DeletionProtection` 开关 | 防止误删集群，删除前必须关闭 |

## 集群类型对比

| 类型 | 最佳场景 | Master 运维 | 升级 | 费用 |
|:-----|:---------|:-----------|:-----|:-----|
| MANAGED_CLUSTER（托管） | 生产环境首选 | 腾讯云运维 | 一键升级 | 集群管理费 + 节点费 |
| INDEPENDENT_CLUSTER（独立） | 完全控制 Master | 你运维 3 台 CVM | 手动升级 Master | 节点费（含 Master CVM） |

> 99% 场景选托管集群。独立集群仅在需要自定义 Master 组件时使用。
> 托管集群（MANAGED_CLUSTER）的 Master 由腾讯云运维，无需也无法用 tccli 扩缩容 Master。**独立集群**（INDEPENDENT_CLUSTER）需自行维护 Master，其 Master 扩缩容见 [独立集群 Master 运维](master-ops.md)。

## 集群状态生命周期

| 状态 | 含义 | 触发 | 用户可执行操作 |
|:-----|:-----|:-----|:--------------|
| Creating | 创建中 | `CreateCluster` | 等待 |
| Running | 正常运行 | 创建/升级完成 | 全部操作 |
| Upgrading | 升级中 | `UpdateClusterVersion` | 等待 / 暂停 |
| Deleting | 删除中 | `DeleteCluster` | 等待 |
| Abnormal | 异常 | Master/etcd 组件故障、节点大面积失联 | 诊断 + 修复 |

> 完整状态机（23 个状态）见 [状态机参考](../reference/states.md)。多数操作要求集群 `Running` 才能执行。

## 不适用场景

- 只有一两个容器，不需要 K8s 编排 → 用 [CVM](https://cloud.tencent.com/product/cvm) + Docker Compose
- 不想管理集群，只要 Serverless 容器 → [EKS 弹性集群](../specialized/eks-cluster.md)
- 需要边缘计算（IDC/门店）→ [边缘集群](../specialized/edge-cluster.md)

## 快速检查

```bash
# 查看账号下的集群
tccli tke DescribeClusters --region <REGION> --Limit 5 \
  --filter "Clusters[].{id:ClusterId,name:ClusterName,status:ClusterStatus}"
# expected: 集群列表，status 含 Running
```

## 集群生命周期故事线

集群从创建到销毁的完整流程，按顺序阅读。**集群 Running 只是第 1 步**——能跑 Pod 的完整集群须走到第 3 步（加节点）+ 第 6 步（开端点连通 kubectl）。

| 阶段 | 文档 | 做什么 | 产物 |
|------|------|--------|------|
| 0. 前置 | [准备 VPC 与子网](../../getting-started/prepare-vpc.md) | 创建 VPC + 子网（集群与节点的前置） | VpcId + SubnetId |
| 1. 创建 | [创建集群](create.md) | `CreateCluster` 控制面（空集群） | ClusterId，Running 但 0 节点 |
| 2. 连接 | [集群认证](../security/auth.md) | `DescribeClusterKubeconfig` 获取 kubeconfig | kubectl 可连控制面 |
| 3. 加节点 | [创建节点池](../nodes/nodepool-create.md) | 给集群加工作节点（节点从子网分配 IP） | ClusterRunningNodeNum ≥ 1，能跑 Pod |
| 4. 查询 | [查询和过滤集群](query.md) | 列表查询、JMESPath 投影、单集群健康全貌 | 集群状态全貌 |
| 5. 配置 | [配置集群属性与运行时](configure.md) | 改等级/标签/镜像/组件参数/运行时/Master 组件 | 集群属性变更 |
| 6. 网络 | [管理端点](../networking/endpoints.md) | 公网/内网访问端点（kubectl 远程连的前置） | ClusterExternalEndpoint |
| 7. 升级 | [升级集群版本](upgrade.md) | K8s 版本升级（不可回滚） | ClusterVersion 提升 |
| 8. 维护 | [维护窗口](maintenance-window.md) | 控制自动升级时段与排除项 | 维护窗口配置 |
| 9. 调度 | [调度策略](scheduling.md) | 调度器插件配置（Pod 调度行为） | SchedulerPolicy |
| 10. 备份 | [备份存储位置](backup.md) | 灾备存储位置（Velero 后端） | BackupStorageLocation |
| 11. 销毁 | [删除集群](delete.md) | 删除保护、残留资源清理 | 集群销毁 |

> **职责边界**：本文档域管集群（控制面）本身的生命周期。节点池归 [节点](../nodes/index.md)，网络端点/路由归 [网络](../networking/index.md)，认证/审计归 [集群加固](../security/index.md)——这些是集群的关联 noun，按用户任务归入对应目录，本域在故事线中指路。

**仅独立集群**：

- [独立集群 Master 运维](master-ops.md) — 扩缩容独立集群 Master/etcd 节点（托管集群 Master 由腾讯云运维，无需此步）

**参考**：

- [状态机](../reference/states.md) — 集群完整状态枚举（23 个状态）
- [配额和限制](../reference/quotas.md) — 集群数/节点数上限
