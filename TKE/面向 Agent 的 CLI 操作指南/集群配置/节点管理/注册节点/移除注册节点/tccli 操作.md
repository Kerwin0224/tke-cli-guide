# 移除注册节点

> 对照官方：[移除注册节点](https://cloud.tencent.com/document/product/457/79767) · page_id `79767` · tccli ≥3.1.107 · API 2018-05-25

## 概述

移除已注册到 TKE 集群的外部节点。支持两种粒度：删除单个注册节点（`DeleteExternalNode`）或删除整个节点池（`DeleteExternalNodePool`）。

删除节点前，建议先驱逐（`DrainExternalNode`）节点上的 Pod，避免业务中断。使用 `Force=true` 可强制删除，绕过删除保护（节点池级别）并跳过优雅终止。删除操作不可逆，执行前务必确认目标节点或节点池。

> 注意：`DeleteExternalNode` 对不存在的节点返回成功（幂等），不会报错。`DeleteExternalNodePool` 删除节点池时会同时删除池内所有注册节点。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters
#    tke:DescribeExternalNodePools
#    tke:DescribeExternalNode
#    tke:DrainExternalNode
#    tke:DeleteExternalNode
#    tke:DeleteExternalNodePool
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "<ClusterId>",
            "ClusterName": "<ClusterName>",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.32.2"
        }
    ]
}
```

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "<ClusterId>",
            "ClusterName": "<ClusterName>",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.32.2"
        }
    ]
}
```

```bash
# 5. 确认目标节点池存在
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回节点池列表，找到目标 NodePoolId
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "<NodePoolId>",
            "Name": "<NodePoolName>",
            "LifeState": "normal",
            "NodeCount": 0,
            "NodeType": "CPU",
            "ClusterId": null
        }
    ]
}
```

```bash
# 6. 确认目标节点存在（如仅删除单个节点）
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 返回节点列表，找到目标节点 Name
```

**预期输出**（含节点时）：

```json
{
    "TotalCount": 1,
    "Nodes": [
        {
            "Name": "<NodeName>",
            "NodePoolId": "<NodePoolId>",
            "IP": "10.0.0.1",
            "Location": "IDC",
            "Status": "Running"
        }
    ]
}
```

**预期输出**（空节点池时）：

```json
{
    "TotalCount": 0,
    "Nodes": []
}
```

## 控制台与 CLI 参数映射

以下说明移除注册节点相关 API 的主要参数及与控制台的对应关系。

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点池列表 | `DescribeExternalNodePools` | 是 |
| 查看注册节点列表 | `DescribeExternalNode` | 是 |
| 驱逐节点（Pod 迁移） | `DrainExternalNode` | 是 |
| 删除单个注册节点 | `DeleteExternalNode` | 是（对不存在节点返回成功） |
| 删除节点池及所有节点 | `DeleteExternalNodePool` | 是（同上） |

### DrainExternalNode 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameter.ClusterId` |
| `Name` | String | 是 | 节点名称（非节点池 ID） | 填节点池 ID → `InvalidParameter.NodeName` |

### DeleteExternalNode 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameter.ClusterId` |
| `Names` | Array | 是 | 节点名称列表，支持批量删除 | 节点不存在 → 幂等返回成功（不报错） |
| `Force` | Boolean | 否 | 默认 `false`。`true` 时跳过优雅终止 | 设为 `false` 时若节点不可达可能删除失败 |

### DeleteExternalNodePool 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameter.ClusterId` |
| `NodePoolIds` | Array | 是 | 节点池 ID 列表，支持批量删除。格式 `np-xxxxxxxx` | 节点池不存在 → 幂等返回成功 |
| `Force` | Boolean | 否 | 默认 `false`。`true` 时跳过删除保护检查并强制删除节点池及所有节点 | 设 `false` 且删除保护开启 → `OperationDenied` |

## 操作步骤

### 步骤 A：删除单个注册节点

#### 选择依据

- **节点驱逐**：删除前先驱逐 Pod 可避免业务中断。`DrainExternalNode` 会将节点上的 Pod 迁移到其他可用节点。
- **删除幂等性**：`DeleteExternalNode` 对不存在的节点返回成功，不会报错。两次删除同一节点，第二次也返回 exit 0。
- **参数注意**：`DeleteExternalNode` 的 `--Names` 参数填**节点名称**（如 `node-xxx`），不是节点池 ID（`np-xxx`）。需先通过 `DescribeExternalNode` 确认节点名称。

#### 步骤 A-1：驱逐节点上的 Pod（推荐）

```bash
tccli tke DrainExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --Name <NodeName>
# expected: exit 0，返回 RequestId，节点上的 Pod 被迁移到其他节点
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 步骤 A-2：删除注册节点

```bash
cat > delete-node.json <<'EOF'
{
    "ClusterId": "<ClusterId>",
    "Names": ["<NodeName>"]
}
EOF
tccli tke DeleteExternalNode --region <Region> \
    --cli-input-json file://delete-node.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> `DeleteExternalNode` 对不存在的节点返回成功（幂等）。如果误填节点池 ID 而非节点名称，API 同样返回成功但不会删除任何节点——务必确认 `Names` 中的值为节点名称。

### 步骤 B：删除整个节点池

#### 选择依据

- **直接删除 vs 先逐节点驱逐**：删除节点池（`DeleteExternalNodePool`）会同时删除池内所有注册节点，操作不可逆。生产环境建议先逐节点驱逐，确认 Pod 已重新调度后再删除节点池。
- **Force 参数**：`Force=true` 跳过删除保护检查，会强制删除节点池及其所有节点。仅在确认可以安全删除时使用。

