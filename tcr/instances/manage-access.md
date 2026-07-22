---
doc_type: How-to
subtype: 6A
fused: true
---
# 管理实例访问

> 控制台: [内网访问](https://console.cloud.tencent.com/tcr/privateaccess) · [公网访问](https://console.cloud.tencent.com/tcr/publicaccess) · [用户级账号](https://console.cloud.tencent.com/tcr/token)
> 官方文档: [企业版快速入门](https://cloud.tencent.com/document/product/1141/39287)（**先内网后公网**）· [产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731)（VPC 接入配额 basic=5/standard=10/premium=20；服务账号配额 basic=10/standard=20/premium=100）
> 配置 TCR 企业版实例的网络访问、登录凭证和白名单策略。

## 触发条件

- `tccli tcr DescribeInternalEndpoints --RegistryId "<ID>"` 返回 `AccessVpcSet` 空或无目标 VPC，且客户端在腾讯云 VPC 内需 `docker login`/`pull`
- `tccli tcr DescribeExternalEndpointStatus --RegistryId "<ID>"` 返回 `Status` 非 `Opened`（如 `Closed`/`Opening`），本地或外网 CI `docker login` 报 `connection refused`
- `tccli tcr DescribeInstanceToken --RegistryId "<ID>"` 返回 `Tokens: []` 或 `Enabled: false`，`docker login` 返回 `unauthorized`
- `docker login <REGISTRY_DOMAIN>` 报 `403 Forbidden`，`DescribeSecurityPolicies` 返回的白名单不含当前出口 IP

## 概述

TCR 实例创建后**默认拒绝全部公网与内网访问**。登录/推拉前须配置访问：① 开访问端点 → ② 创建登录 Token → ③（公网时）配置白名单。

官方与控制台推荐顺序：**先内网（VPC），再按需公网**。内网更快、无公网流量费、可限定 VPC；公网把独享实例暴露到互联网，须尽快配白名单，正式启用前勿长期 `0.0.0.0/0`。

| 操作 | 作用 | 是否必须 | 副作用 |
|------|------|:---:|------|
| 开启内网访问 | 允许指定 VPC 内访问 | 腾讯云内推拉推荐 | 须配 VPC 私有域解析；占用 VPC 接入配额（见 [配额](../reference/quotas.md)） |
| 开启公网访问 | 允许通过互联网访问 | 仅本地/外网 CI 需要 | 无白名单时全互联网可尝试登录；产生 COS 公网流量费 |
| 创建 Token | 生成 docker login 凭证 | 是 | Token 泄露 = 可 push/pull；值只显示一次 |
| 配置白名单 | 限制公网访问来源 IP | 公网场景建议配置 | 白名单外 IP 被拒绝 |

> **为什么会出现 docker**：`docker login` 用于确认端点与 Token 是否真正可用。TCCLI 只管理端点/凭证/白名单，**不提供** Registry 协议登录或镜像传输（无 daemon / 无 `docker login` 等价 Action）——能力终止处才接 docker。

## 准备工作

```bash
# 确认实例存在且为 Running
tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]'
# expected: { "Registries": [{ "Status": "Running" }] }
```

## 配置项

### 步骤 1：决策 — 内网优先，公网按需 {#步骤-1决策-内网优先公网按需}

#### 为什么先内网

- **内网 vs 公网**：内网只允许指定 VPC 访问（更快、无公网流量费、可收紧数据面）；公网让任意互联网客户端可达（本地笔记本、外网 CI）
- **官方口径**（企业版快速入门）：建议内网推拉；开启公网会暴露独享实例，完成内网配置后应尽快关闭不必要的公网入口
- **默认推荐**：
  - 推拉端在腾讯云 VPC / TKE → **只开内网**（见步骤 2）
  - 本地调试或外网 CI → 再开公网 + 白名单（步骤 3–4）
  - 两者可并存，互不冲突
- **可修改**：公网/内网可随时 `Manage*Endpoint --Operation Create|Delete`（**仅** `Create`/`Delete`；传 `Open`/`Close` 等返回 `InvalidParameter`：`not support operation, must Create or Delete`）

### 步骤 2：开启内网访问（推荐优先） {#步骤-2开启内网访问推荐优先}

```bash
tccli tcr ManageInternalEndpoint \
  --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --Operation Create \
  --VpcId "<VPC_ID>" \
  --SubnetId "<SUBNET_ID>"
# expected: exit 0
# --Operation 仅 Create|Delete；Open/Close 非法
```

## 跨字段约束

| `Operation` | `VpcId` | `SubnetId` | 关系 |
|:------------|:--------|:-----------|:-----|
| `Create` | 必填 | 必填，且必须属于该 VPC | 创建指定 VPC/子网的内网访问链路 |
| `Delete` | 按待删除端点传原 VPC | 按待删除端点传原子网 | 删除同一对 VPC/子网标识的链路 |

`VpcId` 与 `SubnetId` 不是互斥字段；它们共同标识接入网络。`Open`/`Close` 不是合法 `Operation`。

验证:

```bash
tccli tcr DescribeInternalEndpoints \
  --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>"
# expected: AccessVpcSet 含目标 VpcId/SubnetId；项字段 Status 示例为 Running（官方示例；空接入时 AccessVpcSet 可为 null、TotalCount=0）
```

> 内网域名解析：链路建立后，若 VPC 内无法解析实例域名，须在控制台「管理自动解析」或 Private DNS 配置（见官方内网访问控制）。TCCLI 侧对应 `CreateInternalEndpointDns` 等；本页步骤只保证接入链路创建。

### 步骤 3：按需开启公网访问

仅当客户端不在已接入 VPC 内时执行。

```bash
tccli tcr ManageExternalEndpoint \
  --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --Operation Create
# expected: exit 0
```

验证:

```bash
tccli tcr DescribeExternalEndpointStatus \
  --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>"
# expected: Status: "Opened"
```

### 步骤 4：创建访问 Token

Token 是 docker login 的密码。两种类型：

| TokenType | 有效期 | 适用 | 能否删除 |
|:----------|:-------|:-----|:--------:|
| `temp`（默认） | 1 小时自动过期 | 临时推送、调试 | 自动过期，无需删 |
| `longterm` | 长期有效 | 生产 CI/CD | ✅ `DeleteInstanceToken` |

> 生产 CI/CD 用 `longterm`（持久）；临时推送用 `temp`（默认，过期自动失效）。详见 [推送拉取镜像](../images/push-pull.md)。

```bash
tccli tcr CreateInstanceToken \
  --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --TokenType longterm \
  --Desc "CI/CD Token"
# expected: { "Token": "<LONG_TOKEN_STRING>", "Username": "..." }（tccli 默认剥离 Response 包装层）
```

> ⚠️ **Token 只显示一次。** 将返回的 `Username` 与 `Token` 直接写入密钥管理服务或 docker credential store，不要输出到日志。保存 `TokenId`（Create 响应字段名），离开后无法再次获取 Token 值。
>
> **字段名分叉**：`CreateInstanceToken`（`longterm`）返回顶层 `TokenId`；`DescribeInstanceToken` 列表项字段是 **`Id`**（不是 `TokenId`），值与 Create 的 `TokenId` 相同。`DeleteInstanceToken` / `ModifyInstanceToken` 入参仍用 **`--TokenId`**，传入列表里的 `Id`。`temp` 创建时常返回 `TokenId: ""`，且通常**不会**出现在 `DescribeInstanceToken` 列表中（约 1 小时自动过期）。

#### Token 生命周期闭环 {#token-生命周期闭环}

```
创建(CreateInstanceToken) → 使用(docker login) → [temp 自动过期 / longterm 禁用或删除]
```

- **禁用长期 Token**（保留凭证记录，可再启用）：`tccli tcr ModifyInstanceToken --region <REGION> --RegistryId "<ID>" --TokenId "<TOKEN_ID>" --Enable false`（`<TOKEN_ID>` = `DescribeInstanceToken` → `Tokens[].Id`；`ModifyFlag` 默认 `2` 表示启停，改描述用 `1`）
- **删除长期 Token**（彻底移除凭证）：`tccli tcr DeleteInstanceToken --region <REGION> --RegistryId "<ID>" --TokenId "<TOKEN_ID>"`，expected exit 0
- **查询 Token 列表**：`tccli tcr DescribeInstanceToken --region <REGION> --RegistryId "<ID>"` → `Tokens[].Id` / `Enabled` / `Desc`（列表无 `TokenType` 字段）

> 长期凭证泄露后，先 `ModifyInstanceToken --Enable false` 禁用止损，再 `DeleteInstanceToken` 删除。temp 凭证 1 小时自动过期，无需手动清理。

获取实例访问域名:

```bash
tccli tcr DescribeInstances \
  --region ap-guangzhou \
  --Registryids '["<REGISTRY_ID>"]'
# expected: Registries[].PublicDomain 如 "xxx.tencentcloudcr.com"（内网亦用该域名 + VPC 解析）
```

登录测试（能力边界：docker，非 tccli）:

> docker CLI（Registry 登录，非 tccli；TCCLI 不提供 docker daemon / Registry 协议登录能力）
```bash
printf '%s' "$TCR_TOKEN" | docker login <REGISTRY_DOMAIN> --username "$TCR_USERNAME" --password-stdin
# expected: Login Succeeded
```

### 步骤 5：配置公网白名单（开了公网时）

建议限制公网访问来源。不要用 `0.0.0.0/0`（对全互联网开放）除非你清楚知道风险；正式启用前删除过宽规则。

> 白名单策略（`CreateSecurityPolicy` / `DeleteSecurityPolicy` / `ModifySecurityPolicy` / 多策略批量 `CreateMultipleSecurityPolicy`）的完整命令见 [访问控制 — 公网白名单](../access/manage.md#createsecuritypolicy公网白名单)。本页覆盖实例级访问开关（端点 + Token）；白名单写操作见访问控制。

#### 为什么不用 0.0.0.0/0

- **`<YOUR_IP>/32` vs `0.0.0.0/0`**: `/32` 只允许你自己访问；`0.0.0.0/0` 对全互联网开放
- **安全风险**: `0.0.0.0/0` + Token 泄露 = 任何人可读写你的镜像仓库
- **默认推荐**: 开发期用 IP 白名单；生产环境优先 VPC 内网，公网仅放行 CI 出口
- **可修改**： 随时可以修改/删除白名单规则（见 [访问控制](../access/manage.md)）

## 验证 — 综合检查 {#验证-综合检查}

```bash
# 内网（若已配）
tccli tcr DescribeInternalEndpoints --region ap-guangzhou --RegistryId "<ID>"
# expected: AccessVpcSet 非空且含目标 VPC

# 公网（若已配）
tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou --RegistryId "<ID>"
# expected: Status: "Opened"（未开公网则为 Closed，属预期）

tccli tcr DescribeInstanceToken --region ap-guangzhou --RegistryId "<ID>"
# expected: Tokens[] 含长期凭证；项字段 Id/Enabled/Desc（列表用 Id，非 TokenId）

tccli tcr DescribeSecurityPolicies --region ap-guangzhou --RegistryId "<ID>"
# expected: 公网 Opened 时返回 SecurityPolicySet；Closed → ResourceNotFound（Failed to get security group id；先 ManageExternalEndpoint --Operation Create）
```

## 清理

```bash
# 关闭公网
tccli tcr ManageExternalEndpoint --RegistryId "<ID>" --Operation Delete
# expected: exit 0, 公网端点关闭

# 关闭内网
tccli tcr ManageInternalEndpoint --RegistryId "<ID>" --Operation Delete --VpcId "<VPC_ID>" --SubnetId "<SUBNET_ID>"
# expected: exit 0, 内网端点关闭

# 禁用 Token
tccli tcr ModifyInstanceToken --RegistryId "<ID>" --TokenId "<TOKEN_ID>" --Enable false
# expected: exit 0, Token 禁用

# 白名单删除（写操作见访问控制）
tccli tcr DeleteSecurityPolicy --RegistryId "<ID>" --PolicyIndex <INDEX>
# expected: exit 0
```

完整白名单运维见 [访问控制](../access/manage.md)。

## 外网端点字段契约

| 字段 | 所属 Action | 必填 | 说明 |
|:---|:---|:---:|:---|
| `RegistryId` | `ManageExternalEndpoint` | 是 | 目标企业版实例 ID |

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `docker login` 提示 `unauthorized` | `tccli tcr DescribeInstanceToken` 确认 Token 存在且 Enabled | Token 不存在或已禁用 | `CreateInstanceToken` 新建 |
| `docker login` 连接超时（公网） | `tccli tcr DescribeExternalEndpointStatus` | 公网访问未开启 | `ManageExternalEndpoint --Operation Create` |
| `docker login` 连接超时（VPC 内） | `tccli tcr DescribeInternalEndpoints` | 内网未接入或域名未解析 | `ManageInternalEndpoint` + 私有域解析 |
| `docker login` 返回 `403 Forbidden` | `tccli tcr DescribeSecurityPolicies` | 当前 IP 不在白名单 | 添加当前 IP 到白名单，或临时删除白名单 |
| `DescribeSecurityPolicies` → `ResourceNotFound`（消息含 `Failed to get security group id from registry`） | `DescribeExternalEndpointStatus` | 公网端点未开（`Status=Closed`）或实例无安全组绑定（basic 常见） | 先 `ManageExternalEndpoint --Operation Create`；仍失败则查规格/公网是否支持白名单 |
| `InvalidParameter`（`not support operation, must Create or Delete`） | 检查 `--Operation` | 传了 `Open`/`Close` 等非法值 | 只用 `Create` 或 `Delete` |
| `ManageExternalEndpoint` / `ManageInternalEndpoint` 报错 | `tccli tcr DescribeInstances` | 实例状态不是 Running，或 RegistryId 不存在 | 等待实例就绪；假 ID 可能返回 `InternalError.DbError` / `FailedOperation.ErrorGetDBDataError` |
| `CreateSecurityPolicy` CidrBlock 错误 | 检查 CIDR 格式 | 格式不是 `IP/掩码` | 使用格式如 `1.2.3.4/32` |
| `LimitExceeded`（VPC 接入） | `DescribeInternalEndpoints` 看条数 | 达规格 VPC 配额（basic=5） | 删闲置接入或升规格，见 [配额](../reference/quotas.md) |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| Token 创建成功但 docker login 失败 | 检查 Token 是否已保存 (只显示一次) | Token 值丢失 | 创建新 Token，旧的无法恢复 |
| 公网已开启但无法访问 | `curl https://<REGISTRY_DOMAIN>/v2/` | 安全组/防火墙阻断 | 检查客户端到 443 端口的网络连通性 |
| 白名单配置后自己被锁 | 通过腾讯云内网操作 | 当前 IP 不在白名单 | 从腾讯云 CVM 登录修改白名单；或提工单 |

## 收尾确认

> docker CLI（Registry 登录可用性确认，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
# 至少一条可达路径（内网 AccessVpcSet 或公网 Opened）+ Token Enabled
tccli tcr DescribeInternalEndpoints --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --filter "AccessVpcSet[].{vpc:VpcId,subnet:SubnetId,status:Status}"
# expected: 内网路径时非空

tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --filter "Status"
# expected: 公网路径时 "Opened"；仅内网时可为 Closed

tccli tcr DescribeInstanceToken --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --filter "Tokens[0].{enabled:Enabled,id:Id,desc:Desc}"
# expected: enabled=true, id/desc 与配置一致（Enabled/Id/Desc 在 Tokens[] 内，非顶层；DescribeInstanceToken 响应无 TokenType 字段，类型在 CreateInstanceToken 入参）

# 公网路径时核对白名单
tccli tcr DescribeSecurityPolicies --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --filter "SecurityPolicySet[].CidrBlock"
# expected: 公网 Opened 时包含你的出口 IP（如 "203.0.113.10/32"）

# docker login 端到端确认可达
printf '%s' "$TCR_TOKEN" | docker login <REGISTRY_DOMAIN> --username "$TCR_USERNAME" --password-stdin
# expected: Login Succeeded
```

> （内网已接入 **或** 公网 Opened）+ Token Enabled +（公网时白名单含 IP）+ docker Login Succeeded = 访问管理配置完成，可进入推送镜像。路径与凭证缺一则 docker login 失败。

---

## 下一步

- [推送镜像](../images/push-pull.md) — docker push 你的第一个镜像
- [创建命名空间和仓库](../repositories/manage.md) — 组织你的镜像
- [配额与限制](../reference/quotas.md) — VPC / 服务账号配额

## Action 字段契约

| 字段 | 所属 Action | 必填 | 说明 |
|:---|:---|:---:|:---|
| `VpcId` | `ManageInternalEndpoint` | 是 | 端点 VPC |
| `SubnetId` | `ManageInternalEndpoint` | 是 | 端点子网 |
