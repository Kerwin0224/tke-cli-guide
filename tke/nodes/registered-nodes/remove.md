---
doc_type: How-to
subtype: 6A
fused: true
---
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)
>
> 配额：无额外配额限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：DeleteExternalNode 不可逆、数据丢失；未 Drain 先 Delete 致 Pod 残留孤儿；DeleteExternalNodePool 误删致全池节点丢失且不可恢复。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

# 移除注册节点

> 优雅驱逐节点上的 Pod 并删除注册节点或节点池。
> 控制台: [容器服务 - 节点池 - 注册节点](https://console.cloud.tencent.com/tke2/nodepool)

移除注册节点分两步：先 `DrainExternalNode` 优雅驱逐 Pod，再 `DeleteExternalNode` 删除节点。删除节点池用 `DeleteExternalNodePool`。

## 触发条件

- 注册节点需下线（机器回收、维护或替换）。
- 注册节点池需删除。

## 准备工作

- 已注册节点（见[创建注册节点（专线版）](dedicated-line.md)）。
- 已配置 tccli 凭证（见[配置凭证](../../../getting-started/credentials.md)）。

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `ClusterId` | DrainExternalNode / DeleteExternalNode / DeleteExternalNodePool | 是 | 集群 ID |
| `Name` | DrainExternalNode | 是 | 节点名（**单数**） |
| `Names` | DeleteExternalNode | 是 | 节点名数组（**复数** `Names[]`，勿与 Drain 的 `Name` 混用） |
| `NodePoolIds` | DeleteExternalNodePool | 是 | 节点池 ID 数组 |
| `Force` | DeleteExternalNode / DeleteExternalNodePool | 否 | 是否强制删除（默认 false 优雅） |

> `DrainExternalNode` 用单数 `Name`；`DeleteExternalNode` 用 `Names[]`；`DeleteExternalNodePool` 用 `NodePoolIds[]`。后两者支持 `Force`。

## 操作步骤

### 步骤 1：优雅驱逐节点 Pod

```bash
tccli tke DrainExternalNode --ClusterId <CLUSTER_ID> --Name <NODE_NAME> --region <REGION>
# expected: exit 0
```

### 步骤 2：删除节点

```bash
tccli tke DeleteExternalNode --ClusterId <CLUSTER_ID> --Names '["<NODE_NAME>"]' --Force false --region <REGION>
# expected: exit 0
```

### 步骤 3：删除节点池（如不再需要）

```bash
tccli tke DeleteExternalNodePool --ClusterId <CLUSTER_ID> --NodePoolIds '["<NODEPOOL_ID>"]' --Force false --region <REGION>
# expected: exit 0
```

## 验证

```bash
# 节点已删除
tccli tke DescribeExternalNode --ClusterId <CLUSTER_ID> \
  --NodePoolId <NODEPOOL_ID> --region <REGION> \
  --filter "Nodes[?Name=='<NODE_NAME>']"
# expected: 返回空（节点已移除）

# 节点池已删除
tccli tke DescribeExternalNodePools --ClusterId <CLUSTER_ID> --region <REGION> \
  --filter "NodePoolSet[?NodePoolId=='<NODEPOOL_ID>']"
# expected: 返回空（节点池已移除）
```

## 清理

> **不可逆警告**：注册节点与节点池删除后不可恢复，集群侧记录随之清除。`DrainExternalNode` 是删除前置——先驱逐 Pod 再 `DeleteExternalNode`，避免 Pod 被强制终止导致业务中断。节点池删除后关联节点一并清除且计费即止。

删除即清理，本篇操作步骤即清理操作，无独立回滚命令：

```bash
# 删除节点（须先 DrainExternalNode 驱逐 Pod，见 ## 操作步骤 步骤 1）
tccli tke DeleteExternalNode --ClusterId <CLUSTER_ID> --Names '["<NODE_NAME>"]' --Force false --region <REGION>
# expected: exit 0

# 删除节点池（关联节点一并清除，不可恢复）
tccli tke DeleteExternalNodePool --ClusterId <CLUSTER_ID> --NodePoolIds '["<NODEPOOL_ID>"]' --Force false --region <REGION>
# expected: exit 0
```

> 如需重新接入节点，见[创建注册节点（专线版）](dedicated-line.md)。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `DeleteExternalNode` 报节点不存在 | `Names[]` 用了节点池 ID 而非节点名 | 用 `DescribeExternalNode` 返回的节点名 |
| `Force=true` 后 Pod 丢失 | 未先驱逐直接强制删除 | 先 `DrainExternalNode` 再删除 |

## 收尾确认

```bash
tccli tke DescribeExternalNodePools --ClusterId "<CLUSTER_ID>" --region ap-guangzhou \
  --filter "NodePoolSet[?NodePoolId=='<NODEPOOL_ID>'].LifeState"
# expected: 空（节点池已删除，关联节点清零，无持续计费）
```

## 下一步

- 重新接入：[创建注册节点（专线版）](dedicated-line.md)
- 编辑节点池：[编辑注册节点池](edit-pool.md)
