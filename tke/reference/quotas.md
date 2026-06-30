---
doc_type: Reference
subtype: 8C
---
# TKE 配额与限制

> 资源配额与 API 限频。配额默认值来自腾讯云官方文档，API 限频来自 TKE API 概览页实测。超限信号列给出触发限制时看到的错误码。

## 资源配额

| 限制项 | 默认值 | 作用域 | 可调整 | 超限信号 |
|:-------|:------:|:-------|:------:|:---------|
| 集群数 | 50 | 每地域每账号 | 是（提工单） | `InternalError.QuotaMaxClsLimit` |
| 集群节点数 | 5000 | 每集群 | 是（提工单） | `InternalError.QuotaMaxNodLimit` |
| 节点池数 | 20 | 每集群 | 是（提工单） | `LimitExceeded` |
| 每节点池节点数 | 受 ASG MaxSize 限制 | 每节点池 | 是（改 ASG） | 扩容时 ASG 达上限 |
| 重启节点上限 | 100 | 每次请求 | 否 | `InvalidParameterValue` |
| 集群等级 | L5 ~ L100 | 每集群 | 是（`ModifyClusterLevel`） | `InvalidParameterValue.ClusterLevel` |

> 集群等级（L5/L20/L50/L100）决定集群管理费与可管理节点规模，等级越高费用越高但支持节点越多。详见 [集群等级说明](https://cloud.tencent.com/document/product/457)。

## API 限频

> 来源：腾讯云 TKE API 概览页。多数接口 20 次/秒，写操作与高频查询有差异。

| 接口类别 | 限频 (次/秒) | 超限信号 |
|:---------|:----------:|:---------|
| `DescribeClusters` / `DescribeClusterStatus` / `DescribeClusterNodePools` | 20 | `RequestLimitExceeded` |
| `DescribeClusterEndpoints` / `DescribeClusterKubeconfig` | 20 | `RequestLimitExceeded` |
| `CreateCluster` / `CreateClusterNodePool` | 20 | `RequestLimitExceeded` |
| `DeleteCluster` | 50 | `RequestLimitExceeded` |
| `DescribeClusterSecurity` | 100 | `RequestLimitExceeded` |
| 边缘集群查询（`DescribeEdgeClusterInstances` 等） | 10 | `RequestLimitExceeded` |

> 通用默认限频 10 次/秒，TKE 多数接口放宽到 20 次/秒。批量操作时串行调用或加间隔，避免触发 `RequestLimitExceeded`。

## 查询配额用量

```bash
# 集群存量 (核对是否接近 50 上限)
tccli tke DescribeClusters --region <REGION> --output text --filter "TotalCount"
# expected: 数字 ≤ 50

# 节点池存量 (核对是否接近 20 上限)
tccli tke DescribeClusterNodePools --region <REGION> --ClusterId "<CLUSTER_ID>" --output text --filter "TotalCount"
# expected: 数字 ≤ 20

# 集群节点数 (核对 ClusterRunningNodeNum)
tccli tke DescribeClusterStatus --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].{state:ClusterState,running:ClusterRunningNodeNum,init:ClusterInitNodeNum}"
# expected: running 节点数 ≤ 5000
```

## 资源用量查询

> 集群资源用量（CRD/Pod/节点）、Pod 计费与抵扣、预留实例利用率的查询。

```bash
# 集群资源用量 (CRD 等存量, 核对是否接近上限)
tccli tke DescribeResourceUsage --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, CRDUsage 含 Name/Usage/Details[]
```
```json
{
    "CRDUsage": {"Name": "CRD", "Usage": 10, "Details": [{"Name": "example-crd.example.com", "Usage": 2}]}
}
```

```bash
# 节点资源 (含预留实例, 需 NodeName)
tccli tke DescribePostNodeResources --ClusterId "<CLUSTER_ID>" --NodeName "<NODE_NAME>" --region <REGION>
# expected: exit 0, ReservedInstanceSet[]

# 按规格查询 Pod (CPU/Mem/GPU 规格 + 可用区, 创建前估算)
tccli tke DescribePodsBySpec --Cpu 2 --Memory 4 --Zone "<ZONE>" --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, 可创建 Pod 数

# Pod 计费信息 (按 Namespace+Name 或 Uids)
tccli tke DescribePodChargeInfo --ClusterId "<CLUSTER_ID>" --Namespace "<NS>" --Name "<POD_NAME>" --region <REGION>
# expected: exit 0, Pod 计费详情

# Pod 抵扣率 (按可用区, 预留实例抵扣情况)
tccli tke DescribePodDeductionRate --Zone "<ZONE>" --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, 抵扣率

# 预留实例利用率明细 (Filters 过滤, 属预留实例域)
tccli tke DescribeRIUtilizationDetail --Limit 20 --region <REGION>
# expected: exit 0, 预留实例利用明细
```

> `DescribeResourceUsage` 返回 CRD 等资源存量（核对是否接近上限）。`DescribePodsBySpec` 按规格估算可创建 Pod 数。`DescribePodChargeInfo`/`DescribePodDeductionRate` 是 Pod 计费与预留实例抵扣（预留实例域，见 [排除说明](#)）。`DescribeRIUtilizationDetail` 属预留实例（D34 已排除，仅查询参考）。

## 相关文档

- [错误码](error-codes.md) — `LimitExceeded` / `QuotaMaxClsLimit` 的诊断
- [创建集群](../clusters/create.md) — 集群数配额检查
- [创建节点池](../nodes/nodepool-create.md) — 节点池配额检查
