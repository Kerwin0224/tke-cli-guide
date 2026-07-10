---
doc_type: How-to
subtype: 6A
fused: false
---
# 个人版全功能操作

> 控制台：企业版侧栏无独立「个人版」管理页（`/tcr/personal` 当前不可用）。个人版规格见 [购买页对比](https://buy.cloud.tencent.com/tcr)；公开镜像浏览见 [公有镜像](https://console.cloud.tencent.com/tcr/publicimage)（域名 `ccr.ccs.tencentyun.com`）。日常管理以本页 API / docker 为准。
> TCR 个人版的用户、命名空间、仓库管理与镜像推送。个人版 API 形态与企业版完全不同——所有 Action 带 `Personal` 后缀，全局服务不传 `--region`。

## 触发条件

- `tccli tcr DescribeUserQuotaPersonal` 返回 `Data.LimitInfo` 显示个人版配额（无个人版用户时 `CreateUserPersonal` 报 `ResourceNotFound.ErrNoUser`）
- `docker login ccr.ccs.tencentyun.com` 报 `unauthorized`（个人版用户未创建或密码错），或 `DescribeNamespacePersonal` 返回 `Data.NamespaceCount: 0`（无命名空间无法 push）
- 个人开发/小团队场景，企业版实例成本过高，`DescribeInstances` 返回 `TotalCount: 0`


## 概述

个人版操作链路：创建用户（docker login 凭证）→ 创建命名空间 → 创建仓库 → docker 推送/拉取。

| 操作 | 接口 | 作用 |
|:-----|:-----|:-----|
| 创建用户 | `CreateUserPersonal` | docker login 的账号密码 |
| 创建命名空间 | `CreateNamespacePersonal` | 仓库分组 |
| 创建仓库 | `CreateRepositoryPersonal` | 镜像存放单元 |
| 查询命名空间 | `DescribeNamespacePersonal` | 看已有命名空间 |
| 推送/拉取 | docker CLI | 镜像传输 |

> 个人版无实例概念，所有操作直接在账号下的个人版空间进行。配额（不可调额）：命名空间 **10**/地域；仓库 **广州 500 / 其他地域 100**；单镜像 Tag **100**。不够则升企业版。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

docker --version
# expected: Docker version 20+
```

### 资源检查

```bash
# 查看个人版配额（Data.LimitInfo[]）
tccli tcr DescribeUserQuotaPersonal
# expected: Data.LimitInfo 含 namespace/repo 配额

# 查看已有命名空间（必填 Namespace/Limit/Offset）
tccli tcr DescribeNamespacePersonal --Namespace "" --Limit 10 --Offset 0
# expected: Data.NamespaceInfo（数组）+ Data.NamespaceCount
```

## 关键字段

> 完整入参以各 Action 的 `help --detail` 为准。个人版 Action 入参极简。

### CreateUserPersonal

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| Password | string | 是 | docker login 密码，8-32 字符 | `InvalidParameter` |

> `CreateUserPersonal` 仅需 `Password`。用户名由系统分配（腾讯云账号 ID）。创建后用该用户名 + 密码 docker login。

### CreateNamespacePersonal

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| Namespace | string | 是 | 小写字母/数字/`-`，账号内唯一 | `InvalidParameter` / `LimitExceeded` |

### CreateRepositoryPersonal

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| Namespace | string | 是 | 须已存在的命名空间 | `ResourceNotFound` |
| RepoName | string | 是 | 仓库名，命名空间内唯一 | `InvalidParameter` |

> 个人版 `CreateRepositoryPersonal` 入参是 `RepoName`（非 `RepositoryName`）。与企业版 `CreateRepository` 的字段名不同。

## 操作步骤

### 步骤 1：创建个人版用户

```bash
tccli tcr CreateUserPersonal --Password "<PASSWORD>"
# expected: exit 0, 返回用户名（系统分配）
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<PASSWORD>` | docker login 密码 | 8-32 字符 | 自定义 |

> 用户名由系统分配（腾讯云账号 ID）。若已创建过用户，再次 `CreateUserPersonal` 会报错——用 `ModifyUserPasswordPersonal` 改密。

```bash
# 修改个人版用户密码（仅 Password，用户名系统分配不可改）
tccli tcr ModifyUserPasswordPersonal --Password "<NEW_PASSWORD>"
# expected: exit 0; 账号未初始化个人版用户报 ResourceNotFound.ErrNoUser
```

### 步骤 2：创建命名空间

```bash
tccli tcr CreateNamespacePersonal --Namespace "<NAMESPACE_NAME>"
# expected: exit 0
```

> 个人版命名空间名直接用于镜像地址：`<个人版域名>/<namespace>/<repo>:<tag>`。个人版域名是 `ccr.ccs.tencentyun.com`（与企业版的 `xxx.tencentcloudcr.com` 不同）。

### 步骤 3：创建仓库

```bash
tccli tcr CreateRepositoryPersonal --Namespace "<NAMESPACE_NAME>" --RepoName "<REPO_NAME>"
# expected: exit 0
```

### 步骤 4：docker login + 推送

```bash
# 登录个人版（域名 ccr.ccs.tencentyun.com）
docker login ccr.ccs.tencentyun.com -u "<USERNAME>" -p "<PASSWORD>"
# expected: Login Succeeded

# 打标签并推送
docker tag alpine:latest ccr.ccs.tencentyun.com/<NAMESPACE_NAME>/<REPO_NAME>:v1
docker push ccr.ccs.tencentyun.com/<NAMESPACE_NAME>/<REPO_NAME>:v1
# expected: digest: sha256:...
```

> 个人版域名是 `ccr.ccs.tencentyun.com`（固定），与企业版每实例独立域名不同。

## 高级管理

> 仓库查询过滤、镜像复制、生命周期策略、应用触发器属进阶操作，见 [个人版高级管理](advanced.md)。

## 验证

```bash
# 查命名空间
tccli tcr DescribeNamespacePersonal --Namespace "<NAMESPACE_NAME>" --Limit 10 --Offset 0
# expected: Data.NamespaceInfo 含目标命名空间

# 查镜像版本（RepoName 须为 "namespace/repo"；无 --Namespace，传了报 Unknown options）
tccli tcr DescribeImagePersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>" --Limit 10 --Offset 0
# expected: 含刚推送的 tag；仓库不存在报 ResourceNotFound.ErrNoRepo

# 查仓库详情（RepoName 须为 "namespace/repo" 格式，无 Limit/Offset 参数）
tccli tcr DescribeRepositoryPersonal --RepoName "<NAMESPACE_NAME>/<REPO_NAME>"
# expected: exit 0, 返回 Data 对象含仓库属性；仓库不存在报 ResourceNotFound.ErrNoRepo
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 命名空间存在 | `DescribeNamespacePersonal` | 含目标命名空间 |
| 仓库存在 | `DescribeRepositoryPersonal` | 含目标仓库 |
| 镜像版本 | `DescribeImagePersonal --RepoName ns/repo` | 含推送的 tag |
| docker 本地 | `docker images ccr.ccs.tencentyun.com/<ns>/<repo>` | 含推送的 tag |

## 清理

> **副作用警告**：删除命名空间会级联删除其下所有仓库与镜像，不可恢复。

```bash
# 1. 删除镜像
tccli tcr DeleteImagePersonal --Namespace "<NAMESPACE_NAME>" --RepoName "<REPO_NAME>" --Tag "<TAG>"
# expected: exit 0

# 2. 删除仓库
tccli tcr DeleteRepositoryPersonal --Namespace "<NAMESPACE_NAME>" --RepoName "<REPO_NAME>"
# expected: exit 0

# 3. 删除命名空间（须无仓库）
tccli tcr DeleteNamespacePersonal --Namespace "<NAMESPACE_NAME>"
# expected: exit 0

# 4. 验证已删
tccli tcr DescribeNamespacePersonal --Namespace "<NAMESPACE_NAME>" --Limit 10 --Offset 0
# expected: Data.NamespaceInfo 不含目标
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `LimitExceeded` | `DescribeUserQuotaPersonal` 看配额 | 命名空间/仓库达上限 | 删除闲置资源或升级企业版 |
| `InvalidParameter` (Password) | 检查密码长度 | 密码 <8 或 >32 字符 | 用 8-32 字符密码 |
| `ResourceNotFound` (Namespace) | `DescribeNamespacePersonal` 核对 | 命名空间不存在 | 先 `CreateNamespacePersonal` |
| `ResourceInUse` | 查命名空间下仓库 | 删除命名空间时仍有仓库 | 先删所有仓库再删命名空间 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| docker login 失败 `unauthorized` | 核对用户名密码 | 密码错或未创建用户 | `CreateUserPersonal` 或 `ModifyUserPasswordPersonal` |
| docker push 报 `denied` | `DescribeRepositoryPersonal` 看权限 | 仓库私有且无权限 | 个人版默认创建者有权限，核对账号 |
| 个人版域名连不上 | `docker login ccr.ccs.tencentyun.com` | 用错域名（用了企业版域名） | 个人版用 `ccr.ccs.tencentyun.com` |

## 收尾确认

```bash
# ② 业务可用性端到端：docker login + push 真正可达（Verify 查 TCR 侧记录存在，这里查端到端镜像传输成功）
docker login ccr.ccs.tencentyun.com -u "<USERNAME>" -p "<PASSWORD>"
# expected: Login Succeeded

docker push ccr.ccs.tencentyun.com/<NAMESPACE_NAME>/<REPO_NAME>:v1
# expected: digest: sha256:... Push complete

# ④ 衔接下一步前置：镜像在个人版可被拉取（进高级管理/触发器前须确认镜像可用）
docker pull ccr.ccs.tencentyun.com/<NAMESPACE_NAME>/<REPO_NAME>:v1
# expected: Pull complete / Status: Image is up to date
```

> docker login 成功 + push 成功 + pull 成功 = 个人版管理闭环完成，可进入高级管理（触发器/复制）。push 失败说明用户/命名空间/仓库配置有缺。

---

## 下一步

- [个人版概览](index.md) — 个人版与企业版差异
- [个人版高级管理](advanced.md) — 仓库查询/镜像复制/生命周期/触发器
- [TCR 企业版](../instances/index.md) — 升级路径
- [推送拉取镜像](../images/push-pull.md) — 企业版推送（对比）
- [故障排查](../troubleshooting.md) — docker login/push 失败诊断
