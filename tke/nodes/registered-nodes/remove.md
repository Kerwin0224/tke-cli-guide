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

移除单个注册节点时，先 `DrainExternalNode` 优雅驱逐 Pod，再 `DeleteExternalNode` 删除节点。删除整个节点池前，也必须先枚举池内节点并逐一完成驱逐；`DeleteExternalNodePool --Force false` 在节点上仍有 Pod 时会删除失败。

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
| `Names` | DeleteExternalNode | 是 | 节点名数组（**复数** `Names[]`，不要与 Drain 的 `Name` 混用） |
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

先查询池内全部节点：

```bash
tccli tke DescribeExternalNode --ClusterId <CLUSTER_ID> \
  --NodePoolId <NODEPOOL_ID> --region <REGION> \
  --filter "Nodes[].{name:Name,status:Status,unschedulable:Unschedulable}"
# expected: 记录返回的每个 Name；空数组表示池内已无注册节点
```

若返回节点，对每个 `<POOL_NODE_NAME>` 逐一驱逐，并等待其不再承载业务 Pod。`DrainExternalNode` 成功响应只有 `RequestId`，不代表驱逐已完成；还需在集群中确认工作负载已迁移，并检查 PDB 与本地数据约束：

```bash
tccli tke DrainExternalNode --ClusterId <CLUSTER_ID> \
  --Name <POOL_NODE_NAME> --region <REGION>
# expected: exit 0；随后 DescribeExternalNode 中该节点 Status 可能为 Draining

kubectl get pods -A --field-selector spec.nodeName=<POOL_NODE_NAME>
# expected: 除明确允许保留的 DaemonSet/静态 Pod 外，无业务 Pod
```

全部节点完成上述检查后，再删除节点池：

```bash
tccli tke DeleteExternalNodePool --ClusterId <CLUSTER_ID> --NodePoolIds '["<NODEPOOL_ID>"]' --Force false --region <REGION>
# expected: exit 0；若节点上仍有 Pod，非强制删除会失败
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

> **不可逆警告**：注册节点与节点池删除后不可恢复。删除整个节点池同样必须按步骤 3 先枚举节点、逐一驱逐并确认工作负载迁移，再执行非强制删除。

删除操作即清理动作，无独立回滚命令：

- 只下线单个节点时，执行步骤 1、步骤 2，并用“验证”中的节点查询确认记录为空。
- 删除整个节点池时，只执行步骤 3 的流程，不要另起一套删除步骤。
- 节点池查询为空只证明 TKE 中该节点池记录已不可见，不证明外部机器、Kubernetes Node 或关联云资源已经删除，也不证明账单已经停止。分别检查这些资源，并按对应产品的计费规则确认结算状态。

> 如需重新接入节点，见[创建注册节点（专线版）](dedicated-line.md)。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `DeleteExternalNode` 报节点不存在 | `Names[]` 用了节点池 ID 而非节点名 | 用 `DescribeExternalNode` 返回的节点名 |
| `Force=true` 后 Pod 丢失 | 未先驱逐直接强制删除 | 先 `DrainExternalNode` 再删除 |

## 下一步

- 重新接入：[创建注册节点（专线版）](dedicated-line.md)
- 编辑节点池：[编辑注册节点池](edit-pool.md)
