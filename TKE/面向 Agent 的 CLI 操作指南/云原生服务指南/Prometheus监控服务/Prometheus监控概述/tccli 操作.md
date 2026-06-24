# Prometheus监控概述

> 对照官方：[Prometheus监控概述](https://cloud.tencent.com/document/product/457/84543) · page_id `84543`

## 概述

腾讯云 Prometheus 监控服务（Tencent Cloud Managed Service for Prometheus，TMP）是全托管、免运维的 Prometheus 服务，与 TKE 深度集成。通过安装集群代理（Cluster Agent）即可自动采集 TKE 集群的监控数据，无需自行搭建和维护 Prometheus Server。支持数据采集、存储、告警、可视化一体化的监控方案。

**核心特点**：

| 特性 | 说明 |
|------|------|
| 全托管 | 无需部署 Prometheus Server，自动扩缩容、高可用 |
| TKE 集成 | 通过 `CreatePrometheusClusterAgent` 一键关联 TKE 集群 |
| 兼容生态 | 原生 PromQL 查询、Alertmanager 告警规则、Grafana 可视化 |
| 数据存储 | 支持 15 天到 6 个月不等的保留时长，按量或包年包月计费 |
| 采集配置 | 通过 ServiceMonitor/PodMonitor CRD 或自定义 scrape config 管理采集目标 |
| 告警 | 内置告警模板 + 自定义规则，支持 Webhook/短信/邮件等通知渠道 |

**API 入口**：所有 Prometheus 相关操作使用 `tccli monitor` 服务的以下 API Action：

| Action | 用途 | 是否幂等 |
|--------|------|:--:|
| `CreatePrometheusInstance` | 创建托管 Prometheus 实例 | 否 |
| `DescribePrometheusInstances` | 查询实例列表/详情 | 是 |
| `DestroyPrometheusInstance` | 销毁实例 | 是 |
| `CreatePrometheusClusterAgent` | 关联 TKE 集群到 Prometheus 实例 | 否 |
| `DescribePrometheusClusterAgents` | 查询已关联的集群 | 是 |
| `DescribePrometheusConfig` | 查询采集配置 | 是 |
| `ModifyPrometheusConfig` | 修改采集配置 | 否 |
| `DescribePrometheusAlertHistory` | 查询告警历史 | 是 |

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 monitor 服务权限
#    需要以下 Action 名：
#    monitor:CreatePrometheusInstance, monitor:DescribePrometheusInstances
#    monitor:DestroyPrometheusInstance, monitor:CreatePrometheusClusterAgent
#    monitor:DescribePrometheusClusterAgents, monitor:DescribePrometheusConfig
#    monitor:ModifyPrometheusConfig, monitor:DescribePrometheusAlertHistory
# 验证：执行 DescribePrometheusInstances 确认权限
tccli monitor DescribePrometheusInstances --region <Region>
# expected: exit 0，返回实例列表（可为空）

# 4. 检查 TKE 权限（关联集群时需要）
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 5. 查询已有 Prometheus 实例
tccli monitor DescribePrometheusInstances --region <Region>
# expected: 可查看已有实例信息，确认是否需要新建

# 6. 查询可关联的 TKE 集群
tccli tke DescribeClusters --region <Region>
# expected: 至少有 1 个可关联的集群（ClusterStatus: "Running"）

# 7. 查询目标集群的 agent 关联状态
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId '<InstanceId>' \
    --ClusterId '<ClusterId>'
# expected: 返回当前关联状态
```

### 版本与规格选择

- **实例类型**：`basic`（基础版）/ `standard`（标准版）/ `premium`（高级版），按数据存储时长和功能区分
- **数据保留时长**：15 天（基础版）/ 30 天（标准版）/ 90 天（高级版）/ 180 天（旗舰版）
- **计费模式**：按量计费（后付费）或包年包月（预付费），详见 [计费方式和资源使用](../计费方式和资源使用/tccli%20操作.md)

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看实例列表 | `tccli monitor DescribePrometheusInstances --region <Region>` | 是 |
| 创建监控实例 | `tccli monitor CreatePrometheusInstance --region <Region> --cli-input-json file://input.json` | 否 |
| 关联 TKE 集群 | `tccli monitor CreatePrometheusClusterAgent --region <Region> --cli-input-json file://input.json` | 否 |
| 管理采集配置 | `tccli monitor ModifyPrometheusConfig --region <Region> --cli-input-json file://input.json` | 否 |
| 查看告警历史 | `tccli monitor DescribePrometheusAlertHistory --region <Region> --cli-input-json file://input.json` | 是 |
| 销毁实例 | `tccli monitor DestroyPrometheusInstance --region <Region> --InstanceId '<InstanceId>'` | 是 |

## 操作步骤

### 架构概述

Prometheus 监控服务的典型使用流程：

```
创建实例 → 关联集群 → 配置采集 → 配置告警 → 日常运维/监控
```

1. **创建实例**：通过 `CreatePrometheusInstance` 创建托管 Prometheus 实例，选择规格和存储时长
2. **关联集群**：通过 `CreatePrometheusClusterAgent` 在 TKE 集群中安装 agent，实现数据采集
3. **配置采集**：通过 `ModifyPrometheusConfig` 配置 scrape targets、ServiceMonitor 等采集规则
4. **精简指标**：通过采集配置过滤不需要的指标，降低存储成本
5. **告警配置**：基于 Prometheus Alertmanager，配置告警规则和通知渠道
6. **查看告警**：通过 `DescribePrometheusAlertHistory` 查询告警历史

### 实例生命周期

```
Creating → Running → Terminating → （销毁）
                     → Isolated （欠费隔离，7 天后自动销毁）
```

- **Creating**：实例创建中，约 2-5 分钟
- **Running**：正常运行，可关联集群、配置采集
- **Isolated**：账户欠费隔离，停止数据采集和查询，保留数据 7 天
- **Terminating**：正在销毁中

### 数据采集

采集数据通过安装在 TKE 集群内的 agent 组件完成。agent 自动同步监控实例中配置的 scrape 规则，采集到的数据上报到托管 Prometheus 实例中存储。

采集配置支持两种方式：
- **CRD 方式**：通过 ServiceMonitor / PodMonitor CRD 声明采集规则
- **配置文件方式**：直接在监控实例中配置 scrape_config

## 验证

```bash
# 验证 monitor 服务可访问
tccli monitor DescribePrometheusInstances --region <Region>
# expected: exit 0，返回 TotalCount（可为 0）

# 验证可查看 TKE 集群（关联时需要）
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 清理

> **警告**：销毁 Prometheus 实例会删除所有历史监控数据且不可恢复。生产环境销毁前务必确认已备份必要的监控数据和告警规则。

```bash
# 查询当前实例
tccli monitor DescribePrometheusInstances --region <Region>
# 确认 InstanceId、InstanceName、关联的集群数

# 销毁实例
tccli monitor DestroyPrometheusInstance --region <Region> \
    --InstanceId '<InstanceId>'
# expected: exit 0，返回 RequestId

# 验证已销毁
tccli monitor DescribePrometheusInstances --region <Region>
# expected: 列表中不再包含该实例
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribePrometheusInstances` 返回 `UnauthorizedOperation.CamNoAuth` | 检查 CAM 策略 | 缺少 `monitor:DescribePrometheusInstances` 权限 | 联系管理员授权 `monitor:DescribePrometheusInstances` |
| `CreatePrometheusInstance` 返回 `FailedOperation` | 检查参数和地域 | 地域不支持该实例类型或参数格式错误 | 确认地域支持 Prometheus 监控服务，检查 input JSON 格式 |
| `DescribePrometheusClusterAgents` 返回 `ResourceNotFound` | 检查 InstanceId | 实例不存在或已销毁 | 先用 `DescribePrometheusInstances` 确认 InstanceId |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 实例创建后长时间处于 `Creating` | `tccli monitor DescribePrometheusInstances --region <Region>` 查看 `InstanceStatus` | 后端创建缓慢（正常）或资源不足（异常） | 等待 5 分钟；超时后保留 region、InstanceId、RequestId 登录控制台查看 |
| 关联集群后无数据上报 | `tccli monitor DescribePrometheusClusterAgents --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` 查看 agent 状态 | agent 未正确安装或采集配置为空 | 检查集群中 agent 组件状态，确认采集规则已配置 |

## 下一步

- [创建监控实例](../创建监控实例/tccli%20操作.md) — 创建第一个 Prometheus 托管实例
- [关联集群](../关联集群/tccli%20操作.md) — 将 TKE 集群接入 Prometheus 监控
- [数据采集配置](../数据采集配置/tccli%20操作.md) — 配置指标采集规则
- [精简监控指标](../精简监控指标/tccli%20操作.md) — 过滤不需要的指标，控制成本
- [告警历史](../告警历史/tccli%20操作.md) — 查询和处理告警
- [计费方式和资源使用](../计费方式和资源使用/tccli%20操作.md) — 了解计费模型

## 控制台替代

控制台：容器服务控制台 → 运维功能 → Prometheus 监控。
