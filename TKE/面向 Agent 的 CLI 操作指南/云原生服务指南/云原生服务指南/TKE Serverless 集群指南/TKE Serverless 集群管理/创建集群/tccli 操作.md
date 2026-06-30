# 创建集群（tccli）

> 对照官方：[创建集群](https://cloud.tencent.com/document/product/457/39813) · page_id `39813`

## 概述

通过 `CreateEKSCluster` API 创建 TKE Serverless 集群。相比标准集群，Serverless 集群无需管理物理节点，集群创建后自动生成超级节点，每个超级节点映射到一个 VPC 子网。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户调用 API 不受影响。需要超级节点的用户应创建 TKE 标准集群并添加超级节点。

## 前置条件

- [环境准备](../../环境准备.md)
- 已注册腾讯云账号并完成实名认证
- 已通过 [CAM 控制台](https://console.cloud.tencent.com/cam) 开启 TKE 相关服务权限
- 已有可用的 VPC 和至少一个子网（子网需有充足 IP 地址）
- 确认当前地域集群数量未达到[购买限制](../购买限制/tccli%20操作.md)的上限（5 个）

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 创建 Serverless 集群 | `CreateEKSCluster` | 否（同名可能冲突） |
| 查询集群创建状态 | `DescribeEKSClusters` | 是 |
| 删除集群 | `DeleteEKSCluster` | 否（删除后不可恢复） |

## 操作步骤

### 步骤 1：准备创建参数

`CreateEKSCluster` 关键参数：

| 参数 | 必填 | 说明 |
|------|------|------|
| `ClusterName` | 是 | 集群名称，最长 60 字符 |
| `VpcId` | 是 | 所属 VPC ID |
| `SubnetIds` | 是 | 子网 ID 列表（多个可用区建议配置多个子网） |
| `K8SVersion` | 否 | Kubernetes 版本，支持 v1.16+ |
| `ClusterDesc` | 否 | 集群描述信息 |
| `ServiceSubnetId` | 否 | Service CIDR 子网 ID |
| `DnsServers` | 否 | DNS 服务器列表 |
| `NeedDeleteCbs` | 否 | 删除集群时是否自动删除关联的 CBS |
| `EnableVpcCoreDNS` | 否 | 是否开启 VPC 内网 CoreDNS |

### 步骤 2：创建集群

使用 `--cli-input-json` 创建集群：

```bash
tccli tke CreateEKSCluster --cli-input-json '{
    "ClusterName": "<ClusterName>",
    "VpcId": "<VpcId>",
    "SubnetIds": ["<SubnetId-1>", "<SubnetId-2>"],
    "K8SVersion": "1.28",
    "ClusterDesc": "<ClusterDesc>",
    "ServiceSubnetId": "<ServiceSubnetId>",
    "DnsServers": [
        {"Domain": "cluster.local", "DnsServers": ["10.0.0.10"]}
    ]
}' --region <Region> --output json
```

示例输入（使用占位符）：

```bash
tccli tke CreateEKSCluster --cli-input-json '{
    "ClusterName": "my-serverless-cluster",
    "VpcId": "vpc-of73262z",
    "SubnetIds": ["subnet-rdmcho9m"],
    "K8SVersion": "1.28",
    "ClusterDesc": "TKE Serverless 生产集群"
}' --region ap-guangzhou --output json
```

> **注意：** 建议为容器网络配置多个可用区的子网，以提高工作负载分布和高可用性。请确保子网 IP 充足，不足会导致大规模工作负载创建失败。

示例输出：

```json
{
    "ClusterId": "cls-example",
    "RequestId": "abc123-def456-ghi789"
}
```

### 步骤 3：等待集群就绪

使用轮询查询集群状态直到 `Status` 为 `Running`：

```bash
tccli tke DescribeEKSClusters \
    --ClusterIds '["cls-example"]' \
    --region ap-guangzhou \
    --output json
```

Running 状态的示例输出：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "my-serverless-cluster",
            "ClusterDesc": "TKE Serverless 生产集群",
            "VpcId": "vpc-of73262z",
            "SubnetIds": ["subnet-rdmcho9m"],
            "Status": "Running",
            "CreatedTime": "2024-01-15T10:30:00Z",
            "K8SVersion": "1.28.3",
            "ClusterVersion": "1.28"
        }
    ],
    "RequestId": "abc123-def456-ghi789"
}
```

> **提示：** 创建过程通常需要 2-5 分钟。可使用简单的 shell 循环轮询 `Status` 字段直到值为 `Running`。

### 步骤 4：连接集群

集群就绪后，获取 kubeconfig 连接集群。参见[连接集群](../连接集群/tccli%20操作.md)。

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 集群创建成功 | `tke DescribeEKSClusters --ClusterIds '["cls-example"]' --region ap-guangzhou` | `Status: "Running"` |
| 集群信息完整 | 同上，检查 `VpcId`, `SubnetIds`, `K8SVersion` | 与创建参数一致 |

## 清理

如果不需要该集群，可以删除：

```bash
tccli tke DeleteEKSCluster \
    --ClusterId cls-example \
    --region ap-guangzhou \
    --output json
```

> **注意：** 删除集群会释放超级节点资源，集群内的所有 Pod 将被终止。请确认已备份必要数据。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| `CreateEKSCluster` 报 `LimitExceeded.EKSCluster` | 当前地域集群数已达上限（5 个） | 删除不再使用的集群或[提交工单](https://console.cloud.tencent.com/workorder/category)提升配额 |
| `CreateEKSCluster` 报 `InvalidParameter` | 参数格式错误或取值非法 | 检查 `VpcId`/`SubnetIds` 格式（以 `vpc-`/`subnet-` 开头），确认子网 IP 充足 |
| 集群长时间处于 `Creating` 状态 | 后端资源分配中，偶有延迟 | 等待 10 分钟，若仍异常 [在线咨询](https://cloud.tencent.com/online-service?from=doc_457) |
| `DescribeEKSClusters` 报 `ResourceNotFound` | 集群 ID 不存在或已删除 | 检查 `ClusterId` 是否正确 |
| 集群创建后状态为 `Idle` | 新建集群 7 天内无 Pod 创建/删除操作 | 在集群中创建 Pod 或等待自动激活 |

## 下一步

- [连接集群](../连接集群/tccli%20操作.md) — 获取 kubeconfig、连接集群
- [工作负载管理](../../Kubernetes%20对象管理/工作负载管理/tccli%20操作.md) — 在集群中部署工作负载
- [集群生命周期](../集群生命周期/tccli%20操作.md) — 了解集群状态流转

## 控制台替代

[容器服务控制台 → 集群 → 新建 → Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster/create) — 通过控制台表单创建，含向导式配置。
