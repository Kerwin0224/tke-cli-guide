---
doc_type: How-to
subtype: 6A
fused: true
---
# 版本保留策略

> 配置自动保留规则，按策略清理旧镜像版本。**该操作可能批量删除镜像**——匹配的 tag 会被永久删除，不可恢复。

## 概述

保留规则按 `CronSetting` 定时执行，按 `RetentionRule` 保留指定数量/时间的镜像，其余删除。

| 规则类型 | 字段 | 含义 | 示例取值 |
|:---------|:----|:-----|:-----------|
| 保留最新 N 个 | `latestPushedK` | 保留最近推送的 N 个版本 | `10` |
| 保留最近 N 天 | `nDays` | 保留 N 天内推送的版本 | `30` |

> 规则作用于命名空间级别（`NamespaceId`）。规则创建后按 `CronSetting` 定时执行，也可 `CreateTagRetentionExecution` 手动触发。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tcr DescribeInstanceStatus --region <REGION> --RegistryIds '["<REGISTRY_ID>"]' \
  --filter "RegistryStatusSet[0].Status"
# expected: "Running"
```

### 资源检查

```bash
# 确认命名空间存在（保留规则绑到命名空间）
tccli tcr DescribeNamespaces --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "NamespaceList[].{name:Name,id:NamespaceId}"
# expected: 含目标命名空间及其 NamespaceId

# 查看已有保留规则
tccli tcr DescribeTagRetentionRules --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "TotalCount"
# expected: 已有规则数
```

## 关键字段

> 来源：`tccli tcr CreateTagRetentionRule --generate-cli-skeleton`（实测）。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| NamespaceId | int | 是 | 命名空间 ID | `ResourceNotFound` |
| CronSetting | string | 是 | Cron 表达式，如 `0 2 * * *`（每天 2 点） | `InvalidParameterValue` |
| RetentionRule | object | 是 | `{Key, Value}` | `InvalidParameterValue` |
| RetentionRule.Key | string | 是 | `latestPushedK` / `nDays` | `InvalidParameterValue` |
| RetentionRule.Value | int | 是 | 保留数量或天数 | `InvalidParameterValue` |
| AdvancedRuleItems | list | 否 | 高级规则（按 tag/仓库过滤） | `InvalidParameterValue` |
| Disabled | boolean | 否 | 是否禁用规则 | — |

> ⚠️ `NamespaceId` 是整数 ID（非命名空间名），从 `DescribeNamespaces` 响应的 `NamespaceId` 字段获取。

## 操作步骤

### 步骤 1：决策 — 保留策略

#### 为什么选 latestPushedK vs nDays

- **latestPushedK（保留 N 个）**: 保留最近 N 个版本，数量固定。适合持续集成（每次推送保留最新 10 个）
- **nDays（保留 N 天）**: 保留 N 天内版本，时间固定。适合按时间回滚的需求
- **默认推荐**: `latestPushedK` + `Value=10`——多数场景保留最新 10 个够用
- **能改吗?**: 能，`ModifyTagRetentionRule` 修改规则

```bash
# 修改保留规则（RegistryId + RetentionId 定位 + 新 CronSetting/RetentionRule）
tccli tcr ModifyTagRetentionRule --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --RetentionId <RETENTION_ID> --NamespaceId <NAMESPACE_ID> \
  --CronSetting "0 3 * * *" \
  --RetentionRule '{"Key":"nDays","Value":30}'
# expected: exit 0
```

> `ModifyTagRetentionRule` 用 `RetentionId`（整数，来自 `DescribeTagRetentionRules`）定位规则，`RetentionRule`/`CronSetting` 覆盖式更新。`NamespaceId` 仍需传（规则绑命名空间）。改后用 `DescribeTagRetentionRules` 确认新值，或 `CreateTagRetentionExecution` 手动触发验证效果。

### 步骤 2：创建 — 最小化

```bash
tccli tcr CreateTagRetentionRule --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceId <NAMESPACE_ID> \
  --CronSetting "0 2 * * *" \
  --RetentionRule '{"Key":"latestPushedK","Value":10}'
# expected: exit 0, 返回 RetentionId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<REGISTRY_ID>` | 实例 ID | `tcr-xxxxxxxx` | `tccli tcr DescribeInstances` |
| `<NAMESPACE_ID>` | 命名空间 ID | 整数 | `tccli tcr DescribeNamespaces` → `NamespaceList[].NamespaceId` |

### 步骤 3：创建 — 增强：按仓库过滤

```bash
# 只对匹配的仓库应用规则（AdvancedRuleItems）
tccli tcr CreateTagRetentionRule --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceId <NAMESPACE_ID> \
  --CronSetting "0 2 * * *" \
  --RetentionRule '{"Key":"latestPushedK","Value":5}' \
  --AdvancedRuleItems '[{"RepositoryFilter":{"Decoration":"matches","Pattern":"prod-*"}}]'
# expected: exit 0
```

