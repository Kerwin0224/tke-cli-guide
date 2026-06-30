# Prometheus 监控概述（tccli）

> 对照官方：[Prometheus 监控概述](https://cloud.tencent.com/document/product/457/84543) · page_id `84543`

## 概述

Prometheus 监控服务（TencentCloud Managed Service for Prometheus，TMP）是基于开源 Prometheus 构建的高可用、全托管监控服务。与腾讯云容器服务（TKE）深度集成，兼容开源生态组件，结合腾讯云可观测平台告警与 Prometheus Alertmanager 能力，提供免搭建的高效运维能力。

通过 `tccli monitor` 系列 API 可完全替代控制台完成 TMP 实例的全生命周期管理：创建实例、关联集群、配置数据采集、精简指标、管理告警、查看用量及销毁实例。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已开通 Prometheus 监控服务（首次使用需授权服务角色 `TKE_QCSLinkedRoleInPrometheusService`）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看实例列表 | `tccli monitor DescribePrometheusInstances --region <Region>` | 是 |
| 查看实例详情 | `tccli monitor DescribePrometheusInstanceDetail --region <Region> --InstanceId <InstanceId>` | 是 |
| 查看实例用量 | `tccli monitor DescribePrometheusInstanceUsage --region <Region> --InstanceIds '["<InstanceId>"]'` | 是 |
| 查看可用地域 | `tccli monitor DescribePrometheusRegions` | 是 |
| 查看可用区 | `tccli monitor DescribePrometheusZones --region <Region>` | 是 |
| 查看概览统计 | `tccli monitor DescribePrometheusInstancesOverview` | 是 |

## 操作步骤

### 1. 查看当前区域 Prometheus 实例列表

```bash
tccli monitor DescribePrometheusInstances --region ap-guangzhou
```

```json
{
    "Response": {
        "InstanceSet": [
            {
                "InstanceId": "prom-example01",
                "InstanceName": "prod-monitoring",
                "InstanceStatus": 2,
                "VpcId": "vpc-example",
                "SubnetId": "subnet-example",
                "Zone": "ap-guangzhou-3",
                "DataRetentionTime": 15,
                "InstanceChargeType": 2,
                "SpecName": "标准版",
                "GrafanaInstanceId": "grafana-example",
                "RemoteWrite": "https://prom-example01.ap-guangzhou.prometheus.tencent clouds.com/api/v1/write"
            },
            {
                "InstanceId": "prom-example02",
                "InstanceName": "test-monitoring",
                "InstanceStatus": 2,
                "VpcId": "vpc-example",
                "SubnetId": "subnet-example",
                "Zone": "ap-guangzhou-4",
                "DataRetentionTime": 15,
                "InstanceChargeType": 2,
                "SpecName": "标准版"
            }
        ],
        "TotalCount": 2,
        "RequestId": "abc123-..."
    }
}
```

**实例状态码说明：**

| 状态码 | 含义 |
|--------|------|
| 1 | 正在创建 |
| 2 | 运行中 |
| 3 | 异常 |
| 4 | 重建中 |
| 5 | 销毁中 |
| 6 | 已停机 |

### 2. 按条件过滤实例

```bash
# 按实例名称过滤
tccli monitor DescribePrometheusInstances --region ap-guangzhou --InstanceName "prod-monitoring" --filter "TotalCount, InstanceSet[0].InstanceId, InstanceSet[0].InstanceStatus"

# 按运行中状态过滤
tccli monitor DescribePrometheusInstances --region ap-guangzhou --InstanceStatus '[2]' --filter "TotalCount"
```

### 3. 查看单实例详情

```bash
tccli monitor DescribePrometheusInstanceDetail --region ap-guangzhou --InstanceId prom-example01
```

```json
{
    "Response": {
        "InstanceId": "prom-example01",
        "InstanceName": "prod-monitoring",
        "VpcId": "vpc-example",
        "SubnetId": "subnet-example",
        "InstanceStatus": 2,
        "ChargeStatus": 1,
        "EnableGrafana": 1,
        "GrafanaURL": "https://prom-example01.ap-guangzhou.prometheus.tencentclouds.com",
        "InstanceChargeType": 1,
        "SpecName": "标准版",
        "DataRetentionTime": 15,
        "ExpireTime": "",
        "AutoRenewFlag": 0,
        "RequestId": "abc123-..."
    }
}
```

### 4. 查看可用地域和可用区

```bash
# 查看 Prometheus 支持的地域
tccli monitor DescribePrometheusRegions --filter "RegionSet[].Region"

# 查看广州地域支持的可用区
tccli monitor DescribePrometheusZones --region ap-guangzhou --filter "ZoneSet[].Zone"
```

### 特征与能力概览

| 能力 | CLI 操作 | 说明 |
|------|---------|------|
| 实例管理 | `CreatePrometheusInstance` / `DestroyPrometheusInstance` / `DescribePrometheusInstances` | 创建、销毁、查询实例 |
| 集群关联 | `CreatePrometheusClusterAgent` / `DescribePrometheusClusterAgents` / `DeletePrometheusClusterAgent` | 关联/解关联 TKE 集群 |
| 数据采集 | `CreatePrometheusConfig` / `DescribePrometheusConfig` / `ModifyPrometheusConfig` | ServiceMonitor、PodMonitor、RawJobs 配置 |
| 预聚合 | `CreatePrometheusRecordRuleYaml` / `DescribePrometheusRecordRules` | 管理预聚合规则 |
| 告警策略 | `CreatePrometheusAlertPolicy` / `DescribePrometheusAlertPolicy` / `ModifyPrometheusAlertPolicy` | 管理 TMP 告警策略 |
| 告警历史 | `DescribeAlarmHistories` | 查询告警历史记录 |
| 用量查询 | `DescribePrometheusInstanceUsage` | 查询按量计费用量 |

## 验证

```bash
# 确认实例列表可正常访问
tccli monitor DescribePrometheusInstances --region ap-guangzhou --filter "TotalCount"
```

## 清理

概念页无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribePrometheusInstances` 返回空列表 | `tccli configure list` 确认 region 为目标地域 | 目标地域暂无 Prometheus 实例，或 region 配置错误 | 确认 `--region` 为目标地域；如确实无实例，参见 [创建监控实例](../创建监控实例/tccli%20操作.md) |
| `DescribePrometheusInstances` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 未授权服务角色 `TKE_QCSLinkedRoleInPrometheusService`（此为环境限制，非命令错误） | 登录 [CAM 控制台](https://console.cloud.tencent.com/cam/role) 为 TKE 服务授权角色 |
| 接口调用超时或不可达 | `curl -s -o /dev/null -w '%{http_code}' https://monitor.tencentcloudapi.com` | 网络环境限制或 CAM 策略阻断（此为环境限制） | 检查 `monitor.tencentcloudapi.com` 可达性及 CAM 权限配置 |

## 下一步

- [创建监控实例](../创建监控实例/tccli%20操作.md) 创建首个 TMP 实例
- [关联集群](../关联集群/tccli%20操作.md) 将 TKE 集群关联到 TMP 实例
- [数据采集配置](../数据采集配置/tccli%20操作.md) 配置 ServiceMonitor / PodMonitor

## 控制台替代

控制台：容器服务控制台 → 云原生服务 → Prometheus 监控 → 查看实例列表及详情。
