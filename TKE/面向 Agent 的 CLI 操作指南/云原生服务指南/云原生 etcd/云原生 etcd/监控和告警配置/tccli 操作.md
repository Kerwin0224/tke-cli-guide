# 监控和告警配置

> 对照官方：[监控和告警配置](https://cloud.tencent.com/document/product/457/58179) · page_id `58179`

## 概述

云原生 etcd 默认提供四维度监控指标：节点资源、业务指标、实例级指标和实例接口指标，全部支持告警配置。可额外关联 Prometheus 监控服务（TMP）获取更多 etcd 原生指标和自定义 Grafana 大盘。

> 前提：已有运行中的 etcd 集群。若无，先执行 [创建集群](../创建集群/tccli%20操作.md)。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 至少一个状态为 `Running` 的 etcd 实例

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要 cetcd:DescribeEtcdInstances
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0

# 4. 确认有运行中的 etcd 实例
tccli cetcd DescribeEtcdInstances --region <Region> | jq '.InstanceSet[] | select(.Status=="Running") | .InstanceId'
# expected: 至少返回 1 个 InstanceId
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看实例详情（含监控入口） | `DescribeEtcdInstance` | 是 |
| 查看实例列表 | `DescribeEtcdInstances` | 是 |
| 配置 Prometheus 监控 | `AssociateEtcdWithPrometheus` | 否 |
| 获取监控数据 | `DescribeEtcdMonitorData` | 是 |

## 操作步骤

### 步骤 1：查看实例 ID 和当前状态

```bash
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0，返回所有实例
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "etcd-example",
            "InstanceName": "my-etcd",
            "Status": "Running",
            "EtcdVersion": "v3.5.12-tke.1"
        }
    ]
}
```

### 步骤 2：查看实例详情（获取监控入口）

```bash
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回实例完整配置
```

### 步骤 3：获取实例监控数据

```bash
tccli cetcd DescribeEtcdMonitorData --region <Region> \
    --InstanceId INSTANCE_ID \
    --MetricName CpuUsage \
    --StartTime START_TIME \
    --EndTime END_TIME
# expected: exit 0，返回监控指标数据
```

### 步骤 4：（可选）关联 Prometheus 监控实例

```bash
tccli cetcd AssociateEtcdWithPrometheus --region <Region> \
    --InstanceId INSTANCE_ID \
    --PrometheusInstanceId PROMETHEUS_INSTANCE_ID
# expected: exit 0
```

> **注意**：Prometheus 实例必须与 etcd 集群在同一 VPC，且已开启 Grafana。默认 Grafana Dashboard 不可修改，可复制后自定义。

## 验证

```bash
# 确认实例运行中
tccli cetcd DescribeEtcdInstance --region <Region> --InstanceId INSTANCE_ID | jq '.Status'
# expected: "Running"
```

监控指标采集间隔为 15 秒，控制台聚合粒度支持 1 分钟或 5 分钟。指标数据有 2-3 分钟延迟属正常现象。

## 清理

如需解除 Prometheus 关联：

```bash
tccli cetcd DisassociateEtcdFromPrometheus --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `AssociateEtcdWithPrometheus` 返回 `InvalidParameter` | 检查 PrometheusInstanceId 和 etcd InstanceId | Prometheus 实例不存在或不在同 VPC | 确认 Prometheus 实例与 etcd 在同一 VPC，且已开启 Grafana |
| `DescribeEtcdMonitorData` 返回空数据 | 确认 MetricName 是否正确 | 指标名错误或实例刚创建 | 等待 2-3 分钟后重试，确认 MetricName 使用正确值（如 `CpuUsage`） |

### 监控异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 监控数据缺失 | 执行 `tccli cetcd DescribeEtcdInstance --region <Region> --InstanceId INSTANCE_ID` 确认 Status | 实例异常或 Prometheus 关联失效 | 检查实例状态，重新关联 Prometheus |
| 告警未触发 | 登录控制台检查告警策略配置 | 告警阈值设置不当或通知渠道未配置 | 在 [云监控控制台](https://console.cloud.tencent.com/monitor) 检查告警策略 |

## 下一步

- [节点扩容和升降配](../节点扩容和升降配/tccli%20操作.md) — 调整节点规格
- [自动压缩管理](../自动压缩管理/tccli%20操作.md) — 配置自动压缩防止性能退化
- [快照管理](../快照管理/tccli%20操作.md) — 配置数据备份
- [腾讯云 Prometheus 监控文档](https://cloud.tencent.com/document/product/1416) — 更多 Prometheus 配置

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，点击实例进入 详情页 > 实例监控。
