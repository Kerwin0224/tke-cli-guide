---
doc_type: How-to
subtype: 6A
fused: true
---
# 配置 Prometheus 告警

> 创建和管理 TKE Prometheus 告警策略与告警规则，配置全局通知渠道，查询告警历史。
> 控制台: [容器服务 - 监控](https://console.cloud.tencent.com/tke2/monitor)
>
> 官方文档：[可观测体系概述](https://cloud.tencent.com/document/product/457/118975)
>
> 配额：告警规则数与通知渠道数以产品实际限制为准，TKE 侧无额外配额限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)

> ⚠️ 本文档所有 Action 属 **TKE 2018-05-25（默认版本）**。
> AlertPolicy / AlertRule / GlobalNotification 三层操作同处一篇。

## 触发条件

- `DescribePrometheusAlertPolicy` 返回空，需为 Prometheus 实例创建告警策略监控指标越阈
- 告警触发但通知未送达，`DescribePrometheusGlobalNotification` → `Enabled=false` 或接收组为空
- `DescribePrometheusAlertHistory` 无记录，规则 PromQL 永假或 Agent 采集未生效 — 看 [故障恢复](#故障恢复)


## 概述

TKE Prometheus 告警是三层模型：

| 层 | Action 前缀 | 作用 | 寻址键 |
|:---|:-----------|:-----|:-------|
| 告警策略 AlertPolicy | `*PrometheusAlertPolicy` | 告警规则的管理单元（一个 policy 含多条 rule） | `AlertIds[]` 或 `Names[]` |
| 告警规则 AlertRule | `*PrometheusAlertRule` | 单条告警规则（PromQL + 阈值 + 持续时间） | `AlertIds[]` |
| 全局通知 GlobalNotification | `*PrometheusGlobalNotification` | 告警触发后的通知渠道（WebHook/AlertManager/电话/接收组） | `InstanceId` |

**AlertPolicy ≠ AlertRule**：两套独立 Action，入参结构几乎相同（都含 `AlertRule.{Name, Rules[], Id, TemplateId}`），但删除寻址不同——Policy 支持 `AlertIds[]` 或 `Names[]` 双寻址，Rule 仅 `AlertIds[]`。命名易混，调用前用 `--generate-cli-skeleton` 确认。

通知层（GlobalNotification）是实例级单例配置：一个 Prometheus 实例配一条全局通知，所有告警规则触发后都走这条通知渠道。

## 准备工作

### 环境检查

```bash
tccli tke DescribePrometheusInstancesOverview --region <REGION>
# expected: exit 0, TotalCount >= 1, 含可用 InstanceId
```

> ⚠️ 此步返回 `UnauthorizedOperation`（Prometheus 域 CAM 拦截）。授权前无法继续。

### 资源检查

告警规则依赖已采集的指标。确认 Agent 已安装、采集目标 up（见 [Prometheus Agent 管理](prometheus-agent.md)），否则规则因无数据永不触发。

## 关键字段

### 告警策略/规则 AlertPolicy & AlertRule

| 字段 | 类型 | 必填 | 约束 | 填错的影响 |
|:-----|:-----|:----:|:-----|:-----------|
| `InstanceId` | String | 是 | Prometheus 实例 | — |
| `AlertRule.Name` | String | 是 | 策略/规则名 | — |
| `AlertRule.Rules[].Name` | String | 是 | 单条规则名 | — |
| `AlertRule.Rules[].Rule` | String | 是 | PromQL 表达式 | 规则不触发或报错 |
| `AlertRule.Rules[].For` | String | 否 | 持续时间，如 `5m` | 阈值抖动误报 |
| `AlertRule.Rules[].Labels[]` | Array | 否 | 标签{Name,Value} | 路由失效 |
| `AlertRule.Rules[].Annotations[]` | Array | 否 | 注解{Name,Value} | 告警信息不全 |
| `AlertRule.Rules[].RuleState` | Integer | 否 | 规则状态 | — |
| `AlertIds[]` | Array | 是（Delete） | 删除用 ID 列表 | 删错目标 |
| `Names[]` | Array | 是（DeletePolicy 可选） | 删除用名列表（仅 Policy） | — |

### 全局通知 GlobalNotification

| 字段 | 类型 | 必填 | 约束 | 填错的影响 |
|:-----|:-----|:----:|:-----|:-----------|
| `Notification.Enabled` | Boolean | 是 | 是否启用通知 | 通知不送达 |
| `Notification.Type` | String | 是 | 通知类型 | — |
| `Notification.WebHook` | String | 否 | WebHook URL | WebHook 通知失败 |
| `Notification.AlertManager` | Object | 否 | {Url,ClusterType,ClusterId} | AM 转发失败 |
| `Notification.NotifyWay[]` | Array | 否 | 通知方式列表 | 渠道选错 |
| `Notification.ReceiverGroups[]` | Array | 否 | 接收组 ID 列表 | 收件人错 |
| `Notification.RepeatInterval` | String | 否 | 重复间隔，如 `5m` | 重复打扰或漏报 |
| `Notification.TimeRangeStart/End` | String | 否 | 通知时段 | 非时段告警静默 |
| `Notification.Phone*` | 多个 | 否 | 电话通知参数（CircleTimes/InnerInterval 等） | 电话策略不符 |

## 操作步骤

### 步骤 1：创建告警策略

```bash
tccli tke CreatePrometheusAlertPolicy --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --AlertRule '{"Name":"<POLICY_NAME>","Rules":[{"Name":"<RULE_NAME>","Rule":"up == 0","For":"5m"}]}'
# expected: exit 0, 返回 RequestId（含 AlertId）
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<PROM_INSTANCE_ID>` | Prometheus 实例 | 已存在 | `DescribePrometheusInstancesOverview` |
| `<POLICY_NAME>` | 策略名 | 实例内唯一 | 自定义 |
| `<RULE_NAME>` | 规则名 | 策略内唯一 | 自定义 |

> `AlertRule` 是单数入参名，但其内 `Rules[]` 是复数数组——一个策略可含多条规则。AlertRule（`CreatePrometheusAlertRule`）入参结构相同，是另一套独立 Action。

### 步骤 2：配置全局通知

```bash
tccli tke CreatePrometheusGlobalNotification --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --Notification '{"Enabled":true,"Type":"amp","RepeatInterval":"5m","NotifyWay":["SMS","EMAIL"],"ReceiverGroups":["<RG_ID>"]}'
# expected: exit 0, 返回 RequestId
```

通知是实例级单例：一个实例配一条全局通知，所有告警规则触发后都走它。`Type: "amp"` 表示走 TKE AMP（Alert Manager Provider），`WebHook`/`AlertManager` 是另两种类型。电话通知需配 `PhoneCircleTimes`/`PhoneInnerInterval` 等 Phone* 参数。

### 步骤 3：修改告警策略

```bash
tccli tke ModifyPrometheusAlertPolicy --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --AlertRule '{"Name":"<POLICY_NAME>","Id":"<ALERT_ID>","Rules":[{"Name":"<RULE_NAME>","Rule":"up == 0","For":"10m"}]}'
# expected: exit 0, 返回 RequestId
```

Modify 为覆盖式：`AlertRule.Id` 指定改哪条，`Rules[]` 整体替换。

### 步骤 3b：AlertRule 独立 Action 流程（与 AlertPolicy 平行）

> AlertPolicy 与 AlertRule 是两套独立 Action，入参结构几乎相同（都含 `AlertRule.{Name,Rules[],Id,TemplateId}`），但删除寻址不同。AlertRule 用 `InstanceId`+`AlertRule`+分页/`Filters` 查询。
>
> `Create` 返回 `Id`、`Describe` 返回 `AlertRules[]`+`Total`。

```bash
# 1. 创建告警规则（独立 Action，AlertRule 单数入参含 Rules[]）
tccli tke CreatePrometheusAlertRule --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --AlertRule '{"Name":"<RULE_NAME>","Rules":[{"Name":"<RULE_NAME>","Rule":"up == 0","For":"5m"}]}'
# expected: CAM 拦截 AuthFailure.UnauthorizedOperation；授权后返回 Id

# 2. 查询告警规则（InstanceId + Filters/Offset/Limit）
tccli tke DescribePrometheusAlertRule --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --Filters '[{"Name":"Name","Values":["<RULE_NAME>"]}]' --Offset 0 --Limit 20
# expected: CAM 拦截 UnauthorizedOperation（云API网关层）；授权后返回 AlertRules[]+Total

# 3. 修改告警规则（AlertRule.Id 寻址，覆盖式）
tccli tke ModifyPrometheusAlertRule --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --AlertRule '{"Name":"<RULE_NAME>","Id":"<ALERT_ID>","Rules":[{"Name":"<RULE_NAME>","Rule":"up == 0","For":"10m"}]}'
# expected: CAM 拦截 AuthFailure.UnauthorizedOperation；授权后 exit 0
```

> AlertPolicy vs AlertRule 命令一一对应：`Create/Describe/Modify/Delete` × `PrometheusAlertPolicy|PrometheusAlertRule`。删除寻址不同——Policy 支持 `Names[]`/`AlertIds[]` 双寻址，Rule 仅 `AlertIds[]`（删规则须先从 `DescribePrometheusAlertRule` 取 AlertId）。混淆会报 `Unknown options`。

### 步骤 3c：修改全局通知

```bash
# 修改全局通知（实例级单例，Notification 整体覆盖）
tccli tke ModifyPrometheusGlobalNotification --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --Notification '{"Enabled":true,"Type":"amp","RepeatInterval":"10m","NotifyWay":["SMS","EMAIL"],"ReceiverGroups":["<RG_ID>"]}'
# expected: CAM 拦截 AuthFailure.UnauthorizedOperation；授权后 exit 0
```

> `ModifyPrometheusGlobalNotification` 是实例级单例的覆盖式更新。一个实例只有一条全局通知，所有告警规则触发后都走它。修改前可先 `DescribePrometheusGlobalNotification` 取当前配置，避免遗漏已有渠道。

## 验证

```bash
tccli tke DescribePrometheusAlertPolicy --region <REGION> --InstanceId <PROM_INSTANCE_ID>
# expected: exit 0, 返回告警策略列表，含目标 Name
```

```bash
tccli tke DescribePrometheusAlertHistory --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --RuleName <RULE_NAME> \
  --StartTime <START_TIME> --EndTime <END_TIME>
# expected: exit 0, 返回告警触发历史记录
```

```bash
tccli tke DescribePrometheusGlobalNotification --region <REGION> --InstanceId <PROM_INSTANCE_ID>
# expected: exit 0, 返回全局通知配置
```

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 策略存在 | `DescribePrometheusAlertPolicy --InstanceId <PROM_INSTANCE_ID>` | 含目标 Name |
| 规则存在 | `DescribePrometheusAlertRule --InstanceId <PROM_INSTANCE_ID>` | 含目标规则 |
| 通知配置 | `DescribePrometheusGlobalNotification --InstanceId <PROM_INSTANCE_ID>` | Enabled=true，渠道正确 |
| 告警已触发 | `DescribePrometheusAlertHistory --InstanceId <PROM_INSTANCE_ID> --RuleName <RULE_NAME>` | 含触发记录 |
| 通知已送达 | 查对应渠道（SMS/EMAIL/接收组）日志 | 收到通知 |

> `DescribePrometheusAlertHistory` 的 `StartTime`/`EndTime` 为 ISO8601 格式。

## 清理

> **副作用警告**：删除告警策略/规则后，对应指标即使越阈也不再告警，可能漏报生产故障。删除全局通知则所有告警静默。建议先确认无生产规则依赖再删。
>
> ⚠️ **高危操作**：告警策略 PromQL 误配（如永假表达式）会导致漏报；通知渠道配置错误或全局通知关闭会导致告警无法送达。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

```bash
# 删除告警策略（AlertIds 或 Names 二选一）
tccli tke DeletePrometheusAlertPolicy --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --Names '["<POLICY_NAME>"]'
# expected: exit 0

# 删除告警规则（仅 AlertIds）
tccli tke DeletePrometheusAlertRule --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --AlertIds '["<ALERT_ID>"]'
# expected: exit 0
```

> 注意 `DeletePrometheusAlertPolicy` 支持 `Names[]` 或 `AlertIds[]`，而 `DeletePrometheusAlertRule` 仅支持 `AlertIds[]`——删除规则必须先有 AlertId（从 Describe 取）。

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| `AuthFailure.UnauthorizedOperation` (tke:CreatePrometheusAlertPolicy 等) | 查账号 CAM | 写告警配置需 `tke:ActionName` 权限 | 申请写权限。此为环境限制 |
| `UnauthorizedOperation` 您未授权访问该接口 (DescribePrometheus*) | 同上 | 读操作在云 API 网关层被拦 | 申请 `tke:DescribePrometheus*` 读权限 |
| `Unknown options` | `tccli tke <Action> --generate-cli-skeleton` | 混用 AlertPolicy/AlertRule（删除寻址不同） | Policy 用 Names/AlertIds，Rule 仅 AlertIds |
| WebHook 通知失败 | `DescribePrometheusGlobalNotification` 查 WebHook | URL 不通或返回非 2xx | 校验 WebHook 端点 |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| 告警未触发 | `DescribePrometheusAlertHistory --RuleName <RULE_NAME>` | PromQL 永假、`For` 过长、或指标未采集 | 校验 Rule 表达式，确认 [Agent 采集](prometheus-agent.md) 正常 |
| 告警触发但通知未送达 | `DescribePrometheusGlobalNotification` | `Enabled=false`、时段外、接收组为空 | 启用通知，检查 TimeRange 与 ReceiverGroups |
| 规则冲突 | `DescribePrometheusAlertPolicy` | 同名策略/规则已存在 | 先删后建，或 Modify 覆盖 |
| 通知重复轰炸 | 查 `RepeatInterval` | 间隔过短 | 调大 `RepeatInterval` |

## 收尾确认

```bash
# 一次性核对：告警规则已创建（出参 AlertRules[]+Total，结构以实际响应为准）
tccli tke DescribePrometheusAlertRule --region <REGION> --InstanceId <PROM_INSTANCE_ID>   --filter "{total:Total,rules:AlertRules[].{name:Name}}"
# expected: total>=1 → 告警配置闭环完成
```

---

## 下一步

- [Prometheus 入门](prometheus.md) — 创建实例（告警的前置）
- [Prometheus 配置与模板](prometheus-config.md) — 告警模板（Template）的配置侧
- [故障排查](../troubleshooting.md) — 通用排障入口
