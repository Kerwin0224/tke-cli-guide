---
doc_type: How-to
subtype: 6A
fused: true
---
# 删除集群

> ⚠️ **不可逆操作。** 删除集群会销毁所有工作节点 (CVM)，数据无法恢复。

## 副作用

删除集群时会影响以下资源:

| 资源 | 是否自动删除 | 需手动清理 |
|------|:----------:|------|
| 工作节点 (CVM) | ✅ 自动销毁 | — |
| 托管 Master | ✅ 自动回收 | — |
| CBS 云硬盘 | ❌ **保留** | `tccli cbs DeleteDisks` |
| 弹性公网 IP (EIP) | ❌ **保留** | `tccli vpc ReleaseAddresses` |
| CLB 负载均衡 | ❌ **保留** | 控制台或 CLB API |
| 集群内创建的 VPC 子网 | ❌ **保留** | `tccli vpc DeleteSubnet` |

> **计费提示**: 集群管理费在删除后立即停止。但保留的 CBS/EIP/CLB 会**持续扣费**。

## 准备工作

```bash
# 确认要删除的集群
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: 确认 ClusterName 与预期一致

# 检查删除保护状态
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterDeletionProtection" --output text
# expected: true（已开启删除保护，需先关闭）或 false（已关闭，可直接删）
```

## 操作步骤

### 步骤 1：关闭删除保护

如果创建时开启了删除保护，需要先关闭:

```bash
tccli tke DisableClusterDeletionProtection \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: exit 0
```

### 步骤 2：删除 — 最小化

`InstanceDeleteMode` 必填（不传报 `the following arguments are required: --InstanceDeleteMode`），控制节点 CVM 销毁方式：`terminate`（销毁）/ `retain`（保留）。最小化用 `terminate` 销毁节点，CBS 盘和 EIP 默认保留（手动清理）。

```bash
tccli tke DeleteCluster \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --InstanceDeleteMode terminate
# expected: exit 0
```

### 步骤 3：删除 — 增强：级联删除所有关联资源

连 CBS 盘、EIP 一起删（真正的一键清理）:

```bash
tccli tke DeleteCluster \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --InstanceDeleteMode terminate \
  --ResourceDeleteOptions '[
    {"ResourceType":"CBS","DeleteMode":"terminate","SkipDeletionProtection":false},
    {"ResourceType":"EIP","DeleteMode":"terminate","SkipDeletionProtection":false}
  ]'
# expected: exit 0
```

> ⚠️ `SkipDeletionProtection: false` 表示不跳过云硬盘的删除保护。如果 CBS 盘也开启了删除保护，需先手动关闭。

### 步骤 4：验证

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: { "TotalCount": 0, "Clusters": [] }
```

### 步骤 5：清理残留资源

即使用了 Enhanced 模式，也建议检查残留:

```bash
# 检查 CBS 盘（按标签或创建时间过滤）
tccli cbs DescribeDisks --region ap-guangzhou

# 检查 EIP
tccli vpc DescribeAddresses --region ap-guangzhou

# 检查 CLB
# CLB 需通过 CLB API 或控制台检查
```

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| 删除保护阻止 | 错误码 `OperationDenied.ClusterInDeletionProtection`（消息含 `cluster in deletion protection`） | 未关闭删除保护 | `tccli tke DisableClusterDeletionProtection --ClusterId "<ID>"` 先关保护；若该接口被 CAM 拒（要求 `request_tag` 但接口无 Tags 入参），需控制台关闭或调整 CAM 策略 |
| `ResourceNotFound.ClusterId` | `tccli tke DescribeClusters --ClusterIds '["<ID>"]'` | 集群 ID 错误或已删除 | 检查集群 ID 是否正确 |
| `AuthFailure` | `tccli cvm DescribeRegions` | 凭证过期 | `tccli configure` |
| `InternalError.ClusterDeletion` | 查看错误详情的 RequestId | 服务端删除异常 | 等待 5 分钟后重试；仍失败则提交工单附带 RequestId |
| CBS 盘删除失败 | `tccli cbs DescribeDisks` → 检查 `DeleteWithInstance` | CBS 开启了删除保护 | `tccli cbs ModifyDiskAttributes --DeleteWithInstance true` |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| 命令返回成功但集群仍在列表中 | `tccli tke DescribeClusters --ClusterIds '["<ID>"]'` | 异步删除进行中 | 等待 1-2 分钟后重试查询 |
| 集群删除后 CBS 盘仍存在 | `tccli cbs DescribeDisks` | 未使用 Enhanced 删除模式 | `tccli cbs TerminateDisks --DiskIds '["<ID>"]'` |
| 删除后 EIP 仍在扣费 | `tccli vpc DescribeAddresses` | EIP 未随集群释放 | `tccli vpc ReleaseAddresses --AddressIds '["<ID>"]'` |

## 开启删除保护（反操作）

> 删除保护的开启是 `DeleteCluster` 的逆操作——给已创建集群补开保护以防误删。与 [§步骤 1](#步骤-1关闭删除保护) 的关闭对称，入参同为 `ClusterId`。

```bash
tccli tke EnableClusterDeletionProtection \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: { "RequestId": "..." }
```

验证保护已生效：

```bash
tccli tke DescribeClusterStatus --region ap-guangzhou \
  --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterDeletionProtection" --output text
# expected: True（布尔值，开启后再次删除会被 OperationDenied.ClusterInDeletionProtection 拒绝）
```

> 幂等：对已开启保护的集群再次 `Enable` 仍返回 RequestId，不报错。

## 下一步

- [创建集群](create.md) — 重新创建一个新集群
- [查询集群](query.md) — 确认其他集群状态

## 控制台替代方案

[容器服务控制台 - 集群列表](https://console.cloud.tencent.com/tke2/cluster)

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `DisableClusterDeletionProtection` | 主操作 | 2018-05-25 | 前置：解除删除保护 |
| `EnableClusterDeletionProtection` | 主操作 | 2018-05-25 | 反操作：开启删除保护（防误删） |
| `DeleteCluster` | 主操作 | 2018-05-25 | 删除集群（支持级联删 CBS/EIP） |
| `DescribeClusters` | 验证 | 2018-05-25 | 确认目标集群/验证已删除 |
| `DescribeClusterStatus` | 验证 | 2018-05-25 | 检查删除保护状态 |
| `DescribeRegions` | 验证 | 2018-05-25 | 凭证有效性检查 |
| `cbs:DescribeDisks` | 跨产品 | cbs | 检查残留 CBS 云硬盘 |
| `cbs:TerminateDisks` | 跨产品 | cbs | 销毁残留 CBS 盘 |
| `vpc:DescribeAddresses` | 跨产品 | vpc | 检查残留 EIP |
| `vpc:ReleaseAddresses` | 跨产品 | vpc | 释放残留 EIP |