#### 步骤 B-1：查询节点池中的节点

```bash
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 列出节点池内所有节点
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "Nodes": []
}
```

#### 步骤 B-2：逐节点驱逐（推荐）

```bash
# 对每个 NODE_NAME 执行：
tccli tke DrainExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --Name <NodeName>
# expected: exit 0
```

#### 步骤 B-3：删除节点池

```bash
cat > delete-pool.json <<'EOF'
{
    "ClusterId": "<ClusterId>",
    "NodePoolIds": ["<NodePoolId>"],
    "Force": true
}
EOF
# ⚠️ Force=true 强制删除，跳过删除保护检查
tccli tke DeleteExternalNodePool --region <Region> \
    --cli-input-json file://delete-pool.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | `tccli tke DescribeExternalNodePools --region <Region> --ClusterId <ClusterId>` |
| `<NodeName>` | 节点名称 | 字符串，非节点池 ID | `tccli tke DescribeExternalNode --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` |

## 验证

### 删除后验证

```bash
# 1. 确认节点已删除（如删除单个节点）
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 目标节点不在返回列表中
```

**预期输出**（节点已删除）：

```json
{
    "TotalCount": 0,
    "Nodes": []
}
```

```bash
# 2. 确认节点池已删除（如删除节点池）
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 目标节点池不在返回列表中

# 3. 确认所有注册节点已清除（如删除全部）
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId <ClusterId>
# expected: TotalCount 为 0
```

**预期输出**（节点池已删除）：

```json
{
    "TotalCount": 0,
    "NodePoolSet": []
}
```

| 验证维度 | 命令 | 预期 |
|------|------|------|
| 节点已删除 | `DescribeExternalNode` 指定 `NodePoolId` | 目标节点不出现在 `Nodes` 列表中 |
| 节点池已删除 | `DescribeExternalNodePools` | 目标节点池不出现在 `NodePoolSet` 列表中 |
| 所有节点已清空 | `DescribeExternalNode` 不指定 `NodePoolId` | `TotalCount=0` 且 `Nodes` 为空数组 |

## 清理

> **⚠️ 警告**：`DeleteExternalNodePool` 删除节点池时会同时删除池内所有注册节点，操作不可逆。使用 `Force=true` 跳过删除保护检查，会强制删除节点池及所有节点。被驱逐的 Pod 需要集群中有其他可用节点才能重新调度，否则 Pod 将处于 Pending 状态。`DrainExternalNode` 驱逐的是节点上的 Pod，不删除节点本身——先驱逐再删除节点是安全操作顺序。

### 1. 清理前状态检查

```bash
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# 确认是待删除的目标节点池，记录 NodePoolId、Name、NodeCount
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "<NodePoolId>",
            "Name": "<NodePoolName>",
            "LifeState": "normal",
            "NodeCount": 0,
            "NodeType": "CPU"
        }
    ]
}
```

### 2. 验证已删除

```bash
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 目标节点池不在 NodePoolSet 中
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "NodePoolSet": []
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DrainExternalNode` 返回 `InvalidParameter.NodeName` | `tccli tke DescribeExternalNode --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 列出所有节点 | `Name` 参数填了节点池 ID 或其他字符串 | `Name` 应填节点名称（从 `DescribeExternalNode` 输出中的 `Name` 字段获取），不是节点池 ID |
| `DeleteExternalNodePool` 返回 `OperationDenied` | `tccli tke DescribeExternalNodePools --region <Region> --ClusterId <ClusterId>` 查看 `DeletionProtection` | 节点池开启了删除保护 | 使用 `Force=true` 跳过删除保护检查 |
| 删除后节点仍出现在列表中 | `tccli tke DescribeExternalNode --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` | 删除操作是异步的，状态更新有延迟 | 等待 1-2 分钟后重试 |
| `DeleteExternalNode` 传入不存在的节点名 | 直接检查入参 | 此为正常幂等行为——API 返回成功 | 无需修复。若需确认节点是否真实存在，先用 `DescribeExternalNode` 验证 |

### 驱逐后 Pod 无法重新调度

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 驱逐后处于 Pending | `kubectl get pods -A \| grep Pending` 查看 Pending Pod | 集群中无其他可用节点或资源不足 | 添加新节点或扩容现有节点池；检查 Pod 的 `nodeSelector`/`nodeAffinity` 是否仅匹配已删除的节点 |
| 有状态 Pod 驱逐后数据丢失 | `kubectl describe pod <PodName>` 查看卷挂载 | 使用 `emptyDir` 等临时卷的 Pod 被驱逐 | 确保有状态 Pod 使用持久化存储（PVC）；`emptyDir` 数据随 Pod 迁移丢失是正常行为 |

## 下一步

- [编辑注册节点池](https://cloud.tencent.com/document/product/457/79765) — 修改节点池配置
- [创建注册节点（专线版）](https://cloud.tencent.com/document/product/457/57917) — 重新创建注册节点
- [创建注册节点（公网版）](https://cloud.tencent.com/document/product/457/101532) — 公网版创建注册节点
- [注册节点常见问题](https://cloud.tencent.com/document/product/457/79750) — 排障与 FAQ

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 注册节点](https://console.cloud.tencent.com/tke2/cluster)：选择节点池或节点执行删除操作。
