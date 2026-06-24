# 购买指南

> 对照官方：[购买指南](https://cloud.tencent.com/document/product/457/73706) · page_id `73706`

## 概述

云原生 etcd 支持两种计费模式：**包年包月**（预付费）和**按量计费**（后付费）。实例规格由 CPU、内存和副本数三个维度决定。购买前需确认配额是否足够。

| 计费模式 | 付费方式 | 计费周期 | 适用场景 |
|---------|---------|---------|---------|
| 包年包月（PREPAID） | 预付费，购买时付款 | 按月 | 长期稳定业务，单价比按量计费更低 |
| 按量计费（POSTPAID_BY_HOUR） | 后付费，按实际用量结算 | 按小时 | 短期或临时使用，释放即停费 |

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

# 3. 检查 CAM 权限 — 需要 cetcd:DescribeEtcdQuota
tccli cetcd DescribeEtcdQuota --region <Region>
# expected: exit 0，返回 QuotaLimit
```

### 资源检查

```bash
# 4. 查询当前实例配额
tccli cetcd DescribeEtcdQuota --region <Region>
# expected: 返回 QuotaLimit，确认未达上限
```

**预期输出**：

```json
{
    "QuotaLimit": 21
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看配额 | `DescribeEtcdQuota` | 是 |
| 查看可用版本 | `DescribeEtcdAvailableVersions` | 是 |
| 创建实例（包年包月） | `CreateEtcdInstance --ChargeType PREPAID` | 否 |
| 创建实例（按量计费） | `CreateEtcdInstance --ChargeType POSTPAID_BY_HOUR` | 否 |

## 关键字段说明

以下说明计费相关的 `CreateEtcdInstance` 参数。完整参数见 `tccli cetcd CreateEtcdInstance --help`。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ChargeType` | String | 是 | `PREPAID`（包年包月）或 `POSTPAID_BY_HOUR`（按量计费）。注意不是 `POSTPAID` | 填 `POSTPAID` → `InvalidParameter: supported values: PREPAID, POSTPAID_BY_HOUR` |
| `ChargePrepaid.Period` | Integer | PREPAID 时必填 | 购买时长，单位月。取值 1-36 | 不填则 `InvalidParameter` |
| `ChargePrepaid.RenewFlag` | String | PREPAID 时必填 | 自动续费标识：`NOTIFY_AND_MANUAL_RENEW`（通知但不自动续费）、`NOTIFY_AND_AUTO_RENEW`（通知且自动续费）、`DISABLE_NOTIFY_AND_MANUAL_RENEW`（不通知不自动续费） | 不填则 `InvalidParameter` |
| `Size` | Integer | 是 | etcd 集群节点数，Raft 协议要求奇数，最小 3 | Size=1 无法保证高可用 |
| `Cpu` | Integer | 是 | CPU 核数。此参数在 skeleton 解析中不在顶层 | 不填 → `InvalidParameter: "Cpu: Required value"` |

## 操作步骤

### 查询当前实例数和配额

```bash
# 查看已有实例数
tccli cetcd DescribeEtcdInstances --region <Region> | jq '.TotalCount'
# expected: 当前实例数

# 查看配额上限
tccli cetcd DescribeEtcdQuota --region <Region> | jq '.QuotaLimit'
# expected: 配额上限值（如 21）
```

## 验证

- `DescribeEtcdQuota` 返回 `QuotaLimit` > 当前实例数，即可创建新实例。
- 配额不足时需 [提交工单](https://console.cloud.tencent.com/workorder) 申请提升。

## 清理

本页面为只读操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEtcdQuota` 返回 `AuthFailure` | `tccli configure list` 检查凭据 | CAM 权限不足 | 确认已授予 `cetcd:DescribeEtcdQuota` 权限 |
| 配额已满无法创建 | 执行 `tccli cetcd DescribeEtcdInstances --region <Region>` 查看当前实例数 | 已达 QuotaLimit 上限（此为环境限制，非命令错误） | 删除不再使用的实例后重试，或 [提交工单](https://console.cloud.tencent.com/workorder) 申请提升配额 |

## 下一步

- [创建集群](../创建集群/tccli%20操作.md) — 创建新 etcd 实例
- [包年包月实例退费说明](../包年包月实例退费说明/tccli%20操作.md) — 了解退费规则
- [节点扩容和升降配](../节点扩容和升降配/tccli%20操作.md) — 调整节点规格

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)。
