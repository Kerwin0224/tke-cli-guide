---
doc_type: Overview
---
# 专用工作负载

> 特殊场景：边缘计算、存量 EKS、无集群容器实例。通用免 CVM 算力优先标准集群 + 虚拟节点。

## 是什么

TKE 标准集群覆盖通用场景。本目录覆盖边缘与存量 EKS / 容器实例；**新建免 CVM、仍要 K8s 编排**时，主路径是 [标准集群](../clusters/create.md) + [虚拟节点](../nodes/virtual-nodes.md)，不是本目录里的 `CreateEKSCluster`。

## 触发条件

- 你要在边缘位置（IoT/门店/CDN）运行 K8s 集群，节点处于弱网环境 — 新建走 [标准集群](../clusters/create.md) + [注册节点公网版](https://cloud.tencent.com/document/product/457/57916)；已有 TKEEdge 资源的查询、迁移和受保护清理见 [边缘集群](edge-cluster.md)
- 你已有存量 EKS 集群（`DescribeEKSClusters` 返回资源），需查询/扩缩容/删除 — 去 [EKS / 容器实例](eks-cluster.md)
- 你要运行单次或常驻容器，不要 K8s 控制面 — 用 `CreateEKSContainerInstances`，见 [容器实例](eks-cluster.md#创建容器实例-部署-pod)
- 你要新建免 CVM、仍要 K8s 编排，不确定用 `CreateEKSCluster` 还是 `CreateCluster` + 虚拟节点 — 看 [选型指南](#选型指南)

## 选型指南

| 场景 | 方案 | 何时选择 | 计费 | 状态 |
|:-----|:-----|:---------|:-----|:-----|
| 标准 Web 服务/微服务 | [TKE 标准集群](../clusters/index.md) | 固定流量，需完整 K8s | 集群管理费 + 节点费 | — |
| 边缘计算（IoT/门店/CDN） | [边缘集群 TKEEdge](edge-cluster.md) | 节点在弱网/边缘位置 | 集群费 + 边缘节点费 | ⚠️ 已迁移到[注册节点公网版](https://cloud.tencent.com/document/product/457/57916) |
| **新建**免 CVM、要 K8s 编排 | [虚拟节点](../nodes/virtual-nodes.md)（先 [CreateCluster](../clusters/create.md)） | 标准集群内按 Pod 用量扩容 | Pod 按用量 | **推荐新建路径**（两步，非一次 CreateCluster） |
| 无集群、单次/常驻容器 | [容器实例](eks-cluster.md#创建容器实例-部署-pod) | 不要 K8s 控制面 | 按实例规格用量 | 控制台「容器实例」CPU/GPU |
| 存量 EKS 集群运维 | [EKS / 容器实例](eks-cluster.md) | 已有 `DescribeEKSClusters` 资源 | 按 Pod 用量 | ⚠️ 新建入口已关闭 |

## 核心概念

| 概念 | 含义 | 主 Action | 为什么重要 |
|:-----|:-----|:----------|:-----|
| TKEEdge（存量） | 已下线的边缘 K8s 集群 | `DescribeTKEEdgeClusters` / `*TKEEdge*` 存量运维接口；`CreateTKEEdgeCluster` 仅用于识别旧脚本，禁止调用新建 | 新建改用标准集群 + 注册节点公网版 |
| 虚拟/超级节点 | 标准集群内免 CVM 容量 | `CreateCluster` + `*VirtualNode*` / `CreateNodePool Type=Super` | **新建**免 CVM 编排负载的推荐路径 |
| EKS 集群（存量） | 独立 Serverless K8s 控制面 | `*EKSCluster*` | 新建已关；勿与 `CreateCluster` 混用 |
| EKS 容器实例 | 无集群容器（控制台 CPU/GPU 实例） | `*EKSContainerInstance*` | 无 `ClusterId`；与虚拟节点、EKS 集群都不同 |

## 标准集群 vs 专用工作负载

| 维度 | 标准集群 + CVM | 标准集群 + 虚拟节点 | EKS 集群（存量） | 容器实例 | 边缘集群 |
|:-----|:---------------|:--------------------|:-----------------|:---------|:---------|
| 首个操作 | `CreateCluster` | `CreateCluster` 再虚拟节点 | `DescribeEKSClusters` | `CreateEKSContainerInstances` | 新建：`CreateCluster` 后接注册节点；存量：`DescribeTKEEdgeClusters` |
| 节点 | 自管/托管 CVM | eklet 虚拟节点 | 无自管 CVM | 无 K8s Node | 边缘机器注册 |
| 计费 | 集群费 + CVM | 集群费 + Pod 用量 | 按 Pod 用量 | 按实例规格 | 集群费 + 边缘机 |
| 适用 | 生产常态 | **新建**免 CVM 编排 | 存量运维 | 无编排任务 | 边缘场景 |

## 不适用场景

- 标准数据中心容器工作负载 → 用 [标准集群](../clusters/index.md)，不需专用
- 已有自建 K8s → 不需 EKS，考虑标准集群的 [注册节点池](../nodes/registered-nodes/overview.md)
- **新建**免 CVM、要编排 → [虚拟节点](../nodes/virtual-nodes.md)（先 `CreateCluster`），**不要** `CreateEKSCluster`
- 不要集群、只要容器 → [容器实例](eks-cluster.md#创建容器实例-部署-pod)，**不要**当成虚拟节点

## 快速检查

```bash
# 查看已有的 EKS 集群（返回 Clusters[]）
tccli tke DescribeEKSClusters --region ap-guangzhou --Limit 3 \
  --filter "Clusters[].{id:ClusterId,status:Status}"
# expected: EKS 集群列表，Status 含 Running

# 查看边缘集群（须用 Edge 支持地域；ap-guangzhou 返回 UnsupportedRegion）
tccli tke DescribeTKEEdgeClusters --region <EDGE_REGION> --Limit 1
# expected: TotalCount + Clusters（无边缘集群则 0）；UnsupportedRegion → 换 Edge 地域（如 ap-beijing）
# <EDGE_REGION> 定义见 [边缘集群](edge-cluster.md)
```

## 文档

- [边缘集群](edge-cluster.md) — TKEEdge 存量查询、迁移与受保护清理；新建改用标准集群 + 注册节点公网版
- [EKS / 容器实例](eks-cluster.md) — 存量 EKS 集群运维 + 控制台「CPU/GPU 实例」（`CreateEKSContainerInstances`）
- [虚拟节点 (超级节点)](../nodes/virtual-nodes.md) — 标准集群内免 CVM 容量（新建推荐；先 `CreateCluster`）
- [标准集群概览](../clusters/index.md) — 对比标准集群
- [节点池](../nodes/index.md) — 标准集群的节点管理
