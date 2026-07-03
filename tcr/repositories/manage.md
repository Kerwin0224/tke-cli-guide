---
doc_type: How-to
subtype: 6A
fused: false
---
# 管理命名空间和仓库

> 在 TCR 企业版实例下创建/查询/删除命名空间与仓库。命名空间是仓库的分组，仓库是镜像的存放单元。

## 概述

TCR 的两级结构：实例 → 命名空间 → 仓库 → 镜像版本。

| 资源 | 作用 | 唯一性 | 配额（basic） |
|:-----|:-----|:-----|:-------------|
| 命名空间 | 仓库分组，控制可见性（Public/Private） | 实例内唯一 | 50/实例 |
| 仓库 | 镜像存放单元 | 命名空间内唯一 | 1000/实例 |
| 镜像版本 | 具体镜像 tag | 仓库内唯一 | 无限制 |

> 命名空间名直接用于镜像地址：`<domain>/<namespace>/<repo>:<tag>`。命名空间创建后**不可改名**，只能删除重建。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tcr DescribeInstances --region <REGION> --Registryids '["<REGISTRY_ID>"]' \
  --filter "Registries[0].Status"
# expected: "Running"
```

### 资源检查

```bash
# 确认实例 Running 且有命名空间配额
tccli tcr DescribeNamespaces --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "TotalCount"
# expected: 数字 < 50（basic 配额）
```

## 关键字段

### CreateNamespace

> 来源：`tccli tcr CreateNamespace --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| NamespaceName | string | 是 | 2-30 字符，小写字母/数字/`-`/`_`，实例内唯一 | `InvalidParameter.InstanceName` / `LimitExceeded.Namespace` |
| IsPublic | boolean | 是 | `true`（公开，任何人可拉）/ `false`（私有，需凭证） | `InvalidParameterValue` |
| TagSpecification | object | 否 | 标签 | — |
| IsAutoScan | boolean | 否 | 自动安全扫描，默认 false | — |
| IsPreventVUL | boolean | 否 | 漏洞阻断开关，开启后推送镜像含指定等级漏洞会被拒，默认 false | — |
| Severity | string | 否（`IsPreventVUL=true` 时需设置） | 阻断漏洞等级枚举：`low` / `medium` / `high`。设 `high` 仅阻断高危漏洞，`low` 三级全阻断 | `InvalidParameter`（传非枚举值） |

### CreateRepository

> 来源：`tccli tcr CreateRepository --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| NamespaceName | string | 是 | 须已存在的命名空间 | `ResourceNotFound` |
| RepositoryName | string | 是 | 2-128 字符，命名空间内唯一 | `InvalidParameter` / `LimitExceeded.Repository` |
| BriefDescription | string | 否 | 仓库简述 | — |
| Description | string | 否 | 仓库详细描述 | — |

## 操作步骤

### 步骤 1：决策 — 命名空间可见性

#### 为什么选 Public vs Private

- **Private（推荐）**: 需凭证才能 push/pull，适合生产镜像。安全默认
- **Public**: 任何人可 `docker pull`（不可 push），适合开源镜像分发
- **默认推荐**: Private——除非确需公开分发，否则保持私有
- **能改吗?**: 命名空间可见性可后续 `ModifyNamespace`（如存在）修改，但镜像若已被公开拉取无法撤回

### 步骤 2：创建命名空间 — 最小化

```bash
tccli tcr CreateNamespace --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" --IsPublic false
# expected: exit 0, 返回 NamespaceId + RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<REGISTRY_ID>` | 实例 ID | `tcr-xxxxxxxx` | `tccli tcr DescribeInstances` → `Registries[].RegistryId` |
| `<NAMESPACE_NAME>` | 命名空间名 | 2-30 字符，小写字母数字`-`_，实例内唯一 | 自定义，如 `prod`、`team-backend` |

### 步骤 3：创建仓库 — 最小化

```bash
tccli tcr CreateRepository --region <REGION> \
  --RegistryId "<REGISTRY_ID>" \
  --NamespaceName "<NAMESPACE_NAME>" --RepositoryName "<REPOSITORY_NAME>"
# expected: exit 0
```

> 仓库也可在首次 `docker push` 时自动创建（若命名空间允许），但显式创建可控制可见性与描述。

### 步骤 4：查询

```bash
# 命名空间列表
tccli tcr DescribeNamespaces --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "NamespaceList[].{name:Name,public:Public}"
# expected: 命名空间名 + 可见性

# 仓库列表
tccli tcr DescribeRepositories --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --NamespaceName "<NAMESPACE_NAME>" --Limit 20 \
  --filter "RepositoryList[].{name:Name,ns:Namespace}"
# expected: 仓库列表
```

## 验证

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 命名空间存在 | `DescribeNamespaces` → `NamespaceList[].Name` | 含目标命名空间 |
| 仓库存在 | `DescribeRepositories` → `RepositoryList[].Name` | 含目标仓库 |
| 可见性 | `DescribeNamespaces` → `Public` | 与创建参数 `IsPublic` 一致 |

### 修改命名空间属性

