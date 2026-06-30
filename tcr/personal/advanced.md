---
doc_type: How-to
subtype: 6A
fused: false
---
# 个人版高级管理

> 个人版仓库查询过滤、镜像复制、生命周期策略、应用触发器。核心 CRUD 见 [个人版全功能](manage.md)。

> 个人版 API 形态与企业版不同——所有 Action 带 `Personal` 后缀。`RepoName` 格式是 `<namespace>/<repo>`（含斜杠）。

## 概述

| 任务 | 接口 | 用途 |
|:-----|:-----|:-----|
| 仓库查询 | `DescribeRepositoryFilterPersonal` 等 | 过滤/收藏/所有者查询 |
| 仓库属性 | `ModifyRepositoryAccessPersonal` 等 | 改可见性/描述/批量删 |
| 镜像复制 | `DuplicateImagePersonal` | 同账号内复制镜像 |
| 生命周期 | `ManageImageLifecycleGlobalPersonal` | 自动清理旧镜像 |
| 应用触发器 | `CreateApplicationTriggerPersonal` | 推送后触发 TKE 部署 |

## 决策依据

### 仓库可见性（Public vs Private）

- **Public (1)**: 公开仓库，任何人可拉取（docker pull 无需凭证）。开源镜像/共享制品
- **Private (0)**: 私有仓库，需凭证拉取。生产/私有制品（默认推荐）
- **决策依据**: 含敏感信息或生产镜像用 Private；公共基础镜像用 Public

### 生命周期策略（全局 vs 仓库级）

- **全局策略**: `ManageImageLifecycleGlobalPersonal`，对所有仓库生效，无 `RepoName`
- **仓库级策略**: `DescribeImageLifecyclePersonal` 查单仓库，粒度更细
- **Type 选择**: `global_keep_last_days`（按天保留）/ `global_keep_last_nums`（按个数保留）
- **决策依据**: 统一清理用全局；个别仓库特殊保留期用仓库级

### 应用触发器（何时用）

- **触发器**: 推送镜像后自动更新 TKE 工作负载，实现 push 即部署
- **何时用**: CI/CD 无独立部署步骤时，用触发器联动 TKE
- **不用触发器**: 有成熟 CI/CD（Jenkins/GitLab CI）时，由 CI/CD 控制部署，避免重复触发

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `RepoName` | 多数 | 是 | `<namespace>/<repo>`（含斜杠） |
| `Namespace` | 部分查询 | 是 | 命名空间名 |
| `Public` | ModifyRepositoryAccessPersonal | 是 | 1 公开 / 0 私有 |
| `SrcImage`/`DestImage` | DuplicateImagePersonal | 是 | `<ns>/<repo>:<tag>` |
| `Type`/`Val` | ManageImageLifecycleGlobalPersonal | 是 | 策略类型/保留值 |
| `TriggerName` | 触发器类 | 是 | 触发器名 |
| `ClusterId`/`WorkloadName` | CreateApplicationTriggerPersonal | 是 | 关联 TKE 集群/工作负载 |

> 参数名实测自各 Action `--generate-cli-skeleton`（P7）。

## 操作步骤

### 步骤 1：仓库查询与校验

```bash
# 校验命名空间是否存在 (建仓前用)
tccli tcr ValidateNamespaceExistPersonal --Namespace "<NAMESPACE_NAME>"
# expected: exit 0, Data.IsExist=true/false
```
```json
{"Data": {"IsExist": false, "IsPreserved": false}, "RequestId": "..."}
```

```bash
# 校验仓库是否存在
tccli tcr ValidateRepositoryExistPersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>"
# expected: exit 0, Data.IsExist

# 按条件过滤仓库 (Public/Namespace 过滤)
tccli tcr DescribeRepositoryFilterPersonal --Namespace "<NAMESPACE_NAME>" --Limit 10 --Offset 0
# expected: exit 0, Data.RepoInfo[] + Server (ccr.ccs.tencentyun.com)

# 查询收藏的仓库
tccli tcr DescribeFavorRepositoryPersonal --Limit 10 --Offset 0
# expected: exit 0, Data.RepoInfo[]

# 查询仓库所有者
tccli tcr DescribeRepositoryOwnerPersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" --Limit 10
# expected: exit 0, Data
```

