# 计费方式和资源使用（tccli）

> 对照官方：[计费方式和资源使用](https://cloud.tencent.com/document/product/457/71905) · page_id `71905`

## 概述

Prometheus 监控服务（TMP）的费用由两部分组成：**TMP 服务本身**（数据点采集和存储）和**实际消耗的云资源**（TKE Serverless 集群、CLB 负载均衡）。通过 `tccli monitor` 可查询实例用量和费用估算数据。

TMP 免费指标存储时长调整为 15 天（自 2022-10-27 起），超出 15 天的免费指标存储也会产生费用。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已创建 Prometheus 监控实例（参见 [创建监控实例](../创建监控实例/tccli%20操作.md)）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看实例列表和规格 | `tccli monitor DescribePrometheusInstances --region <Region>` | 是 |
| 查看实例详情 | `tccli monitor DescribePrometheusInstanceDetail --InstanceId <InstanceId>` | 是 |
| 查看实例用量 | `tccli monitor DescribePrometheusInstanceUsage --InstanceIds '["<InstanceId>"]'` | 是 |
| 查看实例费用估算 | `tccli monitor DescribePrometheusInstanceUsage` 计算每日数据点 | 是 |

## 操作步骤

### 1. 查看实例基本信息和计费配置

```bash
tccli monitor DescribePrometheusInstanceDetail --region ap-guangzhou --InstanceId prom-example01
```

```json
{
    "Response": {
        "InstanceId": "prom-example01",
        "InstanceName": "prod-monitoring",
        "InstanceStatus": 2,
        "ChargeStatus": 1,
        "InstanceChargeType": 1,
        "SpecName": "标准版",
        "DataRetentionTime": 15,
        "AutoRenewFlag": 0,
        "VpcId": "vpc-example",
        "SubnetId": "subnet-example",
        "RequestId": "abc123-..."
    }
}
```

**关键计费字段：**

| 字段 | 说明 |
|------|------|
| `InstanceChargeType` | `1` = 包年包月，`2` = 按量计费 |
| `ChargeStatus` | 计费状态 |
| `DataRetentionTime` | 数据保留天数，超过该天数数据自动清理 |
| `AutoRenewFlag` | `0` = 不自动续费，`1` = 自动续费，`2` = 禁止自动续费 |

### 2. 查看按量计费用量

```bash
tccli monitor DescribePrometheusInstanceUsage --region ap-guangzhou --InstanceIds '["prom-example01"]'
```

```json
{
    "Response": {
        "UsageSet": [
            {
                "InstanceId": "prom-example01",
                "CalcDate": "2026-06-16",
                "Total": 156000.0,
                "Basic": 80000.0,
                "Fee": 76000.0
            },
            {
                "InstanceId": "prom-example01",
                "CalcDate": "2026-06-15",
                "Total": 148000.0,
                "Basic": 78000.0,
                "Fee": 70000.0
            }
        ],
        "RequestId": "def456-..."
    }
}
```

**用量字段说明：**

| 字段 | 说明 |
|------|------|
| `CalcDate` | 计费日期 |
| `Total` | 总数据点采集量（个/天） |
| `Basic` | 基础免费指标用量（个/天） |
| `Fee` | 付费指标用量（个/天） |

**每日数据点计算公式：**

```
每日数据点 = 采集速率 × 86400
```

其中采集速率可在控制台「收费指标采集速率」列查看，该值基于实例上报量估算。

### 3. 查看所有实例列表（含规格信息）

```bash
tccli monitor DescribePrometheusInstances --region ap-guangzhou --filter "InstanceSet[].{InstanceId:InstanceId,InstanceName:InstanceName,InstanceChargeType:InstanceChargeType,SpecName:SpecName,DataRetentionTime:DataRetentionTime}"
```

```json
{
    "InstanceSet": [
        {"InstanceId": "prom-example01", "InstanceName": "prod-monitoring", "InstanceChargeType": 1, "SpecName": "标准版", "DataRetentionTime": 15},
        {"InstanceId": "prom-example02", "InstanceName": "test-monitoring", "InstanceChargeType": 2, "SpecName": "标准版", "DataRetentionTime": 15}
    ]
}
```

### 资源清单

创建 Prometheus 监控实例时，系统会自动创建以下关联资源：

| 资源类型 | 说明 | 计费方式 |
|---------|------|---------|
| **TMP 实例** | Prometheus 监控服务本身，按数据点量计费 | 包年包月 / 按量计费 |
| **TKE Serverless 集群** | 用于数据采集的 eks 集群，名称等于 TMP 实例 ID，描述为「Prometheus监控专用，请勿修改或删除」 | 按量计费 |
| **内网 CLB** | 默认创建，用于采集器与用户集群网络连通 | 按量计费 |
| **公网 CLB** | 仅在关联边缘集群或跨 VPC 无网络连通时创建 | 按量计费 |

**TKE Serverless 集群资源与每日费用参考：**

| 瞬时 Series 量级 | 预估资源 | 每日刊例价（元） |
|-----------------|---------|----------------|
| < 50 万 | 1.25核, 1.6GiB | ~1.3 |
| 100 万 | 0.5核 1.5GiB x 2 | ~5.5 |
| 500 万 | 1核 3GiB x 3 | ~11 |
| 2000 万 | 1核 6GiB x 5 | ~30 |

### 4. 查看特殊属性（归档存储、创建来源等）

```bash
tccli monitor DescribePrometheusInstances --region ap-guangzhou --InstanceIds '["prom-example01"]' --filter "InstanceSet[0].InstanceAttributes"
```

常见属性 key：
- `LongTermStorageRetentionTime` — 归档存储时长（天）
- `CreatedFrom` — 创建来源（`0` 控制台，`1` TKE 集群详情页，`2` 新建集群页）
- `FreeTrialExpireAt` — 免费试用到期时间（RFC3339 格式）
- `ResourcePackageID` — 关联的资源包 ID

## 验证

```bash
# 确认计费用量在可接受范围
tccli monitor DescribePrometheusInstanceUsage --region ap-guangzhou --InstanceIds '["prom-example01"]' --filter "UsageSet[-1].{Date:CalcDate,Total:Total,Basic:Basic,Fee:Fee}"
```

## 清理

**注意：** 用户不能直接从 TKE Serverless 集群或 CLB 控制台删除关联资源。要销毁所有关联资源，必须删除 Prometheus 监控实例（参见 [销毁监控实例](../销毁监控实例/tccli%20操作.md)）。

腾讯云不会主动回收监控实例，请在不再使用时及时销毁以避免持续计费。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribePrometheusInstanceUsage` 返回空 | `tccli monitor DescribePrometheusInstanceDetail --InstanceId <InstanceId>` 检查 `InstanceChargeType` | 实例为包年包月类型时不显示按量用量（此为产品设计，非命令错误） | 包年包月实例费用在购买时确定，无需关注按量用量 |
| 用量突增 | `tccli monitor DescribePrometheusInstanceUsage --InstanceIds '["<InstanceId>"]'` 对比近几日用量趋势 | 采集配置中 `interval` 过短或存在冗余未过滤指标 | 检查 [精简监控指标](../精简监控指标/tccli%20操作.md) 过滤非必要指标，增大采集间隔 |
| 基础指标也产生费用 | `tccli monitor DescribePrometheusInstanceDetail --InstanceId <InstanceId>` 检查 `DataRetentionTime` | 免费指标存储时长 15 天，`DataRetentionTime` 超过 15 天的部分按存储量计费 | 调低 `DataRetentionTime` 至 15 天或清理不必要的长周期存储 |

## 下一步

- [销毁监控实例](../销毁监控实例/tccli%20操作.md) 删除不再需要的实例避免持续计费
- [精简监控指标](../精简监控指标/tccli%20操作.md) 减少付费指标用量
- 参见 [TKE Serverless 集群定价](https://cloud.tencent.com/document/product/457/92840) 和 [CLB 计费](https://cloud.tencent.com/document/product/214/42935)

## 控制台替代

控制台：Prometheus 监控 → 实例列表（查看规格和用量）→ 点击实例名 → 集群监控/关联集群/数据采集配置页均可查看「收费指标采集速率」。
