---
doc_type: How-to
subtype: 6A
fused: true
---
# 管理实例访问

> 控制台: [公网访问](https://console.cloud.tencent.com/tcr/publicaccess) · [内网访问](https://console.cloud.tencent.com/tcr/privateaccess) · [用户级账号](https://console.cloud.tencent.com/tcr/token)
> 配置 TCR 企业版实例的网络访问、登录凭证和白名单策略。

## 触发条件

- `tccli tcr DescribeExternalEndpointStatus --RegistryId "<ID>"` 返回 `Status` 非 `Opened`（如 `Closed`/`Opening`），`docker login` 报 `connection refused`
- `tccli tcr DescribeInstanceToken --RegistryId "<ID>"` 返回 `Tokens: []` 或 `Enabled: false`，`docker login` 返回 `unauthorized`
- `docker login <REGISTRY_DOMAIN>` 报 `403 Forbidden`，`DescribeSecurityPolicies` 返回的白名单不含当前出口 IP


## 概述

TCR 实例默认不对外开放访问。你需要按顺序配置: ① 开启访问端点 → ② 创建登录 Token → ③ (可选) 配置白名单限制访问来源。

| 操作 | 作用 | 是否必须 | 副作用 |
|------|------|:---:|------|
| 开启公网访问 | 允许通过互联网访问 | 是 (docker login) | 无白名单时全互联网可访问 ⚠️ |
| 开启内网访问 | 允许 VPC 内访问 | 否 | 需要配置 VPC 私有域解析 |
| 创建 Token | 生成 docker login 凭证 | 是 | Token 泄露 = 任何人可 push/pull |
| 配置白名单 | 限制公网访问来源 IP | 否（公网场景建议开） | 白名单外 IP 被拒绝 |

## 准备工作

```bash
# 确认实例存在且为 Running
tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]'
# expected: { "Registries": [{ "Status": "Running" }] }
```

## 配置项

### 步骤 1：决策 — 公网 vs 内网

#### 为什么先开公网

- **公网 vs 内网**: 公网让任何地方都能 `docker login`；内网只允许 VPC 内访问（更安全但限制位置）
- **安全风险**: 公网 + 无白名单 = 任何知道你 Registry 域名的人都可以尝试登录
- **默认推荐**: 先开公网（开发阶段），上线后:
  - CI/CD 在腾讯云 → 切换到内网访问
  - 外部 CI/CD → 公网 + 白名单只允许 CI IP
- **能改吗?**: 公网/内网可以随时开关，互不冲突

### 步骤 2：开启公网访问

```bash
tccli tcr ManageExternalEndpoint \
  --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --Operation Open
# expected: exit 0
```

验证:

```bash
tccli tcr DescribeExternalEndpointStatus \
  --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>"
# expected: Status: "Opened"
```

### 步骤 3：创建访问 Token

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

> ⚠️ **Token 只显示一次。** 保存好返回的 `Token` 值与 `TokenId`（Create 响应字段名），离开后就无法再次获取 Token 值。
>
> **字段名分叉（实测）**：`CreateInstanceToken`（`longterm`）返回顶层 `TokenId`；`DescribeInstanceToken` 列表项字段是 **`Id`**（不是 `TokenId`），值与 Create 的 `TokenId` 相同。`DeleteInstanceToken` / `ModifyInstanceToken` 入参仍用 **`--TokenId`**，传入列表里的 `Id`。`temp` 创建时常返回 `TokenId: ""`，且通常**不会**出现在 `DescribeInstanceToken` 列表中（约 1 小时自动过期）。

#### Token 生命周期闭环

```
创建(CreateInstanceToken) → 使用(docker login) → [temp 自动过期 / longterm 禁用或删除]
```

- **禁用长期 Token**（保留凭证记录，可再启用）：`tccli tcr ModifyInstanceToken --region <REGION> --RegistryId "<ID>" --TokenId "<TOKEN_ID>" --Enable false`（`<TOKEN_ID>` = `DescribeInstanceToken` → `Tokens[].Id`）
- **删除长期 Token**（彻底移除凭证）：`tccli tcr DeleteInstanceToken --region <REGION> --RegistryId "<ID>" --TokenId "<TOKEN_ID>"`，expected exit 0
- **查询 Token 列表**：`tccli tcr DescribeInstanceToken --region <REGION> --RegistryId "<ID>"` → `Tokens[].Id` / `Enabled` / `Desc`

> 长期凭证泄露后，先 `ModifyInstanceToken --Enable false` 禁用止损，再 `DeleteInstanceToken` 删除。temp 凭证 1 小时自动过期，无需手动清理。

获取实例的公网域名:

```bash
tccli tcr DescribeInstances \
  --region ap-guangzhou \
  --Registryids '["<REGISTRY_ID>"]'
# expected: Registries[].PublicDomain 如 "xxx.tencentcloudcr.com"
```

登录测试:

```bash
docker login <REGISTRY_DOMAIN> --username <USERNAME> --password <TOKEN>
# expected: Login Succeeded
```

### 步骤 4：配置白名单

强烈建议: 限制公网访问来源。不要用 `0.0.0.0/0`（对全互联网开放）除非你清楚知道风险。

> 白名单策略（`CreateSecurityPolicy` / `DeleteSecurityPolicy` / `ModifySecurityPolicy` / 多策略批量 `CreateMultipleSecurityPolicy`）的完整命令见 [访问控制 — 公网白名单](../access/manage.md#createsecuritypolicy公网白名单)。本篇聚焦实例级访问开关（端点 + Token），白名单写操作归访问控制篇，避免双篇重复。

#### 为什么不用 0.0.0.0/0

- **`<YOUR_IP>/32` vs `0.0.0.0/0`**: `/32` 只允许你自己访问；`0.0.0.0/0` 对全互联网开放
- **安全风险**: `0.0.0.0/0` + Token 泄露 = 任何人可读写你的镜像仓库
- **默认推荐**: 开发期用 IP 白名单；生产环境用 VPC 内网 + 白名单
- **能改吗?**: 随时可以修改/删除白名单规则（见 [访问控制](../access/manage.md)）

### 步骤 5：开启内网访问 (可选)

如果你在腾讯云 VPC 内使用 TCR，开启内网访问更安全:

```bash
tccli tcr ManageInternalEndpoint \
  --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --Operation Open \
  --VpcId "<VPC_ID>" \
  --SubnetId "<SUBNET_ID>"
# expected: exit 0
```

## 验证 — 综合检查

```bash
# 确认所有访问配置
tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou --RegistryId "<ID>"
# expected: Status: "Opened"

tccli tcr DescribeInternalEndpoints --region ap-guangzhou --RegistryId "<ID>"
# expected: 如果配置了内网，AccessVpcSet 非空

tccli tcr DescribeInstanceToken --region ap-guangzhou --RegistryId "<ID>"
# expected: Tokens[] 含长期凭证；项字段 Id/Enabled/Desc（列表用 Id，非 TokenId）

tccli tcr DescribeSecurityPolicies --region ap-guangzhou --RegistryId "<ID>"
# expected: 公网 Opened 时返回 SecurityPolicySet；Closed → ResourceNotFound（Failed to get security group id；先 Open）
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

# 白名单删除（写操作归访问控制篇）
tccli tcr DeleteSecurityPolicy --RegistryId "<ID>" --PolicyIndex <INDEX>  # 见 access/manage.md
```

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `docker login` 提示 `unauthorized` | `tccli tcr DescribeInstanceToken` 确认 Token 存在且 Enabled | Token 不存在或已禁用 | `CreateInstanceToken` 新建 |
| `docker login` 连接超时 | `tccli tcr DescribeExternalEndpointStatus` | 公网访问未开启 | `ManageExternalEndpoint --Operation Open` |
| `docker login` 返回 `403 Forbidden` | `tccli tcr DescribeSecurityPolicies` | 当前 IP 不在白名单 | 添加当前 IP 到白名单，或临时删除白名单 |
| `DescribeSecurityPolicies` → `ResourceNotFound`（消息含 `Failed to get security group id from registry`） | `DescribeExternalEndpointStatus` | 公网端点未开（`Status=Closed`）或实例无安全组绑定（basic 常见） | 先 `ManageExternalEndpoint --Operation Open`；仍失败则查规格/公网是否支持白名单 |
| `ManageExternalEndpoint` 报错 | `tccli tcr DescribeInstances` | 实例状态不是 Running | 等待实例就绪 |
| `CreateSecurityPolicy` CidrBlock 错误 | 检查 CIDR 格式 | 格式不是 `IP/掩码` | 使用格式如 `1.2.3.4/32` |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| Token 创建成功但 docker login 失败 | 检查 Token 是否已保存 (只显示一次) | Token 值丢失 | 创建新 Token，旧的无法恢复 |
| 公网已开启但无法访问 | `curl https://<REGISTRY_DOMAIN>/v2/` | 安全组/防火墙阻断 | 检查客户端到 443 端口的网络连通性 |
| 白名单配置后自己被锁 | 通过腾讯云内网操作 | 当前 IP 不在白名单 | 从腾讯云 CVM 登录修改白名单；或提工单 |

## 收尾确认

```bash
# ③ 跨步骤汇总：公网端点 Opened + Token Enabled + 白名单含 IP 三条同时满足才闭环（Verify 逐项查，这里一次性核对三步产物）
tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --filter "Status"
# expected: "Opened"

tccli tcr DescribeInstanceToken --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --filter "Tokens[0].{enabled:Enabled,id:Id,desc:Desc}"
# expected: enabled=true, id/desc 与配置一致（Enabled/Id/Desc 在 Tokens[] 内，非顶层；DescribeInstanceToken 响应无 TokenType 字段，类型在 CreateInstanceToken 入参）

tccli tcr DescribeSecurityPolicies --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --filter "SecurityPolicySet[].CidrBlock"
# expected: 包含你的出口 IP（如 "203.0.113.10/32"）

# ② 业务可用性端到端：三步配置后 docker login 真正可达（Verify 查字段存在，这里查端到端登录成功）
docker login <REGISTRY_DOMAIN> --username <USERNAME> --password <TOKEN>
# expected: Login Succeeded
```

> 公网 Opened + Token Enabled + 白名单含 IP + docker Login Succeeded = 访问管理闭环完成，可进入推送镜像。三条配置缺一则 docker login 失败。

---

## 下一步

- [推送镜像](../images/push-pull.md) — docker push 你的第一个镜像
- [创建命名空间和仓库](../repositories/manage.md) — 组织你的镜像
