---
doc_type: How-to
subtype: 6A
fused: true
---
# 查询与删除 AI Skill

> 查询 TCR 企业版实例下的 AI Skill 列表、版本、详情、下载链接，并删除 Skill。
> 控制台: [容器镜像服务 - AI Skills](https://console.cloud.tencent.com/tcr)
> 官方文档: [AI Skills 管理](https://cloud.tencent.com/document/product/1141/129802)

AI Skill 是可被智能体调用的工具包，挂在企业版实例下。每个 Skill 含多个版本，可获取预签名下载链接分发使用。查询与删除通过 5 个 Action 完成。

| Action | 用途 | 必填入参 |
|:-------|:-----|:---------|
| `ListSkills` | 列出 Skill | `RegistryId` |
| `ListSkillVersions` | 列出某 Skill 的版本 | `RegistryId` + `SkillName` |
| `DescribeSkillDetail` | 查 Skill 详情（类型/运行时/状态） | `RegistryId` + `SkillName` + `SkillVersion` |
| `DescribeSkillDownloadInfo` | 获取预签名下载链接 | `RegistryId` + `SkillName` + `SkillVersion` |
| `DeleteSkill` | 删除指定 Skill 版本 | `RegistryId`（调用时显式传 `Items[]` 指定目标） |

## 触发条件

- 你要查看实例下有哪些 AI Skill — `ListSkills`
- 你要获取某 Skill 的版本历史 — `ListSkillVersions`
- 你要查看某个 Skill 版本的完整详情 — `DescribeSkillDetail`
- 你要获取 Skill 下载链接以分发使用 — `DescribeSkillDownloadInfo`
- 你要删除某个 Skill — `DeleteSkill`

## 准备工作

- 已有 TCR 企业版实例且开启 AI 特性（`DescribeInstances` 的 `AIFeature=true`），并拿到 `RegistryId`
- 已配置 tccli 凭证（见 [配置凭证](../../getting-started/credentials.md)）

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `RegistryId` | 全部 | 是 | 企业版实例 ID |
| `SearchKey` | ListSkills | 否 | 模糊查询（匹配 Skill 名） |
| `SkillName` | ListSkills | 否 | 可选名称筛选 |
| `SkillName` | ListSkillVersions / DescribeSkillDetail / DescribeSkillDownloadInfo | 是 | Skill 名称 |
| `SkillType` | ListSkills | 否 | 按类型过滤，枚举：`MCP Server` |
| `Status` | ListSkills | 否 | 按状态过滤，枚举：`active` |
| `Offset` / `Limit` | ListSkills / ListSkillVersions | 否 | 分页；`Limit` 默认 20 |
| `SkillVersion` | DescribeSkillDetail / DescribeSkillDownloadInfo / DeleteSkill.Items | 是 | Skill 版本 |
| `Items[]` | DeleteSkill | API 可选；删除操作须显式传入 | 删除项数组，每项含 `SkillName` + `SkillVersion`；显式指定删除目标 |

## 操作步骤

### 步骤 1：列出 Skill

```bash
tccli tcr ListSkills --region <REGION> --RegistryId <REGISTRY_ID> --Limit 20
# expected: exit 0, { "TotalCount": <N>, "SkillList": [...], "RequestId": "..." }
```

`SkillList[]` 每项含：`SkillName` / `Description` / `SkillType` / `Tags` / `LatestVersion` / `Status` / `UpdateTime`。无 Skill 时返回 `TotalCount: 0, SkillList: []`。

按类型或状态过滤：

```bash
tccli tcr ListSkills --region <REGION> --RegistryId <REGISTRY_ID> \
  --SkillType "MCP Server" --Status active
# expected: TotalCount 与 SkillList 仅含匹配项
```

### 步骤 2：列出 Skill 版本

```bash
tccli tcr ListSkillVersions --region <REGION> --RegistryId <REGISTRY_ID> \
  --SkillName <SKILL_NAME> --Limit 20
# expected: exit 0, { "TotalCount": <N>, "VersionList": [...], "RequestId": "..." }
```

`VersionList[]` 每项含：`Version` / `Size` / `PushTime`（注意：Skill 版本无 `IsRecommended` 字段，与 AI 模型版本不同）。

### 步骤 3：查看 Skill 详情

```bash
tccli tcr DescribeSkillDetail --region <REGION> --RegistryId <REGISTRY_ID> \
  --SkillName <SKILL_NAME> --SkillVersion <SKILL_VERSION>
# expected: exit 0, { "Skill": {...}, "RequestId": "..." }
```

`Skill` 含：`SkillName` / `SkillVersion` / `Description` / `Tags` / `SkillType` / `Runtime`（运行时）/ `Status` / `UpdateTime`。

### 步骤 4：获取下载链接

```bash
tccli tcr DescribeSkillDownloadInfo --region <REGION> --RegistryId <REGISTRY_ID> \
  --SkillName <SKILL_NAME> --SkillVersion <SKILL_VERSION>
# expected: exit 0, { "PreSignedDownloadURL": "https://...", "RequestId": "..." }
```

`PreSignedDownloadURL` 是预签名下载链接，有时效（以实际返回为准）。`SkillVersion` 不存在时链接为空或报错，先用 `ListSkillVersions` 核对可用版本。

## 验证

```bash
# 删除前确认目标 Skill 版本存在
tccli tcr ListSkillVersions --region <REGION> --RegistryId <REGISTRY_ID> \
  --SkillName <SKILL_NAME> --filter "VersionList[].Version"
# expected: ["<版本1>", "<版本2>", ...] 含待删 SkillVersion
```

## 清理

`DeleteSkill` 通过 `Items[]` 嵌套数组删除，每项用 `SkillName` + `SkillVersion` 定位。虽然 API 将 `Items` 标为可选，删除操作应显式传入 `Items`，避免删除目标不明确。

```bash
tccli tcr DeleteSkill --region <REGION> --RegistryId <REGISTRY_ID> \
  --Items '[{"SkillName":"<SKILL_NAME>","SkillVersion":"<SKILL_VERSION>"}]'
# expected: exit 0, { "RequestId": "..." }（删除成功仅返回 RequestId）
```

> 删除不可恢复。删除后先用 `ListSkillVersions` 确认目标版本不再返回；仅当删除的是最后一个版本时，才能进一步确认 `ListSkills` 不再返回该 Skill。

## 故障恢复

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| `the following arguments are required: --SkillName` | tccli 客户端报错 | `ListSkillVersions` 缺必填 `SkillName` | 补全 `--SkillName` |
| `ListSkills` 返回 `TotalCount: 0` | 响应正常 | 类型/状态不匹配，或实例无 Skill | 用 `DescribeInstances` 确认 `AIFeature=true`；去掉过滤条件全量列举 |
| `DescribeSkillDetail` 返回错误 | 响应 | `SkillName`/`SkillVersion` 不存在 | 用 `ListSkillVersions` 核对版本 |
| 下载链接为空 | `DescribeSkillDownloadInfo` | `SkillVersion` 不存在 | 用 `ListSkillVersions` 核对可用版本 |

> 查询类对不存在的资源多返回空列表（`TotalCount: 0`），不返回 `ResourceNotFound`。`RegistryId` 错误以实际返回的 `Error.Code` 为准。

## 收尾确认

```bash
# 确认目标版本已删除
# 若版本较多，请按实际总数调整 Limit 或继续翻页，避免只检查第一页
tccli tcr ListSkillVersions --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --SkillName "<SKILL_NAME>" --Limit 100 \
  --filter "VersionList[?Version=='<SKILL_VERSION>'].Version"
# expected: []（目标 SkillVersion 不再返回）
```

如果删除前确认目标版本是该 Skill 的最后一个版本，可再运行：

```bash
tccli tcr ListSkills --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --SearchKey "<SKILL_NAME>" \
  --filter "SkillList[?SkillName=='<SKILL_NAME>'].SkillName"
# expected: []（仅适用于已删除最后一个版本）
```

## 下一步

- 查询与删除 AI 模型：[查询与删除 AI 模型](models.md)
- 管理镜像仓库与命名空间：[管理命名空间和仓库](../repositories/manage.md)
- AI 模型与 Skill 概览：[AI 模型与 Skill](index.md)
