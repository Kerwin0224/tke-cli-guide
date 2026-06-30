# 节点池概述（tccli）

> 对照官方：[节点池概述](https://cloud.tencent.com/document/product/457/43719) · page_id `43719`  

## 概述

**节点池**对同配置节点分组管理，支持弹性伸缩、Label/Taint、批量升级。官方建议单集群节点池不超过 **20** 个；计费模式为池级属性。本页保留官方功能点表与伸缩原理说明，并用 tccli 列出池清单。

## 前置条件

- [环境准备](../../../../环境准备.md)

## 控制台与 CLI 参数映射

| 功能点 | tccli / 专页 |
|--------|----------------|
| 创建 | [创建节点池](../创建节点池/tccli%20操作.md) `CreateClusterNodePool` |
| 查看 | [查看节点池](../查看节点池/tccli%20操作.md) |
| 调整 | [调整节点池](../调整节点池/tccli%20操作.md) `ModifyClusterNodePool` |
| 删除 | [删除节点池](../删除节点池/tccli%20操作.md) |
| 伸缩记录 | [查看节点池伸缩记录](../查看节点池伸缩记录/tccli%20操作.md) |

## 操作步骤

### 简介

节点池提高异构节点分组、扩缩容与标签运维效率。

### 功能点及注意事项（节选）

| 功能点 | 功能说明 | 注意事项 |
| --- | --- | --- |
| 创建节点池 | 新增节点池 | 单集群不建议超过 20 个节点池；包年包月勿将按量节点转包月，宜新建包月池。 |
| 删除节点池 | 可选是否销毁池内节点 | 删除后节点均不在集群内。 |
| 节点池开启弹性伸缩 | 随负载自动调节点数 | **勿在伸缩组控制台单独开关弹性伸缩。** |
| 调整节点池节点数量 | 直接调期望节点数 | 开启弹性伸缩后不建议手动调大小；缩容优先用弹性缩容（先不可调度再驱逐）。 |
| 调整节点池配置 | 名称、OS、ASG 范围、Label/Taint | Label/Taint 变更对池内全部节点生效，可能重调度。 |

### 节点池内节点种类

| 节点类型 | 节点来源 | 弹性伸缩 | 移除方式 |
| --- | --- | --- | --- |
| 伸缩组内节点 | 弹性扩容或手动调数量 | 是 | 弹性缩容或手动调数量 |
| 伸缩组外节点 | 手动加入池 | 否 | 手动移除 |

### tccli：列出节点池

```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "NodePoolSet[].{NodePoolId:NodePoolId,Name:Name,LifeState:LifeState}"
```

```json
{
  "NodePoolSet": [],
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>",
  "AutoscalingGroupId": "<AutoscalingGroupId>",
  "Labels": []
}
```

## 验证

### Control plane (tccli)

- 返回 `NodePoolSet` 与控制台节点池列表一致。

## 清理

只读。

## 排障

勿在 ASG 控制台单独改期望实例数，避免与 TKE 数据不一致。

## 下一步

- [创建节点池](../创建节点池/tccli%20操作.md)

## 控制台替代

**节点管理 → 节点池**。
