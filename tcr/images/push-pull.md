---
doc_type: How-to
subtype: 6A
fused: true
---
# 推送和拉取镜像

> 用 tccli 获取访问凭证、docker CLI 推送/拉取镜像、tccli 验证。跨工具操作——tccli 管 TCR 侧，docker 管镜像传输。

## 概述

完整流程：tccli 取 Token → docker login → docker tag/push 或 pull → tccli DescribeImages 验证。

| 步骤 | 工具 | 作用 |
|:-----|:-----|:-----|
| 取访问凭证 | tccli | `CreateInstanceToken` 拿 Username + Token |
| 登录仓库 | docker | `docker login` 用凭证建立会话 |
| 推送镜像 | docker | `docker tag` + `docker push` |
| 拉取镜像 | docker | `docker pull` |
| 验证 | tccli | `DescribeImages` 确认镜像版本存在 |

> 镜像地址格式：`<REGISTRY_DOMAIN>/<NAMESPACE>/<REPO>:<TAG>`，其中 `REGISTRY_DOMAIN` 是实例的 `PublicDomain`（如 `xxx.tencentcloudcr.com`）。

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
# 1. 实例 Running
tccli tcr DescribeInstanceStatus --region <REGION> --RegistryIds '["<REGISTRY_ID>"]' \
  --filter "RegistryStatusSet[0].Status"
# expected: "Running"

# 2. 公网访问已开启（push/pull 前提）
tccli tcr DescribeExternalEndpointStatus --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "Status"
# expected: "Opened"（未开启见 [访问管理](../instances/manage-access.md)）

# 3. 命名空间与仓库存在
tccli tcr DescribeNamespaces --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "NamespaceList[].Name"
# expected: 含目标命名空间
```

## 关键字段

### tccli: CreateInstanceToken

> 来源：`tccli tcr CreateInstanceToken --generate-cli-skeleton` + 响应。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| TokenType | string | 否 | `temp`（默认，临时 1 小时）/ `longterm`（长期，CI/CD）；
| Desc | string | 否 | 凭证描述 | — |

> 响应字段：`Username`（docker login 用户名）、`Token`（docker login 密码）、`ExpTime`（过期时间戳）、`TokenId`。临时 Token 1 小时过期，CI/CD 用长期凭证见 [访问控制](../access/manage.md)。

### docker: login / tag / push / pull

| 命令 | 作用 | 关键参数 |
|:-----|:-----|:---------|
| `docker login` | 登录仓库 | `<REGISTRY_DOMAIN>` -u `<Username>` -p `<Token>` |
| `docker tag` | 给镜像打 TCR 地址标签 | 源镜像 + `<domain>/<ns>/<repo>:<tag>` |
| `docker push` | 推送镜像 | `<domain>/<ns>/<repo>:<tag>` |
| `docker pull` | 拉取镜像 | `<domain>/<ns>/<repo>:<tag>` |

## 操作步骤

### 步骤 1：取访问凭证

```bash
tccli tcr CreateInstanceToken --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --TokenType temp --Desc "push-pull"
# expected: 返回 Username + Token + ExpTime
```

```json
{
    "Username": "100049208872",
    "Token": "eyJhbGciOiJSUzI1NiIsImtpZCI6...",
    "ExpTime": 1782479695904,
    "TokenId": "",
    "RequestId": "xxx"
}
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<REGISTRY_ID>` | 实例 ID | `tcr-xxxxxxxx` | `tccli tcr DescribeInstances` → `Registries[].RegistryId` |
| `<REGISTRY_DOMAIN>` | 实例访问域名 | `xxx.tencentcloudcr.com` | `DescribeInstances` → `Registries[].PublicDomain` |
| `<NAMESPACE_NAME>` | 命名空间 | 须已存在 | `tccli tcr DescribeNamespaces` |
| `<REPOSITORY_NAME>` | 仓库名 | 须已存在 | `tccli tcr DescribeRepositories` |

> 凭证 1 小时过期（`ExpTime` 字段）。过期后 `docker push` 报 `unauthorized`，需重新 `CreateInstanceToken`。

### 步骤 2：docker login

```bash
docker login <REGISTRY_DOMAIN> -u "<USERNAME>" -p "<TOKEN>"
# expected: Login Succeeded
```

> ⚠️ 凭证不应明文出现在脚本中。用环境变量或 docker credential store：`docker login <REGISTRY_DOMAIN> -u "$TCR_USER" -p "$TCR_TOKEN"`。

### 步骤 3：push — 最小化

```bash
# 打标签
docker tag alpine:latest <REGISTRY_DOMAIN>/<NAMESPACE_NAME>/<REPOSITORY_NAME>:v1

# 推送
docker push <REGISTRY_DOMAIN>/<NAMESPACE_NAME>/<REPOSITORY_NAME>:v1
# expected: digest: sha256:... 推送成功
```

### 步骤 4：push — 增强：多架构镜像

```bash
# 用 buildx 构建多架构镜像后推送
docker buildx build --platform linux/amd64,linux/arm64 \
  -t <REGISTRY_DOMAIN>/<NAMESPACE_NAME>/<REPOSITORY_NAME>:v1 --push .
# expected: 推送多架构 manifest
```

### 步骤 5：pull

```bash
docker pull <REGISTRY_DOMAIN>/<NAMESPACE_NAME>/<REPOSITORY_NAME>:v1
# expected: Pull complete
```

### 步骤 6：验证

```bash
# tccli 侧验证镜像版本已上传
tccli tcr DescribeImages --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" \
  --RepositoryName "<REPOSITORY_NAME>" \
  --filter "ImageInfoList[].{tag:ImageVersion,digest:Digest,size:Size}"
