---
doc_type: How-to
subtype: 6B
fused: false
---
# 日志采集

> 安装日志 Agent、采集业务日志与控制面日志到 CLS。配置型操作（开启采集行为，不创建资源）。

## 概述

TKE 日志分两类，采集方式不同：

| 日志类型 | 采集方式 | 内容 | 文档 |
|:---------|:---------|:-----|:-----|
| 业务日志 | CLS Agent（Pod 日志） | 应用 stdout/文件 | 本页 |
| 控制面日志 | `EnableControlPlaneLogs` | Master 组件（apiserver/etcd/...） | 本页 |
| 审计日志 | `EnableClusterAudit` | API 操作记录 | [审计](../security/audit.md) |

> 业务日志与控制面日志都投递到 CLS，但开启接口不同。日志查询在 CLS 服务（跨产品 `tccli cls`）。

## 决策依据

#### CLS Agent vs 控制面日志

- **CLS Agent**: 采集 Pod 业务日志（应用输出），需 `InstallLogAgent` 安装 Agent + `CreateCLSLogConfig` 配置采集规则
- **控制面日志**: 采集 Master 组件日志（apiserver/scheduler/controller-manager/etcd），仅托管集群支持
- **默认推荐**: 业务日志用 CLS Agent；控制面问题排查时开启控制面日志
- **能关闭吗?**: 能。`UninstallLogAgent` / `DisableControlPlaneLogs`

## 配置项

### InstallLogAgent

> 来源：`tccli tke InstallLogAgent --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 作用 | 填错的影响 |
|:------|------|:--------:|:-----|:-----------|
| ClusterId | string | 是 | 集群 ID | `ResourceNotFound` |
| KubeletRootDir | string | 否 | kubelet 根目录，默认 `/var/lib/kubelet` | Agent 装错位置 |
| ClusterType | string | 否 | 集群类型 | 类型不匹配 |

### EnableControlPlaneLogs

> 来源：`tccli tke EnableControlPlaneLogs --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 作用 |
|:------|------|:--------:|:-----|
| ClusterId | string | 是 | 集群 ID |
| ClusterType | string | 否 | 集群类型 |
| Components[] | list | 是 | 组件列表 |
| Components[].Name | string | 是 | 组件名（kube-apiserver/etcd/kube-scheduler/kube-controller-manager） |
| Components[].LogLevel | int | 否 | 日志级别 |
| Components[].LogSetId | string | 是 | CLS 日志集 ID |
| Components[].TopicId | string | 是 | CLS 日志主题 ID |
| Components[].TopicRegion | string | 是 | CLS 主题地域 |

> 控制面日志每个组件需指定独立的 CLS TopicId。须先在 CLS 创建日志集与主题。

## 应用

### 安装 CLS Agent（业务日志）

```bash
tccli tke InstallLogAgent --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0, Agent 以 DaemonSet 部署到集群
```

### 配置业务日志采集规则

```bash
# 创建 CLS 日志采集配置（指定采集哪些 Pod 日志）
tccli tke CreateCLSLogConfig --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --ConfigJson '{"Name":"app-log","InputType":"container","LogSetId":"<LOGSET_ID>","TopicId":"<TOPIC_ID>"}'
# expected: exit 0
```

### 开启控制面日志

```bash
tccli tke EnableControlPlaneLogs --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Components '[{"Name":"kube-apiserver","LogSetId":"<LOGSET_ID>","TopicId":"<TOPIC_ID>","TopicRegion":"ap-guangzhou"}]'
# expected: exit 0
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<LOGSET_ID>` | CLS 日志集 ID | CLS 创建返回 | `tccli cls CreateLogset` |
| `<TOPIC_ID>` | CLS 日志主题 ID | CLS 创建返回 | `tccli cls CreateTopic` |

## 验证

```bash
# 查看日志开关状态
tccli tke DescribeLogSwitches --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: exit 0, 返回 SwitchSet[] (每集群一项, 含 Audit/Event/Log/MasterLog 开关 + ClusterId)
```

```json
{
    "SwitchSet": [
        {"ClusterId": "cls-example", "Audit": false, "Event": false, "Log": true, "MasterLog": false}
    ],
    "RequestId": "xxx"
}
```

> 字段含义：`Audit`=审计日志开关，`Event`=K8s 事件日志开关，`Log`=业务日志（CLS Agent）开关，`MasterLog`=控制面日志开关。任一为 `true` 表示该类日志已开启，`false` 表示未开启。

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| Agent 安装 | `kubectl get ds -n kube-system \| grep logagent` | CLS Agent DaemonSet Running |
| 业务日志投递 | `tccli cls SearchLog --TopicId <ID>` | 有 Pod 日志记录 |
| 控制面日志 | `tccli cls SearchLog --TopicId <API_TOPIC>` | 有 apiserver 日志 |

