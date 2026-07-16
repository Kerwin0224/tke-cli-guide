---
doc_type: How-to
subtype: 6A
fused: false
---
# 管理命名空间和仓库

> 控制台: [容器镜像服务控制台 - 仓库管理](https://console.cloud.tencent.com/tcr/repository)
> 官方文档: [管理命名空间](https://cloud.tencent.com/document/product/1141/41803) · [管理镜像仓库](https://cloud.tencent.com/document/product/1141/41811) · [产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731)
> 在 TCR 企业版实例下创建/查询/删除命名空间与仓库。命名空间是仓库的分组，仓库是镜像的存放单元。

## 触发条件

- `tccli tcr DescribeNamespaces --RegistryId "<ID>"` 返回 `TotalCount: 0`（实例下无命名空间，docker push 报 `project not found`）
- `docker push <DOMAIN>/<NS>/<REPO>:<TAG>` 报 `repository not found`，`DescribeRepositories --NamespaceName "<NS>"` 返回 `RepositoryList: []`（命名空间下无仓库）
- `LimitExceeded.Namespace`（命名空间数达 basic=50 上限）或 `LimitExceeded.Repository`（仓库数达 basic=1000 上限），需清理或提额


## 概述

TCR 的两级结构：实例 → 命名空间 → 仓库 → 镜像版本。

| 资源 | 作用 | 唯一性 | 配额（basic / standard / premium） |
|:-----|:-----|:-----|:-------------|
| 命名空间 | 仓库分组，控制可见性（Public/Private） | 实例内唯一 | 50 / 100 / 500 |
| 仓库 | 镜像存放单元 | 命名空间内唯一 | 1000 / 3000 / 5000 |
| 镜像版本 | 具体镜像 tag | 仓库内唯一 | 无限制 |

> 配额以官方 [104731](https://cloud.tencent.com/document/product/1141/104731) 为准，本仓汇总见 [配额与限制](../reference/quotas.md)。命名空间名直接用于镜像地址：`<domain>/<namespace>/<repo>:<tag>`。命名空间创建后**不可改名**，只能删除重建。
>
> **为什么本篇可能出现 docker**：仓库 CRUD 纯 tccli；`docker login`/`push` 仅在收尾衔接下一步时作为能力边界提示——TCCLI 无 Registry 协议与镜像层传输 Action。

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

> 完整入参以 `tccli tcr CreateNamespace help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` / `InternalError.DbError`（假 ID） |
| NamespaceName | string | 是 | API：长度 **2–30** 字符，只能包含小写字母、数字及分隔符 `.`/`_`/`-`（不能以分隔符开头、结尾或连续），实例内唯一 | `InvalidParameter.InstanceName` / `LimitExceeded.Namespace` / `FailedOperation.ErrorTcrResourceConflict`（重名） |
| IsPublic | boolean | **是** | `true`（公开，任何人可拉）/ `false`（私有，需凭证）。**缺省被 tccli 本地拒绝**：`the following arguments are required: --IsPublic` | `InvalidParameterValue`（非法布尔） |
| TagSpecification | object | 否 | 标签 | — |
| IsAutoScan | boolean | 否 | 自动安全扫描，默认 false | — |
| IsPreventVUL | boolean | 否 | 漏洞阻断开关，开启后推送镜像含指定等级漏洞会被拒，默认 false | — |
| Severity | string | 否（`IsPreventVUL=true` 时需设置） | 阻断漏洞等级枚举：`low` / `medium` / `high`。设 `high` 仅阻断高危漏洞，`low` 三级全阻断 | `InvalidParameter`（传非枚举值） |
| CVEWhitelistItems | list | 否 | 漏洞白名单列表 | — |

### CreateRepository

> 完整入参以 `tccli tcr CreateRepository help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| NamespaceName | string | 是 | 须已存在的命名空间 | `ResourceNotFound` |
| RepositoryName | string | 是 | API：长度 **大于 2 且小于 245** 字符，仅允许小写字母、数字及 `.`/`_`/`-`，命名空间内唯一 | `InvalidParameter` / `LimitExceeded.Repository` |
| BriefDescription | string | 否 | 仓库简述（`ModifyRepository` 时与 `Description` **同为必填**） | — |
| Description | string | 否 | 仓库详细描述（`ModifyRepository` 时与 `BriefDescription` **同为必填**） | — |

## 操作步骤

### 步骤 1：决策 — 命名空间可见性

#### 为什么选 Public vs Private

- **Private（推荐）**: 需凭证才能 push/pull，适合生产镜像。安全默认
- **Public**: 任何人可 `docker pull`（不可 push），适合开源镜像分发
- **默认推荐**: Private——除非确需公开分发，否则保持私有
- **可修改**： 命名空间可见性可后续 `ModifyNamespace` 修改（`IsPublic`/`IsAutoScan`/`IsPreventVUL` 等），但镜像若已被公开拉取无法撤回

### 步骤 2：创建命名空间

```bash
tccli tcr CreateNamespace --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" --IsPublic false
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<REGISTRY_ID>` | 实例 ID | `tcr-xxxxxxxx` | `tccli tcr DescribeInstances` → `Registries[].RegistryId` |
| `<NAMESPACE_NAME>` | 命名空间名 | **2–30** 字符，小写字母/数字及 `.`/`_`/`-`，实例内唯一 | 自定义，如 `prod`、`team-backend` |

### 步骤 3：创建仓库

```bash
tccli tcr CreateRepository --region <REGION> \
  --RegistryId "<REGISTRY_ID>" \
  --NamespaceName "<NAMESPACE_NAME>" --RepositoryName "<REPOSITORY_NAME>"
# expected: exit 0
```

> 仓库也可在首次 `docker push` 时自动创建（若命名空间允许），但显式创建可控制可见性与描述。

### 步骤 4：查询

```bash
# 命名空间列表（顶层 NamespaceList[]；字段 Name/Public/NamespaceId 等）
tccli tcr DescribeNamespaces --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "NamespaceList[].{name:Name,public:Public}" --output text
# expected: 命名空间名 + 可见性；条数可用 --filter TotalCount

# 仓库列表（顶层 RepositoryList[]；Name 常为 `ns/repo` 全路径，另有 Namespace 字段）
tccli tcr DescribeRepositories --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --NamespaceName "<NAMESPACE_NAME>" --Limit 20 \
  --filter "RepositoryList[].{name:Name,ns:Namespace}" --output text
# expected: 仓库列表；text 多键投影列序按投影 key 名字母序（name 在 ns 前），与书写序无关——勿用 awk 按书写序取列；机器解析用 `--output json`（见 [agent-optimization](../../appendix/agent-optimization.md)）
# expected: 全实例条数 --filter TotalCount（可不传 NamespaceName）
```

## 验证

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 命名空间存在 | `DescribeNamespaces` → `NamespaceList[].Name` | 含目标命名空间 |
| 仓库存在 | `DescribeRepositories` → `RepositoryList[].Name` | 含目标仓库 |
| 可见性 | `DescribeNamespaces` → `Public` | 与创建参数 `IsPublic` 一致 |

### 修改命名空间属性

> 命名空间可见性、自动扫描、漏洞阻断可后续修改。`ModifyNamespace` 覆盖式更新，参数以 `tccli tcr ModifyNamespace help --detail` 为准（`IsPublic`/`IsAutoScan`/`IsPreventVUL` 等）。

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

> **副作用警告**：删除命名空间会级联删除其下所有仓库与镜像版本，不可恢复。删除仓库会删除该仓库所有镜像版本，不可恢复。命名空间创建后不可改名，只能删除重建。

```bash
# 1. 删除仓库（先删仓库再删命名空间）
tccli tcr DeleteRepository --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" --RepositoryName "<REPOSITORY_NAME>"
# expected: exit 0（可选 `--ForceDelete false`：有镜像时先检查；默认 true 直接删）

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
| tccli 本地报 `the following arguments are required: --IsPublic` | 检查命令是否漏 `--IsPublic` | `IsPublic` 为必填布尔 | 显式传 `--IsPublic true` 或 `false` |
| `InvalidParameter.InstanceName` | 检查命名空间格式与长度 | 含非法字符、超长（API 上限 30）或以分隔符开头/结尾/连续 | 用小写字母/数字及 `.`/`_`/`-`，**2–30** 字符 |
| `LimitExceeded.Namespace` | `DescribeNamespaces` 看存量 | 命名空间达上限（basic=50） | 删除闲置命名空间或提工单提额 |
| `LimitExceeded.Repository` | `DescribeRepositories` 看存量 | 仓库达上限（basic=1000） | 删除闲置仓库 |
| `ResourceNotFound` | `DescribeInstances` 核对 ID | RegistryId 错或实例非 Running | 确认实例 ID 与状态 |
| `FailedOperation.ErrorTcrResourceConflict`（消息含 `project named ... already exists`） | `DescribeNamespaces` 看重名 | 命名空间名已存在 | 换名或先删旧命名空间 |
| `FailedOperation` | `DescribeInstanceStatus` 查看状态 | 实例非 Running | 等实例 Running 后重试 |

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

> `ModifyRepository` 用 `NamespaceName`+`RepositoryName` 定位（非 ID）。`BriefDescription` 和 `Description` 均为必填（与 `CreateRepository` 不同，Create 时两者可选）。`DeleteRepositoryTags` 批量删 tag（区别于删整个仓库 `DeleteRepository`）。

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

## 收尾确认

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
# ③ 跨步骤汇总：命名空间 + 仓库 一次性核对（Verify 逐项查，这里汇总两步产物）
tccli tcr DescribeNamespaces --region ap-guangzhou --RegistryId "<REGISTRY_ID>" \
  --filter "NamespaceList[?Name=='<NAMESPACE_NAME>'].{name:Name,public:Public}"
# expected: 含目标命名空间, public 与创建参数一致

tccli tcr DescribeRepositories --region ap-guangzhou --RegistryId "<REGISTRY_ID>" \
  --NamespaceName "<NAMESPACE_NAME>" --Limit 20 \
  --filter "RepositoryList[?Namespace=='<NAMESPACE_NAME>' && (Name=='<REPOSITORY_NAME>' || Name=='<NAMESPACE_NAME>/<REPOSITORY_NAME>')].{name:Name,ns:Namespace}"
# expected: 含目标仓库；`Name` 响应常为 `ns/repo` 全路径，过滤时两种写法都兼容

# ④ 衔接下一步前置：命名空间/仓库已建后，访问端点+Token 就绪即可 docker push（见访问管理/推送篇）
# docker login 属能力边界（非 tccli）；此处不强制执行，仅确认资源就绪
```

> 命名空间存在 + 仓库存在 = 仓库管理闭环完成。推送前再确认访问路径（先内网后公网）与 Token，见 [访问管理](../instances/manage-access.md) / [推送拉取镜像](../images/push-pull.md)。

---

## 下一步

- [推送拉取镜像](../images/push-pull.md) — 命名空间/仓库创建后 push/pull
- [访问控制](../access/manage.md) — 配置谁能 push/pull
- [创建实例](../instances/create.md) — 实例生命周期
- [故障排查](../troubleshooting.md) — `denied` / `not found` 诊断
