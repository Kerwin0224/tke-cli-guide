---
doc_type: Quickstart
fused: true
---

# TCR: 5 分钟推送第一份镜像

> 控制台: [TCR 控制台](https://console.cloud.tencent.com/tcr)
> 官方文档: [企业版快速入门](https://cloud.tencent.com/document/product/1141/39287) · [产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731) · [创建企业版实例](https://cloud.tencent.com/document/product/1141/51110)
>
> ⚠️ **计费警告**: 创建企业版实例即开始计费。`basic` 为按量计费最低规格。
> 在 [Step 4: 删除实例](#step-4-删除实例清理) 中销毁可停止计费。用完即删。
>
> **目标读者**: DevOps / SRE — 用 TCCLI + docker 管理 TCR 镜像仓库。
>
> **阅读路径**: 本文 → [TCR 概览](../tcr/index.md) → [创建实例详解](../tcr/instances/create.md)
>
> **时间估计**: ~5 分钟（实例创建约 1 分钟，公网端点生效 1-2 分钟，docker push 取决于网络）
>
> **本文同时使用 TCCLI 和 docker 两个 CLI 工具。**

---

## 触发条件

- 需用 TCCLI 验证创建镜像仓库并推送一份镜像（建实例→开公网→推送→删除闭环）— 本篇是最短路径
- 终端执行 `tccli tcr DescribeInstances` 返回空或需验证 TCR+docker 混合工作流 — 从 [Step 0 环境检查](#step-0-环境检查) 开始
- 要使用 TCCLI 管理镜像仓库 + docker 推送的混合工作流实操 TCR 生命周期 — 本 Quickstart 含完整 docker login/tag/push + TCR 验证

## 准备工作

逐条验证，全部通过后再进入 Step 0。

| # | 条件 | 验证命令 | 预期结果 |
|:--|:-----|:--------|:---------|
| 1 | TCCLI 已安装 (>= 3.0) | `tccli --version` | 输出 `3.x.x` 版本号 |
| 2 | 凭证已配置 | `tccli tcr DescribeRegions` | 返回 `"RequestId"`，无 Error |
| 3 | 目标地域可用 | `tccli tcr DescribeRegions` | `ap-guangzhou` 的 `Status` 为 `alluser` |
| 4 | Docker 已安装 (>= v24) | `docker --version` | `Docker version 24.x.x` 或更高（29.6.0） |
| 5 | Docker daemon 运行中 | `docker info 2>&1` | `Server Version: 29.x.x` 或更高 |

> 未安装 TCCLI → [安装 TCCLI](../getting-started/install.md)。凭证未配置 → [配置凭证](../getting-started/credentials.md)（`tccli configure` 或 `tccli auth login`）。
> Docker 安装参考 [Docker 官方指南](https://docs.docker.com/get-docker/)。

```bash
tccli --version
# expected: 输出 3.x.x 版本号
```
```text
3.1.126.1
```

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
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
# expected: TotalCount 为当前账号可见地域数（随账号/产品开通变化，勿写死）；Regions[] 含 ap-guangzhou 且 Status=alluser
```
```json
{
    "TotalCount": 19,
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
# expected: Tab 分隔；--output text 多键投影列序按投影 key 名字母序（alias,name），
# 非书写序。要固定列序请用 --output json
```
```text
bj	ap-beijing
gz	ap-guangzhou
hk	ap-hongkong
cd	ap-chengdu
nj	ap-nanjing
```

> 本文使用 **ap-guangzhou**（`Status: alluser`，所有用户可用）。
> `Status` 含 `WHITELIST` 的地域需白名单，不适合快速入门。

---

## Step 1: 创建 TCR 企业版实例

### 决策: 实例规格

| 特性 | **basic (基础版)** | standard (标准版) | premium (高级版) |
|:-----|:------------------|:------------------|:-----------------|
| 计费 | 按量 (`RegistryChargeType: 0`) | 按量/包年包月 | 按量/包年包月 |
| 命名空间配额 | 50 | 100 | 500 |
| 镜像仓库配额 | 1000 | 3000 | 5000 |
| VPC 接入配额 | 5 | 10 | 20 |
| 跨地域同步 | 不支持 | 支持 | 支持 |
| 安全扫描 | 基础 | 增强 | 全面 |
| 适用 | 个人/小团队试用 | 企业日常开发 | 核心生产 |

> **推荐**: `basic` — 入门场景用 basic，按量计费、用完即删。

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
> Step 2 和 Step 3 都会用到，下文以 `<PUBLIC_DOMAIN>` 代指。

---

## Step 2: 开启公网访问端点

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

## Step 3: 登录并推送镜像

### 3.1 创建访问 Token

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
> `DescribeInstanceToken` 只能查询长期凭证的 `Id`、描述、启用状态和有效期等元数据，不返回 `Token` 或 `Username` 密文。Token 密文仅在创建响应中返回；若丢失，应创建新 Token、验证登录，再禁用或删除旧 Token。

### 3.2 Docker 登录

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

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
echo "$TCR_TOKEN" | docker login "$TCR_ENDPOINT" -u "$TCR_USERNAME" --password-stdin
# expected: Login Succeeded
```

> ⚠️ 切勿 `docker login -p <TOKEN>` — 明文暴露在 shell 历史。始终用 `--password-stdin`。

> ⚠️ **边界**: 若 `docker login` 报 `TLS handshake timeout`（公网端点已 Open、DNS 已解析、host 侧 `curl` 可连通），多为本机 docker daemon 所在 VM（如 colima）网络栈对 TLS 握手的限制。此为环境边界，命令格式与端点均正确。可换用 Linux 裸机 docker、通过 TKE 节点内网推送，或继续用 TCCLI 侧 `DescribeImages` 验证镜像入库。

### 3.3 创建 TCR 命名空间

这里创建的是 **TCR 实例内用于镜像路径组织和访问控制的命名空间**，不是 Kubernetes Namespace；二者即使同名也不会自动关联。资源模型见 [TCR 概览](../tcr/index.md)。

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

### 3.4 推送镜像

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
# 拉取轻量测试镜像
docker pull alpine:latest
# expected: Status: Downloaded newer image for alpine:latest
```

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
# 打标签（格式: <PUBLIC_DOMAIN>/<NAMESPACE_NAME>/<REPO_NAME>:<TAG>）
docker tag alpine:latest "$TCR_ENDPOINT/<NAMESPACE_NAME>/<REPO_NAME>:<TAG>"
```

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
# 推送
docker push "$TCR_ENDPOINT/<NAMESPACE_NAME>/<REPO_NAME>:<TAG>"
# expected: digest: sha256:... size: ...
```

| 占位符 | 含义 | 获取方式 |
|--------|------|---------|
| `<NAMESPACE_NAME>` | 命名空间名 | Step 3.3 创建的命名空间 |
| `<REPO_NAME>` | 仓库名 | 自定义（如 `alpine`） |
| `<TAG>` | 镜像标签 | 自定义（如 `v1`） |

### 3.5 验证推送

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
| `<NAMESPACE_NAME>` | 命名空间名 | Step 3.3 创建的命名空间 |
| `<REPO_NAME>` | 仓库名 | Step 3.4 使用的仓库名 |

> 镜像索引有约 5 秒缓存延迟。push 成功后若 `DescribeImages` 暂未返回，
> 等待约 5 秒后重查。详见 [故障恢复: exit 0 误判](#exit-0-误判)。

---

## Step 4: 删除实例（清理）

> ⚠️ **不可逆操作**: 实例删除后，所有命名空间、仓库、镜像 tag、
> 访问凭证**永久丢失**，无法恢复。关联 COS 桶加 `--DeleteBucket true` 同时清理，
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
# expected: 返回 RequestId；服务端完成试运行且不实际删除
```
```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> DryRun 响应不列出待删资源。执行前应使用上方 `DescribeInstances` 按目标 `RegistryName` 核对 `RegistryId`、删除保护和实例范围。

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
| `docker login` 报 `TLS handshake timeout` | 公网端点未开启或 DNS 未生效 | Step 2 开启端点，等 `Status=Opened` 且 DNS 解析后再登录 |
| `docker login` 报 `TLS handshake timeout`（公网端点已 Open、DNS 已解析） | 本机 docker daemon 所在 VM 网络栈限制（如 colima） | 此为环境限制，非命令错误。可换用 Linux 裸机 docker 或通过 TKE 节点内网推送 |
| `docker push` 报 `denied: requested access ... denied` | Token 无权限或命名空间不存在 | `DescribeInstanceToken` 查 Token 状态；`DescribeNamespaces` 确认命名空间存在 |
| `docker push` 报 `name unknown: repository` | 仓库/命名空间不存在 | 先 `CreateNamespace`，再 push |
| `CheckInstanceName` 返回 `IsValidated: false` | `RegistryName` 已被占用 | 换名后重新 `CheckInstanceName` 预检 |
| `RegionNotSupport` | 地域不支持 TCR 企业版 | 换 `Status: alluser` 的地域 |

### exit 0 误判

| 症状 | 诊断 | 说明 |
|:-----|:-----|:-----|
| `docker push` 成功但 `DescribeImages` 不显示 | 等 ~5 秒再查 | 镜像索引缓存延迟约 5 秒，非推送失败 |
| 实例创建成功（`Status: Running`）但域名不解析 | `nslookup <PUBLIC_DOMAIN>` | DNS 生效约 1-2 分钟，等待后重试 |
| 公网端点 `Status: Opening` 迟迟不变 `Opened` | `DescribeExternalEndpointStatus` 轮询 | 异步操作，1-2 分钟内完成 |
| `DeleteInstance` 返回 `RequestId` 但 `DescribeInstances` 仍有记录 | 等待约 5 秒后重查 | 删除异步，`TotalCount: 0` 即确认 |

---

## 收尾确认

```bash
# 残留资源核查：DeleteInstance --DeleteBucket true 已声明同删关联 COS 桶。
# TCR 关联桶命名固定为 tcr-<RegistryId>-<AppId>（AppId 见账号信息）；
# 桶清理由 --DeleteBucket true 负责，tccli cos 是对象操作 CLI（无 DescribeBuckets 枚举接口）。
# 已知桶名时可用 tccli cos ls 直接确认桶已不存在：
tccli cos ls -b tcr-<REGISTRY_ID>-<APP_ID>
# expected: 报错 "NoSuchBucket"（桶已随实例删除）；若返回对象列表则需手动清理
# 说明：tccli cos 的子命令为 ls/list/download 等对象操作，不提供桶枚举，
# 残量桶确认须基于已知桶名；否则到 COS 控制台核对 "tcr-" 前缀桶。

# 目标实例最终确认（只按本 Quickstart 记录的唯一名称核对）
tccli tcr DescribeInstances --region ap-guangzhou \
  --Filters '[{"Name":"RegistryName","Values":["<REGISTRY_NAME>"]}]' \
  --filter "TotalCount" --output text
# expected: 0（目标实例已从列表消失；账号内其他实例不影响判据）
```

> 目标实例已删（按 `RegistryName` 查询 `TotalCount=0`）+ 已知关联 COS 桶无残留 = 本 Quickstart 的建实例→推送→删除闭环完成。不要用账号全局实例数作为通过条件，也不要删除与本次 Quickstart 无关的资源。

---

## 下一步

- [TCR 概览](../tcr/index.md) — TCR 对象模型与核心概念
- [创建实例详解](../tcr/instances/create.md) — 完整参数、多规格对比
- [推送和拉取详解](../tcr/images/push-pull.md) — CI/CD 集成、VPC 内网推送
- [访问控制](../tcr/access/manage.md) — Token 管理、VPC 端点、白名单、服务账号
- [TKE 拉取 TCR 镜像](../cross-product/tke-pull-tcr.md) — 跨产品集成

---
