# 集群生命周期

> 对照官方：[集群生命周期](https://cloud.tencent.com/document/product/457/72124) · page_id `72124`

## 概述

TKE Serverless 集群从创建到销毁经历多个生命周期状态。了解这些状态有助于判断集群当前是否可用、是否需要干预。

生命周期状态流转：

```
创建中 (Creating) → 运行中 (Running) → 删除中 (Deleting) → 已删除
                        ↓
                    异常 (Abnormal) → 恢复 → 运行中 (Running)
                        ↓
                    隔离 (Isolated) → 恢复 → 运行中 (Running)
```

各状态说明：

| 状态 | 英文值 | 说明 | 可操作 |
|------|--------|------|:--:|
| 创建中 | `Creating` | 集群正在创建，底层 VPC 网络、控制面初始化中 | 仅可查询 |
| 运行中 | `Running` | 集群正常运行，可部署工作负载 | 全部操作 |
| 删除中 | `Deleting` | 集群正在删除，Pod 被驱逐，资源释放中 | 仅可查询 |
| 异常 | `Abnormal` | 集群出现异常，可能是网络故障或底层资源问题 | 仅查询/删除 |
| 隔离 | `Isolated` | 因欠费被停服隔离 | 仅查询/删除 |

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表及状态 | `DescribeEKSClusters` | 是 |
| 查看单个集群详情 | `DescribeEKSClusters --ClusterIds` | 是 |
| 查看集群状态变化历史 | 无 CLI API（控制台查看） | — |
| 删除集群 | `DeleteEKSCluster` | 否 |
| 进入回收站 | 无 CLI API（系统自动） | — |

## 操作步骤

### 步骤 1：查看所有集群及其生命周期状态

```bash
tccli tke DescribeEKSClusters --region <Region>
# expected: exit 0，返回所有 Serverless 集群及状态
```

**预期输出**：

```json
{
    "TotalCount": 3,
    "Clusters": [
        {
            "ClusterId": "cls-running-xxxxx",
            "ClusterName": "prod-cluster",
            "Status": "Running",
            "CreatedTime": "2025-01-10T08:00:00Z"
        },
        {
            "ClusterId": "cls-creating-xxxxx",
            "ClusterName": "new-cluster",
            "Status": "Creating",
            "CreatedTime": "2025-01-15T12:00:00Z"
        },
        {
            "ClusterId": "cls-abnormal-xxxxx",
            "ClusterName": "test-cluster",
            "Status": "Abnormal",
            "CreatedTime": "2025-01-05T10:00:00Z"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 2：轮询检查集群状态（等待创建完成）

创建集群后，可通过轮询检查状态变化：

```bash
# 轮询直到 Running
while true; do
    STATUS=$(tccli tke DescribeEKSClusters --region <Region> \
        --ClusterIds '["CLUSTER_ID"]' \
        | jq -r '.Clusters[0].Status')
    echo "$(date): Status=$STATUS"
    if [ "$STATUS" = "Running" ]; then
        echo "集群已就绪"
        break
    fi
    if [ "$STATUS" = "Abnormal" ]; then
        echo "集群状态异常，请检查控制台"
        break
    fi
    sleep 15
done
# expected: 最终输出 "集群已就绪"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

### 步骤 3：检查异常集群的详细信息

```bash
# 查看异常集群的完整信息
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，返回完整集群信息，检查字段判断异常原因
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

常见异常原因字段：
- `Status`：`Abnormal` 或 `Isolated`
- 检查 `VpcId`、`SubnetIds` 是否仍有效
- 检查是否因欠费导致 `Isolated`

### 步骤 4：删除不再需要的集群

```bash
tccli tke DeleteEKSCluster --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，集群进入 Deleting 状态
```

> **注意**：删除集群是不可逆操作。集群删除后 Pod 全部终止，仅 PVC 关联的存储数据保留（需手动清理）。

## 验证

```bash
# 验证目标集群状态
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '{ClusterId: .Clusters[0].ClusterId, Status: .Clusters[0].Status, CreatedTime: .Clusters[0].CreatedTime}'
# expected: Status 为 "Running"

# 验证已删除集群不在列表中
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount == 0（已彻底删除后）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

## 清理

本页面为只读操作，无需清理（查询生命周期状态）。

> 若需删除集群，参考步骤 4 使用 `DeleteEKSCluster`。删除前确保已清理所有 PVC 关联的 CBS 磁盘避免持续计费。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEKSClusters` 中集群长期为 `Creating` | 轮询检查，超过 5 分钟 | 后端创建可能因资源不足或网络配置问题卡住 | 等待不超过 10 分钟；超时后记录 ClusterId → 登录控制台查看详细状态或提工单 |
| `DeleteEKSCluster` 返回 `FailedOperation.ClusterInDeletion` | `DescribeEKSClusters` 确认状态 | 该集群正在被删除 | 等待删除完成，不可重复删除 |
| `DeleteEKSCluster` 返回 `FailedOperation.ClusterState` | 检查集群当前状态 | 集群可能为 `Isolated`，需先恢复 | 若是欠费隔离，充值恢复后再删除；若无法恢复，联系售后强制删除 |
| 集群状态为 `Abnormal` 且无法恢复 | 检查 `DescribeEKSClusters` 返回的详细信息 | 底层资源或网络异常 | 若等待 30 分钟仍未恢复，提工单处理或删除集群重建 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 集群 `Running` 但 kubectl 无法连接 | `cat ~/.kube/config` 检查 server 地址 | kubeconfig 过期或网络不通 | 重新获取 kubeconfig：`tccli tke DescribeEKSClusterCredential` |
| 删除集群后 `DescribeEKSClusters` 仍返回该集群（状态 `Deleting`） | 等待 2-5 分钟 | 删除是异步操作，资源释放需要时间 | 等待 `Deleting` 转为不返回 |
| 集群从 `Running` 突然变为 `Isolated` | 登录计费中心检查账户 | 账户欠费导致集群被隔离 | 充值后集群自动恢复（5-10 分钟） |

## 下一步

- [创建 Serverless 集群](../创建集群/tccli%20操作.md) — 新建集群
- [连接集群](../连接集群/tccli%20操作.md) — 获取 kubeconfig 连接集群
- [计费概述](../../购买%20TKE%20Serverless%20集群/计费概述/tccli%20操作.md) — 了解各状态的计费影响
- [欠费说明](../../购买%20TKE%20Serverless%20集群/欠费说明/tccli%20操作.md) — 欠费导致的隔离状态处理

## 控制台替代

[TKE 控制台 → Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster)：集群列表页展示所有集群的当前状态（彩色标签），点击集群名查看详情和状态变更历史。
