---
doc_type: How-to
subtype: 6A
fused: true
---
# 访问控制

> 控制台: [公网访问](https://console.cloud.tencent.com/tcr/publicaccess) · [内网访问](https://console.cloud.tencent.com/tcr/privateaccess) · [用户级账号](https://console.cloud.tencent.com/tcr/token) · [服务级账号](https://console.cloud.tencent.com/tcr/serviceaccount)
> 官方文档: [配置访问权限](https://cloud.tencent.com/document/product/1141) · [VPC 内网访问](https://cloud.tencent.com/document/product/1141/50733) · [产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731)
> 配置谁能访问 TCR 实例：用户级账号（Token）、VPC 内网、公网白名单、服务级账号。**推送/拉取优先走 VPC 内网**，公网作补充。

## 触发条件

- `docker push <DOMAIN>/<NS>/<REPO>` 报 `denied: requested access to the resource is denied`，`DescribeSecurityPolicies` 返回的白名单不含当前出口 IP
- `docker login` 报 `unauthorized`，`DescribeInstanceToken` 返回 `Tokens: []`（需创建 Token）或需长期凭证（CI/CD 场景 `temp` Token 约 1 小时过期，无法覆盖长时任务）
- VPC 内 TKE 集群拉取 TCR 镜像延迟高/走公网，`DescribeInternalEndpoints` 返回 `TotalCount: 0`（需开 VPC 内网接入）


## 概述

推荐决策顺序：**先内网（VPC）→ 再按需开公网白名单 → 再选凭证类型**。

> ⚠️ **创建后默认拒绝全部公网及内网访问**。未配置 VPC 内网或公网白名单前，`docker login`/`push`/`pull` 会失败——须先完成本页访问策略，再推镜像。公网入口开启后尽快配白名单并优先改走内网（公网产生 COS 公网流量费；内网不计该费）。`unauthorized: authentication required`：核对 `docker login` 凭证是否正确、临时 Token 是否过期。

> **VPC 接入配额（按实例规格）**：basic=5 / standard=10 / premium=20（详见 [产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731)）。`CreateSecurityPolicy` 白名单数量受规格限制（basic=5）。

| 访问方式 | 控制台入口 | 适用场景 | 创建方式 | 凭证类型 |
|:--------|:--------|:--------|:--------|:--------|
| VPC 内网 | 内网访问 | VPC 内推送/拉取（优先） | `ManageInternalEndpoint` | DNS + IP（VPC 控制） |
| 公网白名单 | 公网访问 | 开发者本地、无 VPC 时 | `CreateSecurityPolicy` | IP 白名单（需固定出口） |
| 用户级账号 | 用户级账号 | CI/CD、需严格审计身份 | `CreateInstanceToken` | Username + Token |
| 服务级账号 | 服务级账号 | 多租户 / 自动化；自定义用户名与权限范围 | `CreateServiceAccount` | 服务账号凭证 |

> 用户级账号适合需追踪实际操作者的场景；服务级账号可自定义用户名与权限范围，但平台无法验证实际使用者身份——严格审计请用用户级账号。长期凭证用 `CreateInstanceToken --TokenType longterm`，临时用 `--TokenType temp`（默认约 1 小时）。列表字段是 `Tokens[].Id`（非 `TokenId`）；删除/禁用入参仍用 `--TokenId`（传 `Id` 值）。详见 [访问管理 — Token 生命周期](../instances/manage-access.md#token-生命周期闭环)。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tcr DescribeInstanceStatus --region <REGION> --RegistryIds '["<REGISTRY_ID>"]' \
  --filter "RegistryStatusSet[0].Status"
# expected: "Running"
```

### 资源检查

```bash
# 公网访问状态
tccli tcr DescribeExternalEndpointStatus --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "Status"
# expected: "Opened"（白名单需先开公网端点）

# VPC 内网接入状态
tccli tcr DescribeInternalEndpoints --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "TotalCount"
# expected: 已接入的 VPC 数
```

## 关键字段

### CreateInstanceToken

> 详见 [推送拉取镜像](../images/push-pull.md)。入参 `{RegistryId, TokenType, Desc}`，`TokenType=temp`。

### ManageInternalEndpoint（VPC 内网）

> 完整入参以 `tccli tcr ManageInternalEndpoint help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| Operation | string | 是 | `Create`/`Delete` | `InvalidParameterValue` |
| VpcId | string | 是 | VPC ID（Create/Delete 均必填） | `ResourceNotFound` |
| SubnetId | string | 是 | 子网 ID（Create/Delete 均必填） | `ResourceNotFound` |
| RegionId | int | 否 | 地域 ID | — |
| RegionName | string | 否 | 地域名 | — |

### CreateSecurityPolicy（公网白名单）

> 完整入参以 `tccli tcr CreateSecurityPolicy help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| CidrBlock | string | 是 | CIDR，如 `1.2.3.4/32` | `InvalidParameterValue` |
| Description | string | 是 | 描述（不可省略） | `InvalidParameter` / 客户端缺参 |

### CreateServiceAccount

> 完整入参以 `tccli tcr CreateServiceAccount help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| Name | string | 是 | 服务账号名，实例内唯一 | `InvalidParameter` |
| Permissions | list | 是 | 权限列表（命名空间 + 读/写） | `InvalidParameterValue` |
| Description | string | 否 | 描述 | — |
| Duration | int | 否 | 有效期（**单位：天**，非秒），从当前时间起算，优先级高于 `ExpiresAt` | — |
| ExpiresAt | int | 否 | 过期时间戳（**单位：毫秒**） | — |
| Disable | boolean | 否 | 是否禁用 | — |

## 操作步骤

### 步骤 1：决策 — 访问方式选择

#### 四种方式怎么选

- **CI/CD 自动化**: Token（临时）或服务账号（长期）
- **VPC 内 TKE 集群拉取**: VPC 内网（低延迟；凭证仍须配置，见 [TKE 拉取 TCR](../../cross-product/tke-pull-tcr.md)）
- **开发者本地推送**: 公网白名单（需固定 IP）
- **默认推荐**: 生产用 VPC 内网 + 服务账号；本地临时用 Token
- **能切换吗?**: 各方式独立，可同时开启

### 步骤 2：VPC 内网接入

> VPC 内网接入（`ManageInternalEndpoint` 开启实例内网访问 VPC 链接）属实例级访问开关，完整命令见 [实例访问管理 — 开启内网访问](../instances/manage-access.md#步骤-2开启内网访问推荐优先)。本页覆盖访问策略层（白名单/服务账号/DNS）；端点开关见实例访问管理。VPC 内网开通后，在本页 [内网 DNS 解析](#内网-dns-解析) 配置私有域名。

> VPC 内网开通是异步操作，DNS 生效需等待。用 `DescribeInternalEndpoints` 轮询直到出现接入记录。

### 步骤 3：公网白名单

```bash
# 先确认公网端点已开启
tccli tcr DescribeExternalEndpointStatus --region <REGION> --RegistryId "<REGISTRY_ID>"
# expected: Status="Opened"

# 添加白名单 IP
tccli tcr CreateSecurityPolicy --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --CidrBlock "<YOUR_IP>/32" --Description "dev-machine"
# expected: exit 0
```

### 步骤 4：服务账号（长期凭证）

```bash
tccli tcr CreateServiceAccount --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --Name "<SA_NAME>" \
  --Permissions '[{"Resource":"prod","Actions":["tcr:PushRepository","tcr:PullRepository"]}]' \
  --Duration 30
# expected: exit 0, 返回服务账号凭证（Duration=30 表示 30 天；Permissions 用 Resource=命名空间 + Actions 列表，非 NamespaceName/Access）
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<REGISTRY_ID>` | 实例 ID | `tcr-xxxxxxxx` | `tccli tcr DescribeInstances` |
| `<VPC_ID>` | VPC ID | 须存在 | `tccli vpc DescribeVpcs` |
| `<SUBNET_ID>` | 子网 ID | 须在 VPC 内 | `tccli vpc DescribeSubnets` |

### 步骤 5：验证

```bash
# VPC 内网接入状态
tccli tcr DescribeInternalEndpoints --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "AccessVpcSet[].{vpc:VpcId,subnet:SubnetId}"
# expected: 含刚接入的 VPC

# 白名单列表（公网端点须 Opened；Closed 时报 ResourceNotFound: Failed to get security group id from registry）
tccli tcr DescribeSecurityPolicies --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "SecurityPolicySet[].{cidr:CidrBlock,desc:Description}"
# expected: Status=Opened 时含白名单；Closed → ResourceNotFound（先 ManageExternalEndpoint --Operation Create）

# 服务账号列表（RegistryId + EmbedPermission=true 带权限，Filters 按名过滤）
tccli tcr DescribeServiceAccounts --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --EmbedPermission true --Offset 0 --Limit 20
# expected: exit 0，返回 ServiceAccounts[]+TotalCount（EmbedPermission=true 时含 Permissions）
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| VPC 内网 | `DescribeInternalEndpoints` → `AccessVpcSet` | 含目标 VPC |
| 白名单 | `DescribeSecurityPolicies` → `SecurityPolicySet` | 含目标 IP |
| 服务账号 | `DescribeServiceAccounts --EmbedPermission true` | 含目标账号及权限 |
| DNS 解析 | `nslookup <REGISTRY_DOMAIN>` | 解析到 VPC 内网 IP |

## 清理

> **副作用警告**：关闭 VPC 内网会断开该 VPC 内所有拉取访问。删除白名单会阻断对应 IP 的访问。

```bash
# 关闭 VPC 内网（端点开关见实例访问管理）
tccli tcr ManageInternalEndpoint --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --Operation Delete --VpcId "<VPC_ID>" --SubnetId "<SUBNET_ID>"  # Delete 亦必填 SubnetId；见 instances/manage-access.md
# expected: exit 0

# 删除白名单
tccli tcr DeleteSecurityPolicy --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --CidrBlock "<IP>/32"
# expected: exit 0
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound.VpcId` | `tccli vpc DescribeVpcs` | VPC 不存在或跨账号 | 确认 VPC ID 与账号 |
| `ResourceNotFound.SubnetId` | `tccli vpc DescribeSubnets` | 子网不在指定 VPC | 用 VPC 内子网 |
| `InvalidParameterValue.CidrBlock` | 检查 CIDR 格式 | CIDR 格式错 | 用 `IP/掩码` 格式，如 `1.2.3.4/32` |
| `ResourceNotFound`（消息含 `Failed to get security group id from registry`） | `DescribeExternalEndpointStatus` | 公网端点 `Closed` 或实例无安全组绑定（basic 常见） | 先 `ManageExternalEndpoint --Operation Create`；仍失败则查规格是否支持公网白名单 |
| `LimitExceeded` | `DescribeSecurityPolicies` 看数量 | 白名单达上限（basic=5） | 删除闲置白名单或升规格 |
| `FailedOperation` | `DescribeInstanceStatus` 查看状态 | 实例非 Running | 等实例 Running |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| VPC 内网开通但 DNS 不解析 | `DescribeInternalEndpoints` | DNS 生成延迟 | 等 1-2 分钟；查 PrivateDNS |
| 白名单添加但 docker push 仍 `denied` | `DescribeExternalEndpointStatus` | 公网端点未开启 | 先 `ManageExternalEndpoint --Operation Create` |
| 服务账号创建但拉取失败 | `DescribeServiceAccounts` 看权限 | 权限不含目标命名空间 | 补 `Permissions` 命名空间 |

## 内网 DNS 与多策略白名单

> 内网端点的自定义 DNS 解析（`InternalEndpointDns`）与批量安全策略管理（`MultipleSecurityPolicy`）。属访问控制的进阶操作。

### 内网 DNS 解析

```bash
# 查询内网 DNS 解析状态 (注意: 入参是 VpcSet[] 嵌套数组, 非 RegistryId)
tccli tcr DescribeInternalEndpointDnsStatus --region <REGION> \
  --VpcSet '[{"InstanceId":"<REGISTRY_ID>","VpcId":"<VPC_ID>","RegionName":"<REGION>"}]'
# expected: exit 0, 返回各 VPC 的 DNS 解析状态
```

> ⚠️ `DescribeInternalEndpointDnsStatus` 必填 `--VpcSet`（嵌套数组，含 InstanceId/VpcId/RegionName），缺失报 `the following arguments are required: --VpcSet`（exit 252）。注意用 `InstanceId`（非 RegistryId）。

```bash
# 创建内网 DNS 解析 (用内网 IP 解析到自定义/默认域名)
tccli tcr CreateInternalEndpointDns --region <REGION> \
  --InstanceId "<REGISTRY_ID>" --VpcId "<VPC_ID>" --EniLBIp "<ENI_LB_IP>" \
  --RegionName "<REGION>" --RegionId <REGION_ID> --UsePublicDomain false
# expected: exit 0

# 删除内网 DNS 解析
tccli tcr DeleteInternalEndpointDns --region <REGION> \
  --InstanceId "<REGISTRY_ID>" --VpcId "<VPC_ID>" --EniLBIp "<ENI_LB_IP>" --RegionName "<REGION>"
# expected: exit 0
```

> `CreateInternalEndpointDns` 的 `EniLBIp` 是内网端点 IP，`UsePublicDomain=true` 用公网域名解析，`RegionId` 是数字 ID（见 [实例同步](../replication/manage.md) 地域表）。

### 多策略白名单（批量）

```bash
# 批量创建安全策略 (SecurityGroupPolicySet[] 多条)
tccli tcr CreateMultipleSecurityPolicy --region <REGION> \
  --RegistryId "<REGISTRY_ID>" \
  --SecurityGroupPolicySet '[{"PolicyIndex":1,"CidrBlock":"<IP1>/32","Description":"office"},{"PolicyIndex":2,"CidrBlock":"<IP2>/32","Description":"ci"}]'
# expected: exit 0

# 批量删除安全策略
tccli tcr DeleteMultipleSecurityPolicy --region <REGION> \
  --RegistryId "<REGISTRY_ID>" \
  --SecurityGroupPolicySet '[{"PolicyIndex":1,"CidrBlock":"<IP1>/32"}]'
# expected: exit 0

# 修改单条安全策略 (按 PolicyIndex)
tccli tcr ModifySecurityPolicy --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --PolicyIndex 1 --CidrBlock "<NEW_IP>/32" --Description "<DESC>"
# expected: exit 0
```

> `CreateMultipleSecurityPolicy`/`DeleteMultipleSecurityPolicy` 用 `SecurityGroupPolicySet[]`（批量，含 PolicyIndex/CidrBlock/Description/PolicyVersion）；`ModifySecurityPolicy` 单条修改用 `PolicyIndex` 定位。区别于 `CreateSecurityPolicy`（单条）。

## 服务账号管理

> 服务账号（ServiceAccount）的修改、改密、删除。创建见 [步骤 4](#步骤-4服务账号长期凭证)。

```bash
# 修改服务账号 (描述/有效期/权限)
tccli tcr ModifyServiceAccount --RegistryId "<REGISTRY_ID>" --Name "<SA_NAME>" \
  --Description "<DESC>" --Duration <DURATION> --region <REGION>
# expected: exit 0

# 修改服务账号密码 (Random=true 随机生成, false 用指定 Password)
tccli tcr ModifyServiceAccountPassword --RegistryId "<REGISTRY_ID>" --Name "<SA_NAME>" --Random true --region <REGION>
# expected: exit 0, Random=true 时返回新密码

# 删除服务账号
tccli tcr DeleteServiceAccount --RegistryId "<REGISTRY_ID>" --Name "<SA_NAME>" --region <REGION>
# expected: exit 0
```

> `ModifyServiceAccountPassword` 的 `Random=true` 随机生成密码（返回新密码），`false` 用 `--Password` 指定。`Duration` 是有效期**天数**（非秒），`ExpiresAt` 是过期时间戳（毫秒）。

## 收尾确认

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
# 汇总核对：白名单 + 服务账号 + VPC 内网（字段名 SecurityPolicySet 非 Policies）
tccli tcr DescribeSecurityPolicies --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "SecurityPolicySet[].{ip:CidrBlock,desc:Description}"
# expected: 含你的出口 IP（如 "203.0.113.10/32"）

tccli tcr DescribeServiceAccounts --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --EmbedPermission true --Offset 0 --Limit 20 \
  --filter "ServiceAccounts[].{name:Name,perms:Permissions[].{res:Resource,actions:Actions}}"
# expected: 含目标服务账号及其资源权限（Permissions 用 Resource/Actions，非 NamespaceName/Access）

tccli tcr DescribeInternalEndpoints --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "AccessVpcSet[].{vpc:VpcId,subnet:SubnetId}"
# expected: 含已接入的 VPC（未开内网则空，仅白名单场景可豁免此项）

# 端到端：用配置的访问方式 docker login 成功
printf '%s\n' "<TOKEN_OR_SA_PASSWORD>" | docker login <REGISTRY_DOMAIN> --username "<USERNAME>" --password-stdin
# expected: Login Succeeded
```

> 白名单含 IP + 服务账号带权限 + VPC 内网接入（如开）+ docker Login Succeeded = 访问控制配置完成。任一缺失则 docker login/push 失败。

---

## 下一步

- [推送拉取镜像](../images/push-pull.md) — Token 获取与 docker login
- [访问管理](../instances/manage-access.md) — 实例级访问配置
- [创建实例](../instances/create.md) — 实例生命周期
- [故障排查](../troubleshooting.md) — 访问失败诊断
