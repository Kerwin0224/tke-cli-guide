---
doc_type: How-to
subtype: 6A
fused: true
---
# 管理 Prometheus 配置与模板

> 管理 TKE Prometheus 的采集配置、记录规则、聚合模板、告警模板与 Grafana Dashboard，覆盖配置生命周期与模板跨实例同步。
> 控制台: [容器服务 - 监控](https://console.cloud.tencent.com/tke2/monitor)

> ⚠️ 本文档所有 Action 属 **TKE 2018-05-25（默认版本）**。入参骨架、错误码来自真机实跑（P2）。
> ⚠️ **Prometheus 域 CAM 全拦截（实测）**：当前账号对 Prometheus 全部 Action 返回 `UnauthorizedOperation`，无可用实例。出参结构标 **blocked**，授权后回填。错误码与诊断已入 [§故障恢复](#故障恢复)。
> ⚠️ 本文档为 **Fused Runbook**：合并 Config / RecordRule / Temp / Template / Dashboard 五类操作于一篇，单会话读者无需切页。

## 概述

TKE Prometheus 配置分五个正交维度，各有独立 CRUD：

| 维度 | Action 前缀 | 作用范围 | 寻址键 |
|:-----|:-----------|:---------|:-------|
| 采集配置 Config | `*PrometheusConfig` | 单实例+单集群 | `InstanceId`+`ClusterId` |
| 记录规则 RecordRule | `*PrometheusRecordRuleYaml` | 单实例 | `InstanceId`+`Name` |
| 聚合模板 Temp | `*PrometheusTemp` | 全局（跨实例复用采集/记录规则） | `TemplateId` |
| 告警模板 Template | `*PrometheusTemplate` | 全局（跨实例复用告警规则） | `TemplateId` |
| Dashboard | `*PrometheusDashboard` | 单实例 | `InstanceId`+`DashboardName` |

**Temp ≠ Template**：Temp 装采集与记录规则模板（ServiceMonitors/PodMonitors/RawJobs/RecordRules），Template 装告警规则模板（AlertRules）。两套各自 `Sync`，结构一致但内容不同。模板（Temp/Template）是全局资源，不绑实例，`Describe` 不需 `InstanceId`；Config/RecordRule/Dashboard 绑实例，需 `InstanceId`。

## 准备工作

### 环境检查

```bash
tccli tke DescribePrometheusInstancesOverview --region <REGION>
# expected: exit 0, TotalCount >= 1, 含可用 InstanceId
```

> ⚠️ 本环境此步返回 `UnauthorizedOperation`（Prometheus 域 CAM 拦截）。授权前无法继续实例级操作；模板级操作（Temp/Template）不依赖实例，可先行。

### 资源检查

```bash
tccli tke DescribeClusters --region <REGION> --filter "Clusters[*].ClusterId" --output text
# expected: exit 0, 含目标 ClusterId
```

## 关键字段

> 参数名实测自各 Action 的 `--generate-cli-skeleton`（P7）。注意多组同名入参契约不同：Create 的采集器是对象数组（含 Name/Config），Delete 的同名入参是字符串数组（仅 Name）。

### 采集配置 Config

| 字段 | 类型 | 必填 | 约束 | 填错的影响 |
|:-----|:-----|:----:|:-----|:-----------|
| `InstanceId` | String | 是 | Prometheus 实例 | 实例不存在被拒 |
| `ClusterId` | String | 是 | 已存在集群 | 集群不存在被拒 |
| `ClusterType` | String | 是 | `MANAGED_CLUSTER`/`INDEPENDENT_CLUSTER` | 类型错被拒 |
| `ServiceMonitors[]` | Array | 否 | Create=对象{Name,Config,TemplateId}；Delete=字符串(Name) | Create/Delete 混用会参数错 |
| `PodMonitors[]` | Array | 否 | 同上 | 同上 |
| `RawJobs[]` | Array | 否 | 同上 | 同上 |
| `Probes[]` | Array | 否 | 同上 | 同上 |

### 记录规则 RecordRule

| 字段 | 类型 | 必填 | 约束 | 填错的影响 |
|:-----|:-----|:----:|:-----|:-----------|
| `InstanceId` | String | 是 | 实例 | — |
| `Content` | String | 是（Create/Modify） | YAML 字符串，规则名在内 | 格式错被拒 |
| `Name` | String | 是（Modify/Delete） | 规则名 | 改/删错目标 |

### 模板 Temp / Template

| 字段 | 类型 | 必填 | 约束 | 填错的影响 |
|:-----|:-----|:----:|:-----|:-----------|
| `Template.Name` | String | 是 | 模板名 | — |
| `Template.Level` | String | 是（Create，Modify 无） | 模板层级 | Create 缺被拒 |
| `Template.RecordRules[]` | Object | 否（Temp） | Temp 专用，采集/记录规则 | — |
| `Template.AlertRules[]` | Object | 否（Template） | Template 专用，含 Rule/For/Labels/Annotations | — |
| `TemplateId` | String | 是（Modify/Delete/Sync） | 模板 ID | 操作错目标 |
| `Targets[]` | Array | 是（Sync） | 同步目标 Region/InstanceId/ClusterId | 同步错实例 |

## 操作步骤

### 步骤 1：创建采集配置 — 最小化

```bash
tccli tke CreatePrometheusConfig --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --ClusterType MANAGED_CLUSTER --ClusterId <CLUSTER_ID> \
  --ServiceMonitors '[{"Name":"<SM_NAME>","Config":"<SM_YAML>"}]'
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<PROM_INSTANCE_ID>` | Prometheus 实例 | 已存在 | `DescribePrometheusInstancesOverview` |
| `<CLUSTER_ID>` | 集群 | 已存在 | `tccli tke DescribeClusters --region <REGION>` |
| `<SM_NAME>` | ServiceMonitor 名 | 集群内唯一 | 自定义 |
| `<SM_YAML>` | ServiceMonitor 配置 | 合法 YAML 字符串 | 自定义 |

### 步骤 2：创建记录规则 — 增强

```bash
tccli tke CreatePrometheusRecordRuleYaml --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --Content 'record: job:node_cpu:rate5m\nexpr: sum by (job) (rate(node_cpu[5m]))'
# expected: exit 0, 返回 RequestId
```

`Content` 是含 `record:`/`expr:` 的 YAML 字符串，规则名在 `record:` 行内（非独立 Name 参数）。Modify 时需额外传 `Name` 指定改哪条。

### 步骤 3：创建模板并同步到实例 — 增强（Temp）

```bash
# 创建聚合模板（全局，不绑实例）
tccli tke CreatePrometheusTemp --region <REGION> \
  --Template '{"Name":"<TEMP_NAME>","Level":"cluster","RecordRules":[{"Name":"rr1","Config":"<YAML>"}]}'
# expected: exit 0, 返回 TemplateId

# 同步模板到目标实例
tccli tke SyncPrometheusTemp --region <REGION> \
  --TemplateId <TEMPLATE_ID> \
  --Targets '[{"Region":"<REGION>","InstanceId":"<PROM_INSTANCE_ID>","ClusterId":"<CLUSTER_ID>","ClusterType":"MANAGED_CLUSTER"}]'
# expected: exit 0, 返回 RequestId
```

模板是复用配置的载体：先建模板，再 `Sync` 推送到多个实例，避免逐实例重复配。Template（告警模板）流程相同，把 `*PrometheusTemp` 换成 `*PrometheusTemplate`、`RecordRules` 换成 `AlertRules`。

### 步骤 3b：告警模板 Template 全流程（与 Temp 平行）

> Temp 装采集/记录规则，Template 装告警规则（AlertRules）。两套各自 CRUD+Sync，命令结构对称。参数以 `--generate-cli-skeleton` 实测为准（Template 用 `TemplateId` 寻址，`Describe` 用 `Filters`+分页）。

```bash
# 1. 创建告警模板（全局，不绑实例，Template 含 Name/Level/AlertRules）
tccli tke CreatePrometheusTemplate --region <REGION> \
  --Template '{"Name":"<TPL_NAME>","Level":"cluster","AlertRules":[{"Name":"<ALERT_NAME>","Rule":"up == 0","For":"5m"}]}'
# expected: exit 0, 返回 TemplateId

# 2. 查询告警模板（Filters + Offset/Limit，不需 InstanceId）
tccli tke DescribePrometheusTemplates --region <REGION> \
  --Filters '[{"Name":"Name","Values":["<TPL_NAME>"]}]' --Offset 0 --Limit 20
# expected: exit 0, 返回告警模板列表

# 3. 修改告警模板（TemplateId 寻址，Template 整体覆盖）
tccli tke ModifyPrometheusTemplate --region <REGION> \
  --TemplateId <TEMPLATE_ID> \
  --Template '{"Name":"<TPL_NAME>","AlertRules":[{"Name":"<ALERT_NAME>","Rule":"up == 0","For":"10m"}]}'
# expected: exit 0

# 4. 同步告警模板到目标实例（Targets 结构同 SyncPrometheusTemp）
tccli tke SyncPrometheusTemplate --region <REGION> \
  --TemplateId <TEMPLATE_ID> \
  --Targets '[{"Region":"<REGION>","InstanceId":"<PROM_INSTANCE_ID>","ClusterId":"<CLUSTER_ID>","ClusterType":"MANAGED_CLUSTER"}]'
# expected: exit 0

# 5. 删除告警模板
tccli tke DeletePrometheusTemplate --region <REGION> --TemplateId <TEMPLATE_ID>
# expected: exit 0
```

> Temp 与 Template 命令一一对应：`Create/Describe/Modify/Sync/Delete` × `PrometheusTemp|PrometheusTemplate`。区别仅内容字段——Temp 用 `RecordRules`，Template 用 `AlertRules`；`Describe` 也不同（`DescribePrometheusTemp` 单数 vs `DescribePrometheusTemplates` 复数 + Filters）。

### 步骤 4：创建 Grafana Dashboard — 增强

```bash
tccli tke CreatePrometheusDashboard --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --DashboardName <DASH_NAME> \
  --Contents '["<DASHBOARD_JSON>"]'
# expected: exit 0, 返回 RequestId
```

`Contents` 是 JSON 字符串数组，每项是一个 Grafana Dashboard 的 JSON 定义。

## 验证

```bash
tccli tke DescribePrometheusConfig --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --ClusterId <CLUSTER_ID> --ClusterType MANAGED_CLUSTER
# expected: exit 0, 返回 ServiceMonitors/PodMonitors 等当前配置
```

```bash
tccli tke DescribePrometheusRecordRules --region <REGION> --InstanceId <PROM_INSTANCE_ID>
# expected: exit 0, 返回记录规则列表
```

```bash
tccli tke DescribePrometheusTemp --region <REGION> --Limit 20
# expected: exit 0, 返回模板列表（不需 InstanceId，模板是全局的）
```

```bash
# 查询实例全局配置（InstanceId 定位，DisableStatistics 控制是否禁用统计）
tccli tke DescribePrometheusGlobalConfig --region <REGION> --InstanceId <PROM_INSTANCE_ID>
# expected: exit 0, 返回实例全局配置

# 修改采集配置（覆盖式，ServiceMonitors 等为对象数组{Name,Config,TemplateId}，与 Create 同结构）
tccli tke ModifyPrometheusConfig --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --ClusterType MANAGED_CLUSTER --ClusterId <CLUSTER_ID> \
  --ServiceMonitors '[{"Name":"<SM_NAME>","Config":"<SM_YAML>"}]'
# expected: exit 0

# 修改聚合模板（TemplateId 寻址，Template 整体覆盖，无 Level 字段）
tccli tke ModifyPrometheusTemp --region <REGION> \
  --TemplateId <TEMPLATE_ID> \
  --Template '{"Name":"<TEMP_NAME>","RecordRules":[{"Name":"rr1","Config":"<YAML>"}]}'
# expected: exit 0

# 修改记录规则（Name 指定改哪条 + Content 新 YAML）
tccli tke ModifyPrometheusRecordRuleYaml --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --Name <RULE_NAME> \
  --Content 'record: job:node_cpu:rate5m\nexpr: sum by (job) (rate(node_cpu[5m]))'
# expected: exit 0
```

> `ModifyPrometheusConfig` 是覆盖式（同 `Create` 的对象数组结构），调用前先 `DescribePrometheusConfig` 取当前值再改，避免误删已有采集器。`ModifyPrometheusTemp` 用 `TemplateId` 寻址且无 `Level` 字段（`Level` 仅 Create 必填）。`ModifyPrometheusRecordRuleYaml` 需 `Name`+`Content` 双参定位修改目标。

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 采集配置生效 | `DescribePrometheusConfig --InstanceId <PROM_INSTANCE_ID> --ClusterId <CLUSTER_ID>` | 含创建的 ServiceMonitor |
| 记录规则存在 | `DescribePrometheusRecordRules --InstanceId <PROM_INSTANCE_ID>` | 含创建的规则 |
| 模板存在 | `DescribePrometheusTemp` | 含创建的 TemplateId |
| 模板同步状态 | `DescribePrometheusTempSync` / `DescribePrometheusTemplateSync` | 目标实例同步成功 |
| 全局配置 | `DescribePrometheusGlobalConfig --InstanceId <PROM_INSTANCE_ID>` | 返回实例全局配置 |

> ⚠️ **出参结构 blocked（实测 CAM 拦截）**：上述命令本环境均返回 `UnauthorizedOperation`，输出结构未经真机验证，授权后须用真实响应回填。

## 清理

> **副作用警告**：删除采集配置会停止对应目标的指标采集，历史数据保留但不再增长；删除模板不影响已同步到实例的配置（同步是复制非引用）。

```bash
# 删除采集配置（传 Name 字符串数组）
tccli tke DeletePrometheusConfig --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --ClusterType MANAGED_CLUSTER --ClusterId <CLUSTER_ID> \
  --ServiceMonitors '["<SM_NAME>"]'
# expected: exit 0

# 删除记录规则
tccli tke DeletePrometheusRecordRuleYaml --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --Names '["<RULE_NAME>"]'
# expected: exit 0

# 删除模板
tccli tke DeletePrometheusTemp --region <REGION> --TemplateId <TEMPLATE_ID>
# expected: exit 0
```

> 注意 `DeletePrometheusConfig` 的 `ServiceMonitors` 是**字符串数组**（Name 列表），与 `CreatePrometheusConfig` 的对象数组不同——混用会参数错误。

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| `AuthFailure.UnauthorizedOperation` (tke:CreatePrometheusConfig 等) | 查账号 CAM | 写 Prometheus 配置需对应 `tke:ActionName` 权限 | 申请写权限。此为环境限制 |
| `UnauthorizedOperation` 您未授权访问该接口 (DescribePrometheus*) | 同上 | 读操作在云 API 网关层被拦 | 申请 `tke:DescribePrometheus*` 读权限 |
| `Unknown options` | `tccli tke <Action> --generate-cli-skeleton` | Create/Delete 同名入参类型不同（对象 vs 字符串数组）混用 | 按 [§关键字段](#关键字字段) 区分对象/字符串数组 |
| 模板同步失败 | `DescribePrometheusTempSync` | `Targets[].InstanceId`/`ClusterId` 不存在或地域不符 | 核对目标实例与集群 |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| 采集配置已建但无数据 | `DescribePrometheusTargets`（见 [Agent 管理](prometheus-agent.md)） | ServiceMonitor YAML 中 selector/端口不符 | 修正 Config YAML |
| 记录规则不生效 | `DescribePrometheusRecordRules` | `Content` YAML 格式错或 expr 语法错 | 校验 YAML 与 PromQL |
| 模板同步后实例配置未变 | `DescribePrometheusConfig` | 同步是覆盖式但目标 Version 落后 | 重新 `SyncPrometheusTemp` 指定新 Version |
| Dashboard 创建后空白 | — | `Contents` JSON 非合法 Grafana 定义 | 校验 Dashboard JSON |

## 下一步

- [Prometheus 入门](prometheus.md) — 创建实例（配置的前置）
- [Prometheus Agent 管理](prometheus-agent.md) — 采集目标状态排查
- [Prometheus 告警配置](prometheus-alerting.md) — 告警模板（Template）的告警侧

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `CreatePrometheusConfig` | 主操作 | 2018-05-25 | 创建采集配置（4 类采集器，对象数组） |
| `DescribePrometheusConfig` | 验证 | 2018-05-25 | 查询集群采集配置 |
| `ModifyPrometheusConfig` | 主操作 | 2018-05-25 | 修改采集配置（覆盖式） |
| `DeletePrometheusConfig` | 清理 | 2018-05-25 | 删除采集配置（字符串数组） |
| `DescribePrometheusGlobalConfig` | 验证 | 2018-05-25 | 查询实例全局配置 |
| `CreatePrometheusRecordRuleYaml` | 主操作 | 2018-05-25 | 创建记录规则（Content YAML） |
| `DescribePrometheusRecordRules` | 验证 | 2018-05-25 | 查询记录规则 |
| `ModifyPrometheusRecordRuleYaml` | 主操作 | 2018-05-25 | 修改记录规则（Name+Content） |
| `DeletePrometheusRecordRuleYaml` | 清理 | 2018-05-25 | 删除记录规则（Names 数组） |
| `CreatePrometheusTemp` | 主操作 | 2018-05-25 | 创建聚合模板（采集+记录规则） |
| `DescribePrometheusTemp` | 验证 | 2018-05-25 | 查询模板（全局，不需 InstanceId） |
| `ModifyPrometheusTemp` | 主操作 | 2018-05-25 | 修改模板（TemplateId，无 Level） |
| `DeletePrometheusTemp` | 清理 | 2018-05-25 | 删除模板 |
| `SyncPrometheusTemp` | 主操作 | 2018-05-25 | 同步模板到实例（Targets） |
| `CreatePrometheusTemplate` | 主操作 | 2018-05-25 | 创建告警模板（AlertRules） |
| `DescribePrometheusTemplates` | 验证 | 2018-05-25 | 查询告警模板 |
| `ModifyPrometheusTemplate` | 主操作 | 2018-05-25 | 修改告警模板 |
| `DeletePrometheusTemplate` | 清理 | 2018-05-25 | 删除告警模板 |
| `SyncPrometheusTemplate` | 主操作 | 2018-05-25 | 同步告警模板到实例 |
| `CreatePrometheusDashboard` | 主操作 | 2018-05-25 | 创建 Grafana Dashboard（Contents JSON） |