### 步骤 2：修改仓库属性

```bash
# 修改仓库可见性 (Public: 1 公开 / 0 私有)
tccli tcr ModifyRepositoryAccessPersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" --Public 0
# expected: exit 0

# 修改仓库描述
tccli tcr ModifyRepositoryInfoPersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" --Description "<DESC>"
# expected: exit 0

# 批量删除仓库
tccli tcr BatchDeleteRepositoryPersonal --RepoNames '["<NAMESPACE_NAME>/<REPO_NAME>"]'
# expected: exit 0
```

### 步骤 3：镜像复制与批量删除

```bash
# 复制镜像 (同账号内, SrcImage/DestImage 格式 ns/repo:tag)
tccli tcr DuplicateImagePersonal --SrcImage "<SRC_NS>/<SRC_REPO>:<TAG>" --DestImage "<DEST_NS>/<DEST_REPO>:<TAG>"
# expected: exit 0

# 按标签过滤镜像
tccli tcr DescribeImageFilterPersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" --Tag "<TAG>"
# expected: exit 0, Data

# 批量删除镜像
tccli tcr BatchDeleteImagePersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" --Tags '["<TAG1>","<TAG2>"]'
# expected: exit 0
```

### 步骤 4：生命周期策略

```bash
# 查询全局生命周期策略 (无入参, 账号级)
tccli tcr DescribeImageLifecycleGlobalPersonal
# expected: exit 0, Data.StrategyInfo[] (Type 如 global_keep_last_days)
```
```json
{"Data": {"TotalCount": 2, "StrategyInfo": [{"Username": "1000000000", "RepoName": "", "Type": "global_keep_last_days"}]}}
```

```bash
# 设置全局生命周期 (Type=global_keep_last_days, Val=保留天数)
tccli tcr ManageImageLifecycleGlobalPersonal --Type global_keep_last_days --Val 30
# expected: exit 0

# 删除全局生命周期 (无入参)
tccli tcr DeleteImageLifecycleGlobalPersonal
# expected: exit 0

# 查询仓库级生命周期
tccli tcr DescribeImageLifecyclePersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>"
# expected: exit 0, Data
```

### 步骤 5：应用触发器（推送后触发 TKE 部署）

```bash
# 创建应用触发器 (关联 TKE 集群, 推送镜像后自动更新工作负载)
tccli tcr CreateApplicationTriggerPersonal \
  --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" \
  --TriggerName "<TRIGGER_NAME>" \
  --InvokeMethod "void" \
  --ClusterId "<CLUSTER_ID>" --Namespace "<K8S_NAMESPACE>" \
  --WorkloadType "<WORKLOAD_TYPE>" --WorkloadName "<WORKLOAD_NAME>"
# expected: exit 0
```

> `CreateApplicationTriggerPersonal` 关联 TKE 集群（`ClusterId`/`Namespace`/`WorkloadType`/`WorkloadName`）。

```bash
# 查询触发器
tccli tcr DescribeApplicationTriggerPersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" --Limit 10
# expected: exit 0, Data

# 修改触发器
tccli tcr ModifyApplicationTriggerPersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" --TriggerName "<TRIGGER_NAME>" --InvokeMethod "void"
# expected: exit 0

# 查询触发器执行日志
tccli tcr DescribeApplicationTriggerLogPersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" --Limit 10
# expected: exit 0, Data (含触发记录与状态)

# 删除触发器
tccli tcr DeleteApplicationTriggerPersonal --TriggerName "<TRIGGER_NAME>"
# expected: exit 0
```

