---
doc_type: Overview
---
# AI 模型与 Skill

> 在 TCR 企业版实例中管理 AI 模型仓库与 AI Skill，支撑模型分发与智能体工具复用。
> 控制台: [容器镜像服务 - AI 模型 / AI Skills](https://console.cloud.tencent.com/tcr)
> 官方文档: [AI 模型管理](https://cloud.tencent.com/document/product/1141/129812) · [AI Skills 管理](https://cloud.tencent.com/document/product/1141/129802)

TCR 企业版将 AI 模型与 AI Skill 作为一类特殊制品托管：模型以模型仓库（含多版本与元数据）形式管理，Skill 以可下载的智能体工具包形式管理。二者均挂载在企业版实例下，通过 `RegistryId` 定位。TCR AI 无独立 Create Action，模型内容通过 `docker push` 推送创建。

## 触发条件

- 你要查询企业版实例下的 AI 模型仓库或某模型的多版本与版本详情 — 用 `ListAIModels`/`ListAIModelVersions`/`DescribeAIModelVersionDetail`，见 [查询与删除 AI 模型](models.md)
- 你要删除某个 AI 模型版本 — 用 `DeleteAIModel`，见 [查询与删除 AI 模型](models.md)
- 你要查询 AI Skill 列表/版本/详情，或获取 Skill 预签名下载链接 — 用 `ListSkills`/`ListSkillVersions`/`DescribeSkillDetail`/`DescribeSkillDownloadInfo`，见 [查询与删除 AI Skill](skills.md)
- 你要删除某个 AI Skill — 用 `DeleteSkill`，见 [查询与删除 AI Skill](skills.md)
- 你遇到"如何创建/推送 AI 模型"的困惑 — TCR AI 无独立 Create Action，模型内容通过 `docker push` 推送创建，见 [查询与删除 AI 模型](models.md)

## 核心概念

| 概念 | 含义 |
|:-----|:-----|
| AI 模型仓库 | 一个模型的多版本集合，每条记录含框架、精度、参数规模等元数据 |
| 推荐版本 | 模型版本中 `IsRecommended=true` 的 tag（`ListAIModelVersions` 返回） |
| AI Skill | 智能体可调用的工具包，含版本与预签名下载链接 |
| 命名空间 | 模型所属分组（与镜像命名空间同体系）；Skill 不依赖命名空间，按 `SkillName` 定位 |

## Action 全景

共 9 个 Action，分模型（4）与 Skill（5）两组，均挂实例下。

### AI 模型（4）

| Action | 用途 | 性质 |
|:-------|:-----|:-----|
| `ListAIModels` | 列出模型仓库 | 查询 |
| `ListAIModelVersions` | 列出某仓库版本 | 查询 |
| `DescribeAIModelVersionDetail` | 查版本详情 | 查询 |
| `DeleteAIModel` | 删除模型版本 | 删除 |

### AI Skill（5）

| Action | 用途 | 性质 |
|:-------|:-----|:-----|
| `ListSkills` | 列出 Skill | 查询 |
| `ListSkillVersions` | 列出某 Skill 版本 | 查询 |
| `DescribeSkillDetail` | 查 Skill 详情 | 查询 |
| `DescribeSkillDownloadInfo` | 获取下载链接 | 查询 |
| `DeleteSkill` | 删除 Skill | 删除 |

## 快速开始

- 查询与删除模型：[查询与删除 AI 模型](models.md)
- 查询与删除 Skill：[查询与删除 AI Skill](skills.md)
