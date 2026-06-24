# 节点概述（tccli）

> 对照官方：[节点概述](https://cloud.tencent.com/document/product/457/32201) · page_id `32201`  

## 概述

TKE **节点**是集群中运行工作负载的算力单元。官方将节点分为 **普通节点**、**原生节点**、**超级节点**、**注册节点** 四类，能力与适用场景不同。本页保留官方分类表与操作关系图说明；用 `DescribeClusterInstances` 查看当前集群 **Worker** 节点，并按类型跳转到同目录专页。

## 前置条件

- [环境准备](../../../环境准备.md)
- 目标集群 `<ClusterId>`，地域 `ap-guangzhou`

## 控制台与 CLI 参数映射

| 节点类型 | 官方专页 | tccli / 专页 | 幂等 |
|----------|----------|----------------|------|
| 普通节点 | [普通节点](https://cloud.tencent.com/document/product/457/84660) | `DescribeClusterInstances` · [节点池概述](../普通节点/节点池概述/tccli%20操作.md) | 是 |
| 原生节点 | [原生节点概述](../原生节点/原生节点概述/tccli%20操作.md) | 节点池 `type=Native`（见原生节点专页） | 是 |
| 超级节点 | [超级节点概述](../超级节点/超级节点概述/tccli%20操作.md) | `DescribeClusterVirtualNodePools` | 是 |
| 注册节点 | [注册节点概述](../注册节点/注册节点概述/tccli%20操作.md) | `DescribeExternalNodePools` | 是 |

## 操作步骤

### 简介

节点是 Kubernetes 集群中承载 Pod 的实体；不同节点类型对应不同的资源形态、计费与运维模型（见下表）。

### 节点分类

| **节点类型** | **特点** | **适用场景** |
| --- | --- | --- |
| [普通节点](https://cloud.tencent.com/document/product/457/84660) | 适配腾讯云 CVM 数十种机型；基于弹性伸缩提供自动缩容。 | 资源与运维管控强、OS 偏定制化。 |
| [原生节点](../原生节点/原生节点概述/tccli%20操作.md) | TKE Insight 资源大盘；专有调度器；基础设施声明式 API。 | 降本、提升利用率、简化运维。 |
| [超级节点](../超级节点/超级节点概述/tccli%20操作.md) | Serverless 理念；单 Pod 独占轻量 VM；秒级扩缩容。 | 弹性业务、强隔离、轻量运维。 |
| [注册节点](../注册节点/注册节点概述/tccli%20操作.md) | IDC 接入云端；混合调度；云原生可观测一致。 | 云上云下统一纳管。 |

### 节点相关操作

官方流程图涵盖：新增/移出节点、封锁驱逐、Label/Taint、节点池与超级节点等。对应 **tccli** 入口见 [节点生命周期](../常用操作/节点生命周期/tccli%20操作.md) 与各子目录 **tccli 操作**。

#### tccli：列出集群 Worker 节点

```bash
tccli tke DescribeClusterInstances --ClusterId CLUSTER_ID --region <Region> --output json
# expected: exit 0，返回 InstanceSet 含 InstanceRole=WORKER 的节点
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

```bash
# 筛选 Worker 节点
tccli tke DescribeClusterInstances --ClusterId CLUSTER_ID --region <Region> --output json \
    | jq '[.InstanceSet[] | select(.InstanceRole=="WORKER") | {InstanceId:.InstanceId,InstanceState:.InstanceState}]'
# expected: Worker 节点列表
```

```json
[
  { "InstanceId": "ins-example", "InstanceState": "running" },
  { "InstanceId": "np-example", "InstanceState": "running" }
]
```

`InstanceId` 以 `np-` 前缀的条目通常来自节点池；裸 `ins-` 多为游离普通节点或 Master 以外 Worker。

## 验证

### 控制面（tccli）

- `DescribeClusterInstances` 返回 `TotalCount` ≥ 1 且 `InstanceRole=WORKER` 条目与控制台「节点管理」一致。

## 清理

只读，无资源清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterInstances` 返回空 `InstanceSet` | `tccli tke DescribeClusterVirtualNodePools --ClusterId CLUSTER_ID` 检查超级节点 | 集群仅有超级节点（无 Worker 节点） | 确认集群节点类型；超级节点通过 `DescribeClusterVirtualNodePools` 查看 |
| `DescribeClusterInstances` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <Region>` 列出全部集群 | `ClusterId` 格式错误或不属于当前地域 | 用 `DescribeClusters` 确认正确的集群 ID 和地域 |

## 下一步

- [节点生命周期](../常用操作/节点生命周期/tccli%20操作.md)
- [普通节点 / 节点池概述](../普通节点/节点池概述/tccli%20操作.md)
- [超级节点概述](../超级节点/超级节点概述/tccli%20操作.md)

## 控制台替代

控制台 **集群 → 节点管理** 列表与 `DescribeClusterInstances` 字段对应。