# expected: 含刚推送的 tag
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 镜像版本存在 | `DescribeImages` → `ImageInfoList[].ImageVersion` | 含推送的 tag |
| digest 一致 | `DescribeImages` → `Digest` | 与 docker push 返回的 digest 一致 |
| docker 本地镜像 | `docker images <REGISTRY_DOMAIN>/<NS>/<REPO>` | 含拉取/推送的 tag |
| 公网可达 | `DescribeExternalEndpointStatus` → `Status` | `Opened` |

> ⚠️ push 后立即 `DescribeImages` 可能返回空（服务端索引延迟约 5 秒）。若空，等 5 秒重查。

## 清理

> **副作用警告**：`DeleteImage` 删除指定镜像版本，不可恢复。`docker rmi` 只删本地镜像，不影响 TCR 侧。

```bash
# 1. TCR 侧删除镜像版本
tccli tcr DeleteImage --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" \
  --RepositoryName "<REPOSITORY_NAME>" --ImageVersion "<TAG>"
# expected: exit 0

# 2. 本地清理
docker rmi <REGISTRY_DOMAIN>/<NAMESPACE_NAME>/<REPOSITORY_NAME>:<TAG>
# expected: Untagged + Deleted

# 3. 验证 TCR 侧已删
tccli tcr DescribeImages --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" \
  --RepositoryName "<REPOSITORY_NAME>" --ImageVersion "<TAG>"
# expected: ImageInfoList 为空
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `InvalidParameter` (CreateInstanceToken) | 检查 `TokenType` | `TokenType` 非法（如 `long`/`permanent`） | 用 `temp` 或 `longterm`（
| `unauthorized: authentication required` (docker) | `DescribeInstanceToken` 查状态 | Token 过期或未 login | 重新 `CreateInstanceToken` + `docker login` |
| `denied: requested access to the resource is denied` (docker push) | `DescribeNamespaces` 查权限 | 命名空间 Private 且 Token 无 push 权限 | 配置访问策略，见 [访问控制](../access/manage.md) |
| `unknown: repository not found` (docker push) | `DescribeRepositories` 查仓库 | 仓库不存在 | 先 `CreateRepository` |
| `unknown: repository not found` 或 `project not found` (docker push) | `DescribeNamespaces` 查命名空间 | 命名空间不存在 | 先 [创建命名空间](../repositories/manage.md)，命名空间不存在时 push 报 project not found |
| `ResourceNotFound` (tccli) | 核对 RegistryId/命名空间/仓库 | ID 或名称错 | 确认参数值 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `docker push` 成功但 `DescribeImages` 返回空 | 等 5 秒重查 | 服务端索引延迟 | 等待 5 秒后重新 `DescribeImages` |
| `docker pull` 超时 | `DescribeExternalEndpointStatus` 看公网状态 | 公网访问未开启或网络不通 | 开启公网端点或用 VPC 内网，见 [访问管理](../instances/manage-access.md) |
| `docker login` 成功但 push 报 `denied` | `DescribeNamespaces` → `Public` | 命名空间可见性或 Token 权限不足 | Private 命名空间需 Token 有 push 权限 |
| 多架构 push 部分架构缺失 | `docker manifest inspect <domain>/<ns>/<repo>:<tag>` | buildx 未推某架构 | 重新 `buildx build --platform` 指定缺失架构 |

> docker 侧错误不是 JSON，是 stderr 文本，天然英文。tccli 侧错误用 `--language en-US` 锁定英文便于脚本匹配。

## 镜像 Manifest 与复制

> 查询镜像 manifest（多架构/层信息）、同实例内镜像复制。

```bash
# 查询镜像 manifest (需命名空间/仓库/版本)
tccli tcr DescribeImageManifests --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE>" \
  --RepositoryName "<REPO>" --ImageVersion "<TAG>" --region <REGION>
# expected: exit 0, 镜像 manifest (架构/层/摘要)
```

> ⚠️ `NamespaceName` 必须是实例中已存在的命名空间。传不存在命名空间返回 `InternalError.ErrorTcrUnauthorized: project not found: <namespace>`。先用 `DescribeNamespaces` 确认命名空间存在。

```bash
# 复制镜像 (同实例内, SourceRepo+DestinationRepo)
tccli tcr DuplicateImage --RegistryId "<REGISTRY_ID>" --region <REGION> \
  --SourceNamespace "<SRC_NS>" --SourceRepo "<SRC_REPO>" --SourceReference "<SRC_TAG>" \
  --DestinationNamespace "<DEST_NS>" --DestinationRepo "<DEST_REPO>" --DestinationTag "<DEST_TAG>"
# expected: exit 0
```

> `DuplicateImage` 是同实例内镜像复制（跨命名空间/仓库），区别于 [实例同步](../replication/manage.md)（跨实例/跨地域）。`SourceReference`/`DestinationTag` 是镜像 tag。

## 下一步

- [管理命名空间和仓库](../repositories/manage.md) — push 前创建命名空间/仓库
- [访问控制](../access/manage.md) — `denied` 时的权限配置 + 长期凭证
- [访问管理](../instances/manage-access.md) — 公网/VPC 端点开启
- [实例状态机](../reference/states.md) — push 前确认实例 `Running`
- [故障排查](../troubleshooting.md) — docker login/push 失败诊断

## 控制台替代方案

[容器镜像服务控制台 - 镜像管理](https://console.cloud.tencent.com/tcr/image)
