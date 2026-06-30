# 节点池概述（tccli）

> 对照官方：[节点池概述](https://cloud.tencent.com/document/product/457/104096) · page_id `104096`  

## 概述

**节点池**统一管控一组节点：普通节点池（CVM+ASG）、原生节点池（MachineSet）、超级节点池（虚拟节点）、注册节点池（External）。应用场景包括同质配置、弹性伸缩与批量运维。

## 前置条件

- [环境准备](../../../环境准备.md) · [tccli 专页（TKE）](../../../tccli 专页（TKE）.md)

## 控制台与 CLI 参数映射

| 控制台 / 官方 | tccli / 数据面 |
|---------------|----------------|
| 普通/原生节点池 | `DescribeClusterNodePools` |
| 超级节点池 | `DescribeClusterVirtualNodePools` |
| 注册节点池 | `DescribeExternalNodePools` |

## 操作步骤

### 产品架构（官方）

节点池对节点生命周期、配置与伸缩策略进行分组管理；不同类型对应不同底层实现（ASG / 原生 Machine / 虚拟节点 / 外部 Agent）。

### 查询各类型节点池

```bash
tccli tke DescribeClusterNodePools --ClusterId CLUSTER_ID --region <Region> --output json
# expected: exit 0，返回节点池列表
tccli tke DescribeClusterVirtualNodePools --ClusterId CLUSTER_ID --region <Region> --output json
# expected: exit 0，返回超级节点池列表
tccli tke DescribeExternalNodePools --ClusterId CLUSTER_ID --region <Region> --output json
# expected: exit 0，返回注册节点池列表
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

## 验证

三条 Describe 均返回 `RequestId`。

## 清理

本页以只读查询或声明式配置为主，无额外 API 清理步骤。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePools` 返回空列表 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 查看集群状态 | 集群未创建节点池或仅含其他类型节点（超级/注册） | 确认集群状态为 Running；分别用对应 API 查询各类型节点池 |
| `DescribeClusterVirtualNodePools` 返回 `InvalidParameter` | 检查集群 ID 和地域 | 集群 ID 格式错误或不支持超级节点 | 确认集群 ID 正确且集群版本支持超级节点 |

## 下一步

- [创建节点池](../../../集群配置/节点管理/普通节点/创建节点池/tccli 操作.md) · [原生节点概述](../../../集群配置/节点管理/原生节点/原生节点概述/tccli 操作.md) · [超级节点概述](../../../集群配置/节点管理/超级节点/超级节点概述/tccli 操作.md)

## 控制台替代

容器服务控制台 → **集群** → 目标集群 → **节点管理**，按[官方文档](https://cloud.tencent.com/document/product/457/104096)对应菜单操作。