## 验证

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 仓库可见性已变 | `DescribeRepositoryFilterPersonal --RepoName <R>` | Public 字段为目标值 |
| 镜像已复制 | `DescribeImageFilterPersonal --RepoName <DEST>` | 含目标 tag |
| 全局策略已设 | `DescribeImageLifecycleGlobalPersonal` | StrategyInfo 含新策略 |
| 触发器已建 | `DescribeApplicationTriggerPersonal --RepoName <R>` | 含目标触发器 |
| 触发器触发记录 | `DescribeApplicationTriggerLogPersonal` | 有推送后的触发日志 |

## 清理

```bash
# 删除触发器
tccli tcr DeleteApplicationTriggerPersonal --TriggerName "<TRIGGER_NAME>"
# expected: exit 0

# 删除全局生命周期 (无入参)
tccli tcr DeleteImageLifecycleGlobalPersonal
# expected: exit 0

# 批量删除镜像
tccli tcr BatchDeleteImagePersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" --Tags '["<TAG>"]'
# expected: exit 0
```

> **副作用警告**：批量删除镜像/仓库不可恢复；全局生命周期策略会自动清理所有仓库的旧镜像。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `DuplicateImagePersonal` 报源镜像不存在 | `SrcImage` 格式错或镜像不存在 | 核对 `<ns>/<repo>:<tag>`，`DescribeImageFilterPersonal` 确认 |
| 全局策略不生效 | `Type` 拼错或 `Val` 非法 | 用 `global_keep_last_days`/`global_keep_last_nums`，Val 为正整数 |
| 触发器创建报集群不存在 | `ClusterId` 错或无权限 | 核对 TKE 集群 ID 与访问权限 |
| 触发器未触发部署 | `InvokeMethod`/工作负载参数错 | 查 `DescribeApplicationTriggerLogPersonal` 看触发记录与错误 |
| 仓库改 Public 后仍拉取失败 | 个人版域名用错 | 个人版用 `ccr.ccs.tencentyun.com` |

## 下一步

- [个人版全功能](manage.md) — 核心 CRUD（用户/命名空间/仓库/推送）
- [推送拉取镜像](../images/push-pull.md) — 企业版推送对比
- [故障排查](../troubleshooting.md) — 触发器/复制失败诊断

## Action 清单

| Action | 类型 | 说明 |
|:-------|:-----|:-----|
| `ValidateNamespaceExistPersonal` | 验证 | 校验命名空间存在 |
| `ValidateRepositoryExistPersonal` | 验证 | 校验仓库存在 |
| `DescribeRepositoryFilterPersonal` | 验证 | 过滤查询仓库 |
| `DescribeFavorRepositoryPersonal` | 验证 | 查收藏仓库 |
| `DescribeRepositoryOwnerPersonal` | 验证 | 查仓库所有者 |
| `ModifyRepositoryAccessPersonal` | 主操作 | 改可见性（Public 1/0） |
| `ModifyRepositoryInfoPersonal` | 主操作 | 改描述 |
| `BatchDeleteRepositoryPersonal` | 清理 | 批量删仓库 |
| `DuplicateImagePersonal` | 主操作 | 复制镜像（同账号） |
| `DescribeImageFilterPersonal` | 验证 | 按标签过滤镜像 |
| `BatchDeleteImagePersonal` | 清理 | 批量删镜像 |
| `DescribeImageLifecycleGlobalPersonal` | 验证 | 查全局生命周期 |
| `ManageImageLifecycleGlobalPersonal` | 主操作 | 设全局生命周期（Type/Val） |
| `DeleteImageLifecycleGlobalPersonal` | 清理 | 删全局生命周期 |
| `DescribeImageLifecyclePersonal` | 验证 | 查仓库级生命周期 |
| `CreateApplicationTriggerPersonal` | 主操作 | 建触发器（关联 TKE） |
| `DescribeApplicationTriggerPersonal` | 验证 | 查触发器 |
| `ModifyApplicationTriggerPersonal` | 主操作 | 改触发器 |
| `DescribeApplicationTriggerLogPersonal` | 验证 | 查触发器日志 |
| `DeleteApplicationTriggerPersonal` | 清理 | 删触发器 |
