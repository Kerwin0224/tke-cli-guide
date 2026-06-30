---
doc_type: Overview
---
# 专用工作负载

> 特殊场景的容器方案：边缘计算、Serverless 容器。当标准集群不满足时选这些。

## 这是什么

TKE 标准集群覆盖大多数场景。专用工作负载针对特殊需求：边缘计算（节点在 IDC/门店）、Serverless（不想管节点）。

## 选型指南

| 场景 | 方案 | 何时选择 | 计费 |
|:-----|:-----|:---------|:-----|
| 标准 Web 服务/微服务 | [TKE 标准集群](../clusters/index.md) | 固定流量，需完整 K8s | 集群管理费 + 节点费 |
| 边缘计算（IoT/门店/CDN） | [边缘集群 TKEEdge](edge-cluster.md) | 节点在弱网/边缘位置 | 集群费 + 边缘节点费 |
| 批处理/定时/突发流量 | [EKS 弹性集群](eks-cluster.md) | 不想管节点，按 Pod 秒级计费 | 按 Pod 资源用量 |
| 无集群一次性任务 | [EKS 容器实例](eks-instances.md) | 单次任务，不需集群 | 按容器实例时长 |

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| TKEEdge | 边缘 K8s 集群 | 节点可部署在任意位置，适应弱网 |
| EKS 集群 | Serverless K8s 控制面 | 零运维节点，按 Pod 计费 |
| EKS 容器实例 | 无集群的容器运行 | 单次任务，不需维护集群 |

## 标准集群 vs 专用工作负载

| 维度 | 标准集群 | EKS 集群 | 边缘集群 |
|:-----|:---------|:---------|:---------|
| 节点管理 | 自管/托管 CVM | 无节点（Serverless） | 边缘机器注册 |
| 计费 | 集群费 + CVM | 按 Pod 用量 | 集群费 + 边缘机 |
| 网络 | VPC 内 | VPC 内 | 边缘弱网/公网 |
| 适用 | 生产常态 | 突发/批处理 | 边缘场景 |

## 不适用场景

- 标准数据中心容器工作负载 → 用 [标准集群](../clusters/index.md)，不需专用
- 已有自建 K8s → 不需 EKS，考虑标准集群的 [第三方节点池](../nodes/nodepool-create.md)
- 短暂一次性任务（不需集群）→ [EKS 容器实例](eks-instances.md)，直接创建容器不建集群

## 快速检查

```bash
# 查看已有的 EKS 集群（实测返回 Clusters[]）
tccli tke DescribeEKSClusters --region ap-guangzhou --Limit 3 \
  --filter "Clusters[].{id:ClusterId,status:Status}"
# expected: EKS 集群列表，Status 含 Running

# 查看边缘集群（账号下无则 TotalCount=0）
tccli tke DescribeTKEEdgeClusters --region ap-guangzhou --Limit 1
# expected: TotalCount + Clusters（无边缘集群则 0）
```

## 文档

- [边缘集群](edge-cluster.md) — TKEEdge 创建、管理、注册节点
- [EKS 弹性集群](eks-cluster.md) — Serverless 集群创建与凭证
- [EKS 容器实例](eks-instances.md) — 无集群直接创建容器
- [标准集群概览](../clusters/index.md) — 对比标准集群
- [节点池](../nodes/index.md) — 标准集群的节点管理
