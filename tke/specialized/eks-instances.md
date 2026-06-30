---
doc_type: How-to
subtype: 6A
fused: true
---
# 虚拟节点 (超级节点)

> 在标准 TKE 集群中添加 Serverless 节点 —— Pod 无需 CVM 直接运行。
> 控制台: [容器服务 - 虚拟节点](https://console.cloud.tencent.com/tke2/virtual-node)

## 概述

虚拟节点 (VirtualNode，也称超级节点) 让你在**标准 TKE 集群**中使用 Serverless 能力。它表现为一个"无限容量"的节点：
Pod 调度到虚拟节点时直接使用弹性容器实例 (EKS)，不需要实际的 CVM。

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
# expected: { "Response": { "NodeName": "eklet-xxx" } }
```

### 查询虚拟节点

```bash
tccli tke DescribeClusterVirtualNode \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: 虚拟节点列表，查看节点状态
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
  --SubnetIds '["<SUBNET_ID>"]'

# 查询虚拟节点池
tccli tke DescribeClusterVirtualNodePools \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"

# 修改虚拟节点池
tccli tke ModifyClusterVirtualNodePool \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --NodePoolId "<POOL_ID>" \
  --Labels '[{"Name":"workload-type","Value":"serverless"}]'
```

### 更新 EKS 容器实例

> 修改已创建的 EKS 容器实例（Serverless Pod）的容器定义、重启策略。参数以 `--generate-cli-skeleton` 实测为准（`EksCiId` 定位 + `Containers[]` 覆盖）。

```bash
# 更新容器实例（EksCiId 定位，Containers[] 覆盖式更新镜像/资源，RestartPolicy 重启策略）
tccli tke UpdateEKSContainerInstance --region ap-guangzhou \
  --EksCiId "<EKSCI_ID>" \
  --RestartPolicy "Always" \
  --Containers '[{"Name":"nginx","Image":"nginx:1.25","Cpu":0.5,"Memory":1.0}]'
# expected: exit 0
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<EKSCI_ID>` | EKS 容器实例 ID | `tccli tke DescribeEKSContainerInstances --ClusterId "<CLUSTER_ID>"` → `EksCis[].EksCiId` |

> `UpdateEKSContainerInstance` 用 `EksCiId`（单数）定位单个实例，`Containers[]` 是覆盖式整体替换（非增量），调用前先 `DescribeEKSContainerInstances` 取当前容器定义再改，避免遗漏。`RestartPolicy` 取值如 `Always`/`OnFailure`/`Never`。与 `RestartEKSContainerInstances`（重启，不改定义）区别。

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

## API 参考

完整的虚拟节点 API (含节点池):

| 分类 | API | 说明 |
|------|-----|------|
| 节点 | `CreateClusterVirtualNode` / `DeleteClusterVirtualNode` / `DescribeClusterVirtualNode` | CRUD |
| 排水 | `DrainClusterVirtualNode` | 驱逐 Pod |
| 节点池 | `CreateClusterVirtualNodePool` / `DeleteClusterVirtualNodePool` / `DescribeClusterVirtualNodePools` / `ModifyClusterVirtualNodePool` | 节点池管理 |

## EKS 日志配置

> EKS 容器实例的日志采集配置（投递到 CLS）。

```bash
# 创建 EKS 日志配置 (LogConfig 采集规则 + LogsetId CLS 日志集)
tccli tke CreateEksLogConfig --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --LogConfig "<LOG_CONFIG_JSON>" --LogsetId "<LOGSET_ID>"
# expected: exit 0
```

> `CreateEksLogConfig` 的 `LogConfig` 是采集规则 JSON，`LogsetId` 是 CLS 日志集 ID（见 [CLS 服务](https://cloud.tencent.com/document/product/614)）。EKS 日志区别于普通集群日志（见 [日志采集](../observability/logging.md)）。

## 下一步

- [EKS 弹性集群](eks-cluster.md) — Serverless 集群（需集群时用）
- [边缘集群](edge-cluster.md) — 边缘计算场景
- [创建节点池](../nodes/nodepool-create.md) — 标准节点池

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `CreateEKSContainerInstances` | 主操作 | 2018-05-25 | 创建容器实例（Serverless Pod） |
| `UpdateEKSContainerInstance` | 主操作 | 2018-05-25 | 更新容器实例 |
| `RestartEKSContainerInstances` | 主操作 | 2018-05-25 | 重启容器实例 |
| `CreateClusterVirtualNode` | 主操作 | 2018-05-25 | 创建虚拟节点 |
| `CreateClusterVirtualNodePool` | 主操作 | 2018-05-25 | 创建虚拟节点池 |
| `DrainClusterVirtualNode` | 主操作 | 2018-05-25 | 排水虚拟节点（迁移 Pod） |
| `ModifyClusterVirtualNodePool` | 主操作 | 2018-05-25 | 修改虚拟节点池 |
| `CreateEksLogConfig` | 主操作 | 2018-05-25 | 创建 EKS 日志配置（投递 CLS） |
| `DeleteEKSContainerInstances` | 清理 | 2018-05-25 | 删除容器实例 |
| `DeleteClusterVirtualNode` | 清理 | 2018-05-25 | 删除虚拟节点 |
| `DeleteClusterVirtualNodePool` | 清理 | 2018-05-25 | 删除虚拟节点池 |
| `DescribeEKSContainerInstances` | 验证 | 2018-05-25 | 容器实例列表 |
| `DescribeEKSContainerInstanceEvent` | 验证 | 2018-05-25 | 容器实例事件 |
| `DescribeEksContainerInstanceLog` | 验证 | 2018-05-25 | 容器实例日志 |
| `DescribeEKSContainerInstanceRegions` | 验证 | 2018-05-25 | 支持地域 |
| `DescribeClusterVirtualNode` | 验证 | 2018-05-25 | 虚拟节点状态 |
| `DescribeClusterVirtualNodePools` | 验证 | 2018-05-25 | 虚拟节点池列表 |