> 命名空间可见性、自动扫描、漏洞阻断可后续修改。`ModifyNamespace` 覆盖式更新，参数以 `--generate-cli-skeleton` 为准（`IsPublic`/`IsAutoScan`/`IsPreventVUL` 等）。

```bash
# 修改命名空间可见性（RegistryId + NamespaceName 定位）
tccli tcr ModifyNamespace --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" --IsPublic false
# expected: exit 0; 命名空间不存在报 ResourceNotFound.TcrResourceNotFound
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<REGISTRY_ID>` | 实例 ID | `tccli tcr DescribeInstances` |
| `<NAMESPACE_NAME>` | 命名空间名 | `tccli tcr DescribeNamespaces` → `NamespaceList[].Name` |

> `ModifyNamespace` 用 `NamespaceName`（字符串名）而非 `NamespaceId`，可改 `IsPublic`（可见性）/`IsAutoScan`（自动扫描）/`IsPreventVUL`（漏洞阻断）等。改可见性后用 `DescribeNamespaces` 确认 `Public` 字段。镜像若已被公开拉取，改回 Private 不影响已拉取的副本。

## 清理

> **副作用警告**：删除命名空间会级联删除其下所有仓库与镜像版本，不可恢复。删除仓库会删除该仓库所有镜像版本。

```bash
# 1. 删除仓库（先删仓库再删命名空间）
tccli tcr DeleteRepository --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" --RepositoryName "<REPOSITORY_NAME>"
# expected: exit 0

# 2. 删除命名空间（须无仓库才能删）
tccli tcr DeleteNamespace --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>"
# expected: exit 0

# 3. 验证已删
tccli tcr DescribeNamespaces --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "NamespaceList[?Name=='<NAMESPACE_NAME>']"
# expected: 空数组
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `InvalidParameter.InstanceName` | 检查命名空间格式 | 含非法字符或超长 | 用小写字母/数字/`-`/`_`，2-30 字符 |
| `LimitExceeded.Namespace` | `DescribeNamespaces` 看存量 | 命名空间达上限（basic=50） | 删除闲置命名空间或提工单提额 |
| `LimitExceeded.Repository` | `DescribeRepositories` 看存量 | 仓库达上限（basic=1000） | 删除闲置仓库 |
| `ResourceNotFound` | `DescribeInstances` 核对 ID | RegistryId 错或实例非 Running | 确认实例 ID 与状态 |
| `FailedOperation` | `DescribeInstanceStatus` 看状态 | 实例非 Running | 等实例 Running 后重试 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 命名空间创建成功但 docker push 报 `denied` | `DescribeNamespaces` → `Public` | 命名空间 Private 但 Token 无 push 权限 | 配置访问凭证，见 [访问控制](../access/manage.md) |
| 删除命名空间报 `ResourceInUse` | `DescribeRepositories` 看仓库 | 命名空间下仍有仓库 | 先删所有仓库再删命名空间 |

## 仓库属性与标签管理

> 修改仓库描述、删除仓库标签。

```bash
# 修改仓库属性 (BriefDescription 简述 / Description 详述)
tccli tcr ModifyRepository --RegistryId "<REGISTRY_ID>" --region <REGION> \
  --NamespaceName "<NAMESPACE>" --RepositoryName "<REPO>" \
  --BriefDescription "<BRIEF>" --Description "<DESC>"
# expected: exit 0

# 删除仓库标签 (Tags[] 待删 tag 列表)
tccli tcr DeleteRepositoryTags --RegistryId "<REGISTRY_ID>" --region <REGION> \
  --NamespaceName "<NAMESPACE>" --RepositoryName "<REPO>" --Tags '["<TAG1>","<TAG2>"]'
# expected: exit 0
```

> `ModifyRepository` 用 `NamespaceName`+`RepositoryName` 定位（非 ID）。`DeleteRepositoryTags` 批量删 tag（区别于删整个仓库 `DeleteRepository`）。

### Helm Chart 下载

```bash
# 查询 Helm Chart 下载信息
tccli tcr DescribeChartDownloadInfo --RegistryId "<REGISTRY_ID>" --region <REGION> \
  --NamespaceName "<NAMESPACE>" --ChartName "<CHART>" --ChartVersion "<VERSION>"
# expected: exit 0, 下载地址

# 下载 Helm Chart
tccli tcr DownloadHelmChart --RegistryId "<REGISTRY_ID>" --region <REGION> \
  --NamespaceName "<NAMESPACE>" --ChartName "<CHART>" --ChartVersion "<VERSION>"
# expected: exit 0, 返回下载包
```

> Helm Chart 用 `ChartName`+`ChartVersion`（非 RepositoryName），存放在命名空间下。

## 下一步

- [推送拉取镜像](../images/push-pull.md) — 命名空间/仓库创建后 push/pull
- [访问控制](../access/manage.md) — 配置谁能 push/pull
- [创建实例](../instances/create.md) — 实例生命周期
- [故障排查](../troubleshooting.md) — `denied` / `not found` 诊断

## 控制台替代方案

[容器镜像服务控制台 - 仓库管理](https://console.cloud.tencent.com/tcr/repository)
