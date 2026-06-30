# 计费方式和资源使用

> 对照官方：[计费方式和资源使用](https://cloud.tencent.com/document/product/457/71905) · page_id `71905`

## 概述

腾讯云 Prometheus 监控服务的费用由监控数据存储量决定，支持两种计费模式：**按量计费（后付费）**和**包年包月（预付费）**。了解计费模型有助于在 CLI 创建实例时选择合适的数据保留时长和规格，控制监控成本。

**计费构成**：

| 计费项 | 说明 | 计费方式 |
|--------|------|:--:|
| 数据存储 | 按实际存储的监控数据量计费，受采集指标数量、标签基数、数据保留时长影响 | 按量/包年包月 |
| 数据上报 | 免费（公有云环境 agent 上报） | 免费 |
| 基础监控 | 免费（agent 基础组件运行） | 免费 |
| Grafana 集成 | 可选，使用已有 Grafana 实例或创建新实例 | 独立计费 |

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限
tccli monitor DescribePrometheusInstances --region <Region>
# expected: exit 0，返回实例列表（可为空）
```

### 资源检查

```bash
# 4. 查看当前实例及计费模式
tccli monitor DescribePrometheusInstances --region <Region>
# expected: 返回实例列表，包含 InstanceChargeType 字段
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看实例及计费信息 | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` | 是 |

## 操作步骤

### 计费模式对比

| 维度 | 按量计费 (POSTPAID) | 包年包月 (PREPAID) |
|------|:--:|:--:|
| 付费方式 | 后付费，按小时结算 | 预付费，按月/年购买 |
| 成本特点 | 随数据量波动，适合测试和弹性场景 | 固定费用，适合长期稳定使用 |
| 数据保留 | 实例类型决定（15 天/30 天/90 天/180 天） | 同上 |
| 实例类型 | basic / standard / premium | basic / standard / premium |
| 续费 | 自动结算 | 可选自动续费或手动续费 |
| 适用场景 | 开发测试、短期项目、数据量波动大 | 生产环境、长期监控、数据量稳定 |

### 实例类型与数据保留

| 实例类型 | 数据保留时长 | 适用场景 |
|----------|:--:|------|
| `basic`（基础版） | 15 天 | 开发测试、小规模集群 |
| `standard`（标准版） | 30 天 | 一般生产环境 |
| `premium`（高级版） | 90 天 | 大规模集群、长期分析 |
| 旗舰版 | 180 天 | 合规/审计场景、超大规模 |

在 `CreatePrometheusInstance` 中，实例类型和数据保留时长通过以下方式控制：
- **实例类型**：由控制台选择（CLI 可能通过 `InstanceType` 参数指定，具体查看 help）
- **DataRetentionTime**：直接指定数据保留天数，需与实例类型支持的时长一致

```bash
# 查看 CreatePrometheusInstance 支持的参数
tccli monitor CreatePrometheusInstance --help
# expected: 输出参数列表，包含 PayMode、DataRetentionTime、Zone、VpcId、SubnetId 等
```

### 成本控制策略

**1. 精简监控指标（最有效）**

通过 `metric_relabel_configs` 丢弃不需要的指标，直接减少数据上报量。详见 [精简监控指标](../精简监控指标/tccli%20操作.md)。

```bash
tccli monitor ModifyPrometheusConfig --region <Region> \
    --cli-input-json file://config-with-metric-relabel.json
# 在 scrape_config 中配置 metric_relabel_configs 丢弃低频指标
```

**2. 选择合适的数据保留时长**

按需选择，避免过度保留。测试环境用基础版 15 天，生产环境按实际需求选择。

**3. 降低标签基数**

高基数标签（如 `pod_uid`、`container_id`）会导致时间序列数量激增，通过 `labeldrop` 减少。

**4. 使用 recording rules**

通过 recording rules 预计算聚合指标，仅保留聚合结果而非原始数据点。

### 包年包月参数说明

创建包年包月实例时需额外指定计费周期参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| `InstanceChargeType` | String | `PREPAID` 表示包年包月 |
| `InstanceChargePrepaid.Period` | Integer | 购买时长（月），如 `1`、`3`、`12` |
| `InstanceChargePrepaid.RenewFlag` | String | 续费方式：`NOTIFY_AND_AUTO_RENEW`（自动续费）/ `NOTIFY_AND_MANUAL_RENEW`（手动续费）/ `DISABLE_NOTIFY_AND_MANUAL_RENEW`（到期不续） |

按量计费时只需 `InstanceChargeType` 为 `POSTPAID`，无需额外计费周期参数。

### 实例隔离与销毁

欠费状态下实例的处理流程：

```
正常运行 → 欠费 (Isolated) → 7 天后自动 → 销毁 (Terminated)
```

- 欠费后实例状态变为 `Isolated`（`InstanceStatus: 3`），停止数据采集和查询
- 数据保留 7 天，期间可充值恢复
- 7 天后实例自动销毁，数据永久删除

## 验证

```bash
# 查看当前实例的计费信息
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]' | jq '.InstanceSet[0] | {InstanceName, InstanceChargeType, DataRetentionTime, InstanceStatus}'
# expected: 返回计费类型、数据保留时长和实例状态

# 查看所有实例的计费概览
tccli monitor DescribePrometheusInstances --region <Region> | jq '.InstanceSet[] | {InstanceName, InstanceChargeType, DataRetentionTime}'
# expected: 列出所有实例的计费信息
```

## 清理

> **计费提醒**：实例销毁后停止产生费用。包年包月未到期销毁按比例退款。按量计费销毁后立即停止计费。销毁前确认是否已备份必要的告警规则和监控数据。

```bash
# 查看待清理实例的计费信息
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]' | jq '.InstanceSet[0] | {InstanceName, InstanceChargeType, InstanceStatus}'

# 销毁实例（停止计费）
tccli monitor DestroyPrometheusInstance --region <Region> \
    --InstanceId '<InstanceId>'
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePrometheusInstance` 返回 `FailedOperation.TradeFailed` | 检查账户余额 | 包年包月时余额不足（此为环境限制，非命令错误） | 充值或切换为按量计费 `POSTPAID` |
| `CreatePrometheusInstance` 返回 `InvalidParameter.DataRetentionTime` | 检查 DataRetentionTime 值 | 指定的保留时长不被该实例类型支持 | 选择实例类型支持的保留时长（15/30/90/180） |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 实例被意外隔离（`InstanceStatus: 3`） | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 检查状态 | 账户欠费导致隔离 | 充值后实例自动恢复，无需操作 |
| 按量计费成本超出预期 | 检查数据上报量 | 采集了过多指标或高基数标签导致存储成本上升 | 配置 `metric_relabel_configs` 精简指标（见 [精简监控指标](../精简监控指标/tccli%20操作.md)） |

## 下一步

- [创建监控实例](../创建监控实例/tccli%20操作.md) — 按所选计费模式创建实例
- [精简监控指标](../精简监控指标/tccli%20操作.md) — 控制监控成本的核心手段
- [销毁监控实例](../销毁监控实例/tccli%20操作.md) — 销毁实例停止计费

## 控制台替代

控制台：容器服务控制台 → 运维功能 → Prometheus 监控 → 实例列表 → 详情页查看计费信息。
