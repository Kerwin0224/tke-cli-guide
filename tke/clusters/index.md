---
doc_type: Overview
---
# 集群管理

> TKE 集群（控制面）的创建、查询、配置、升级、维护、备份与销毁。集群是 TKE 的顶级资源，所有操作的基础。
>
> **集群是控制面，运行 Pod 还需节点池**——"可运行 Pod 的集群"完整闭环见下方 [集群生命周期故事线](#集群生命周期故事线)。

## 是什么

集群是容器运行所需云资源的集合（含 CVM、负载均衡等）。**标准集群（Master 托管）**：Master/Etcd 由腾讯云集中维护，你购置并管理工作节点；同一集群可混用普通/原生/超级/注册节点。**Master 自维护（独立）模式已停止新建**，存量独立集群仍可运维。

## 触发条件

- 你要创建首个 TKE 集群（`CreateCluster` 建空控制面，再单独加节点）— 去 [创建集群](create.md)
- 你要查询集群列表或单集群健康全貌（`DescribeClusters` + JMESPath 投影）— 去 [查询和过滤集群](query.md)
- 你要升级集群 K8s 版本或改运行时/控制面参数（`UpdateClusterVersion`/`ModifyClusterExtraArgs`）— 去 [升级集群版本](upgrade.md) 或 [配置集群属性与运行时](configure.md)
- 你要删除集群但被 `DeletionProtection` 拦截，或须清理残留云资源 — 看 [删除集群](delete.md)
- 你要给存量独立集群扩缩容 Master/etcd 节点 — 看 [独立集群 Master 运维](master-ops.md)

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Cluster | 容器运行所需云资源集合；标准集群 = 托管 Master/Etcd + 你管工作节点 | 顶级资源；决定地域、网络、K8s 版本 |
| ClusterType | `MANAGED_CLUSTER`（托管）/ `INDEPENDENT_CLUSTER`（独立，**已停止新建**） | 决定 Master 归谁管：托管=腾讯云管 Master 节点规模（不可改），但控制面参数/组件/加密仍由 tccli 改；新建默认走托管 |
| ClusterLevel | L5 / L20 / L50 / L100 / L200 / L500 / L3000 / L5000 等 | 决定管理费与可管理规模（见 [配额](../reference/quotas.md)） |
| ClusterState / ClusterStatus | 集群生命周期状态（分源） | `DescribeClusterStatus` → `ClusterStatusSet[].ClusterState`；`DescribeClusters` → `Clusters[].ClusterStatus`。非 `Running` 时多数写操作被拒 |
| 删除保护 | `DeletionProtection`（创建/属性）/ `ClusterDeletionProtection`（`DescribeClusterStatus`） | 删除前须先 `DisableClusterDeletionProtection`，否则 `DeleteCluster` 失败 |

## 集群类型对比

| 类型 | 最佳场景 | Master 运维 | 升级 | 费用 | 新建 |
|:-----|:---------|:-----------|:-----|:-----|:-----|
| MANAGED_CLUSTER（托管） | 生产与常规业务 | 腾讯云运维 **节点规模**（不可扩缩容）；控制面参数/组件/etcd 加密仍可由 tccli 改（见下） | API/控制台升级 | 集群管理费 + 节点等云资源费 | ✅ 默认路径 |
| INDEPENDENT_CLUSTER（独立） | 存量：须自行管 Master/Etcd | 你运维 Master CVM | 手动升级 Master | 节点费（含 Master CVM） | ❌ **已停止新建** |

> **新建**选 `MANAGED_CLUSTER`。托管模式下即使删光工作节点，未删的工作负载仍可能触发云资源计费——终止服务须 [删除集群](delete.md)。存量独立集群的 Master 扩缩容见 [独立集群 Master 运维](master-ops.md)。

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

- 只有一两个容器，不需要 K8s 编排 → [CVM](https://cloud.tencent.com/product/cvm) + Docker Compose
- **新建**免 CVM、要编排 → ① 本文 `CreateCluster` ② [虚拟节点](../nodes/virtual-nodes.md)（两步；仅 CreateCluster 不够）；无集群容器 → [容器实例](../specialized/eks-cluster.md#创建容器实例-部署-pod)；存量 EKS → [EKS](../specialized/eks-cluster.md)（新建入口已关闭）
- **新建**边缘/IDC 集群 → [注册节点公网版](https://cloud.tencent.com/document/product/457/57916)；存量见 [边缘集群](../specialized/edge-cluster.md)（已下线）
- **新建**独立集群（`INDEPENDENT_CLUSTER`）→ 已停止新建；改用托管标准集群

## 快速检查

```bash
# 查看账号下的集群
tccli tke DescribeClusters --region <REGION> --Limit 5 \
  --filter "Clusters[].{id:ClusterId,name:ClusterName,status:ClusterStatus}"
# expected: 集群列表，status 含 Running
```

## 集群生命周期故事线

集群从创建到销毁的完整流程，按顺序阅读。**集群 Running 只是第 1 步**——可运行 Pod 的完整集群须走到第 3 步（加节点）+ 第 6 步（开端点连通 kubectl）。

| 阶段 | 文档 | 做什么 | 产物 |
|------|------|--------|------|
| 0. 前置 | [准备 VPC 与子网](../../getting-started/prepare-vpc.md) | 创建 VPC + 子网（集群与节点的前置） | VpcId + SubnetId |
| 1. 创建 | [创建集群](create.md) | `CreateCluster` 控制面（空集群） | ClusterId，Running 但 0 节点 |
| 2. 连接 | [集群认证](../security/auth.md) | `DescribeClusterKubeconfig` 获取 kubeconfig | kubectl 可连控制面 |
| 3. 加节点 | [创建节点池](../nodes/nodepool-create.md) | 给集群加工作节点（节点从子网分配 IP） | ClusterRunningNodeNum ≥ 1，可运行 Pod |
| 4. 查询 | [查询和过滤集群](query.md) | 列表查询、JMESPath 投影、单集群健康全貌 | 集群状态全貌 |
| 5. 运行时 | [配置集群属性与运行时](configure.md) + [升级集群版本](upgrade.md) | 集群运行时行为：版本升级/运行时配置/Master 组件/OS 镜像/自定义参数 | ClusterVersion/运行时/Master 配置变更 |
| 6. 网络 | [管理端点](../networking/endpoints.md) | 公网/内网访问端点（kubectl 远程连接的前置） | ClusterExternalEndpoint |
| 7. 维护 | [维护窗口](maintenance-window.md) | 控制自动升级时段与排除项 | 维护窗口配置 |
| 8. 调度 | [调度策略](scheduling.md) | 调度器插件配置（Pod 调度行为） | SchedulerPolicy |
| 9. 备份 | [备份存储位置](backup.md) | 灾备存储位置（Velero 后端） | BackupStorageLocation |
| 10. 销毁 | [删除集群](delete.md) | 删除保护、残留资源清理 | 集群销毁 |

> **职责边界**：本文档域管集群（控制面）本身的生命周期。节点池归 [节点](../nodes/index.md)，网络端点/路由归 [网络](../networking/index.md)，认证/审计归 [集群加固](../security/index.md)——这些是集群的关联 noun，按用户任务归入对应目录，本域在故事线中指路。

**仅独立集群（节点规模）**：

- [独立集群 Master 运维](master-ops.md) — 扩缩容独立集群 Master/etcd 节点（托管集群 Master/etcd 节点规模由腾讯云运维，无需此步）

**托管集群（控制面仍可由 tccli 改）**：

- [配置集群属性与运行时](configure.md) — `ModifyClusterExtraArgs` 改控制面/etcd 参数、`ModifyMasterComponent` 组件启停演练（均托管专属）
- [集群加密保护](../security/protection.md) — `EnableEncryptionProtection` 为 etcd 存储数据开启 KMS 加密

**参考**：

- [状态机](../reference/states.md) — 集群完整状态枚举（23 个状态）
- [配额和限制](../reference/quotas.md) — 集群数/节点数上限
