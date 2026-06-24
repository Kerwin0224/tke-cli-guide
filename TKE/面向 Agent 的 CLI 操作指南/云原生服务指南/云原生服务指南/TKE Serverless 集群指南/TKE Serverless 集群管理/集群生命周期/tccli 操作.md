# 集群生命周期（tccli）

> 对照官方：[集群生命周期](https://cloud.tencent.com/document/product/457/72124) · page_id `72124`

## 概述

TKE Serverless 集群在其生命周期中会经历多种状态，包括创建中、运行中、闲置、激活中、删除中和异常。了解各状态的含义和转换条件有助于运维管理和故障排查。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。

## 前置条件

- [环境准备](../../环境准备.md)
- 已拥有 Serverless 集群

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查询集群状态 | `DescribeEKSClusters` | 是 |
| 按状态过滤集群 | `DescribeEKSClusters --filter` | 是 |

## 操作步骤

### 查询集群生命周期状态

```bash
tccli tke DescribeEKSClusters \
    --region ap-guangzhou \
    --output json
```

示例输出（展示不同状态的集群）：

```json
{
    "TotalCount": 3,
    "Clusters": [
        {
            "ClusterId": "cls-running001",
            "ClusterName": "production-cluster",
            "Status": "Running",
            "VpcId": "vpc-example",
            "SubnetIds": ["subnet-example-a"],
            "CreatedTime": "2024-01-15T10:30:00Z",
            "K8SVersion": "1.28.3"
        },
        {
            "ClusterId": "cls-creating001",
            "ClusterName": "new-cluster",
            "Status": "Creating",
            "VpcId": "vpc-example",
            "SubnetIds": ["subnet-example-b"],
            "CreatedTime": "2024-03-10T12:00:00Z",
            "K8SVersion": "1.28.3"
        },
        {
            "ClusterId": "cls-idle001",
            "ClusterName": "unused-cluster",
            "Status": "Idle",
            "VpcId": "vpc-example",
            "SubnetIds": ["subnet-example-c"],
            "CreatedTime": "2024-01-01T08:00:00Z",
            "K8SVersion": "1.28.3"
        }
    ],
    "RequestId": "abc123-def456-ghi789"
}
```

### 集群状态一览

| 状态 | 含义 | 说明 |
|------|------|------|
| **Creating** | 创建中 | 集群正在创建，云资源分配中。此时不可操作集群。 |
| **Running** | 运行中 | 集群运行正常，可以创建和管理工作负载。 |
| **Idle** | 闲置 | 集群长期无 Pod 运行，被系统回收为闲置态。原集群信息保留但不可用。 |
| **Activating** | 激活中 | 从闲置状态恢复到运行中。激活完成后恢复所有功能。 |
| **Deleting** | 删除中 | 集群正在删除，云资源正在释放。删除完成后集群不复存在。 |
| **Abnormal** | 异常 | 集群出现问题（如网络不可达）。需排查处理。 |

### 状态流转图

```
创建 → Creating → Running ⇄ Idle
                    ↓           ↑ (Activating)
                 Deleting
                    ↓
                 (消失)

    任何状态 → Abnormal
```

### 闲置（Idle）状态详解

TKE Serverless 引入了闲置集群回收机制以降低资源空置成本。满足以下全部条件的集群进入闲置状态：

| 条件 | 说明 |
|------|------|
| 无用户创建的 Pod | 集群内不存在用户创建的 Pod |
| 连续 7 天无 Pod 操作 | 无 Pod 创建、删除操作 |
| 距离上次激活至少 3 天 | 防止短时间反复切换 |
| 集群创建超过 7 天 | 新集群有保护期 |

#### 闲置状态的限制

- 所有原有集群信息保留
- 集群**不可用**：无法查看监控、集群详情
- APIServer 操作受限
- 只能执行两个操作：**激活**（恢复运行）或**删除**（释放资源）

#### 激活闲置集群

闲置集群激活后会保留所有原有功能与配置。但若激活后的集群：

- 无 Pod 运行
- **连续 3 天**无 Pod 创建/删除操作

将被再次标记为闲置状态。

### 按状态过滤集群

```bash
# 查询所有 Running 状态的集群
tccli tke DescribeEKSClusters \
    --region ap-guangzhou \
    --output json | jq '.Clusters[] | select(.Status == "Running")'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": [],
  "K8SVersion": "<K8SVersion>",
  "Status": "<Status>"
}
```

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 查询集群列表含状态 | `tke DescribeEKSClusters --region ap-guangzhou` | `Clusters[].Status` 包含状态值 |
| 过滤特定状态集群 | `jq '.Clusters[] \| select(.Status == "Running")'` | 仅返回 `Running` 状态的集群 |

## 清理

无需清理。本页为概念参考页，不创建任何资源。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| 集群变为 `Idle` 状态 | 连续 7 天无 Pod 操作 | 通过控制台或 API 激活集群：选择集群 → 更多 → 激活 |
| 集群状态为 `Abnormal` | 网络不可达或其他底层问题 | 检查 VPC 子网配置，联系 [在线支持](https://cloud.tencent.com/online-service?from=doc_457) |
| `Creating` 状态持续超过 10 分钟 | 后端资源创建延迟或异常 | 检查子网 IP 是否充足，联系 [在线支持](https://cloud.tencent.com/online-service?from=doc_457) |
| 集群激活后又变为 `Idle` | 激活后连续 3 天无 Pod 操作 | 在集群中创建至少一个 Pod 并保持运行 |
| `Deleting` 状态持续不消失 | 资源释放延迟 | 等待 30 分钟，仍异常联系 [在线支持](https://cloud.tencent.com/online-service?from=doc_457) |

## 下一步

- [创建集群](../创建集群/tccli%20操作.md) — 创建 Serverless 集群
- [注意事项](../注意事项/tccli%20操作.md) — 了解使用限制和注意事项

## 控制台替代

[容器服务控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) — 集群列表的「状态」列展示生命周期状态，闲置集群可通过 **更多 > 激活** 恢复。
