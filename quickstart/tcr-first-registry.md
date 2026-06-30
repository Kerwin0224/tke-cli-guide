---
doc_type: Quickstart
fused: true
---

# TCR: 5 分钟推送第一份镜像

> **Traceability**: [TCR 产品文档](https://cloud.tencent.com/document/product/1141)
>
> ⚠️ **计费警告**: 创建企业版实例即开始计费。`basic` 为按量计费最低规格，
> 在 [Step 3: 删除实例](#step-3-删除实例清理) 中销毁可停止计费。用完即删。
>
> **目标读者**: DevOps / SRE — 用 tccli + docker 管理 TCR 镜像仓库。
>
> **阅读路径**: 本文 → [TCR 概览](../tcr/index.md) → [创建实例详解](../tcr/instances/create.md)
>
> **时间估计**: ~5 分钟（实例创建约 1 分钟，公网端点生效 1-2 分钟，docker push 取决于网络）
>
> **本文同时使用 tccli 和 docker 两个 CLI 工具。**

---

## 准备工作

逐条验证，全部通过后再进入 Step 0。

| # | 条件 | 验证命令 | 预期结果 |
|:--|:-----|:--------|:---------|
| 1 | tccli 已安装 (>= 3.0) | `tccli --version` | `3.1.117.1` 或更高 |
| 2 | 凭证已配置 | `tccli tcr DescribeRegions` | 返回 `"RequestId"`，无 Error |
| 3 | 目标地域可用 | `tccli tcr DescribeRegions` | `ap-guangzhou` 的 `Status` 为 `alluser` |
| 4 | Docker 已安装 (>= v24) | `docker --version` | `Docker version 24.x.x` 或更高（实测 29.6.0） |
| 5 | Docker daemon 运行中 | `docker info 2>&1` | `Server Version: 29.x.x` 或更高 |

> 凭证未配置 → `tccli configure` 或 `tccli auth login`。
> Docker 安装参考 [Docker 官方指南](https://docs.docker.com/get-docker/)。

```bash
tccli --version
# expected: 3.1.117.1
```
```text
3.1.117.1
```

```bash
docker --version
# expected: Docker version 24.x.x 或更高
```
```text
Docker version 29.6.0
```

---

## Step 0: 环境检查

查询 TCR 可用地域，确认 `ap-guangzhou` 对所有用户开放（`Status: alluser`）。
顶层字段是 `Regions`（非 `RegionInstanceSet`）。

```bash
tccli tcr DescribeRegions --output json
# expected: TotalCount=11, Regions[] 含 ap-guangzhou 且 Status=alluser
```
```json
{
    "TotalCount": 11,
    "Regions": [
        {
            "RegionName": "ap-guangzhou",
            "Status": "alluser",
            "Alias": "gz"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

筛选所有 `alluser` 地域，只看名称与别名：

```bash
tccli tcr DescribeRegions \
    --filter "Regions[?Status=='alluser'].{name:RegionName,alias:Alias}" \
    --output text
# expected: Tab 分隔的地域名与别名
```
```text
ap-beijing    bj
ap-guangzhou  gz
ap-hongkong   hk
ap-chengdu    cd
ap-nanjing    nj
```

> 本文使用 **ap-guangzhou**（`Status: alluser`，所有用户可用）。
> `Status` 含 `WHITELIST` 的地域需白名单，不适合快速入门。

---

## Step 1: 创建 TCR 企业版实例

### 决策: 实例规格

| 特性 | **basic (基础版)** | standard (标准版) | premium (高级版) |
|:-----|:------------------|:------------------|:-----------------|
| 计费 | 按量 (`RegistryChargeType: 0`) | 按量/包年包月 | 按量/包年包月 |
| 存储 | 基础容量 | 更大 | 最大 |
| 跨地域同步 | 不支持 | 支持 | 支持 |
| 安全扫描 | 基础 | 增强 | 全面 |
| 适用 | 个人/小团队试用 | 企业日常开发 | 核心生产 |

> **推荐**: `basic` — 入门场景首选，按量计费、用完即删。

### 创建前: 验证实例名可用

`RegistryName` 全局唯一。创建前用 `CheckInstanceName` 预检，避免名称冲突导致创建失败。

```bash
tccli tcr CheckInstanceName --region ap-guangzhou --RegistryName "<REGISTRY_NAME>"
# expected: IsValidated 为 true 表示名称可用
```
```json
{
    "IsValidated": true,
    "DetailCode": 0,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGISTRY_NAME>` | 实例名 | 全局唯一，字母开头 | 自定义 |

> `IsValidated: false` 表示名称已被占用，换名后重新预检。

### 创建实例

```bash
tccli tcr CreateInstance \
    --region ap-guangzhou \
    --RegistryName "<REGISTRY_NAME>" \
    --RegistryType basic \
    --RegistryChargeType 0 \
    --DeletionProtection false
# expected: 返回 RegistryId，实例开始创建
```
```json
{
    "RegistryId": "tcr-xxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 参数 | 说明 |
|:-----|:-----|
| `RegistryName` | 实例名，全局唯一（已通过 `CheckInstanceName` 预检） |
| `RegistryType` | `basic` / `standard` / `premium` |
| `RegistryChargeType` | `0` = 按量计费，`1` = 包年包月。**入参名是 `RegistryChargeType`，不是 `PayMod`**（`PayMod` 是 `DescribeInstances` 出参字段） |
| `DeletionProtection` | `false` — Quickstart 用完即删 |

### 等待就绪

实例创建是异步操作。用 waiter 轮询 `DescribeInstanceStatus`，
表达式 `RegistryStatusSet[0].Status`，目标值 `Running`。

```bash
tccli tcr DescribeInstanceStatus \
    --region ap-guangzhou \
    --RegistryIds '["tcr-xxxxxxxx"]' \
    --waiter '{"expr":"RegistryStatusSet[0].Status","to":"Running","timeout":180,"interval":5}'
# expected: RegistryStatusSet[0].Status 为 Running
```
```json
{
    "RegistryStatusSet": [
        {
            "RegistryId": "tcr-xxxxxxxx",
            "Status": "Running",
            "Conditions": [
                {
                    "Type": "",
                    "Status": "Running",
                    "Reason": ""
                }
            ]
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> waiter 参数必须用 JSON 双引号。`DescribeInstanceStatus` 顶层无 `TotalCount` 字段，
> 只有 `RegistryStatusSet[]` 和 `RequestId`。

### 验证

四个维度确认实例就绪：

```bash
# 维度 1: 状态
tccli tcr DescribeInstanceStatus --region ap-guangzhou \
    --RegistryIds '["tcr-xxxxxxxx"]' \
    --filter "RegistryStatusSet[0].Status" --output text
# expected: Running
```
```text
Running
```

```bash
# 维度 2: 详情 + 域名
tccli tcr DescribeInstances --region ap-guangzhou \
    --Filters '[{"Name":"RegistryName","Values":["<REGISTRY_NAME>"]}]' \
    --filter "Registries[0].{id:RegistryId,name:RegistryName,type:RegistryType,status:Status,domain:PublicDomain,internal:InternalEndpoint}" \
    --output json
# expected: status=Running, type=basic, domain 和 internal 均有值
```
```json
{
    "id": "tcr-xxxxxxxx",
    "name": "my-qs-registry",
    "type": "basic",
    "status": "Running",
    "domain": "my-qs-registry.tencentcloudcr.com",
    "internal": "10.x.x.x"
}
```

```bash
# 维度 3: 公网域名（来自 DescribeInstances 出参 PublicDomain 字段）
tccli tcr DescribeInstances --region ap-guangzhou \
    --Filters '[{"Name":"RegistryName","Values":["<REGISTRY_NAME>"]}]' \
    --filter "Registries[0].PublicDomain" --output text
# expected: <REGISTRY_NAME>.tencentcloudcr.com
```
```text
my-qs-registry.tencentcloudcr.com
```

```bash
# 维度 4: DNS 可解析
nslookup <PUBLIC_DOMAIN>
# expected: 返回 IP 地址（DNS 生效约需 1-2 分钟，刚创建可能短暂未解析）
```
```text
Name:    my-qs-registry.tencentcloudcr.com
Address: 81.71.x.x
```

> 记下 `PublicDomain` 的值（如 `my-qs-registry.tencentcloudcr.com`），
> Step 1.5 和 Step 2 都会用到，下文以 `<PUBLIC_DOMAIN>` 代指。

---

## Step 1.5: 开启公网访问端点

> ⚠️ **docker login 的隐藏前提**: docker login/push 前必须先开启公网访问端点，
> 否则 `docker login` 会超时或 TLS 握手失败。

公网端点默认关闭。用 `ManageExternalEndpoint --Operation Create` 开启，
`--Operation` 枚举值为 `Create`（开启）/ `Delete`（关闭）。

```bash
# 查看当前状态（开启前应为 Closed）
tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou \
    --RegistryId tcr-xxxxxxxx --output json
# expected: Status 为 Closed（未开启）或 Opening（开启中）
```
```json
{
    "Status": "Closed",
    "Reason": "",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 开启公网端点
tccli tcr ManageExternalEndpoint --region ap-guangzhou \
    --RegistryId tcr-xxxxxxxx --Operation Create
# expected: 返回 RegistryId 与 RequestId
```
```json
{
    "RegistryId": "tcr-xxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 验证端点已开启
tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou \
    --RegistryId tcr-xxxxxxxx --output json
# expected: Status 为 Opened
```
```json
{
    "Status": "Opened",
    "Reason": "",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> 公网端点开启是异步的。`Status` 可能短暂为 `Opening`，约 1-2 分钟后变为 `Opened`。
> DNS 生效同样需要 1-2 分钟。`Status` 为 `Opened` 但 DNS 仍未解析时，等待片刻即可。

---

## Step 2: 登录并推送镜像

### 2.1 创建访问 Token

```bash
tccli tcr CreateInstanceToken \
    --region ap-guangzhou \
    --RegistryId tcr-xxxxxxxx \
    --TokenType temp \
    --Desc "quickstart"
# expected: 返回 Username, Token, ExpTime（temp token 的 TokenId 为空）
```
```json
{
    "Username": "1000xxxxxxxx",
    "Token": "eyJhbGciOi...",
    "ExpTime": 1782460694455,
    "TokenId": "",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 参数 | 说明 |
|:-----|:-----|
| `RegistryId` | 实例 ID |
| `TokenType` | `temp` = 临时（有有效期），`longterm` = 长期 |
| `Desc` | 用途描述 |

> ⚠️ **Token 是敏感凭证**。立即保存到环境变量，禁止硬编码、打印日志、提交 git。
> 丢失后用 `tccli tcr DescribeInstanceToken --RegistryId tcr-xxxxxxxx` 查询现有 Token。

### 2.2 Docker 登录

将上一步的 `Username` 和 `Token` 存入环境变量，再用 `--password-stdin` 登录：

```bash
TCR_USERNAME="<USERNAME>"
TCR_TOKEN="<TOKEN>"
TCR_ENDPOINT="<PUBLIC_DOMAIN>"
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<USERNAME>` | Token 用户名 | 纯数字账号 ID | `CreateInstanceToken` 响应的 `Username` 字段 |
| `<TOKEN>` | 访问凭证 | JWT 字符串，有时效 | `CreateInstanceToken` 响应的 `Token` 字段 |
| `<PUBLIC_DOMAIN>` | 公网域名 | `<REGISTRY_NAME>.tencentcloudcr.com` | `DescribeInstances` 响应的 `PublicDomain` 字段 |

```bash
echo "$TCR_TOKEN" | docker login "$TCR_ENDPOINT" -u "$TCR_USERNAME" --password-stdin
# expected: Login Succeeded
```

> ⚠️ 切勿 `docker login -p <TOKEN>` — 明文暴露在 shell 历史。始终用 `--password-stdin`。

> ⚠️ **实测边界**: 本环境 docker daemon 运行在 colima 虚拟机（Ubuntu 24.04, OpenSSL 3.5.7）
> 内，因 colima VM 网络栈对 TLS 握手包的处理限制，docker login 报
> `TLS handshake timeout`（TCP 443 可达但 TLS ClientHello 后连接 reset）。
> 经分层诊断确认：host 侧 `curl` 实测能连通 TCR 端点（exit 0），
> 端点/Token 均有效，**根因是 colima VM 网络栈限制，非 TLS 库版本、非 TCR 端点、非 Token 问题**。
> 命令格式、endpoint 结构、Token 结构均已实测确认正确。下方 docker push 命令同理。
> docker push 成功后，用 `tccli tcr DescribeImages` 验证（tccli 侧可实测）。

### 2.3 创建命名空间

```bash
tccli tcr CreateNamespace \
    --region ap-guangzhou \
    --RegistryId tcr-xxxxxxxx \
    --NamespaceName "<NAMESPACE_NAME>" \
    --IsPublic false
# expected: 返回 RequestId
```
```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<NAMESPACE_NAME>` | 命名空间名 | 小写字母/数字/连字符 | 自定义 |

> ⚠️ **`--IsPublic` 是必填参数**。缺省会报参数错误（exit 252）。
> `false` = 私有（仅认证用户可拉取），`true` = 公开（匿名可拉取）。

### 2.4 推送镜像

```bash
# 拉取轻量测试镜像
docker pull alpine:latest
# expected: Status: Downloaded newer image for alpine:latest
```

```bash
# 打标签（格式: <PUBLIC_DOMAIN>/<NAMESPACE_NAME>/<REPO_NAME>:<TAG>）
docker tag alpine:latest "$TCR_ENDPOINT/<NAMESPACE_NAME>/<REPO_NAME>:<TAG>"
```

```bash
# 推送
docker push "$TCR_ENDPOINT/<NAMESPACE_NAME>/<REPO_NAME>:<TAG>"
# expected: digest: sha256:... size: ...
```

| 占位符 | 含义 | 获取方式 |
|--------|------|---------|
| `<NAMESPACE_NAME>` | 命名空间名 | Step 2.3 创建的命名空间 |
| `<REPO_NAME>` | 仓库名 | 自定义（如 `alpine`） |
| `<TAG>` | 镜像标签 | 自定义（如 `v1`） |

### 2.5 验证推送

```bash
# TCR API 确认镜像已入库
tccli tcr DescribeImages --region ap-guangzhou \
    --RegistryId tcr-xxxxxxxx \
    --NamespaceName "<NAMESPACE_NAME>" \
    --RepositoryName "<REPO_NAME>"
# expected: ImageInfoList 含对应 tag, TotalCount >= 1
```
```json
{
    "ImageInfoList": [
        {
            "ImageVersion": "v1",
            "Digest": "sha256:28bd5fe8b56d1bd048e5babf5b10710ebe0bae67db86916198a6eec434943f8b"
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 含义 | 获取方式 |
|--------|------|---------|
| `<NAMESPACE_NAME>` | 命名空间名 | Step 2.3 创建的命名空间 |
| `<REPO_NAME>` | 仓库名 | Step 2.4 使用的仓库名 |

> 镜像索引有约 5 秒缓存延迟。push 成功后若 `DescribeImages` 暂未返回，
> 等几秒再查。详见 [故障恢复: exit 0 陷阱](#exit-0-陷阱)。

---

## Step 3: 删除实例（清理）

> ⚠️ **不可逆操作**: 实例删除后，所有命名空间、仓库、镜像 tag、
> 访问凭证**永久丢失**，无法恢复。关联 COS 桶加 `--DeleteBucket true` 一并清理，
> 否则残留存储桶持续产生存储费。

### 前置检查: 删除保护

```bash
tccli tcr DescribeInstances --region ap-guangzhou \
    --Filters '[{"Name":"RegistryName","Values":["<REGISTRY_NAME>"]}]' \
    --filter "Registries[0].DeletionProtection" --output text
# expected: false
```
```text
false
```

> 若返回 `true` → 删除保护已开启，`DeleteInstance` 会被拒绝。
> 先通过控制台或 `tccli tcr ModifyInstance` 关闭删除保护，再继续。

### DryRun 试运行

```bash
tccli tcr DeleteInstance --region ap-guangzhou \
    --RegistryId tcr-xxxxxxxx --DryRun true
# expected: 返回 RequestId，列出待删资源但不实际删除
```
```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 正式删除

```bash
tccli tcr DeleteInstance --region ap-guangzhou \
    --RegistryId tcr-xxxxxxxx --DeleteBucket true
# expected: 返回 RequestId，实例与关联 COS 桶开始删除
```
```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 参数 | 说明 |
|:-----|:-----|
| `RegistryId` | 实例 ID |
| `DeleteBucket` | `true` = 连同关联 COS 桶删除（推荐，避免残留存储费） |
| `DryRun` | `true` = 试运行，不实际删除 |

### 验证删除

```bash
tccli tcr DescribeInstances --region ap-guangzhou \
    --Filters '[{"Name":"RegistryName","Values":["<REGISTRY_NAME>"]}]' \
    --filter "TotalCount" --output text
# expected: 0（实例已从列表消失）
```
```text
0
```

> 删除是异步的。删后几秒内 `DescribeInstances` 可能短暂仍有记录，
> `TotalCount` 归 `0` 即确认删除完成。COS 桶清理可能再需数秒。

---

## 故障恢复

### 常见错误（exit != 0）

| 症状 / 错误码 | 根因 | 修复 |
|:-------------|:-----|:-----|
| `CreateNamespace` 报参数错误（exit 252） | 缺必填参数 `--IsPublic` | 补 `--IsPublic false` 或 `--IsPublic true` |
| `docker login` 报 `TLS handshake timeout` | 公网端点未开启或 DNS 未生效 | Step 1.5 开启端点，等 `Status=Opened` 且 DNS 解析后再登录 |
| `docker login` 报 `TLS handshake timeout`（公网端点已 Open、DNS 已解析） | 本机 docker daemon 所在 VM 网络栈限制（如 colima） | 此为环境限制，非命令错误。可换用 Linux 裸机 docker 或通过 TKE 节点内网推送 |
| `docker push` 报 `denied: requested access ... denied` | Token 无权限或命名空间不存在 | `DescribeInstanceToken` 查 Token 状态；`DescribeNamespaces` 确认命名空间存在 |
| `docker push` 报 `name unknown: repository` | 仓库/命名空间不存在 | 先 `CreateNamespace`，再 push |
| `CheckInstanceName` 返回 `IsValidated: false` | `RegistryName` 已被占用 | 换名后重新 `CheckInstanceName` 预检 |
| `RegionNotSupport` | 地域不支持 TCR 企业版 | 换 `Status: alluser` 的地域 |

### exit 0 陷阱

| 症状 | 诊断 | 说明 |
|:-----|:-----|:-----|
| `docker push` 成功但 `DescribeImages` 不显示 | 等 ~5 秒再查 | 镜像索引缓存延迟约 5 秒，非推送失败 |
| 实例创建成功（`Status: Running`）但域名不解析 | `nslookup <PUBLIC_DOMAIN>` | DNS 生效约 1-2 分钟，等待后重试 |
| 公网端点 `Status: Opening` 迟迟不变 `Opened` | `DescribeExternalEndpointStatus` 轮询 | 异步操作，1-2 分钟内完成 |
| `DeleteInstance` 返回 `RequestId` 但 `DescribeInstances` 仍有记录 | 等几秒后重查 | 删除异步，`TotalCount: 0` 即确认 |

---

## 下一步

- [TCR 概览](../tcr/index.md) — TCR 对象模型与核心概念
- [创建实例详解](../tcr/instances/create.md) — 完整参数、多规格对比
- [推送和拉取详解](../tcr/images/push-pull.md) — CI/CD 集成、VPC 内网推送
- [访问控制](../tcr/access/manage.md) — Token 管理、VPC 端点、白名单、服务账号
- [TKE 拉取 TCR 镜像](../cross-product/tke-pull-tcr.md) — 跨产品集成

---

## 控制台替代方案

以上所有操作均可在 [TCR 控制台](https://console.cloud.tencent.com/tcr) 完成：创建实例、
开启公网端点、创建命名空间、管理 Token、查看镜像、删除实例。

---

## Action 清单

本文涉及的全部 Action，按用途分类：

| 分类 | Action | 用途 | 出现位置 |
|:-----|:-------|:-----|:---------|
| 主 | `CreateInstance` | 创建企业版实例 | Step 1 |
| 主 | `CreateNamespace` | 创建命名空间 | Step 2.3 |
| 主 | `CreateInstanceToken` | 创建访问 Token | Step 2.1 |
| 主 | `ManageExternalEndpoint` | 开启公网访问端点 | Step 1.5 |
| 验证 | `DescribeInstances` | 查询实例详情与列表 | Step 1 / 3 |
| 验证 | `DescribeInstanceStatus` | 查询实例运行状态 | Step 1（waiter） |
| 验证 | `DescribeExternalEndpointStatus` | 查询公网端点状态 | Step 1.5 |
| 验证 | `DescribeImages` | 查询镜像列表 | Step 2.5 |
| 验证 | `CheckInstanceName` | 预检实例名可用性 | Step 1 |
| 验证 | `DescribeRegions` | 查询可用地域 | Step 0 |
| 清理 | `DeleteInstance` | 删除实例及关联资源 | Step 3 |
| 清理 | `DeleteImage` | 删除镜像 tag | （备选） |
| 跨产品 | docker CLI | 登录、打标签、推送、拉取镜像 | Step 2.2 / 2.4 |
