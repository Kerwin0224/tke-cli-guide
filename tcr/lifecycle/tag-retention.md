---
doc_type: How-to
subtype: 6A
fused: true
---
# 版本保留策略

> 配置自动保留规则，按策略清理旧镜像版本。**该操作可能批量删除镜像**——匹配的 tag 会被永久删除，不可恢复。
> 控制台: [版本保留](https://console.cloud.tencent.com/tcr/tagretention)（底层层数据回收见 [制品清理](https://console.cloud.tencent.com/tcr/gc)）
> 官方文档: [自动删除镜像版本](https://cloud.tencent.com/document/product/1141/50613) · [镜像版本不可变](https://cloud.tencent.com/document/product/1141/58200)

## 触发条件

- `tccli tcr DescribeImages --RegistryId "<ID>"` 返回的 `ImageInfoList` 版本数持续增长，旧镜像堆积占用存储（需定时清理）
- `tccli tcr DescribeTagRetentionRules --RegistryId "<ID>"` 返回 `TotalCount: 0`，命名空间下无保留规则，旧版本无人清理
- 镜像存储用量接近配额，`DescribeInstanceStatus` 返回的存储用量告警，需删旧留新


## 概述

保留规则按 `CronSetting` 定时执行，按 `RetentionRule` 保留指定数量/时间的镜像，其余删除。

| 规则类型 | 字段 | 含义 | 示例取值 |
|:---------|:----|:-----|:-----------|
| 保留最新 N 个 | `latestPushedK` | 保留最近推送的 N 个版本 | `10` |
| 保留最近 N 天 | `nDaysSinceLastPush` | 保留 N 天内推送的版本 | `30` |

> 规则作用于命名空间级别（`NamespaceId`）。**单个命名空间暂只能创建一条**保留规则。规则创建后按 `CronSetting` 定时执行，也可 `CreateTagRetentionExecution` 手动触发。创建后**不可修改**生效的命名空间。

> **删除边界**：版本保留删除的是**镜像版本信息**，**不删除**底层镜像层数据；要释放 COS 空间须再执行 GC / 制品清理（见 [生命周期概览](index.md)）。
>
> **Digest 风险**：多个 Tag 可能指向同一 Digest。删除其中一个 Tag 时，可能连带清理该 Digest 上的全部 Tag——仓库若存在「同 Digest、多 Tag」，启用保留规则前先评估，或暂时禁用已有规则。

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

> 完整入参以 `tccli tcr CreateTagRetentionRule help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| NamespaceId | int | 是 | 命名空间 ID | `ResourceNotFound` |
| CronSetting | string | 是 | **仅** `manual` / `daily` / `weekly` / `monthly`（非 5 段 Cron 表达式） | `InvalidParameterValue` |
| RetentionRule | object | 否 | `{Key, Value}`；省略时须配合 `AdvancedRuleItems` 或 `manual` 场景 | `InvalidParameterValue` |
| RetentionRule.Key | string | 传对象时是 | `latestPushedK` / `nDaysSinceLastPush` | `InvalidParameterValue` |
| RetentionRule.Value | int | 传对象时是 | 保留数量或天数 | `InvalidParameterValue` |
| AdvancedRuleItems | list | 否 | 高级规则（按 tag/仓库过滤） | `InvalidParameterValue` |
| Disabled | boolean | 否 | 是否禁用规则 | — |

> ⚠️ `NamespaceId` 是整数 ID（非命名空间名），从 `DescribeNamespaces` 响应的 `NamespaceId` 字段获取。

## 操作步骤

### 步骤 1：决策 — 保留策略

#### 为什么选 latestPushedK vs nDaysSinceLastPush

- **latestPushedK（保留 N 个）**: 保留最近 N 个版本，数量固定。适合持续集成（每次推送保留最新 10 个）
- **nDaysSinceLastPush（保留 N 天）**: 保留 N 天内版本，时间固定。适合按时间回滚的需求
- **默认推荐**: `latestPushedK` + `Value=10`——多数场景保留最新 10 个版本
- **可修改**： 能，`ModifyTagRetentionRule` 修改规则

```bash
# 修改保留规则（RegistryId + RetentionId 定位 + 新 CronSetting/RetentionRule）
# CronSetting 仅 manual|daily|weekly|monthly；Key 用 nDaysSinceLastPush（非 nDays）
tccli tcr ModifyTagRetentionRule --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --RetentionId <RETENTION_ID> --NamespaceId <NAMESPACE_ID> \
  --CronSetting daily \
  --RetentionRule '{"Key":"nDaysSinceLastPush","Value":30}'
# expected: exit 0; RetentionId 不存在时报 InvalidParameter
```

> `ModifyTagRetentionRule` 用 `RetentionId`（整数，来自 `DescribeTagRetentionRules`）定位规则，`RetentionRule`/`CronSetting` 覆盖式更新。`NamespaceId` 仍需传（规则绑命名空间）。改后用 `DescribeTagRetentionRules` 确认新值，或 `CreateTagRetentionExecution` 手动触发验证效果。

### 步骤 2：创建保留规则

`CreateTagRetentionRule` 必传 `RegistryId`/`NamespaceId`/`CronSetting`；`RetentionRule` 为常用可选对象（命名空间级全仓库时传 `{Key,Value}`，高级过滤用 `AdvancedRuleItems`）。按场景**二选一**：A 最小化（命名空间级全仓库）或 B 增强（按仓库过滤）。

> ⚠️ **A 与 B 是二选一变体，不是先做 A 再做 B**——两者各调一次 `CreateTagRetentionRule` 会建**两条规则**。规则创建后改配置用 `ModifyTagRetentionRule`（覆盖式，须含原值），**禁用第二次 `CreateTagRetentionRule` 改配置**。

#### 选项 A：最小化（命名空间级全仓库）

```bash
# CronSetting 仅 manual|daily|weekly|monthly（非 5 段 Cron）
tccli tcr CreateTagRetentionRule --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceId <NAMESPACE_ID> \
  --CronSetting daily \
  --RetentionRule '{"Key":"latestPushedK","Value":10}'
# expected: exit 0, 返回 RetentionId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<REGISTRY_ID>` | 实例 ID | `tcr-xxxxxxxx` | `tccli tcr DescribeInstances` |
| `<NAMESPACE_ID>` | 命名空间 ID | 整数 | `tccli tcr DescribeNamespaces` → `NamespaceList[].NamespaceId` |

#### 选项 B：增强（按仓库过滤）

> **与 A 二选一，非在 A 之后执行**。只对匹配的仓库应用规则（`AdvancedRuleItems`）。

```bash
tccli tcr CreateTagRetentionRule --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceId <NAMESPACE_ID> \
  --CronSetting daily \
  --RetentionRule '{"Key":"latestPushedK","Value":5}' \
  --AdvancedRuleItems '[{"RepositoryFilter":{"Decoration":"repoMatches","Pattern":"prod-*"}}]'
# expected: exit 0
```

> `AdvancedRuleItems` 按 `RepositoryFilter`/`TagFilter` 细化规则作用范围。仓库过滤 `Decoration` 为 `repoMatches` / `repoExcludes`；Tag 过滤为 `matches` / `excludes`。

### 步骤 3：手动触发执行（测试规则）

```bash
# 手动触发一次执行（验证规则效果，先在测试仓库验证）
tccli tcr CreateTagRetentionExecution --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --RetentionId <RETENTION_ID>
# expected: exit 0, 返回执行 ID
```

### 步骤 4：验证

```bash
# 查看规则
tccli tcr DescribeTagRetentionRules --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "RetentionPolicyList[].{id:RetentionId,cron:CronSetting,disabled:Disabled}"
# expected: 含刚创建的规则

# 查看执行历史
tccli tcr DescribeTagRetentionExecution --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --RetentionId <RETENTION_ID>
# expected: 执行记录，Status 含 Succeed（枚举: Failed/Succeed/Stopped/InProgress，非 Success）

# 查看单次执行的任务详情（RegistryId + RetentionId + ExecutionId 定位某次执行）
tccli tcr DescribeTagRetentionExecutionTask --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --RetentionId <RETENTION_ID> --ExecutionId <EXECUTION_ID> \
  --Offset 0 --Limit 20
# expected: 返回 RetentionTaskList[]+TotalCount; RetentionId 不存在报 InvalidParameter.ErrorTcrInvalidParameter: no such Retention policy
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 规则存在 | `DescribeTagRetentionRules` → `RetentionPolicyList` | 含目标规则 |
| 规则启用 | `DescribeTagRetentionRules` → `Disabled` | `false` |
| 执行成功 | `DescribeTagRetentionExecution` → `Status` | `Succeed` |
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
| `InvalidParameterValue.CronSetting` | 检查取值 | 非 `manual`/`daily`/`weekly`/`monthly` | 用四选一枚举，**不要**写 5 段 Cron |
| `InvalidParameterValue.RetentionRule` | 检查 Key/Value | Key 拼错或 Value 非数字 | Key 用 `latestPushedK`/`nDaysSinceLastPush`，Value 为正整数 |
| `FailedOperation` | `DescribeInstanceStatus` 查看状态 | 实例非 Running | 等实例 Running |

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
# 创建触发器 (RegistryId + Namespace + 嵌套 Trigger 对象，5 个必填字段: Name/Targets/EventTypes/Condition/Enabled)
tccli tcr CreateWebhookTrigger --RegistryId "<REGISTRY_ID>" --Namespace "<NAMESPACE>" --region <REGION> \
  --Trigger '{"Name":"<TRIGGER_NAME>","EventTypes":["pushImage"],"Condition":".*","Enabled":true,"Targets":[{"Type":"webhook","Address":"<URL>"}]}'
# expected: exit 0 返回 Trigger（含 Id）; 缺 EventTypes/Condition/Enabled 报 InvalidParameter.ErrorTcrInvalidParameter: non zero value required
```

> `CreateWebhookTrigger` 的 `Trigger` 嵌套对象有 5 个必填字段：`Name`/`Targets[]`/`EventTypes[]`（触发动作如 `pushImage`）/`Condition`（正则触发规则）/`Enabled`。缺任一报 `InvalidParameter.ErrorTcrInvalidParameter: non zero value required`。创建后用 `DescribeWebhookTrigger` 查列表，执行日志查 `DescribeWebhookTriggerLog`。
```json
{"TotalCount": 0, "Triggers": [], "RequestId": "..."}
```

```bash
# 修改触发器（RegistryId + Namespace 必填 + 嵌套 Trigger；Trigger 须含 Id 定位目标）
tccli tcr ModifyWebhookTrigger --RegistryId "<REGISTRY_ID>" --Namespace "<NAMESPACE>" --region <REGION> \
  --Trigger '{"Id":<TRIGGER_ID>,"Name":"<TRIGGER_NAME>","EventTypes":["pushImage"],"Condition":".*","Enabled":true,"Targets":[{"Address":"<URL>"}]}'
# expected: exit 0；缺 --Namespace 报 the following arguments are required: --Namespace

# 删除触发器 (按 Namespace + Id)
tccli tcr DeleteWebhookTrigger --RegistryId "<REGISTRY_ID>" --Namespace "<NAMESPACE>" --Id <TRIGGER_ID> --region <REGION>
# expected: exit 0
```

> `DeleteWebhookTrigger` 用 `Id`（Integer，触发器 ID）+ `Namespace` 定位，非触发器名。`ModifyWebhookTrigger` 必填 `--Namespace`，`Trigger` 是嵌套对象（须含 `Id` 及 Name/Targets[]/EventTypes/Condition/Enabled）。触发器执行日志查 `DescribeWebhookTriggerLog`。

```bash
# 查询触发器执行日志（RegistryId + Namespace + Id 定位触发器，分页取日志）
tccli tcr DescribeWebhookTriggerLog --RegistryId "<REGISTRY_ID>" --Namespace "<NAMESPACE>" \
  --Id <TRIGGER_ID> --Offset 0 --Limit 20 --region <REGION>
# expected: 返回 TotalCount+Logs[]; Namespace 不存在报 ResourceNotFound.TcrResourceNotFound: namespace not found
```

## GC 垃圾回收任务

> GC（垃圾回收）清理镜像层占用的存储。删除镜像后，GC 释放底层存储。属生命周期存储管理。GC 完整闭环：创建 → 查状态 → 终止。

### 创建 GC 任务

```bash
# 创建 GC 任务（RegistryId + GCParameters 嵌套配置；建议先 Dryrun=true 预览）
# 字段名是 Dryrun（全小写 r），非 DryRun
tccli tcr CreateGCJob --RegistryId "<REGISTRY_ID>" --region <REGION> \
  --GCParameters '{"Dryrun":false}'
# expected: exit 0，返回 {"RequestId":"..."}（无 JobId 字段，任务用 DescribeGCJobs 查 Jobs[] 状态）
```

> `GCParameters` 含 `Dryrun`（字段名全小写 r，非 `DryRun`）试运行开关。GC 删除镜像层后释放存储，**不可逆**，建议先 `Dryrun=true` 预览影响范围再正式执行。

### 查询 GC 任务

```bash
tccli tcr DescribeGCJobs --RegistryId "<REGISTRY_ID>" --region <REGION>
# expected: exit 0, Jobs[] (无任务则空)
```
```json
{"Jobs": [], "RequestId": "..."}
```

### 终止 GC 任务

```bash
# 终止进行中的 GC 任务
tccli tcr TerminateGCJob --RegistryId "<REGISTRY_ID>" --region <REGION>
# expected: exit 0
```

> GC 任务由系统在删除镜像后触发或手动发起。`DescribeGCJobs` 返回 `Jobs[]`，状态字段是 **`JobStatus`**（非 `Status`；示例值如 `finished`，以实际返回为准），`TerminateGCJob` 终止进行中的 GC（仅 RegistryId）。

## 收尾确认

```bash
# 汇总核对：规则创建 + 执行记录 + 删除效果
# 规则已创建且启用
tccli tcr DescribeTagRetentionRules --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "RetentionPolicyList[0].{id:RetentionId,cron:CronSetting,disabled:Disabled}"
# expected: id=<RETENTION_ID>, cron 与配置一致, disabled=false

# 手动触发的执行已完成（Status=Succeed）
tccli tcr DescribeTagRetentionExecution --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --RetentionId <RETENTION_ID> \
  --filter "RetentionExecutionList[0].{status:Status,start:StartTime}"
# expected: status=Succeed, start 为触发时间（Status 枚举: Failed/Succeed/Stopped/InProgress）

# 镜像数已收敛到保留数量（latestPushedK=Value 的效果）
tccli tcr DescribeImages --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --NamespaceName "<NS>" --RepositoryName "<REPO>" \
  --filter "length(ImageInfoList)"
# expected: ≤ RetentionRule.Value（保留数量上限）
```

> 规则启用 + 执行 Succeed + 镜像数收敛到保留值 = 版本保留配置完成，旧版本已按策略清理。

---

## 下一步

- [不可变标签](immutable-tags.md) — 禁止覆盖（与保留互补）
- [管理命名空间和仓库](../repositories/manage.md) — 创建命名空间
- [推送拉取镜像](../images/push-pull.md) — 手动删除单个版本
- [故障排查](../troubleshooting.md) — 规则不生效诊断
