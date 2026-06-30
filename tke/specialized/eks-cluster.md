---
doc_type: How-to
subtype: 6A
fused: true
---
# EKS 弹性容器实例

> Serverless 容器 —— 无需管理节点，按 Pod 付费。
> 控制台: [容器服务 - EKS](https://console.cloud.tencent.com/tke2/eks)

## 概述

EKS (Elastic Kubernetes Service) 是 TKE 的 Serverless 容器方案。你不需要创建节点、管理节点池 —— 直接部署 Pod，按实际使用的 vCPU/内存/秒计费。

| 特性 | EKS（Serverless） | TKE 标准集群 |
|------|:---:|:---:|
| 是否需要节点 | ❌ 不需要 | ✅ 需要 CVM 节点 |
| 计费方式 | 按 Pod 使用量 | 按 CVM 实例 |
| 冷启动 | 有 (~10s) | 无 (Pod 调度到已有节点) |
| 适合场景 | 批处理、定时任务、突发流量 | 常驻服务 |

## 关键操作

### 创建 EKS 集群

```bash
tccli tke CreateEKSCluster \
  --region ap-guangzhou \
  --ClusterName "<NAME>-eks" \
  --VpcId "<VPC_ID>" \
  --SubnetIds '["<SUBNET_ID>"]'
# expected: { "Response": { "ClusterId": "cls-xxxxxxxx" } }
```

### 查询 EKS 集群

```bash
# 列出所有 EKS 集群
tccli tke DescribeEKSClusters --region ap-guangzhou
# expected: { "Response": { "TotalCount": ..., "Clusters": [...] } }

# 获取凭证
tccli tke DescribeEKSClusterCredential --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 返回 kubeconfig

# 查询支持的地域
tccli tke DescribeEKSContainerInstanceRegions --region ap-guangzhou
# expected: Region 列表
```

### 创建容器实例 (部署 Pod)

```bash
tccli tke CreateEKSContainerInstances \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Containers '[
    {
      "Name": "nginx",
      "Image": "nginx:latest",
      "Cpu": 1,
      "Memory": 2
    }
  ]'
# expected: { "Response": { "EksCiId": "eksci-xxxxxxxx" } }
```

### 管理容器实例

```bash
# 查询实例列表
tccli tke DescribeEKSContainerInstances \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: EKS 容器实例列表

# 查询实例事件
tccli tke DescribeEKSContainerInstanceEvent \
  --region ap-guangzhou \
  --EksCiId "<EKSCI_ID>"
# expected: 事件列表 (用于排查 Pod 无法启动)

# 重启实例
tccli tke RestartEKSContainerInstances --region ap-guangzhou \
  --EksCiIds '["<EKSCI_ID>"]'

# 查询日志
tccli tke DescribeEksContainerInstanceLog \
  --region ap-guangzhou \
  --EksCiId "<EKSCI_ID>"
```

## 更新 EKS 集群与事件持久化

> 修改 EKS 集群属性（名称/描述/子网/CLB/DNS）与开启事件持久化（事件投递到 CLS）。参数以 `--generate-cli-skeleton` 实测为准。

```bash
# 更新 EKS 集群属性（ClusterId 定位，ClusterName/ClusterDesc/SubnetIds 等覆盖式更新）
tccli tke UpdateEKSCluster --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --ClusterName "<NEW_NAME>" --ClusterDesc "<NEW_DESC>"
# expected: exit 0
```

```bash
# 开启事件持久化（ClusterId + CLS LogsetId/TopicId/TopicRegion）
tccli tke EnableEksEventPersistence --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --LogsetId "<CLS_LOGSET_ID>" --TopicId "<CLS_TOPIC_ID>" --TopicRegion "<REGION>"
# expected: exit 0, 集群事件开始投递到 CLS
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<CLS_LOGSET_ID>` | CLS 日志集 ID | `tccli cls DescribeLogsets` |
| `<CLS_TOPIC_ID>` | CLS 日志主题 ID | `tccli cls DescribeTopics` |
| `<TOPIC_REGION>` | CLS 主题所在地域 | 与集群地域一致或 CLS 实例地域 |

> `UpdateEKSCluster` 覆盖式更新集群属性（`SubnetIds[]`/`PublicLB`/`InternalLB`/`ServiceSubnetId`/`DnsServers[]` 等均可改）。`EnableEksEventPersistence` 需先在 CLS 创建日志集与主题，再把三个 ID 传入，集群事件（Pod 创建/失败等）即投递到 CLS 供检索。

## 清理

```bash
# 1. 删除容器实例
tccli tke DeleteEKSContainerInstances --region ap-guangzhou \
  --EksCiIds '["<EKSCI_ID>"]'

# 2. 删除 EKS 集群
tccli tke DeleteEKSCluster --region ap-guangzhou --ClusterId "<CLUSTER_ID>"

# 3. 验证
tccli tke DescribeEKSClusters --region ap-guangzhou
# expected: TotalCount 减少
```

## API 参考

完整的 EKS API 共 14 个操作:

| 分类 | API | 说明 |
|------|-----|------|
| 集群 | `CreateEKSCluster` / `DeleteEKSCluster` / `UpdateEKSCluster` / `DescribeEKSClusters` | 集群 CRUD |
| 凭证 | `DescribeEKSClusterCredential` | 获取 kubeconfig |
| 容器实例 | `CreateEKSContainerInstances` / `DeleteEKSContainerInstances` / `DescribeEKSContainerInstances` / `RestartEKSContainerInstances` / `UpdateEKSContainerInstance` | 实例管理 |
| 地域 | `DescribeEKSContainerInstanceRegions` | 支持的地域 |
| 事件/日志 | `DescribeEKSContainerInstanceEvent` / `DescribeEksContainerInstanceLog` | 事件和日志 |
| 持久化 | `EnableEksEventPersistence` | 开启事件持久化 |

## 下一步

- [边缘集群](edge-cluster.md) — 边缘计算场景
- [EKS 容器实例](eks-instances.md) — 无集群直接创建容器
- [专用工作负载概览](index.md) — EKS/边缘选型
- [标准集群概览](../clusters/index.md) — 对比标准集群

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `CreateEKSCluster` | 主操作 | 2018-05-25 | 创建 EKS Serverless 集群 |
| `UpdateEKSCluster` | 主操作 | 2018-05-25 | 更新 EKS 集群 |
| `CreateEKSContainerInstances` | 主操作 | 2018-05-25 | 部署 Pod（按 Pod 计费） |
| `RestartEKSContainerInstances` | 主操作 | 2018-05-25 | 重启容器实例 |
| `UpdateEKSContainerInstance` | 主操作 | 2018-05-25 | 更新容器实例 |
| `EnableEksEventPersistence` | 主操作 | 2018-05-25 | 开启事件持久化 |
| `DeleteEKSCluster` | 清理 | 2018-05-25 | 删除 EKS 集群 |
| `DeleteEKSContainerInstances` | 清理 | 2018-05-25 | 删除容器实例 |
| `DescribeEKSClusters` | 验证 | 2018-05-25 | EKS 集群列表 |
| `DescribeEKSClusterCredential` | 验证 | 2018-05-25 | EKS 集群凭证 |
| `DescribeEKSContainerInstanceRegions` | 验证 | 2018-05-25 | 支持地域列表 |
| `DescribeEKSContainerInstances` | 验证 | 2018-05-25 | 容器实例列表 |
| `DescribeEKSContainerInstanceEvent` | 验证 | 2018-05-25 | 容器实例事件 |
| `DescribeEksContainerInstanceLog` | 验证 | 2018-05-25 | 容器实例日志 |
