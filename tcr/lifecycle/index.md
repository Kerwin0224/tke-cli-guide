---
doc_type: Overview
---
# 镜像生命周期管理

> 自动化镜像版本管理：保留、保护、清理。决定镜像如何随时间演进与清理。
>
> 官方文档：[自动删除镜像版本](https://cloud.tencent.com/document/product/1141/50613) · [镜像版本不可变](https://cloud.tencent.com/document/product/1141/58200)

## 是什么

TCR 镜像生命周期管理自动化处理镜像版本的保留、保护与清理，避免旧版本堆积导致存储超限或被误覆盖。

## 触发条件

- 你要按规则自动删旧留新镜像版本（保留最近 N 个或按天保留）— 用 `CreateTagRetentionRule`，见 [版本保留](tag-retention.md)
- 你要禁止覆盖已存在的 Tag（防止 `latest` 被推送同名版本覆盖）— 用 `CreateImmutableTagRules`，见 [不可变标签](immutable-tags.md)
- 你删除 Tag 后存储未释放，需回收底层残留镜像层 — 用 `CreateGCJob`，见 [GC 垃圾回收任务](tag-retention.md#gc-垃圾回收任务)
- 你要在镜像推送/删除时触发外部回调 — 用 `CreateWebhookTrigger`，见 [Webhook 触发器](tag-retention.md#webhook-触发器)
- 你混淆「版本保留」与「制品清理 GC」（保留删 Tag 信息，GC 才回收层数据）— 看 [生命周期子主题](#生命周期子主题) 下方说明

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Tag Retention | 自动保留规则 | 避免旧版本堆积导致存储超限 |
| Immutable Tags | Tag 不可变规则 | 防止 `latest` 被意外覆盖 |
| GC | 垃圾回收 | 清理删除 Tag 后残留的镜像层 |
| Webhook | 事件触发器 | 镜像推送/删除时回调通知 |
| Replication | 实例同步 | 多地域镜像自动复制 |

## 生命周期子主题 {#生命周期子主题}

| 子主题 | 作用 | 接口 | 文档 |
|:-------|:-----|:-----|:-----|
| 版本保留 | 按规则自动删旧留新（Tag 层） | `CreateTagRetentionRule` | [版本保留](tag-retention.md) |
| 不可变规则 | 禁止覆盖已存在 Tag | `CreateImmutableTagRules` | [不可变标签](immutable-tags.md) |
| 制品清理 / GC | 回收未引用镜像层，释放 COS | `CreateGCJob` | [版本保留 - GC](tag-retention.md#gc-垃圾回收任务) · 控制台 [制品清理](https://console.cloud.tencent.com/tcr/gc) |
| Webhook 触发 | 推送/删除事件回调 | `CreateWebhookTrigger` | [版本保留 - Webhook](tag-retention.md#webhook-触发器) |
| 实例同步 | 跨地域复制 | `CreateReplicationInstance` | [实例同步](../replication/manage.md) |

> **版本保留 ≠ 制品清理**：保留规则删的是版本/Tag 信息；制品清理（GC）才回收底层层数据。完整闭环：`CreateGCJob` → `DescribeGCJobs` → 必要时 `TerminateGCJob`（见 [GC 垃圾回收任务](tag-retention.md#gc-垃圾回收任务)）。`GCParameters.Dryrun`（字段名全小写 r，非 `DryRun`）可先模拟再正式执行（正式删除层后不可逆）。

## 治理三步联合配置

版本保留 + 不可变规则 + GC 构成完整治理闭环，建议在同一命名空间联合配置：

| 步骤 | 作用 | Action | 文档 |
|:-----|:-----|:-----|:-----|
| 1. 版本保留 | 按规则删旧留新 | `CreateTagRetentionRule` | [版本保留](tag-retention.md) |
| 2. 不可变规则 | 禁止覆盖已存在 Tag | `CreateImmutableTagRules` | [不可变标签](immutable-tags.md) |
| 3. GC 清理 | 清残留镜像层 | `CreateGCJob` | [GC](tag-retention.md#gc-垃圾回收任务) |

> 触发链：保留规则删旧 Tag →（Tag 删除后）GC 清理残留镜像层；不可变规则独立生效防止 Tag 被覆盖。三者作用于同一命名空间，联合配置才完整。

## 不适用场景

- 不需自动清理（手动删镜像）→ 看 [推送拉取](../images/push-pull.md) 的 `DeleteImage`
- 个人版（无生命周期管理）→ 个人版不支持，需企业版
- 镜像版本少（<10）→ 不需保留规则，手动管理即可

## 快速检查

```bash
# 查看已有保留规则
tccli tcr DescribeTagRetentionRules --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "TotalCount"
# expected: 保留规则数

# 查看不可变规则
tccli tcr DescribeImmutableTagRules --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "Total"
# expected: 不可变规则数
```

## 文档

- [版本保留策略](tag-retention.md) — 自动清理旧版本、按规则保留
- [镜像不可变规则](immutable-tags.md) — 禁止覆盖已存在的 Tag
- [推送拉取镜像](../images/push-pull.md) — 手动删除单个镜像版本
- [配额和限制](../reference/quotas.md) — 命名空间/仓库配额
