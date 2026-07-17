---
doc_type: How-to
subtype: 6A
fused: true
---
# 查询与删除 AI 模型

> 查询 TCR 企业版实例下的 AI 模型仓库、版本、详情，并按版本删除模型。
> 控制台: [容器镜像服务 - AI 模型](https://console.cloud.tencent.com/tcr)
> 官方文档: [AI 模型管理](https://cloud.tencent.com/document/product/1141/129812)

AI 模型以模型仓库形式挂在企业版实例下，一个仓库含多个版本。模型内容通过 `docker push` 推送到实例命名空间下的仓库创建（TCR AI 无独立 Create Action）；查询与删除通过 4 个 Action 完成。

| Action | 用途 | 必填入参 |
|:-------|:-----|:---------|
| `ListAIModels` | 列出模型仓库 | `RegistryId` |
| `ListAIModelVersions` | 列出某仓库的版本 | `RegistryId` + `NamespaceName` + `RepositoryName` |
| `DescribeAIModelVersionDetail` | 查版本详情（框架/精度等元数据） | `RegistryId` + `NamespaceName` + `RepositoryName` + `Reference` |
| `DeleteAIModel` | 删除指定版本（按 tag 或 digest） | `RegistryId` + `Items[]` |

## 触发条件

- 你要查看实例下有哪些 AI 模型仓库 — `ListAIModels`
- 你要获取某模型仓库的版本列表与推荐版本 — `ListAIModelVersions`
- 你要查看某个模型版本的详细元数据（框架、精度、参数规模） — `DescribeAIModelVersionDetail`
- 你要删除某个不再需要的模型版本 — `DeleteAIModel`

## 准备工作

- 已有 TCR 企业版实例且开启 AI 特性（`DescribeInstances` 的 `AIFeature=true`），并拿到 `RegistryId`
- 模型内容已通过 `docker push` 推送到实例命名空间下的仓库（`ListAIModels` 才能查到）
- 已配置 tccli 凭证（见 [配置凭证](../../getting-started/credentials.md)）

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `RegistryId` | 全部 | 是 | 企业版实例 ID |
| `Namespace` | ListAIModels | 否 | 按命名空间过滤 |
| `ModelName` | ListAIModels | 否 | 按模型仓库名精确过滤 |
| `SearchKey` | ListAIModels | 否 | 模糊搜索关键字（匹配模型名） |
| `Offset` / `Limit` | ListAIModels / ListAIModelVersions | 否 | 分页；`Limit` 默认 20 |
| `NamespaceName` | ListAIModelVersions / DescribeAIModelVersionDetail / DeleteAIModel.Items | 是 | 模型仓库所属命名空间 |
| `RepositoryName` | ListAIModelVersions / DescribeAIModelVersionDetail / DeleteAIModel.Items | 是 | 模型仓库名（即模型名） |
| `Reference` | DescribeAIModelVersionDetail / DeleteAIModel.Items | 是 | 版本引用（tag 或 digest） |
| `Items[]` | DeleteAIModel | 是 | 删除项数组，每项含 `NamespaceName`/`RepositoryName`/`Reference` |

## 操作步骤

### 步骤 1：列出模型仓库

```bash
tccli tcr ListAIModels --region <REGION> --RegistryId <REGISTRY_ID> --Limit 20
# expected: exit 0, { "TotalCount": <N>, "ModelList": [...], "RequestId": "..." }
```

`ModelList[]` 每项含：`ModelName` / `NamespaceName` / `LatestVersion` / `Kind` / `ImageSize` / `UpdateTime` / `Digest`。无模型时返回 `TotalCount: 0, ModelList: []`。

按命名空间或关键字过滤：

```bash
tccli tcr ListAIModels --region <REGION> --RegistryId <REGISTRY_ID> \
  --Namespace <NAMESPACE> --SearchKey <KEY>
# expected: TotalCount 与 ModelList 仅含匹配项
```

### 步骤 2：列出模型版本

```bash
tccli tcr ListAIModelVersions --region <REGION> --RegistryId <REGISTRY_ID> \
  --NamespaceName <NAMESPACE> --RepositoryName <MODEL_NAME> --Limit 20
# expected: exit 0, { "TotalCount": <N>, "VersionList": [...], "RequestId": "..." }
```

`VersionList[]` 每项含：`Version` / `Size` / `IsRecommended`（bool，推荐版本标记）/ `PushTime`。**推荐版本由 `IsRecommended=true` 判断**，非按时间猜。

```bash
# 用 --filter 取推荐版本
tccli tcr ListAIModelVersions --region <REGION> --RegistryId <REGISTRY_ID> \
  --NamespaceName <NAMESPACE> --RepositoryName <MODEL_NAME> \
  --filter "VersionList[?IsRecommended==`true`].Version"
# expected: ["<推荐版本 tag>"]（无推荐版本时为 []）
```

### 步骤 3：查看版本详情

```bash
tccli tcr DescribeAIModelVersionDetail --region <REGION> --RegistryId <REGISTRY_ID> \
  --NamespaceName <NAMESPACE> --RepositoryName <MODEL_NAME> --Reference <REFERENCE>
# expected: exit 0, { "Model": {...}, "RequestId": "..." }
```

`Model`（ModelDetail）含：`ModelName` / `NamespaceName` / `Version` / `Digest` / `Size` / `Framework`（框架）/ `Precision`（精度）/ `FileFormat` / `ParamSize`（参数规模）/ `Family` / `IsRecommended` / `PushTime`。

## 验证

```bash
# 删除前确认目标版本存在
tccli tcr ListAIModelVersions --region <REGION> --RegistryId <REGISTRY_ID> \
  --NamespaceName <NAMESPACE> --RepositoryName <MODEL_NAME> \
  --filter "VersionList[].Version"
# expected: ["<版本1>", "<版本2>", ...] 含待删 Reference
```

## 清理

`DeleteAIModel` 通过 `Items[]` 嵌套数组批量删除，每项用 `NamespaceName` + `RepositoryName` + `Reference`（tag 或 digest）定位版本。

```bash
tccli tcr DeleteAIModel --region <REGION> --RegistryId <REGISTRY_ID> \
  --Items '[{"NamespaceName":"<NAMESPACE>","RepositoryName":"<MODEL_NAME>","Reference":"<REFERENCE>"}]'
# expected: exit 0, { "RequestId": "..." }（删除成功仅返回 RequestId）
```

> 删除不可恢复。删除后 `ListAIModelVersions` 该版本不再返回；若删的是仓库最后一个版本，`ListAIModels` 中该仓库随之消失。删错版本须重新 `docker push`（TCCLI 无 AI 模型推送能力，模型内容通过 docker push 创建）。

## 故障恢复

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| `the following arguments are required: --RepositoryName` | tccli 客户端报错 | `ListAIModelVersions` 缺必填 `RepositoryName` | 补全 `--NamespaceName` + `--RepositoryName` |
| `ListAIModels` 返回 `TotalCount: 0` | 响应正常 | 命名空间/模型名不匹配，或该实例无 AI 模型 | 用 `DescribeInstances` 确认 `AIFeature=true`；去掉过滤条件全量列举 |
| `ListAIModelVersions` 返回 `TotalCount: 0` | 响应正常 | `NamespaceName`/`RepositoryName` 不存在（不报错，返回空） | 用 `ListAIModels` 核对仓库名 |
| 删除后版本仍存在 | `ListAIModelVersions` 复核 | 删除异步未完成，或 `Reference` 拼错 | 轮询确认；核对 `Reference` 是 tag 还是 digest |

> 查询类对不存在的资源**返回空列表**（`TotalCount: 0`），不返回 `ResourceNotFound`。`RegistryId` 错误以实际返回的 `Error.Code` 为准。

## 收尾确认

```bash
tccli tcr ListAIModels --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "TotalCount"
# expected: TotalCount 反映删除后的仓库数（删最后版本时仓库消失，TotalCount 减 1）
```

## 下一步

- 查询与删除 AI Skill：[查询与删除 AI Skill](skills.md)
- 管理镜像仓库与命名空间：[管理命名空间和仓库](../repositories/manage.md)
- AI 模型与 Skill 概览：[AI 模型与 Skill](index.md)
