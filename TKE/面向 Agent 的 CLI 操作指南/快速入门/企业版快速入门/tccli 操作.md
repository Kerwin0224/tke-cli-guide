# 企业版快速入门（tccli）

> 对照官方：[企业版快速入门](https://cloud.tencent.com/document/product/1141/39287) · page_id `39287`

## 概述

容器镜像服务（TCR）企业版提供 SLA 保障的生产级镜像托管能力，支持独享实例、镜像安全扫描、跨地域同步等高级功能。本文通过 tccli 完成企业版实例创建、网络访问策略配置、命名空间和镜像仓库创建，再通过 `docker login` / `docker push` / `docker pull` 完成镜像推送与拉取（数据面）。

> **注意**：企业版实例创建会产生费用。按量计费实例（`basic` 规格）每小时约 0.05 元，试用完成后请及时清理（见[清理](#清理)节）。

## 前置条件

- [环境准备](../../环境准备.md)
- 已注册腾讯云账号并完成实名认证
- 了解企业版[产品计费](https://cloud.tencent.com/document/product/1141/34951)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tcr:CreateInstance, tcr:DescribeInstances, tcr:DeleteInstance
#    tcr:CreateNamespace, tcr:CreateRepository
#    tcr:CreateSecurityPolicy, tcr:DescribeSecurityPolicies
#    tcr:CreateInstanceToken, tcr:DescribeInstanceTokens
# 验证：执行 DescribeInstances 确认权限
tccli tcr DescribeInstances --region ap-guangzhou
# expected: exit 0，返回实例列表（可为空）

# 4. 检查 Docker（数据面操作需要）
docker --version
# expected: Docker version 2x.x.x
```

```json
{
  "TotalCount": "<TotalCount>",
  "Registries": "<Registries>",
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>"
}
```

### 资源检查

```bash
# 5. 查询已有企业版实例（确认是否需要新建）
tccli tcr DescribeInstances --region ap-guangzhou
# expected: 返回实例列表（可为空）

# 6. 检查是否已有可用实例（如需使用已有实例）
tccli tcr DescribeInstances --region ap-guangzhou \
    --RegistryIds '["<RegistryId>"]'
# expected: Status 为 Running
```

```json
{
  "TotalCount": "<TotalCount>",
  "Registries": "<Registries>",
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 购买企业版实例 | `CreateInstance` | 否 |
| 查看实例列表 | `DescribeInstances` | 是 |
| 查看实例详情 | `DescribeInstances --RegistryIds` | 是 |
| 配置公网访问白名单 | `CreateSecurityPolicy` | 否 |
| 查看访问策略 | `DescribeSecurityPolicies` | 是 |
| 创建命名空间 | `CreateNamespace` | 否 |
| 创建镜像仓库 | `CreateRepository` | 否 |
| 创建访问令牌 | `CreateInstanceToken` | 否 |
| 删除实例 | `DeleteInstance` | 是 |

> 推送/拉取镜像为 Docker 数据面操作，不经过 tccli。

## 操作步骤

### 步骤 1：购买企业版实例

#### 选择依据

- **RegistryType**：选择 `basic`（基础版），适合开发测试。生产环境可选 `standard`（标准版）或 `premium`（高级版）。
- **地域**：选择与 TKE 集群同地域（如 `ap-guangzhou`），减少镜像拉取延迟和跨地域流量费。
- **计费模式**：`basic` 按量计费，每小时约 0.05 元。

```bash
tccli tcr CreateInstance \
    --RegistryName <INSTANCE_NAME> \
    --RegistryType basic \
    --region ap-guangzhou
# expected: exit 0，返回 RegistryId
```

**预期输出**：

```json
{
    "RegistryId": "tcr-xxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

等待实例状态变为 `Running`：

```bash
tccli tcr DescribeInstances --region ap-guangzhou \
    --RegistryIds '["<RegistryId>"]'
# expected: Status 为 Running
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Registries": [
        {
            "RegistryId": "tcr-xxxxxxxx",
            "RegistryName": "<INSTANCE_NAME>",
            "RegistryType": "basic",
            "Status": "Running",
            "PublicDomain": "<instance-name>.tencentcloudcr.com",
            "CreatedAt": "2024-01-01T00:00:00Z"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

记录 `PublicDomain`（如 `<instance-name>.tencentcloudcr.com`），后续 `docker login` 使用。

| 占位符 | 说明 | 约束 | 示例 |
|--------|------|------|------|
| `INSTANCE_NAME` | 实例名称 | 全局唯一，长度 1-100 | `my-tcr-enterprise` |
| `REGISTRY_ID` | 实例 ID | 格式 `tcr-xxxxxxxx` | `tcr-xxxxxxxx` |

### 步骤 2：配置网络访问策略

> **安全警告**：以下示例使用 `0.0.0.0/0` 开放全部公网访问，仅供测试。生产环境请限制为具体 IP 或 IP 段。

创建公网访问白名单：

```bash
tccli tcr CreateSecurityPolicy \
    --RegistryId <RegistryId> \
    --CidrBlock 0.0.0.0/0 \
    --Description "test-allow-all" \
    --region ap-guangzhou
# expected: exit 0，返回 RequestId
```

查看已有策略：

```bash
tccli tcr DescribeSecurityPolicies \
    --RegistryId <RegistryId> \
    --region ap-guangzhou
# expected: 返回 SecurityPolicySet，含新创建的策略
```

**预期输出**：

```json
{
    "SecurityPolicySet": [
        {
            "CidrBlock": "0.0.0.0/0",
            "Description": "test-allow-all",
            "PolicyIndex": 1,
            "PolicyVersion": "1"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 3：创建命名空间

```bash
tccli tcr CreateNamespace \
    --RegistryId <RegistryId> \
    --NamespaceName <NAMESPACE> \
    --IsPublic false \
    --region ap-guangzhou
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- `--IsPublic false`：私有命名空间（推荐）。`--IsPublic true` 为公开，可被匿名拉取。

| 占位符 | 说明 | 约束 | 示例 |
|--------|------|------|------|
| `NAMESPACE` | 命名空间名称 | 小写字母、数字、连字符，长度 1-100 | `my-namespace` |

### 步骤 4：创建镜像仓库

```bash
tccli tcr CreateRepository \
    --RegistryId <RegistryId> \
    --Namespace <NAMESPACE> \
    --Repository <REPO> \
    --BriefDescription "my first enterprise repo" \
    --region ap-guangzhou
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 示例 |
|--------|------|------|------|
| `REPO` | 仓库名称 | 小写字母、数字、连字符、下划线 | `nginx` |

### 步骤 5：获取访问令牌

```bash
tccli tcr CreateInstanceToken \
    --RegistryId <RegistryId> \
    --TokenType longterm \
    --Desc "cli-quickstart-token" \
    --region ap-guangzhou
# expected: exit 0，返回 Username 和 Token（密码）
```

**预期输出**：

```json
{
    "Username": "700001234567",
    "Token": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **重要**：`Token` 仅在创建时返回一次，之后不可查询。请立即记录并妥善保管。

### 步骤 6：登录 Registry 实例

```bash
docker login <PublicDomain> \
    --username <USERNAME> \
    --password <TOKEN>
# expected: Login Succeeded
```

**预期输出**：

```
Login Succeeded
```

- `PublicDomain` 在步骤 1 的 `DescribeInstances` 输出中获取（如 `<instance-name>.tencentcloudcr.com`）
- `USERNAME` 和 `TOKEN` 来自步骤 5 的 `CreateInstanceToken` 输出

### 步骤 7：推送容器镜像

```bash
# 1. 拉取公共镜像（以 nginx 为例）
docker pull nginx:latest
# expected: Status: Downloaded newer image for nginx:latest

# 2. 打标签为企业版地址
docker tag nginx:latest <PublicDomain>/<NAMESPACE>/<REPO>:latest
# expected: 无输出（成功）

# 3. 推送到企业版实例
docker push <PublicDomain>/<NAMESPACE>/<REPO>:latest
# expected: push 成功，显示 sha256 摘要
```

**预期输出**：

```
The push refers to repository [<PublicDomain>/<NAMESPACE>/<REPO>]
latest: digest: sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx size: xxx
```

### 步骤 8：拉取容器镜像

```bash
docker pull <PublicDomain>/<NAMESPACE>/<REPO>:latest
# expected: Status: Downloaded newer image
```

**预期输出**：

```
latest: Pulling from <NAMESPACE>/<REPO>
Digest: sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Status: Downloaded newer image for <PublicDomain>/<NAMESPACE>/<REPO>:latest
```

## 验证

### 控制面（tccli）

```bash
# 1. 验证实例状态
tccli tcr DescribeInstances --region ap-guangzhou \
    --RegistryIds '["<RegistryId>"]'
# expected: Status 为 Running，PublicDomain 非空

# 2. 验证命名空间存在
tccli tcr DescribeNamespaces \
    --RegistryId <RegistryId> \
    --region ap-guangzhou
# expected: 返回 NamespaceSet，含已创建的命名空间

# 3. 验证仓库存在
tccli tcr DescribeRepositories \
    --RegistryId <RegistryId> \
    --Namespace <NAMESPACE> \
    --region ap-guangzhou
# expected: 返回 RepositorySet，含已创建的仓库
```

```json
{
  "TotalCount": "<TotalCount>",
  "Registries": "<Registries>",
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>"
}
```

### 数据面（Docker）

```bash
# 4. 验证本地镜像存在
docker images <PublicDomain>/<NAMESPACE>/<REPO>
# expected: 显示镜像，含 TAG 为 latest

# 5. 验证可重新拉取
docker rmi <PublicDomain>/<NAMESPACE>/<REPO>:latest
docker pull <PublicDomain>/<NAMESPACE>/<REPO>:latest
# expected: Pull 成功
```

## 清理

> **警告**：`DeleteInstance` 将删除实例及其下所有命名空间和镜像，不可恢复。企业版实例会持续计费，确认不再使用后请及时删除。

### 数据面清理（Docker）

```bash
# 1. 删除本地镜像
docker rmi <PublicDomain>/<NAMESPACE>/<REPO>:latest
# expected: Untagged: ..., Deleted: ...
```

### 控制面清理（tccli）

```bash
# 2. 清理前状态检查
tccli tcr DescribeInstances --region ap-guangzhou \
    --RegistryIds '["<RegistryId>"]'
# 确认是待删除的目标实例

# 3. 删除企业版实例
tccli tcr DeleteInstance \
    --RegistryId <RegistryId> \
    --region ap-guangzhou
# ⚠️ 将删除实例及所有命名空间、镜像，不可恢复

# 4. 验证已删除
tccli tcr DescribeInstances --region ap-guangzhou \
    --RegistryIds '["<RegistryId>"]'
# expected: 返回空列表或 ResourceNotFound
```

```json
{
  "TotalCount": "<TotalCount>",
  "Registries": "<Registries>",
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateInstance` 返回 `InvalidParameter` | 检查 `--RegistryName` 格式 | 实例名称包含非法字符或格式不符 | 使用小写字母、数字、连字符，长度 1-100 |
| `CreateInstance` 返回 `FailedOperation.QuotaExceeded` | `tccli tcr DescribeInstances --region ap-guangzhou` 统计已有实例数 | 实例数量达到配额上限 | 删除不再使用的实例后重试，或联系客服提升配额 |
| `DescribeInstances` 显示 `Status` 为 `Deploying` 未变为 `Running` | 等待 2-5 分钟后重试 | 实例创建中，状态变化需数分钟 | 每 30 秒轮询一次；超过 10 分钟则保留 RegistryId → 登录 [TCR 控制台](https://console.cloud.tencent.com/tcr/instance) 查看详细状态 |
| `CreateSecurityPolicy` 返回 `LimitExceeded.PolicysLimit` | `tccli tcr DescribeSecurityPolicies --RegistryId <RegistryId>` 查看现有策略数 | 公网白名单策略数量达到上限 | 删除不再需要的策略后重试 |
| `docker login` 返回 `unauthorized` | 重新执行 `docker login`；确认 PublicDomain 和 Token | Token 过期、无效，或域名拼写错误 | 重建 Token：`tccli tcr CreateInstanceToken --RegistryId <RegistryId> --TokenType longterm --region ap-guangzhou` |
| `docker push` 返回 `denied` / `401` | `docker login` 确认登录状态；验证 PublicDomain | 未登录、命名空间不存在、或使用错误域名 | 确认 `docker login` 成功；检查 push 路径与已创建的命名空间/仓库一致 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `docker pull` 超时 | `curl -v https://<PublicDomain>/v2/` 测试连通性 | 公网白名单未配置或 CIDR 不含当前 IP | `tccli tcr DescribeSecurityPolicies --RegistryId <RegistryId>` 确认白名单；添加当前出口 IP |
| `DeleteInstance` 提示实例被保护 | `tccli tcr DescribeInstances --RegistryIds '["<RegistryId>"]'` 检查实例属性 | 实例开启了删除保护 | 在控制台关闭删除保护后重试 |
| 实例创建成功但访问域名无法解析 | `nslookup <PublicDomain>` | DNS 尚未生效（通常 1-2 分钟） | 等待 2 分钟后重试 |

## 下一步

- [个人版快速入门](../个人版快速入门/tccli%20操作.md) -- page_id `63910`
- [创建简单的 Nginx 服务](../入门示例/创建简单的%20Nginx%20服务/tccli%20操作.md) -- page_id `7851`（在 TKE 集群中使用 TCR 镜像部署应用）
- [TCR 企业版操作指南](https://cloud.tencent.com/document/product/1141/39288)
- [TCR 镜像同步](https://cloud.tencent.com/document/product/1141/41364) -- 跨地域同步镜像

## 控制台替代

[容器镜像服务控制台 → 实例列表 → 新建](https://console.cloud.tencent.com/tcr/instance)：购买企业版实例并按向导配置网络、命名空间与仓库。
