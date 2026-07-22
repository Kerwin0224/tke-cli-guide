---
doc_type: Reference
subtype: 8B
---
# TCR 共享字段参考

> TCR 多个 Action 共用的嵌套字段结构。统一在此引导一次，各文档引用本页。
>
> 官方文档：[API 概览](https://cloud.tencent.com/document/product/1141/41813)

## 何时查本页

- 写 `CreateNamespace`/`ModifyNamespace`/`CreateServiceAccount`/`ManageReplication`/`CreateTagRetentionRule` 等命令时，遇到 `Filter`/`TagSpecification`/`Permission`/`CVEWhitelistItem`/`RetentionRule`/`ReplicationRule` 等参数
- 完整字段结构以 `tccli tcr <Action> help --detail` 为准

## Filter 查询过滤

`Filters` 是部分查询 Action 提供的过滤参数。本节以 `DescribeNamespaces` 为例；其他 Action 是否支持、参数名和合法 `Name` 以该 Action 的 `help --detail` 为准。

| 字段 | 类型 | 说明 |
|------|------|------|
| Name | string | `DescribeNamespaces` 当前只明确支持 `Tags`（值格式 `tagKey:tagVal`） |
| Values | array | 属性值，同 Filter 内 OR，多 Filter 间 AND |

```bash
# 按命名空间名查询使用顶层参数，不使用 Filter
tccli tcr DescribeNamespaces --region <REGION> \
  --RegistryId "<REGISTRY_ID>" \
  --NamespaceName "prod"
# expected: NamespaceList[] 返回目标实例中名称为 prod 的 TCR 命名空间（若存在）
```

> `RegistryId` 是本 Action 的必填顶层参数，`NamespaceName` 是可选顶层参数；它们都不是 `DescribeNamespaces.Filters[].Name`。该 Action 如需按标签过滤，可传 `--Filters '[{"Name":"Tags","Values":["team:backend"]}]'`。其他 Describe Action 支持的 Filter `Name` 不同，须逐 Action 用 `tccli tcr <Action> help --detail` 核对，不可迁移复用。

### DescribeNamespaces 专属筛选

| 字段 | 类型 | 说明 |
|:-----|:-----|:-----|
| `KmsSignPolicy` | boolean | 设为 `true` 时，仅返回已启用 KMS 镜像签名的命名空间；该字段是顶层筛选参数，不是通用 `Filters[].Name` |

## TagSpecification 云标签

| 字段 | 类型 | 说明 |
|------|------|------|
| ResourceType | string | 资源类型（默认 `instance`） |
| Tags | array | `Tag` 数组（Key/Value） |

```bash
# TagSpecification 为单对象（非数组）——展开时无 .0 下标
--cli-unfold-argument \
  --TagSpecification.ResourceType instance \
  --TagSpecification.Tags.0.Key team \
  --TagSpecification.Tags.0.Value backend
# JSON 等价: --TagSpecification '{"ResourceType":"instance","Tags":[{"Key":"team","Value":"backend"}]}'
```

## Permission 服务账号权限

用于 `CreateServiceAccount`/`ModifyServiceAccount`，控制服务账号对资源的操作权限。

| 字段 | 类型 | 说明 |
|------|------|------|
| Resource | string | 资源路径（目前仅支持 Namespace，如 `prod/`） |
| Actions | array | 动作列表：`tcr:PushRepository`/`tcr:PullRepository`/`tcr:DeleteRepository` 等 |

```bash
--Permissions.0.Resource prod/ --Permissions.0.Actions '["tcr:PushRepository","tcr:PullRepository"]'
```

## CVEWhitelistItem 漏洞白名单 {#cvewhitelistitem-漏洞白名单}

用于 `CreateNamespace`/`ModifyNamespace`，配置命名空间漏洞扫描的白名单 CVE。

| 字段 | 类型 | 说明 |
|------|------|------|
| CVEID | string | 漏洞 ID（如 `CVE-2024-12345`） |

```bash
# 创建命名空间时配置漏洞白名单
tccli tcr CreateNamespace --region <REGION> \
  --RegistryId "<REGISTRY_ID>" \
  --NamespaceName "<NAMESPACE>" \
  --IsPublic false \
  --CVEWhitelistItems '[{"CVEID":"CVE-2024-12345"},{"CVEID":"CVE-2024-67890"}]'
# expected: { "RequestId": "..." }；IsPublic 为必填（true 公开 / false 私有）
```

> 漏洞扫描阻断等级（`Severity`）在创建命名空间时配置（`low`/`medium`/`high`），白名单是补充——命中白名单的 CVE 即使达阻断等级也不阻断推送。

## RetentionRule 版本保留规则

用于 `CreateTagRetentionRule`/`ModifyTagRetentionRule`，定义镜像版本保留策略。

| 字段 | 类型 | 说明 |
|------|------|------|
| Key | string | 策略：`latestPushedK`（保留最新 K 个）/`nDaysSinceLastPush`（保留最近 N 天） |
| Value | int | 对应值（K 或 N） |

```bash
--RetentionRule '{"Key":"latestPushedK","Value":10}'
# 保留最新推送的 10 个版本
```

## ReplicationRule 实例同步规则

用于 `ManageReplication`，定义跨地域同步规则。

| 字段 | 类型 | 说明 |
|------|------|------|
| Name | string | 规则名称 |
| DestNamespace | string | 目标命名空间 |
| Override | bool | 是否覆盖目标已有镜像 |
| Filters | array | 同步过滤条件 |
| Deletion | bool | 是否同步删除事件 |

```bash
# Rule 是单个 ReplicationRule 对象（非数组）——展开时无 .0 下标；Filters 必填
--cli-unfold-argument \
  --Rule.Name prod-sync \
  --Rule.DestNamespace prod \
  --Rule.Override false \
  --Rule.Deletion false \
  --Rule.Filters.0.Type name \
  --Rule.Filters.0.Value '**'
# JSON 等价:
# --Rule '{"Name":"prod-sync","DestNamespace":"prod","Override":false,"Filters":[{"Type":"name","Value":"**"}],"Deletion":false}'
```

> `Override=true` 会覆盖目标端同名镜像，`Deletion=true` 会传播删除事件；启用前先确认目标命名空间和回滚方案。`Filters` 为必填（Type 为 `name`/`tag`/`resource`）。

详见 [实例同步](../replication/manage.md)。

## 传参模式速查

| 场景 | 写法 |
|------|------|
| 单值标量 | `--RegistryId "<ID>"` |
| 对象数组 | `--cli-unfold-argument --Filters.0.Name x --Filters.0.Values.0 y` 或 `--Filters '[{"Name":"x","Values":["y"]}]'` |
| 嵌套对象 | `--RetentionRule '{"Key":"latestPushedK","Value":10}'` |
