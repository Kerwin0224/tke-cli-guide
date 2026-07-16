---
doc_type: How-to
subtype: 6B
fused: false
---
# 日志采集

> 控制台: [容器服务控制台 - 日志采集](https://console.cloud.tencent.com/tke2/cluster)
> 安装日志 Agent、采集业务日志与托管组件日志（`EnableControlPlaneLogs`，当前为 CA/Karpenter）到 CLS。配置型操作（开启采集行为，不创建资源）。
>
> 官方文档：[可观测体系概述](https://cloud.tencent.com/document/product/457/118975)
>
> 配额：CLS 日志集/主题存储限制见 CLS 服务配额，TKE 侧无额外配额限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- `DescribeLogSwitches` → `Log.Enable=false`（业务日志未采），需装 Agent 并建采集规则
- Pod 应用日志无法在 CLS 检索，`kubectl get ds -n kube-system | grep logagent` 无 CLS Agent DaemonSet
- 托管组件（`cluster-autoscaler`/`kapenter`）日志未采，需 `EnableControlPlaneLogs` — 看 [开启控制面日志](#开启控制面日志)
- `DescribeLogSwitches` → `MasterLog` 仅表示开关态只读字段；**当前 CLI 无 `EnableMasterLog` Action**（勿与 `EnableControlPlaneLogs` 混用）


## 概述

TKE 日志分多路，采集方式不同：

| 日志类型 | 采集方式 | 内容 | 文档 |
|:---------|:---------|:-----|:-----|
| 业务日志 | CLS Agent（Pod 日志） | 应用 stdout/文件 | 本页 |
| 托管组件日志 | `EnableControlPlaneLogs` | **当前支持** `cluster-autoscaler` / `kapenter`（不是 apiserver/etcd） | 本页 |
| 审计日志 | `EnableClusterAudit` | API 操作记录 | [审计](../security/audit.md) |
| 事件日志 | `EnableEventPersistence` | K8s Event | [审计/加固](../security/audit.md) |

> 业务日志与托管组件日志都投递到 CLS，但开启接口不同。`DescribeLogSwitches` 的 `MasterLog` 是只读开关态，**不等于** `EnableControlPlaneLogs` 所管的组件集。日志查询在 CLS 服务（跨产品 `tccli cls`）。

## 决策依据

#### CLS Agent vs 托管组件日志

- **CLS Agent**: 采集 Pod 业务日志（应用输出），需 `InstallLogAgent` 安装 Agent + `CreateCLSLogConfig` 配置采集规则
- **托管组件日志（`EnableControlPlaneLogs`）**: 官方接口名含 ControlPlane，实际组件列表当前为 **`cluster-autoscaler` / `kapenter`**；`ClusterType` **仅** `tke`
- **默认推荐**: 业务日志用 CLS Agent；要排查 CA/Karpenter 等托管组件时再开 `EnableControlPlaneLogs`
- **可关闭**： 是。`UninstallLogAgent` / `DisableControlPlaneLogs`（须 `ComponentNames`）

## 配置项

### InstallLogAgent

> 完整入参以 `tccli tke InstallLogAgent help --detail` 为准。

| 字段 | 类型 | 必填 | 作用 | 填错的影响 |
|:------|------|:--------:|:-----|:-----------|
| ClusterId | string | 是 | 集群 ID | `ResourceNotFound` |
| KubeletRootDir | string | 否 | kubelet 根目录，默认 `/var/lib/kubelet` | Agent 装错位置 |
| ClusterType | string | 否 | 集群类型：`tke` / `eks`（默认 `tke`） | 类型错被拒 |

### EnableControlPlaneLogs

> 完整入参以 `tccli tke EnableControlPlaneLogs help --detail` 为准。

| 字段 | 类型 | 必填 | 作用 |
|:------|------|:--------:|:-----|
| ClusterId | string | 是 | 集群 ID |
| ClusterType | string | 是 | **仅** `tke`（非 `MANAGED_CLUSTER`/`INDEPENDENT_CLUSTER`） |
| Components[] | list | 是 | 组件列表 |
| Components[].Name | string | 是 | 组件名：当前支持 `cluster-autoscaler` / `kapenter`（**不是** kube-apiserver/etcd 等） |
| Components[].LogLevel | int | 否 | 日志级别 |
| Components[].LogSetId | string | 否 | CLS 日志集 ID；不传则自动创建 |
| Components[].TopicId | string | 否 | CLS 日志主题 ID；不传则自动创建 |
| Components[].TopicRegion | string | 否 | CLS 主题地域（可跨地域投递） |

> 控制面日志接口当前面向插件组件（`cluster-autoscaler`/`kapenter`），`ClusterType` 固定传 `tke`。审计/事件/业务日志走 `EnableClusterAudit` / `EnableEventPersistence` / `InstallLogAgent` 等路径，勿与本接口混用。

## 应用

### 安装 CLS Agent（业务日志）

```bash
tccli tke InstallLogAgent --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0, Agent 以 DaemonSet 部署到集群
```

### 配置业务日志采集规则

```bash
# 创建 CLS 日志采集配置（参数名是 LogConfig，不是 ConfigJson）
# LogConfig 为 LogConfig CR YAML/JSON 字符串；完整样例见 tccli tke CreateCLSLogConfig help --detail
tccli tke CreateCLSLogConfig --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --LogConfig '{"apiVersion":"cls.cloud.tencent.com/v1","kind":"LogConfig","metadata":{"name":"app-log"},"spec":{"clsDetail":{"logType":"minimalist_log"},"inputDetail":{"type":"container_stdout","containerStdout":{"allContainers":true}}}}'
# expected: exit 0；可选 --LogsetId 指定 CLS 日志集
```

### 开启控制面日志

```bash
# ClusterType 仅 tke；Components.Name 当前支持 cluster-autoscaler / kapenter
tccli tke EnableControlPlaneLogs --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --ClusterType tke \
  --Components '[{"Name":"cluster-autoscaler","LogSetId":"<LOGSET_ID>","TopicId":"<TOPIC_ID>","TopicRegion":"ap-guangzhou"}]'
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
# expected: exit 0, 返回 SwitchSet[]（每集群一项；Audit/Event/Log/MasterLog 均为嵌套对象，非顶层布尔）
```

```json
{
    "SwitchSet": [
        {
            "ClusterId": "cls-example",
            "Audit": {"Enable": true, "Status": "opened", "TopicRegion": "ap-guangzhou", "ErrorMsg": "", "LogsetId": "", "TopicId": "", "UpgradeAble": false, "Version": ""},
            "Event": {"Enable": false, "Status": "closed", "ErrorMsg": "", "LogsetId": "", "TopicId": "", "TopicRegion": "", "UpgradeAble": false, "Version": ""},
            "Log": {"Enable": false, "Status": "closed", "ErrorMsg": "", "LogsetId": "", "TopicId": "", "TopicRegion": "", "UpgradeAble": false, "Version": ""},
            "MasterLog": {"Enable": false, "Status": "closed", "ErrorMsg": "", "LogsetId": "", "TopicId": "", "TopicRegion": "", "UpgradeAble": false, "Version": ""}
        }
    ],
    "RequestId": "xxx"
}
```

> 字段含义：`Audit`/`Event`/`Log`/`MasterLog` 各为对象；看 **`Enable`**（是否开启）与 **`Status`**（如 `opened`/`closed`）。`Log`=业务日志（CLS Agent），`Audit`=审计，`Event`=K8s 事件。`MasterLog` 是开关态只读字段（**当前 CLI 无 `EnableMasterLog`**），**不要**用它判断 `EnableControlPlaneLogs`（CA/Karpenter 组件日志）是否已开——组件日志用 `DescribeControlPlaneLogs --ClusterType tke`。勿写成顶层 `Log=false` 布尔。

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| Agent 安装 | `kubectl get ds -n kube-system \| grep logagent` | CLS Agent DaemonSet Running |
| 业务日志投递 | `tccli cls SearchLog --TopicId <ID>` | 有 Pod 日志记录 |
| 托管组件日志 | `DescribeControlPlaneLogs --ClusterType tke` + `tccli cls SearchLog` | Details 含目标组件；CLS 有对应日志 |

> 日志投递有秒级延迟。开启后产生一次操作，再到 CLS 检索确认。

## 回滚

```bash
# 关闭控制面日志（ClusterType 仅 tke；另需 ComponentNames）
tccli tke DisableControlPlaneLogs --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --ClusterType tke \
  --ComponentNames '["cluster-autoscaler"]'
# expected: exit 0

> `DisableControlPlaneLogs` 必填 `--ClusterId` / `--ClusterType tke` / `--ComponentNames`；`ClusterType` **不是** `MANAGED_CLUSTER`/`INDEPENDENT_CLUSTER`。

# 卸载 CLS Agent
tccli tke UninstallLogAgent --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0

# 删除采集配置（参数名是 LogConfigNames，不是 ConfigId；多个名称用逗号分隔）
tccli tke DeleteLogConfigs --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --LogConfigNames "app-log"
# expected: exit 0；可选 --ClusterType tke/eks（默认 tke）
```

> ⚠️ **高危操作**：卸载 CLS Agent 后日志采集中断，审计日志与业务日志将形成盲区，安全事件不可追溯。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)
>
> 卸载 Agent 后已投递的 CLS 日志保留，按 CLS 保留期过期。停止 CLS 计费需删除日志集。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound` (LogSetId/TopicId) | `tccli cls DescribeLogsets` | CLS 日志集/主题不存在 | 先在 CLS 创建 |
| `FailedOperation` | `DescribeClusterStatus` 查看状态 | 集群非 Running | 等集群 Running |
| `UnsupportedOperation` | 查集群类型 | 控制面日志仅托管集群支持 | 用托管集群 |
| `ResourceInUse` | `DescribeLogSwitches` | Agent 已安装或日志已开启 | 先卸载/关闭再重装 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| Agent 安装但 Pod 日志不投递 | `kubectl get ds -n kube-system` | Agent DaemonSet 未就绪或采集配置错 | 查 Agent Pod 日志，核对 `LogConfig` |
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
# 查询控制面日志（ClusterType 仅 tke）
tccli tke DescribeControlPlaneLogs --ClusterId "<CLUSTER_ID>" --ClusterType tke --region <REGION>
# expected: exit 0, Details[] 含组件日志配置
```

> ⚠️ `DescribeControlPlaneLogs` 必填 `--ClusterType tke`（**不是** `MANAGED_CLUSTER`/`INDEPENDENT_CLUSTER`），缺失报 `the following arguments are required: --ClusterType`（exit 252）。

```bash
# 查询日志采集配置
tccli tke DescribeLogConfigs --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, 日志配置列表

# 修改日志采集配置（参数名是 LogConfig，不是 Name/ConfigData）
tccli tke ModifyLogConfig --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --LogConfig '{"apiVersion":"cls.cloud.tencent.com/v1","kind":"LogConfig","metadata":{"name":"app-log"},"spec":{"clsDetail":{"logType":"minimalist_log"},"inputDetail":{"type":"container_stdout","containerStdout":{"allContainers":true}}}}'
# expected: exit 0；可选 --ClusterType tke/eks（默认 tke）
```

> `DescribeControlPlaneLogs` 的 `--ClusterType` **仅** `tke`；传 `MANAGED_CLUSTER`/`INDEPENDENT_CLUSTER` 会报 `InvalidParameter.Param`（`cluster type not supported:…`）。`DescribeLogConfigs` 在未装 CLS CRD 时可能报 `FailedOperation.KubernetesListOperationError`（`logconfigs.cls.cloud.tencent.com` 不存在）——先 `InstallLogAgent` / 装采集组件后再查。

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- tccli管日志采集配置，kubectl查Agent部署状态，非tccli边界 -->
```bash
# 跨步骤汇总三项合一：Agent 运行 + 业务日志可查 + 托管组件日志可查
# 1. CLS Agent DaemonSet Running（业务日志采集器就绪）
kubectl get ds -n kube-system | grep logagent
# expected: DESIRED=CURRENT=READY 节点数

# 2. 业务日志端到端可查（Verify 查开关，此处查 CLS 真有 Pod 日志）
tccli cls SearchLog --region <REGION> --TopicId "<TOPIC_ID>" --Content '"nginx"'
# expected: 命中含应用输出的日志记录

# 3. 托管组件日志可查（EnableControlPlaneLogs 当前组件为 cluster-autoscaler/kapenter，非 apiserver）
tccli tke DescribeControlPlaneLogs --ClusterId "<CLUSTER_ID>" --ClusterType tke --region <REGION>
# expected: Details[] 含目标组件；再按 TopicId 用 cls SearchLog 检索对应组件日志
```

> Agent Running + 业务日志可查 + 托管组件日志可查 = 端到端闭环。Verify 段查开关状态与维度，此处跨步骤汇总 Agent 部署 + 业务日志投递 + 托管组件日志配置三项，确认采集链路完整可用。

---

## 下一步

- [审计日志](../security/audit.md) — 审计日志（区别于业务/控制面日志）
- [Prometheus 监控入门](prometheus.md) — 指标监控
- [故障排查](../troubleshooting.md) — 日志缺失诊断
