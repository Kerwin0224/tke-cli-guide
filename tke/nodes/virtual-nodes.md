---
doc_type: How-to
subtype: 6A
fused: true
---
# 虚拟节点 (超级节点)

> 在**已有标准 TKE 集群**上添加免 CVM 容量：Pod 调度到虚拟/超级节点，按用量计费。
> 控制台: [容器服务 - 虚拟节点](https://console.cloud.tencent.com/tke2/virtual-node)

> ⚠️ **两步，不是一次 `CreateCluster`**：① [创建标准集群](../clusters/create.md)（`CreateCluster`，空控制面）→ ② 本文创建虚拟节点池/节点。`CreateCluster` **不会**单独变成「Serverless 集群」。
>
> 与另外两条路径区分：存量 **EKS 集群**（`CreateEKSCluster`，新建已关）与 **容器实例 CPU/GPU**（`CreateEKSContainerInstances`，无集群）见 [EKS / 容器实例](../specialized/eks-cluster.md)。三者 Action 不同，勿混用。
>
> 官方文档：[超级节点资源规格](https://cloud.tencent.com/document/product/457/39808) · [节点概述](https://cloud.tencent.com/document/product/457/32201)
>
> 配额：超级节点按量计费无上限，需关注成本。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：按量计费无上限、DrainClusterVirtualNode 驱逐 Pod 需确认业务可中断、删除节点池丢失所有虚拟节点。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 触发条件

- 标准集群内需要免 CVM、按 Pod 用量计费的容量 — 创建虚拟节点 / 超级节点池
- `DescribeClusterVirtualNodePools` 返回空，或需新增超级节点池
- 虚拟节点问题（Pod Pending / 排水失败 / 子网 IP 不足）— 看 [故障恢复](#故障恢复)

## 准备工作

- 已创建 TKE **标准**集群 (见 [创建集群](../clusters/create.md)；`CreateCluster`，非 `CreateEKSCluster`)
- 已配置 tccli 凭证 (见 [配置凭证](../../getting-started/credentials.md))

## 概述

虚拟节点（VirtualNode，也称超级节点，节点池 `Type=Super`）把免 CVM 容量加进**标准 TKE 集群**：Pod 调度到虚拟节点后由弹性容器运行时承载，不占用你自管的 CVM。

| 对比项 | 虚拟节点（本文） | EKS 集群（存量） | EKS 容器实例 |
|:-------|:-----------------|:-----------------|:-------------|
| 是否先有标准集群 | 是（`CreateCluster` 之后） | 否（独立控制面） | 否（无 `ClusterId`） |
| 主 Action | `CreateClusterVirtualNode(Pool)` / `CreateNodePool Type=Super` | `*EKSCluster*` | `*EKSContainerInstance*` |
| 控制台 | 集群内「虚拟节点」 | 新建入口已关 | 「容器实例」CPU/GPU |
| 文档 | 本文 | [EKS / 容器实例](../specialized/eks-cluster.md) | 同左 · 创建容器实例段 |

| 特性 | 虚拟节点 | 普通节点 (CVM) |
|------|:---:|:---:|
| 容量 | 按 Pod 规格弹性分配（见下表） | 固定 CVM 规格 |
| 计费 | Pod 按使用量 | CVM 按实例费 |
| 运维 | 零运维 | 需管理 OS/安全/升级 |
| 冷启动 | 有 (~10s) | 无 |

### Pod 资源规格半常量（计费与调度）

> 超级节点/Serverless Pod 须在支持规格内分配；规格是 Pod 内容器可用的**最大**资源量（含 OS/运行时/日志等系统占用）。**所有 Container 的 Request 之和**与**任一 Container 的 Limit**均不可大于所选最大 Pod 规格。可用 Annotation 显式指定；不指定时按 Request/Limit 推算。完整表与 GPU 型号见 [资源规格](https://cloud.tencent.com/document/product/457/39808)。

**Intel CPU（节选）**

| CPU/核 | 内存区间/GiB |
|:------:|:-------------|
| 0.25 | 0.5、1、2 |
| 0.5 | 0.5、1、2、3、4 |
| 1 | 1–8（粒度 1） |
| 2 | 2、4–16（粒度 1） |
| 4 | 8–32（粒度 1） |
| 8 | 16–64（粒度 1） |
| 16 | 16–128（粒度 1） |
| 32 | 64、128、256 |
| 64 | 128、192、256、512 |

另有星星海 AMD 与 GPU（V100/T4 等）规格；GPU 须 Annotation 指定 `gpu-type`。数值为半常量，以官网最新表为准。

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
  --SecurityGroupIds '["<SECURITY_GROUP_ID>"]' \
  --SubnetIds '["<SUBNET_ID>"]' \
  --OS linux \
  --VirtualNodes '[{"NodeName":"<NODE_NAME>"}]'
# expected: { "NodePoolId": "np-xxx" }（OS 默认 linux，传 windows 建 Windows 虚拟节点池）
```

| 参数 | 必填 | 说明 |
|------|:----:|------|
| `ClusterId` | 是 | 集群 ID |
| `Name` | 是 | 节点池名 |
| `SecurityGroupIds` | 是 | 安全组列表 |
| `SubnetIds` | 否 | 子网列表（与 `SubnetAllocationPolicy` 二选一） |
| `VirtualNodes` | 否 | 虚拟节点规格数组（`VirtualNodeSpec`，含 `NodeName` 等） |
| `OS` | 否 | `linux`（默认）/`windows` |
| `Labels`/`Taints` | 否 | 见 [共享字段](../reference/shared-fields.md#label-taint-annotation) |
| `SubnetAllocationPolicy` | 否 | 子网分配策略（多子网均匀分配） |
| `AgentPlugin` | 否 | Agent 插件配置 |

> `VirtualNodes` 指定虚拟节点的规格（NodeName 等）；`SubnetAllocationPolicy` 用于多子网场景自动分配。Windows 虚拟节点池需集群支持 Windows 节点。

```bash
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
# expected: exit 0
```

## 验证

异步操作，检查 ≥4 个维度：

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- kubectl验证虚拟节点已加入集群(kubectl get nodes查K8s Node对象)，非tccli边界 -->
```bash
# 维度1: 虚拟节点已加入集群 (kubectl 查 K8s Node 对象; tccli 查的是节点池抽象, 这里用 kubectl 看节点实物)
kubectl get nodes --show-labels | grep super
# expected: 含虚拟节点 (eklet-xxx), 标签 super=true

# 维度2: 虚拟节点池状态
tccli tke DescribeClusterVirtualNodePools --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "NodePoolSet[].{name:Name,state:LifeState,subnet:SubnetIds}"
# expected: LifeState=normal, SubnetIds 与创建参数一致
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 节点加入集群 | `kubectl get nodes --show-labels` | 含 `eklet-xxx` 虚拟节点，标签 `super=true` |
| 节点池状态 | `DescribeClusterVirtualNodePools` → `LifeState` | `normal` |
| 子网配置 | `DescribeClusterVirtualNodePools` → `SubnetIds` | 与创建参数一致 |
| 节点 Ready | `kubectl get nodes` | 虚拟节点 `Ready`（eklet 注册成功） |

> 虚拟节点在 kubectl get nodes 可见 + LifeState=normal + Ready = 创建成功。

---

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

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- tccli管虚拟节点池CRUD，kubectl验证节点Ready并测试Pod调度到虚拟节点(K8s层观测)，非tccli边界 -->
```bash
# 端到端核对：虚拟节点 eklet-xxx Ready 且 Pod 可调度到虚拟节点（仅查 LifeState 不够，还须确认 Pod 可调度）
kubectl get nodes | grep eklet
# expected: 含 eklet-xxx 虚拟节点且 Ready（非仅节点池 LifeState=normal）

kubectl run nginx-test --image=nginx --restart=Never --overrides='{"spec":{"nodeName":"<EKLET_NODE_NAME>"}}'
# expected: Pod Pending→Running（Pod 成功调度到虚拟节点）
kubectl delete pod nginx-test
```

> 虚拟节点池 LifeState=normal + eklet-xxx 节点 Ready + Pod 可调度到虚拟节点 = 虚拟节点闭环完成。

---

## 下一步

- [创建节点池](nodepool-create.md) — 标准 CVM 节点池
- [扩缩容节点池](nodepool-scale.md) — 节点池弹性伸缩
- [EKS / 容器实例](../specialized/eks-cluster.md) — 存量 EKS 集群运维；无集群容器实例（与本文虚拟节点不同）
