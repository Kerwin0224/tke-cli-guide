---
doc_type: Overview
---
# 镜像生命周期管理

> 自动化镜像版本管理：保留、保护、清理。决定镜像如何随时间演进与清理。

## 这是什么

TCR 镜像生命周期管理自动化处理镜像版本的保留、保护与清理，避免旧版本堆积导致存储超限或被误覆盖。

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Tag Retention | 自动保留规则 | 避免旧版本堆积导致存储超限 |
| Immutable Tags | Tag 不可变规则 | 防止 `latest` 被意外覆盖 |
| GC | 垃圾回收 | 清理删除 Tag 后残留的镜像层 |
| Webhook | 事件触发器 | 镜像推送/删除时回调通知 |
| Replication | 实例同步 | 多地域镜像自动复制 |

## 生命周期子主题

| 子主题 | 作用 | 接口 | 文档 |
|:-------|:-----|:-----|:-----|
| 版本保留 | 按规则自动删旧留新 | `CreateTagRetentionRule` | [版本保留](tag-retention.md) |
| 不可变规则 | 禁止覆盖已存在 Tag | `CreateImmutableTagRules` | [不可变标签](immutable-tags.md) |
| GC 垃圾回收 | 清理未引用镜像层 | `CreateGCJob` | [版本保留 - GC](tag-retention.md#gc-垃圾回收任务) |
| Webhook 触发 | 推送/删除事件回调 | `CreateWebhookTrigger` | [版本保留 - Webhook](tag-retention.md#webhook-触发器) |
| 实例同步 | 跨地域复制 | `CreateReplicationInstance` | [实例同步](../replication/manage.md) |

> GC 手动触发用 `CreateGCJob`（参数以 `--generate-cli-skeleton` 实测为准）：

```bash
# 创建 GC 任务（RegistryId + GCParameters 嵌套配置）
tccli tcr CreateGCJob --RegistryId "<REGISTRY_ID>" --region <REGION> \
  --GCParameters '{"DryRun":false}'
# expected: exit 0, 返回 GC 任务 JobId
```

> `CreateGCJob` 的 `GCParameters` 是嵌套对象（含 `DryRun` 试跑开关）。GC 删除镜像层后释放存储，属不可逆操作，建议先用 `DryRun=true` 预览影响范围。任务状态用 `DescribeGCJobs` 查（见 [版本保留 - GC](tag-retention.md#gc-垃圾回收任务)）。版本保留与不可变规则有完整 API，在各自文档展开。

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
- [配额和限制](../reference/states.md) — 命名空间/仓库配额
