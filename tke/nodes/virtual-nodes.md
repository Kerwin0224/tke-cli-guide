---
doc_type: How-to
subtype: 6A
fused: true
---
# 虚拟节点 (超级节点)

> 在标准 TKE 集群中添加 Serverless 节点 —— Pod 无需 CVM 直接运行。
> 控制台: [容器服务 - 虚拟节点](https://console.cloud.tencent.com/tke2/virtual-node)

## 概述

虚拟节点 (VirtualNode，也称超级节点) 让你在**标准 TKE 集群**中使用 Serverless 能力。它表现为一个"无限容量"的节点：Pod 调度到虚拟节点时直接使用弹性容器实例 (EKS)，不需要实际的 CVM。

> 虚拟节点属**标准 TKE 集群**功能（。与 [EKS 弹性集群](../specialized/eks-cluster.md)（独立 Serverless 集群）不同——虚拟节点是把 Serverless 能力注入已有集群，EKS 是无集群的纯弹性容器实例。

| 特性 | 虚拟节点 | 普通节点 (CVM) |
|------|:---:|:---:|
| 容量 | 无限制 | 固定 CVM 规格 |
| 计费 | Pod 按使用量 | CVM 按实例费 |
| 运维 | 零运维 | 需管理 OS/安全/升级 |
| 冷启动 | 有 (~10s) | 无 |

## 关键操作

### 创建虚拟节点

在已有集群中添加虚拟节点:

```bash
tccli tke CreateClusterVirtualNode \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --NodePoolId "<NODE_POOL_ID>" \
  --SubnetId "<SUBNET_ID>"
# expected: { "NodeName": "eklet-xxx" }
```

### 查询虚拟节点

```bash
tccli tke DescribeClusterVirtualNode \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: exit 0, 虚拟节点列表
```

### 排水虚拟节点 (驱逐 Pod)

维护前将所有 Pod 迁移到其他节点:

```bash
tccli tke DrainClusterVirtualNode \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --NodeName "<NODE_NAME>"
# expected: exit 0
# Pod 会被重新调度到其他节点 (CVM 或虚拟节点)
```

### 管理虚拟节点池

```bash
# 创建虚拟节点池
tccli tke CreateClusterVirtualNodePool \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Name "<POOL_NAME>" \
  --SubnetIds '["<SUBNET_ID>"]' \
  --OS linux
# expected: { "NodePoolId": "np-xxx" }（OS 默认 linux，传 windows 建 Windows 虚拟节点池）
```

> `OS` 枚举：`linux`（默认）/ `windows`。Windows 虚拟节点池用于运行 Windows 容器（需集群支持 Windows 节点）。不传默认 linux。

# 查询虚拟节点池 (返回 TotalCount + NodePoolSet[])
tccli tke DescribeClusterVirtualNodePools \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: { "TotalCount": 0, "NodePoolSet": [], "RequestId": "..." }

# 修改虚拟节点池
tccli tke ModifyClusterVirtualNodePool \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --NodePoolId "<POOL_ID>" \
  --Labels '[{"Name":"workload-type","Value":"serverless"}]'
```

## 清理

```bash
# 1. 排水 (迁移 Pod)
tccli tke DrainClusterVirtualNode --ClusterId "<ID>" --NodeName "<NODE>"

# 2. 删除虚拟节点
tccli tke DeleteClusterVirtualNode --ClusterId "<ID>" --NodeName "<NODE>"
# expected: exit 0

# 3. 删除虚拟节点池
tccli tke DeleteClusterVirtualNodePool --ClusterId "<ID>" --NodePoolId "<POOL>"
# expected: exit 0
```

## 故障恢复

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| Pod 卡在 Pending | `kubectl describe pod <NAME>` → Events | 虚拟节点未创建或子网 IP 不足 | `DescribeClusterVirtualNode` 检查节点状态 |
| 排水失败 | `DrainClusterVirtualNode` 返回错误 | Pod 有 PDB 限制或无法迁移 | 手动 `kubectl delete pod` 或放宽 PDB |
| 虚拟节点异常 | `DescribeClusterVirtualNode` | 子网不可用或欠费 | 检查子网状态，充值 |

## 下一步

- [创建节点池](nodepool-create.md) — 标准 CVM 节点池
- [扩缩容节点池](nodepool-scale.md) — 节点池弹性伸缩
- [EKS 弹性集群](../specialized/eks-cluster.md) — 独立 Serverless 集群（无集群直接跑 Pod）
