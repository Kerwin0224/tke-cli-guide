# 欠费说明

> 对照官方：[欠费说明](https://cloud.tencent.com/document/product/457/86717) · page_id `86717`

## 概述

TKE Serverless 集群按量计费，若账户余额不足导致欠费，腾讯云会按阶段对集群和 Pod 采取限制和回收措施。

欠费处理阶段：
- **欠费后 24 小时内**：集群和 Pod 仍正常运行，正常计费。控制台和 API 均可正常操作。
- **欠费 24 小时后**：**服务自动停止**。所有 Pod 被驱逐并删除。集群资源被冻结，已有数据（PVC 关联的 CBS/CFS）保留。
- **欠费超过 7 天（部分资源 15 天）**：集群资源**进入回收站**，可在回收站保留期内恢复。超期未续费则**彻底销毁**，数据不可恢复。

> **重要**：欠费后尽快充值可以最小化影响。建议设置余额告警和自动充值，避免业务中断。

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
| 查看集群状态（判断是否因欠费停服） | `DescribeEKSClusters` | 是 |
| 查看账户余额 | 无 CLI API（计费中心查看） | — |
| 充值 | 无 CLI API（计费中心操作） | — |
| 设置余额告警 | 无 CLI API（计费中心/云监控配置） | — |
| 恢复集群（充值后） | 无主动恢复 CLI，等待系统自动恢复 | — |

## 操作步骤

### 步骤 1：检查集群当前状态

```bash
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0].Status'
# expected: "Running"（正常）或 "Abnormal"/"Isolated"（已停服）
```

**预期输出**（正常状态）：

```json
"Running"
```

**预期输出**（已停服）：

```json
"Isolated"
```

### 步骤 2：检查集群中 Pod 运行状态

> **需 kubectl 可达环境**

```bash
kubectl get pods -A
# expected: 正常时返回所有 Pod
# 欠费停服后：可能返回 "Unable to connect to the server" 或无 Pod
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：查看账户余额与欠费状态

登录 [计费中心 → 账户总览](https://console.cloud.tencent.com/expense/overview) 查看余额和欠费金额。

### 步骤 4：欠费处理流程

| 阶段 | 时间 | 集群状态 | Pod 状态 | 操作建议 |
|------|------|---------|---------|---------|
| 正常 | — | `Running` | Running | 确保余额充足 |
| 已欠费 | 0-24h | `Running` | Running | **立即充值**，无影响 |
| 停止服务 | 24h+ | `Isolated` | Terminated | 充值后自动恢复 |
| 进入回收站 | 7-15d | Deleted | — | 从回收站恢复 |
| 彻底销毁 | 15d+ | — | — | 不可恢复 |

## 验证

```bash
# 验证集群未被停服
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].Status'
# expected: "Running"

# 验证 Pod 可正常调度
kubectl get pods -A --field-selector=status.phase=Running | head -5
# expected: 返回正常运行的 Pod
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

> 注意：欠费停服后系统自动清理 Pod，但 PVC 关联的 CBS 存储仍会持续计费。若彻底不使用，请在集群停服前主动清理 PVC。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEKSClusters` 返回 `UnauthorizedOperation` 但之前可用 | 登录控制台查看账户状态 | 账户欠费导致 API 访问受限 | 充值后恢复 |
| `kubectl` 连接集群报 `Unable to connect to the server` | `tccli tke DescribeEKSClusters --ClusterIds` 检查集群状态 | 集群因欠费被停服，API Server 不可访问 | 充值后等待集群自动恢复（通常 5-10 分钟） |
| `DescribeEKSClusters` 返回的 `Status` 为 `Isolated` | — | 集群已因欠费超过 24 小时被停服 | 充值后等待系统自动恢复，通常 5-10 分钟内恢复为 `Running` |
| 充值后集群状态未恢复 | 等待 5-10 分钟后重试 `DescribeEKSClusters` | 系统恢复有延迟 | 若超过 30 分钟仍未恢复，提工单联系售后 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 充值后集群 `Running` 但 Pod 未自动恢复 | `kubectl get pods -A` 确认 | 欠费停服期间 Pod 被驱逐删除，不会自动恢复 | 重新部署工作负载（`kubectl apply -f`） |
| 充值后 PVC 数据丢失 | 确认回收站保留期内是否已超时 | 超过保留期（7-15 天）数据被彻底销毁 | 数据不可恢复。建议设置定期备份和余额告警 |
| 设置了余额告警但未收到通知 | 检查 [云监控告警策略](https://console.cloud.tencent.com/monitor/alarm2/policy) | 通知渠道（短信/邮件/微信）未配置或被拦截 | 在云监控中配置并验证通知渠道有效性 |

## 下一步

- [计费概述](../计费概述/tccli%20操作.md) — 了解完整的计费模式
- [集群生命周期](../../TKE%20Serverless%20集群管理/集群生命周期/tccli%20操作.md) — 集群状态变化与欠费的关系
- [创建集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md) — 充值恢复后重建工作负载

## 控制台替代

[TKE 控制台 → Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster)：集群列表页展示集群状态（正常 / 已隔离）。[计费中心 → 账户总览](https://console.cloud.tencent.com/expense/overview) 查看余额和欠费，设置余额告警。
