# 组建集群选型推荐（tccli）

> 对照官方：[组建集群选型推荐](https://cloud.tencent.com/document/product/457/44966) · page_id `44966`

## 概述

TKE 提供三种集群类型：托管集群（MANAGED_CLUSTER）、独立集群（INDEPENDENT_CLUSTER）、弹性集群（EKS）。本文按场景推荐集群选型，并通过 `DescribeClusters` 查看集群属性。

## 前置条件

- [环境准备](../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群类型 | `tccli tke DescribeClusters --filter "Clusters[].{Id:ClusterId,Type:ClusterType}"` | 是 |
| 创建托管集群 | `tccli tke CreateCluster --ClusterType MANAGED_CLUSTER` | 否 |
| 创建弹性集群 | `tccli tke CreateEKSCluster` | 否 |

## 操作步骤

### 选型对比

| 特性 | 托管（MANAGED） | 独立（INDEPENDENT） | 弹性（EKS） |
|------|----------------|--------------------|------------|
| Master 管理 | 腾讯云托管 | 用户自管理 | Serverless |
| Master 费用 | 免费 | 按 CVM 计费 | 无 |
| Worker 节点 | CVM 节点池 | CVM 节点池 | 超级节点 |
| 适用场景 | 标准生产环境 | 特殊合规需求 | 弹性/批处理 |
| 创建命令 | `CreateCluster MANAGED_CLUSTER` | `CreateCluster INDEPENDENT_CLUSTER` | `CreateEKSCluster` |

### 查看现有集群

```bash
tccli tke DescribeClusters --region ap-guangzhou
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

## 验证

```bash
tccli tke DescribeClusters --region ap-guangzhou --filter "length(Clusters)"
```

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回空列表 | `tccli tke DescribeClusters --region ap-guangzhou --filter "TotalCount"` | 当前地域无集群或 CAM 权限不足 | 确认地域正确（如 ap-guangzhou）；CAM 授予 `tke:DescribeClusters` |
| `CreateCluster` 返回 `AuthFailure.UnauthorizedOperation` | `tccli cam DescribeUserPermissions` | 子账号缺少创建集群权限 | CAM 策略添加 `tke:CreateCluster` 及 CVM/VPC 相关权限 |
| 集群 `ClusterStatus` 长时间 `Creating` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].{Status:ClusterStatus,Reason:ClusterReason}"` | CVM 资源不足或 VPC/子网配额超限 | 检查 CVM 库存与子网剩余 IP；必要时更换可用区 |

## 下一步

- [快速创建一个标准集群](../../../快速入门/快速创建一个标准集群/tccli%20操作.md)
- [实现独立集群的 Master 容灾](../实现独立集群的%20Master%20容灾/tccli%20操作.md)

## 控制台替代

控制台：集群列表 → 查看集群类型列。