> 日志投递有秒级延迟。开启后产生一次操作，再到 CLS 检索确认。

## 回滚

```bash
# 关闭控制面日志
tccli tke DisableControlPlaneLogs --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0

# 卸载 CLS Agent
tccli tke UninstallLogAgent --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0

# 删除采集配置
tccli tke DeleteLogConfigs --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --ConfigId "<CONFIG_ID>"
# expected: exit 0
```

> 卸载 Agent 后已投递的 CLS 日志保留，按 CLS 保留期过期。停止 CLS 计费需删除日志集。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound` (LogSetId/TopicId) | `tccli cls DescribeLogsets` | CLS 日志集/主题不存在 | 先在 CLS 创建 |
| `FailedOperation` | `DescribeClusterStatus` 看状态 | 集群非 Running | 等集群 Running |
| `UnsupportedOperation` | 查集群类型 | 控制面日志仅托管集群支持 | 用托管集群 |
| `ResourceInUse` | `DescribeLogSwitches` | Agent 已安装或日志已开启 | 先卸载/关闭再重装 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| Agent 安装但 Pod 日志不投递 | `kubectl get ds -n kube-system` | Agent DaemonSet 未就绪或采集配置错 | 查 Agent Pod 日志，核对 ConfigJson |
| 控制面日志 CLS 无记录 | `tccli cls SearchLog` | TopicId 错或索引未开 | 核对 TopicId，CLS 开启主题索引 |
| 部分节点无业务日志 | `kubectl get ds -n kube-system -o wide` | Agent 未调度到某节点（污点/资源） | 检查节点污点与资源 |

## 集群巡检与日志配置

> 集群健康巡检结果查询、控制面日志查询、日志配置管理。

### 集群巡检

```bash
# 巡检结果概览 (按集群, GroupBy 分组)
tccli tke DescribeClusterInspectionResultsOverview --ClusterIds '["<CLUSTER_ID>"]' --region <REGION>
# expected: exit 0, Statistics[] 含 HealthyLevel(warn/serious/healthy)+Count
```
```json
{
    "Statistics": [{"HealthyLevel": "warn", "Count": 3}, {"HealthyLevel": "serious", "Count": 1}],
    "Diagnostics": []
}
```

```bash
# 指定集群最新巡检结果 (ClusterIds[] + Name 过滤)
tccli tke ListClusterInspectionResults --ClusterIds '["<CLUSTER_ID>"]' --region <REGION>
# expected: exit 0, InspectionResults[] 含 ClusterId/StartTime/EndTime/Statistics
```

```bash
# 巡检结果明细 (按时间范围, ClusterId 必填)
tccli tke ListClusterInspectionResultsItems --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --StartTime "<START_TIME>" --EndTime "<END_TIME>"
# expected: exit 0, 巡检项明细 InspectionResultsItems[]
```

> `HealthyLevel` 状态：`healthy`/`warn`/`serious`/`risk`。三个巡检 Action 层级：`DescribeClusterInspectionResultsOverview` 按 Region/集群批量查概览，`ListClusterInspectionResults` 查指定集群最新一次巡检结果，`ListClusterInspectionResultsItems` 按时间范围查历史明细（`ClusterId` 必填）。集群巡检是集群级配置诊断，与 [节点级健康检查策略](../nodes/health-check.md)（2022-05-01 `HealthCheckPolicy`）层级与版本均不同，勿混。

### 控制面日志与日志配置

```bash
# 查询控制面日志 (需 ClusterType, 区分托管/独立集群)
tccli tke DescribeControlPlaneLogs --ClusterId "<CLUSTER_ID>" --ClusterType "<CLUSTER_TYPE>" --region <REGION>
# expected: exit 0, 控制面组件日志开关
```

> ⚠️ `DescribeControlPlaneLogs` 必填 `--ClusterType`（`MANAGED_CLUSTER`/`INDEPENDENT_CLUSTER`），缺失报 `the following arguments are required: --ClusterType`（exit 252）。

```bash
# 查询日志采集配置
tccli tke DescribeLogConfigs --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, 日志配置列表

# 修改日志采集配置
tccli tke ModifyLogConfig --ClusterId "<CLUSTER_ID>" --region <REGION> --Name "<CONFIG_NAME>" --ConfigData "<CONFIG_JSON>"
# expected: exit 0
```

> 控制面日志仅独立集群（INDEPENDENT_CLUSTER）可查（托管集群 Master 不可见）。`DescribeLogConfigs`/`ModifyLogConfig` 管理业务日志采集配置。

## 下一步

- [审计日志](../security/audit.md) — 审计日志（区别于业务/控制面日志）
- [Prometheus 监控入门](prometheus.md) — 指标监控
- [故障排查](../troubleshooting.md) — 日志缺失诊断

## 控制台替代方案

[容器服务控制台 - 日志采集](https://console.cloud.tencent.com/tke2/cluster)