> `AdvancedRuleItems` 按 `RepositoryFilter`/`TagFilter` 细化规则作用范围。`Decoration` 为 `matches`（匹配）或 `excludes`（排除）。

### 步骤 4：手动触发执行（测试规则）

```bash
# 手动触发一次执行（验证规则效果，先在测试仓库验证）
tccli tcr CreateTagRetentionExecution --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --RetentionId <RETENTION_ID>
# expected: exit 0, 返回执行 ID
```

### 步骤 5：验证

```bash
# 查看规则
tccli tcr DescribeTagRetentionRules --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "RetentionPolicyList[].{id:RetentionId,cron:CronSetting,disabled:Disabled}"
# expected: 含刚创建的规则

# 查看执行历史
tccli tcr DescribeTagRetentionExecution --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --RetentionId <RETENTION_ID>
# expected: 执行记录，Status 含 Success

# 查看单次执行的任务详情（RegistryId + RetentionId + ExecutionId 定位某次执行）
tccli tcr DescribeTagRetentionExecutionTask --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --RetentionId <RETENTION_ID> --ExecutionId <EXECUTION_ID> \
  --Offset 0 --Limit 20
# expected: exit 0, 返回该次执行删除的镜像版本明细 + 失败原因
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 规则存在 | `DescribeTagRetentionRules` → `RetentionPolicyList` | 含目标规则 |
| 规则启用 | `DescribeTagRetentionRules` → `Disabled` | `false` |
| 执行成功 | `DescribeTagRetentionExecution` → `Status` | `Success` |
| 镜像清理 | `DescribeImages` → `ImageInfoList` | 旧版本被删除，保留指定数量 |

## 清理

> **副作用警告**：删除保留规则会停止自动清理，已删除的镜像不可恢复。规则本身删除不影响镜像。

```bash
# 删除规则
tccli tcr DeleteTagRetentionRule --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --RetentionId <RETENTION_ID>
# expected: exit 0

# 验证已删
tccli tcr DescribeTagRetentionRules --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "RetentionPolicyList[?RetentionId==<RETENTION_ID>]"
# expected: 空数组
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound` | `DescribeNamespaces` 核对 ID | NamespaceId 错或命名空间不存在 | 用正确的 NamespaceId（整数） |
| `InvalidParameterValue.CronSetting` | 检查 Cron 格式 | Cron 表达式错 | 用标准 5 段 Cron，如 `0 2 * * *` |
| `InvalidParameterValue.RetentionRule` | 检查 Key/Value | Key 拼错或 Value 非数字 | Key 用 `latestPushedK`/`nDays`，Value 为正整数 |
| `FailedOperation` | `DescribeInstanceStatus` 看状态 | 实例非 Running | 等实例 Running |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 规则创建但未执行 | `DescribeTagRetentionExecution` | Cron 未到时间 | 手动 `CreateTagRetentionExecution` 触发 |
| 执行删除过多镜像 | `DescribeTagRetentionExecutionTask` 看详情 | Value 太小或规则范围太大 | 调大 Value 或收窄 AdvancedRuleItems |
| 执行 `Failed` | `DescribeTagRetentionExecutionTask` → `Reason` | 命名空间无镜像或权限不足 | 查 Reason，确认有镜像且权限正常 |

> ⚠️ 建议先在测试命名空间验证规则，确认删除效果后再应用到生产命名空间。误删不可恢复。

## Webhook 触发器

> Webhook 触发器在镜像推送/拉取等事件发生时回调指定 URL。属生命周期自动化（事件驱动），与版本保留（定时清理）互补。

```bash
# 查询触发器列表 (按命名空间)
tccli tcr DescribeWebhookTrigger --RegistryId "<REGISTRY_ID>" --Namespace "<NAMESPACE>" --Limit 10 --region <REGION>
# expected: exit 0, Triggers[] + TotalCount (无触发器则空)
```

```bash
# 创建触发器 (RegistryId + Namespace + 嵌套 Trigger 对象含 Name/Targets/事件类型)
tccli tcr CreateWebhookTrigger --RegistryId "<REGISTRY_ID>" --Namespace "<NAMESPACE>" --region <REGION> \
  --Trigger '{"Name":"<TRIGGER_NAME>","TriggerType":"push","Targets":[{"Type":"webhook","Address":"<URL>"}]}'
