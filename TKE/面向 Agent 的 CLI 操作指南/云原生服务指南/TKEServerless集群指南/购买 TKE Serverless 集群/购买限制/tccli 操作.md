# 购买限制

> 对照官方：[购买限制](https://cloud.tencent.com/document/product/457/39821) · page_id `39821`

## 概述

TKE Serverless 集群在资源使用上有配额限制。了解这些限制有助于合理规划集群规模和避免创建失败。

主要限制维度：
- **集群配额**：每个地域最多可创建的 Serverless 集群数量。
- **Pod 配额**：每个集群可运行的 Pod 数量上限。
- **资源配额**：vCPU/内存/GPU 的可用总量（底层资源池容量）。
- **子网 IP 配额**：VPC-CNI 模式下每个 Pod 占用子网一个 IP，子网 CIDR 大小决定最大 Pod 数。
- **账号配额**：部分限制为账号级别，与 TKE 标准集群共享。

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
| 查看已有集群数 | `DescribeEKSClusters`（比对 `TotalCount`） | 是 |
| 查看 VPC 子网信息 | `vpc DescribeSubnets` | 是 |
| 查看当前 Pod 数量 | kubectl（`kubectl get pods -A`） | 是 |
| 查看受限提示 | 创建失败时返回的错误信息 | — |
| 提升配额 | 控制台提工单（无 CLI API） | — |

## 操作步骤

### 步骤 1：检查当前地域已有集群数

```bash
tccli tke DescribeEKSClusters --region <Region> | jq '.TotalCount'
# expected: 返回当前地域 Serverless 集群总数
```

**预期输出**：

```json
2
```

与集群配额上限比对。若已达上限，需删除不用的集群或提工单提升配额。

### 步骤 2：检查 VPC 子网 IP 余量

Serverless 集群采用 VPC-CNI 模式，每个 Pod 占用一个子网 IP。子网可用 IP 数直接决定该子网可调度的 Pod 数上限：

```bash
tccli vpc DescribeSubnets --region <Region> \
    --SubnetIds '["subnet-xxxxxxxx"]' \
    | jq '.SubnetSet[] | {SubnetId, CidrBlock, AvailableIpAddressCount}'
# expected: 返回子网 CIDR 和可用 IP 数
```

**预期输出**：

```json
{
    "SubnetId": "subnet-xxxxxxxx",
    "CidrBlock": "10.0.1.0/24",
    "AvailableIpAddressCount": 200
}
```

> 子网中已分配 IP = CIDR 总 IP 数 - 保留 IP（腾讯云预留约 3-5 个）- `AvailableIpAddressCount`。当可用 IP 不足时，新 Pod 将调度失败。

### 步骤 3：检查当前 Pod 运行数量

> **需 kubectl 可达环境**

```bash
kubectl get pods -A --field-selector=status.phase=Running | wc -l
# expected: 返回 Running Pod 数量（含标题行，实际 Pod 数 = 输出 - 1）
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：常见配额限制速查

| 限制项 | 默认上限 | 影响 |
|--------|---------|------|
| 每地域 Serverless 集群数 | 5（部分地域更低） | 超过后 `CreateEKSCluster` 返回配额超限 |
| 每集群最大 Pod 数（受子网 IP 限制） | 子网可用 IP 数 | 超过后 Pod 调度 Pending |
| 单 Pod 最大 vCPU | 32 核（部分地域更高） | 超过规格创建失败 |
| 单 Pod 最大内存 | 128 GiB（部分地域更高） | 超过规格创建失败 |
| 单 Pod 最大 GPU | 8 卡 | 超过规格创建失败 |
| 临时存储 | 30 GB（默认）/ 可声明上限 | 超过后 Pod 被驱逐 |

## 验证

```bash
# 验证集群数未超配额
tccli tke DescribeEKSClusters --region <Region> | jq '.TotalCount'
# expected: 小于集群配额上限

# 验证子网可用 IP 充足
tccli vpc DescribeSubnets --region <Region> \
    --SubnetIds '["subnet-xxxxxxxx"]' \
    | jq '.SubnetSet[0].AvailableIpAddressCount'
# expected: 大于当前/预期 Pod 数

# 验证 Pod 配额充足（与当前运行 Pod 数比较）
kubectl get pods -A --field-selector=status.phase=Running --no-headers | wc -l
# expected: 小于子网可用 IP 数
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

本页面为只读操作，无需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEKSCluster` 返回 `LimitExceeded.ClusterQuota` | `tccli tke DescribeEKSClusters --region <Region>` 查看已有集群数 | 该地域 Serverless 集群数已达配额上限 | 删除不用的集群后重试，或提工单申请提升配额 |
| `CreateEKSCluster` 返回 `LimitExceeded.SubnetIp` | `tccli vpc DescribeSubnets` 查看子网可用 IP | 所选子网可用 IP 不足 | 选择 CIDR 更大的子网，或清理子网中不再使用的 IP |
| `CreateEKSCluster` 返回 `ResourceInsufficient` | 换可用区或地域重试 | 底层资源池容量不足 | 等待资源补充后重试，或换其他可用区/地域创建 |
| Pod 调度 Pending 且 Events 提示 `Insufficient Subnet IP` | `tccli vpc DescribeSubnets` 确认可用 IP | 子网 IP 耗尽 | 扩展子网 CIDR、添加子网、或清理无用 Pod/服务 |
| Pod 创建报 `LimitExceeded.PodQuota` | `kubectl get pods -A --no-headers` 统计 Pod 数 | 集群 Pod 数超配额 | 删除无用 Pod，或提工单申请提升 Pod 配额 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 集群状态异常后无法创建新集群 | `tccli tke DescribeEKSClusters` 检查是否有 `Abnormal` 集群 | 异常集群仍占用配额 | 等待异常集群恢复或删除异常集群后重试 |
| Pod 运行正常但部分新 Pod 创建失败 | 分可用区检查子网 IP 余量 | 个别可用区 IP 不足 | 为集群添加其他可用区子网，使 Pod 可调度到其他可用区 |

## 下一步

- [创建 Serverless 集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md) — 在配额内创建集群
- [计费概述](../计费概述/tccli%20操作.md) — 了解创建集群后的计费影响
- [欠费说明](../欠费说明/tccli%20操作.md) — 欠费导致的资源限制

## 控制台替代

[TKE 控制台 → 创建 Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster/create)：创建页面实时校验集群数是否超配额，子网选择器显示可用 IP 数，超出限制时发出提示。
