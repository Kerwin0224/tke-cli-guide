# 个人版快速入门（tccli）

> 对照官方：[个人版快速入门](https://cloud.tencent.com/document/product/1141/63910) · page_id `63910`

## 概述

容器镜像服务（TCR）个人版是面向个人开发者和小型团队的免费镜像托管服务。本文通过 tccli 完成个人版的命名空间创建、镜像仓库创建，再通过 `docker login` / `docker push` / `docker pull` 完成镜像推送与拉取（数据面）。个人版域名为 `ccr.ccs.tencentyun.com`。

> **注意**：个人版为免费服务，无 SLA 保障。生产环境推荐使用 [企业版](../企业版快速入门/tccli%20操作.md)。

## 前置条件

- [环境准备](../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tcr:CreateNamespacePersonal, tcr:CreateRepositoryPersonal
#    tcr:ValidateNamespaceExistPersonal, tcr:DescribeRepositoryPersonal
# 验证：执行以下命令确认权限
tccli tcr ValidateNamespaceExistPersonal --Namespace test --region ap-guangzhou
# expected: exit 0（即使命名空间不存在也返回正常，仅权限错误才报错）

# 4. 检查 Docker（数据面操作需要）
docker --version
# expected: Docker version 2x.x.x
```

### 资源检查

```bash
# 5. 确认个人版服务已开通（首次使用需在控制台开通）
#    开通入口：https://console.cloud.tencent.com/tcr
#    开通后无需 tccli 额外操作
```

> 个人版无需创建实例，服务开通后即可直接使用。首次开通需在控制台完成。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 开通个人版服务 | 控制台操作（无 API） | — |
| 初始化个人版密码 | 控制台操作 | — |
| 创建命名空间 | `CreateNamespacePersonal` | 否 |
| 验证命名空间是否存在 | `ValidateNamespaceExistPersonal` | 是 |
| 创建镜像仓库 | `CreateRepositoryPersonal` | 否 |
| 查询镜像仓库 | `DescribeRepositoryPersonal` | 是 |
| 删除命名空间 | `DeleteNamespacePersonal` | 是 |
| 删除镜像仓库 | `DeleteRepositoryPersonal` | 是 |

> 推送/拉取镜像为 Docker 数据面操作，不经过 tccli。

## 操作步骤

### 步骤 1：开通个人版服务（控制台）

按官方文档在 [容器镜像服务控制台](https://console.cloud.tencent.com/tcr) 完成个人版开通和初始化。CLI 无需重复操作。

> **预期输出**：控制台显示"初始化完成"，域名 `ccr.ccs.tencentyun.com` 可用。

### 步骤 2：创建命名空间

命名空间是组织镜像仓库的顶层分组，名称需符合规则（小写字母、数字、连字符，长度 1-100）。

```bash
tccli tcr CreateNamespacePersonal \
    --Namespace <NAMESPACE> \
    --region ap-guangzhou
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

验证命名空间已创建：

```bash
tccli tcr ValidateNamespaceExistPersonal \
    --Namespace <NAMESPACE> \
    --region ap-guangzhou
# expected: Data.IsExist = true
```

**预期输出**：

```json
{
    "Data": {
        "IsExist": true
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 示例 |
|--------|------|------|------|
| `NAMESPACE` | 命名空间名称 | 小写字母、数字、连字符，长度 1-100 | `my-namespace` |

### 步骤 3：创建镜像仓库

> 此步骤为可选。也可以跳过仓库创建，直接在 `docker push` 时自动创建仓库。

个人版仓库名格式为 **`{Namespace}/{ImageName}`**（单参数 `RepoName`）：

```bash
tccli tcr CreateRepositoryPersonal \
    --RepoName "<NAMESPACE>/<REPO>" \
    --Public 0 \
    --region ap-guangzhou
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- `--Public 0`：私有仓库（推荐）。`--Public 1` 为公开仓库。
- 创建后可通过 `DescribeRepositoryPersonal` 查询仓库详情。

| 占位符 | 说明 | 约束 | 示例 |
|--------|------|------|------|
| `NAMESPACE` | 命名空间名称 | 同步骤 2 | `my-namespace` |
| `REPO` | 镜像仓库名称 | 小写字母、数字、连字符、下划线 | `nginx` |

### 步骤 4：获取访问凭证并登录

在 [TCR 访问凭证](https://console.cloud.tencent.com/tcr/token) 页面获取用户名和密码（长期访问令牌）后执行：

```bash
docker login ccr.ccs.tencentyun.com \
    --username <TCR_USERNAME> \
    --password <TCR_TOKEN>
# expected: Login Succeeded
```

**预期输出**：

```
Login Succeeded
```

> **注意**：用户名格式为腾讯云 UIN（10000xxxxxxx），密码为长期访问令牌。令牌可在控制台[访问凭证](https://console.cloud.tencent.com/tcr/token)页面生成或重置。

### 步骤 5：推送容器镜像

```bash
# 1. 拉取公共镜像（以 nginx 为例）
docker pull nginx:latest
# expected: Status: Downloaded newer image for nginx:latest

# 2. 打标签为 TCR 个人版地址
docker tag nginx:latest ccr.ccs.tencentyun.com/<NAMESPACE>/<REPO>:latest
# expected: 无输出（成功）

# 3. 推送到 TCR 个人版
docker push ccr.ccs.tencentyun.com/<NAMESPACE>/<REPO>:latest
# expected: push 成功，显示 sha256 摘要
```

**预期输出**：

```
The push refers to repository [ccr.ccs.tencentyun.com/<NAMESPACE>/<REPO>]
latest: digest: sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx size: xxx
```

### 步骤 6：拉取容器镜像

```bash
docker pull ccr.ccs.tencentyun.com/<NAMESPACE>/<REPO>:latest
# expected: Status: Downloaded newer image
```

**预期输出**：

```
latest: Pulling from <NAMESPACE>/<REPO>
Digest: sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Status: Downloaded newer image for ccr.ccs.tencentyun.com/<NAMESPACE>/<REPO>:latest
```

## 验证

### 控制面（tccli）

```bash
# 1. 验证命名空间存在
tccli tcr ValidateNamespaceExistPersonal \
    --Namespace <NAMESPACE> \
    --region ap-guangzhou
# expected: Data.IsExist = true

# 2. 查询镜像仓库
tccli tcr DescribeRepositoryPersonal \
    --RepoName "<NAMESPACE>/<REPO>" \
    --region ap-guangzhou
# expected: 返回仓库信息，含 ImageCount
```

**预期输出**：

```json
{
    "Data": {
        "Namespace": "<NAMESPACE>",
        "RepoName": "<NAMESPACE>/<REPO>",
        "Public": 0,
        "CreationTime": "2024-01-01 00:00:00",
        "ImageCount": 1
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 数据面（Docker）

```bash
# 3. 验证本地镜像存在
docker images ccr.ccs.tencentyun.com/<NAMESPACE>/<REPO>
# expected: 显示镜像，含 TAG 为 latest

# 4. 验证可重新拉取
docker rmi ccr.ccs.tencentyun.com/<NAMESPACE>/<REPO>:latest
docker pull ccr.ccs.tencentyun.com/<NAMESPACE>/<REPO>:latest
# expected: Pull 成功
```

## 清理

> **警告**：删除仓库将同时删除仓库内所有镜像 Tag，不可恢复。删除前确认镜像已无业务依赖。

### 数据面清理（Docker）

```bash
# 1. 删除本地镜像（释放磁盘空间）
docker rmi ccr.ccs.tencentyun.com/<NAMESPACE>/<REPO>:latest
# expected: Untagged: ..., Deleted: ...
```

### 控制面清理（tccli）

```bash
# 2. 删除镜像仓库（可选，删除仓库及所有镜像）
tccli tcr DeleteRepositoryPersonal \
    --RepoName "<NAMESPACE>/<REPO>" \
    --region ap-guangzhou
# ⚠️ 将删除仓库及所有镜像 Tag，不可恢复

# 3. 验证仓库已删除
tccli tcr DescribeRepositoryPersonal \
    --RepoName "<NAMESPACE>/<REPO>" \
    --region ap-guangzhou
# expected: 返回空数据或资源不存在

# 4. 删除命名空间（需先清空命名空间下所有仓库）
tccli tcr DeleteNamespacePersonal \
    --Namespace <NAMESPACE> \
    --region ap-guangzhou
# expected: exit 0，返回 RequestId

# 5. 验证命名空间已删除
tccli tcr ValidateNamespaceExistPersonal \
    --Namespace <NAMESPACE> \
    --region ap-guangzhou
# expected: Data.IsExist = false
```

```json
{
  "Data": "<Data>",
  "RepoName": "<RepoName>",
  "RepoType": "<RepoType>",
  "Server": "<Server>",
  "CreationTime": "<CreationTime>",
  "Description": "<Description>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateNamespacePersonal` 返回 `InvalidParameter` | 检查 `--Namespace` 参数格式 | 命名空间名称包含非法字符（大写字母、特殊字符）或长度超限 | 使用小写字母、数字、连字符，长度 1-100 |
| `CreateNamespacePersonal` 返回 `NamespaceAlreadyExists` | 尝试不同命名空间名 | 命名空间已存在 | 更换命名空间名称，或使用 `ValidateNamespaceExistPersonal` 确认 |
| `CreateRepositoryPersonal` 返回 `InvalidParameter.RepoName` | 检查 `--RepoName` 格式 | 仓库名格式错误，必须是 `namespace/repo` | 使用格式 `<NAMESPACE>/<REPO>`，不要使用单独的 `--Namespace` 参数 |
| `docker login` 返回 `unauthorized` | 重新执行 `docker login` 确认凭据 | TCR 访问令牌过期或密码错误 | 在 [TCR 控制台](https://console.cloud.tencent.com/tcr/token) 重新生成访问令牌 |
| `docker push` 返回 `denied` | `docker images` 确认标签；`docker login` 确认登录状态 | 未登录、命名空间不存在、或无推送权限 | 确认已 `docker login`；检查 `RepoName` 中的命名空间与已创建的命名空间一致 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `docker pull` 从个人版拉取速度慢 | `docker pull` 观察下载速度 | 个人版无带宽保障，公网下载速度受限 | 使用企业版获得更好的下载性能；或就近选择地域 |
| `DescribeRepositoryPersonal` 返回空（仓库不存在） | 确认 `RepoName` 拼写 | 仓库未创建或在其他命名空间下 | 确认命名空间和仓库名拼写；使用 `DescribeRepositoryFilterPersonal` 模糊搜索 |
| `DeleteNamespacePersonal` 提示命名空间非空 | `tccli tcr DescribeRepositoryFilterPersonal --region ap-guangzhou \| grep <NAMESPACE>` | 命名空间下仍有镜像仓库 | 先删除命名空间下所有仓库，再删除命名空间 |

## 下一步

- [企业版快速入门](../企业版快速入门/tccli%20操作.md) -- page_id `39287`（生产环境推荐）
- [创建简单的 Nginx 服务](../入门示例/创建简单的%20Nginx%20服务/tccli%20操作.md) -- page_id `7851`（在 TKE 集群中使用 TCR 镜像部署应用）
- [TCR 个人版操作指南](https://cloud.tencent.com/document/product/1141/63911)

## 控制台替代

[容器镜像服务控制台 → 个人版](https://console.cloud.tencent.com/tcr/repository)：创建命名空间与仓库，在「访问凭证」获取登录命令。
