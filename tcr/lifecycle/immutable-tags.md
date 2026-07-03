---
doc_type: How-to
subtype: 6B
fused: false
---
# 镜像不可变规则

> 配置不可变规则，禁止覆盖已存在的镜像 Tag。防止 `latest` 等标签被意外覆盖。配置型操作。

## 概述

不可变规则匹配的 Tag 一旦存在，禁止再次 push 覆盖。与版本保留互补：保留管"删旧"，不可变管"不覆盖"。

| 规则维度 | 作用 | 示例 |
|:---------|:-----|:-----|
| 仓库匹配 | 按仓库名模式限定 | `prod-*` 匹配所有 prod 开头仓库 |
| Tag 匹配 | 按Tag 名模式限定 | `latest`/`v*` |
| 装饰 | 匹配或排除 | `matches`（匹配）/ `excludes`（排除） |

> 规则作用于命名空间级别。规则创建后立即生效，匹配的 Tag 被 push 覆盖时报 `denied`。

## 决策依据

#### 不可变 vs 版本保留

- **不可变规则**: 禁止覆盖已存在 Tag（防止 `latest` 被改）。Push 时拒绝
- **版本保留**: 自动删旧留新（清理堆积）。定时执行删除
- **默认推荐**: 生产命名空间对 `latest`/`v*` 设不可变；同时配版本保留清理旧版本
- **能关闭吗?**: 能，`DeleteImmutableTagRules` 或 `ModifyImmutableTagRules --Disabled true`

## 配置项

> 来源：`tccli tcr CreateImmutableTagRules --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 作用 | 填错的影响 |
|:------|------|:--------:|:-----|:-----------|
| RegistryId | string | 是 | 实例 ID | `ResourceNotFound` |
| NamespaceName | string | 是 | 命名空间名 | `ResourceNotFound` |
| Rule | object | 是 | 规则对象 | `InvalidParameterValue` |
| Rule.RepositoryPattern | string | 否 | 仓库匹配模式，如 `**`（全部）/`prod-*` | 作用范围错 |
| Rule.TagPattern | string | 否 | Tag 匹配模式，如 `latest`/`v*` | 作用范围错 |
| Rule.RepositoryDecoration | string | 否 | `matches`/`excludes` | 匹配逻辑错 |
| Rule.TagDecoration | string | 否 | `matches`/`excludes` | 匹配逻辑错 |
| Rule.Disabled | boolean | 否 | 是否禁用 | 规则不生效 |

> `Pattern` 支持 glob：`**` 匹配全部，`prod-*` 匹配前缀，`v*` 匹配 v 开头。`Decoration=matches` 表示匹配 Pattern 的生效；`excludes` 表示排除。

## 应用

### 开启不可变规则 — Minimal

```bash
# 命名空间下所有仓库的 latest Tag 不可覆盖
tccli tcr CreateImmutableTagRules --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" \
  --Rule '{"RepositoryPattern":"**","TagPattern":"latest","RepositoryDecoration":"matches","TagDecoration":"matches"}'
# expected: exit 0, 返回 RuleId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<REGISTRY_ID>` | 实例 ID | `tcr-xxxxxxxx` | `tccli tcr DescribeInstances` |
| `<NAMESPACE_NAME>` | 命名空间名 | 须已存在 | `tccli tcr DescribeNamespaces` |

### Enhanced: 版本号 Tag 不可变

```bash
# 所有 v 开头的 Tag（版本号）不可覆盖，防止发版后被篡改
tccli tcr CreateImmutableTagRules --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" \
  --Rule '{"RepositoryPattern":"**","TagPattern":"v*","RepositoryDecoration":"matches","TagDecoration":"matches"}'
# expected: exit 0
```

## 验证

```bash
# 查看规则（响应含 Rules/EmptyNs/Total）
tccli tcr DescribeImmutableTagRules --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "Rules[].{repo:RepositoryPattern,tag:TagPattern,disabled:Disabled}"
# expected: 规则列表

# 验证不可变生效：尝试覆盖 latest 应被拒绝
docker push <REGISTRY_DOMAIN>/<NAMESPACE_NAME>/<REPO>:latest
# expected: denied（规则生效）
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 规则存在 | `DescribeImmutableTagRules` → `Rules` | 含目标规则 |
| 规则启用 | `DescribeImmutableTagRules` → `Disabled` | `false` |
| 覆盖被拒 | `docker push <existing-tag>` | `denied` |
| 新 Tag 允许 | `docker push <new-tag>` | 成功 |

## 回滚

```bash
# 禁用规则（不删除，保留配置）
tccli tcr ModifyImmutableTagRules --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" \
  --Rule '{"RuleId":<RULE_ID>,"Disabled":true}'
# expected: exit 0

# 或删除规则
tccli tcr DeleteImmutableTagRules --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" --RuleId <RULE_ID>
# expected: exit 0
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound` | `DescribeNamespaces` 核对 | 命名空间不存在 | 先 `CreateNamespace` |
| `InvalidParameterValue` | 检查 Rule JSON | Pattern/Decoration 拼错 | Decoration 用 `matches`/`excludes` |
| `LimitExceeded` | `DescribeImmutableTagRules` 看数量 | 规则达上限 | 删除闲置规则 |
| `FailedOperation` | `DescribeInstanceStatus` 看状态 | 实例非 Running | 等实例 Running |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 规则创建但仍能覆盖 latest | `DescribeImmutableTagRules` → `Disabled` | 规则被禁用或 Pattern 不匹配 | 确认 `Disabled=false`，Pattern 匹配目标仓库 |
| docker push 报 `denied` 但非不可变原因 | `DescribeSecurityPolicies` 查白名单 | 权限不足非不可变 | 区分 `denied` 原因：不可变 vs 权限 |
| 规则误拦截正常推送 | `DescribeImmutableTagRules` 看 Pattern | Pattern 范围太宽 | 收窄 Pattern 或用 `excludes` |

## 下一步

- [版本保留策略](tag-retention.md) — 与不可变互补的自动清理
- [管理命名空间和仓库](../repositories/manage.md) — 命名空间管理
- [推送拉取镜像](../images/push-pull.md) — `denied` 错误诊断
- [故障排查](../troubleshooting.md) — 推送被拒诊断

## 控制台替代方案

[容器镜像服务控制台 - 不可变规则](https://console.cloud.tencent.com/tcr/immutable)