# expected: exit 0, 返回触发器 Id
```

> `CreateWebhookTrigger` 的 `Trigger` 是嵌套对象（含 `Name`/`TriggerType`/`Targets[]`），`Targets[].Address` 是回调 URL。创建后用 `DescribeWebhookTrigger` 查列表，执行日志查 `DescribeWebhookTriggerLog`。
```json
{"TotalCount": 0, "Triggers": [], "RequestId": "..."}
```

```bash
# 修改触发器 (嵌套 Trigger 对象, 含 Name/Targets/事件类型)
tccli tcr ModifyWebhookTrigger --RegistryId "<REGISTRY_ID>" --region <REGION> \
  --Trigger '{"Name":"<TRIGGER_NAME>","Targets":[{"Address":"<URL>"}]}'
# expected: exit 0

# 删除触发器 (按 Namespace + Id)
tccli tcr DeleteWebhookTrigger --RegistryId "<REGISTRY_ID>" --Namespace "<NAMESPACE>" --Id <TRIGGER_ID> --region <REGION>
# expected: exit 0
```

> `DeleteWebhookTrigger` 用 `Id`（Integer，触发器 ID）+ `Namespace` 定位，非触发器名。`ModifyWebhookTrigger` 的 `Trigger` 是嵌套对象（Name/Targets[]/事件类型）。触发器执行日志查 `DescribeWebhookTriggerLog`。

```bash
# 查询触发器执行日志（RegistryId + Namespace + Id 定位触发器，分页取日志）
tccli tcr DescribeWebhookTriggerLog --RegistryId "<REGISTRY_ID>" --Namespace "<NAMESPACE>" \
  --Id <TRIGGER_ID> --Offset 0 --Limit 20 --region <REGION>
# expected: exit 0, 返回触发器每次回调的日志（成功/失败 + 状态码）
```

## GC 垃圾回收任务

> GC（垃圾回收）清理镜像层占用的存储。删除镜像后，GC 释放底层存储。属生命周期存储管理。

```bash
# 查询 GC 任务列表
tccli tcr DescribeGCJobs --RegistryId "<REGISTRY_ID>" --region <REGION>
# expected: exit 0, Jobs[] (无任务则空)
```
```json
{"Jobs": [], "RequestId": "..."}
```

```bash
# 终止进行中的 GC 任务
tccli tcr TerminateGCJob --RegistryId "<REGISTRY_ID>" --region <REGION>
# expected: exit 0
```

> GC 任务由系统在删除镜像后触发或手动发起。`DescribeGCJobs` 查任务状态（Running/Success/Failed），`TerminateGCJob` 终止进行中的 GC（仅 RegistryId）。

## 下一步

- [不可变标签](immutable-tags.md) — 禁止覆盖（与保留互补）
- [管理命名空间和仓库](../repositories/manage.md) — 创建命名空间
- [推送拉取镜像](../images/push-pull.md) — 手动删除单个版本
- [故障排查](../troubleshooting.md) — 规则不生效诊断

## 控制台替代方案

[容器镜像服务控制台 - 版本保留](https://console.cloud.tencent.com/tcr/retention)

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `CreateTagRetentionRule` | 主操作 | TCR | 创建版本保留规则（latestPushedK/nDays） |
| `ModifyTagRetentionRule` | 主操作 | TCR | 修改保留规则 |
| `CreateTagRetentionExecution` | 主操作 | TCR | 手动触发保留规则执行 |
| `CreateWebhookTrigger` | 主操作 | TCR | 创建 Webhook 触发器 |
| `ModifyWebhookTrigger` | 主操作 | TCR | 修改触发器（嵌套 Trigger 对象） |
| `DescribeTagRetentionRules` | 验证 | TCR | 查询保留规则列表 |
| `DescribeTagRetentionExecution` | 验证 | TCR | 查询保留执行历史 |
| `DescribeTagRetentionExecutionTask` | 验证 | TCR | 查询执行任务详情 |
| `DescribeGCJobs` | 验证 | TCR | 查询 GC 垃圾回收任务 |
| `DescribeWebhookTrigger` | 验证 | TCR | 查询 Webhook 触发器列表 |
| `DescribeWebhookTriggerLog` | 验证 | TCR | 查询触发器执行日志 |
| `DescribeImages` | 验证 | TCR | 查询镜像版本（TCR 镜像查询，非 TKE） |
| `DescribeNamespaces` | 验证 | TCR | 查询命名空间（含 NamespaceId） |
| `DescribeInstances` | 验证 | TCR | 查询实例 |
| `DescribeInstanceStatus` | 验证 | TCR | 查询实例状态 |
| `DeleteTagRetentionRule` | 清理 | TCR | 删除保留规则 |
| `TerminateGCJob` | 清理 | TCR | 终止进行中的 GC 任务 |
| `DeleteWebhookTrigger` | 清理 | TCR | 删除触发器（按 Id+Namespace） |
