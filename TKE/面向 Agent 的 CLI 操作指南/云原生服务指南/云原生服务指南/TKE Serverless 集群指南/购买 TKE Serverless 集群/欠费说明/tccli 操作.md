# 欠费说明（tccli）

> 对照官方：[欠费说明](https://cloud.tencent.com/document/product/457/86717) · page_id `86717`

## 概述

TKE Serverless 集群**不收取集群管理费**，仅对集群内的计算资源和关联产品进行计费。当账户发生欠费时，不同类型资源的处理方式不同：按量计费资源会被停止，包年包月资源继续运行。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。

## 前置条件

- [环境准备](../../环境准备.md)
- 已注册腾讯云账号

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查看集群状态 | `DescribeEKSClusters` | 是 |
| 查看容器实例状态 | `DescribeEKSContainerInstances` | 是 |

## 操作步骤

### 查询集群与容器实例运行状态

欠费后，按量计费的容器实例会被停止。可通过以下命令检查集群和容器实例的状态：

```bash
# 查看集群状态
tccli tke DescribeEKSClusters \
    --region ap-guangzhou \
    --output json
```

示例输出（正常运行的集群）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "production-serverless",
            "Status": "Running",
            "VpcId": "vpc-example",
            "SubnetIds": ["subnet-example"],
            "CreatedTime": "2024-01-15T10:30:00Z",
            "K8SVersion": "1.28.3"
        }
    ],
    "RequestId": "abc123-def456-ghi789"
}
```

### 欠费处理机制

#### 按量计费资源（按量计费 Pod）

| 阶段 | 处理方式 |
|------|---------|
| 账户余额为负（欠费） | 下一个结算周期自动**停止**所有按量计费 Pod，释放底层计算资源 |
| Pod YAML / 数据资源 | **不删除**，保留在集群中 |
| 账户恢复（充值后余额为正） | 若 Pod `restartPolicy` 为 `Always`，Pod 自动重新创建并启动 |

#### 包年包月资源（包年包月超级节点 / 预留券）

| 阶段 | 处理方式 |
|------|---------|
| 账户余额为负（欠费） | **不受影响**，继续正常运行 |
| 超级节点 | 仍可调度 Pod |
| 预留券 | 仍可抵扣同地域、同规格的按量计费 Pod |
| 超出预购范围的资源 | 进入 `Pending` 状态（同地域内按量资源被停止） |

#### 集群控制面

| 阶段 | 处理方式 |
|------|---------|
| 任意阶段 | **始终保持正常** — TKE Serverless 不收取集群管理费，欠费不影响控制面 |

#### 关联产品

集群内使用的以下产品遵循各自独立的欠费策略：

| 产品 | 欠费处理 |
|------|---------|
| **负载均衡 CLB** | 按 CLB 欠费策略处理（可能被隔离/释放）——参见 [CLB 欠费说明](https://cloud.tencent.com/document/product/214/42934) |
| **云硬盘 CBS** | 按 CBS 欠费策略处理 —— 参见 [CBS 欠费说明](https://cloud.tencent.com/document/product/362/3064) |
| **文件存储 CFS** | 按 CFS 欠费策略处理 |

### 欠费恢复后的操作

充值恢复账户后，检查容器实例是否已自动恢复：

```bash
# 查看容器实例状态
tccli tke DescribeEKSContainerInstances \
    --region ap-guangzhou \
    --output json
```

示例输出（恢复后的 Running 状态）：

```json
{
    "TotalCount": 2,
    "EksCiSet": [
        {
            "EksCiId": "eksci-example01",
            "EksCiName": "my-deployment-pod-001",
            "Status": "Running",
            "Cpu": 2.0,
            "Memory": 4.0,
            "CreationTime": "2024-01-15T10:35:00Z"
        },
        {
            "EksCiId": "eksci-example02",
            "EksCiName": "my-deployment-pod-002",
            "Status": "Running",
            "Cpu": 2.0,
            "Memory": 4.0,
            "CreationTime": "2024-01-15T10:35:00Z"
        }
    ],
    "RequestId": "abc123-def456-ghi789"
}
```

若带 `restartPolicy: Always` 的 Pod 未自动恢复，可使用 `kubectl` 手动触发重建（注意：kubectl 不可达时可通过 `CreateEKSContainerInstances` 重新创建）。

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 集群状态正常 | `tke DescribeEKSClusters --region ap-guangzhou` | `Status: "Running"` |
| 容器实例已恢复 | `tke DescribeEKSContainerInstances --region ap-guangzhou` | `Status: "Running"` |

## 清理

无需清理。本页为欠费概念参考页，不创建任何资源。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| 充值后 Pod 仍为停止状态 | `restartPolicy` 非 `Always` | 使用 `kubectl apply` 或 API 重新创建 Pod |
| 充值后部分 Pod 仍 `Pending` | 包年包月资源外的按量计费 Pod 未恢复 | 确认按量计费账户余额已恢复，等待下一个调度周期 |
| 欠费后 Service（CLB）不可用 | CLB 遵循独立的欠费策略，可能已被隔离 | 充值后前往 [CLB 控制台](https://console.cloud.tencent.com/clb) 检查并恢复 |
| 欠费后数据丢失 | 按量计费的临时存储（每个 Pod 的临时镜像存储）在 Pod 停止时被释放 | 重要数据应使用持久化存储（CBS/CFS）而非 Pod 临时存储 |

## 下一步

- [创建集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md) — 创建 Serverless 集群
- [计费概述](../计费概述/tccli%20操作.md) — 了解计费模式

## 控制台替代

[费用中心 → 收支明细](https://console.cloud.tencent.com/expense/overview) — 查看账户余额和欠费状态。
