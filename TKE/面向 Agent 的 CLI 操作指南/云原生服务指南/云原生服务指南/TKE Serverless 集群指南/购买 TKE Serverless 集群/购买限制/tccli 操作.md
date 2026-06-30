# 购买限制（tccli）

> 对照官方：[购买限制](https://cloud.tencent.com/document/product/457/39821) · page_id `39821`

## 概述

TKE Serverless 集群在资源创建和使用上有一系列配额限制。了解这些限制有助于在规划集群架构时避免资源不足或创建失败。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。

## 前置条件

- [环境准备](../../环境准备.md)
- 已注册腾讯云账号并完成实名认证

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查看当前集群数量 | `DescribeEKSClusters` | 是 |
| 创建集群（受配额限制） | `CreateEKSCluster` | 否 |

## 操作步骤

### 1. 查询当前集群数量（判断是否接近配额上限）

```bash
tccli tke DescribeEKSClusters \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TotalCount": 3,
    "Clusters": [
        {
            "ClusterId": "cls-example01",
            "ClusterName": "production-cluster",
            "Status": "Running",
            "VpcId": "vpc-example",
            "SubnetIds": ["subnet-example-a"],
            "CreatedTime": "2024-01-15T10:30:00Z",
            "K8SVersion": "1.28.3"
        },
        {
            "ClusterId": "cls-example02",
            "ClusterName": "staging-cluster",
            "Status": "Running",
            "VpcId": "vpc-example",
            "SubnetIds": ["subnet-example-b"],
            "CreatedTime": "2024-02-20T08:00:00Z",
            "K8SVersion": "1.28.3"
        },
        {
            "ClusterId": "cls-example03",
            "ClusterName": "dev-cluster",
            "Status": "Creating",
            "VpcId": "vpc-example",
            "SubnetIds": ["subnet-example-c"],
            "CreatedTime": "2024-03-10T12:00:00Z",
            "K8SVersion": "1.28.3"
        }
    ],
    "RequestId": "abc123-def456-ghi789"
}
```

### 2. 配额限制说明

| 资源 | 限制 | 说明 |
|------|------|------|
| 每个地域最多集群数 | **5** | 包含 Creating 和 Running 状态的集群 |
| 每个集群最多 Pod 数 | **500** | 所有命名空间、任意状态的 Pod |
| 每个工作负载最多 Pod 副本数 | **100** | 工作负载内任意状态的 Pod |
| 每个地域最多容器实例数 | **500** | 任意状态的容器实例 |

### 3. 安全组限制

每个 Pod 等同于一台 CVM 实例，所有副本共享同一安全组。因此：

- 每个安全组可关联的实例数上限适用于 Pod
- 若一个 Deployment 配置 50 个副本且绑定同一安全组，该安全组将占用 50 个实例配额

### 4. CLB 限制

每个绑定到 CLB 的 Pod 等同于 CLB 后端的一台 CVM 实例：

- 每个负载均衡转发规则可绑定的后端服务器数上限适用于 Pod
- 使用 LoadBalancer 类型 Service 时，需确保 CLB 后端配额足够容纳所有 Pod 副本

### 5. 提升配额

若默认配额不满足业务需求，可提交工单申请提升：

```bash
# 以下命令仅供参考操作流程，实际需通过控制台提交工单
```

**操作路径：** [提交工单](https://console.cloud.tencent.com/workorder/category) → 选择「其他问题」 → 新建工单 → 填写申请：

1. **问题描述：** 申请提升 TKE Serverless 配额
2. **目标地域：** 如 `ap-guangzhou`
3. **提升对象：** 如「集群数量上限」或「单集群 Pod 数上限」
4. **目标配额：** 如「集群数提升至 10」
5. **联系电话：** 以便客服回访

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 当前集群数 | `tke DescribeEKSClusters --region ap-guangzhou` | `TotalCount` 应小于 5 |
| 创建集群测试 | `tke CreateEKSCluster ...` | 若已达配额上限，返回错误码 `LimitExceeded` |

## 清理

无需清理。本页为配额概念参考页，不创建任何资源。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| `CreateEKSCluster` 报 `LimitExceeded.EKSCluster` | 单地域集群数已达上限（默认 5） | 清理不再使用的集群，或[提交工单](https://console.cloud.tencent.com/workorder/category)申请提升上限 |
| Pod 创建失败，提示超出配额 | 集群内 Pod 总数已达 500，或单个工作负载 Pod 数超过 100 | 删除不需要的 Pod 或工作负载，或[提交工单](https://console.cloud.tencent.com/workorder/category)申请提升上限 |
| 安全组关联失败 | 安全组关联的实例数超出上限 | 减少该安全组关联的 Pod 数量，或使用多个安全组分散关联 |
| CLB 后端绑定失败 | CLB 转发规则绑定的后端服务器数超出上限 | 减少绑定到同一 CLB 转发规则的 Pod 数，或使用多个 Service 分散流量 |

## 下一步

- [创建集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md) — 在配额范围内创建 Serverless 集群
- [计费概述](../计费概述/tccli%20操作.md) — 了解计费模式与费用构成

## 控制台替代

[容器服务控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) — 集群列表页面展示当前地域集群数量，新建集群时若超限将提示。
