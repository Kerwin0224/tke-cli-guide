---
doc_type: How-to
subtype: 6A
fused: true
---
# 访问控制

> 配置谁能访问 TCR 实例：Token（CI/CD）、VPC 内网、公网白名单、服务账号。四种方式参数与失败模式各异。

## 概述

四种访问方式，用途不同：

| 访问方式 | 适用场景 | 创建方式 | 凭证类型 |
|:--------|:--------|:--------|:--------|
| Instance Token | CI/CD 自动化 | `CreateInstanceToken` | Username + Token（临时，1 小时） |
| VPC 内网 | VPC 内拉取，低延迟 | `ManageInternalEndpoint` | DNS + IP（无认证，VPC 控制） |
| 公网白名单 | 开发者本地推送 | `CreateSecurityPolicy` | IP 白名单（需固定 IP） |
| 服务账号 | K8s 内 Pod 拉取 | `CreateServiceAccount` | 服务账号凭证 |

> 长期凭证（CI/CD）用 `CreateInstanceToken --TokenType longterm`（可禁用/删除），临时凭证用 `--TokenType temp`（默认，1 小时过期）。详见 [访问管理 — Token 生命周期](../instances/manage-access.md#token-生命周期闭环)。

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

> 来源：`tccli tcr ManageInternalEndpoint --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| Operation | string | 是 | `Open`/`Close` | `InvalidParameterValue` |
| VpcId | string | Open 必填 | VPC ID | `ResourceNotFound` |
| SubnetId | string | Open 必填 | 子网 ID | `ResourceNotFound` |
| RegionId | int | 否 | 地域 ID | — |
| RegionName | string | 否 | 地域名 | — |

### CreateSecurityPolicy（公网白名单）

> 来源：`tccli tcr CreateSecurityPolicy --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| CidrBlock | string | 是 | CIDR，如 `1.2.3.4/32` | `InvalidParameterValue` |
| Description | string | 否 | 描述 | — |

### CreateServiceAccount

> 来源：`tccli tcr CreateServiceAccount --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| Name | string | 是 | 服务账号名，实例内唯一 | `InvalidParameter` |
| Permissions | list | 是 | 权限列表（命名空间 + 读/写） | `InvalidParameterValue` |
| Description | string | 否 | 描述 | — |
| Duration | int | 否 | 有效期秒数 | — |
| Disable | boolean | 否 | 是否禁用 | — |

## 操作步骤

### 步骤 1：决策 — 访问方式选择

#### 四种方式怎么选

- **CI/CD 自动化**: Token（临时）或服务账号（长期）
- **VPC 内 TKE 集群拉取**: VPC 内网（低延迟，免认证）
- **开发者本地推送**: 公网白名单（需固定 IP）
- **默认推荐**: 生产用 VPC 内网 + 服务账号；本地临时用 Token
- **能切换吗?**: 各方式独立，可同时开启

### 步骤 2：VPC 内网接入 — 最小化

> VPC 内网接入（`ManageInternalEndpoint` 开启实例内网访问 VPC 链接）属实例级访问开关，完整命令见 [实例访问管理 — 开启内网访问](../instances/manage-access.md#步骤-5开启内网访问-可选)。本篇聚焦访问策略层（白名单/服务账号/DNS），端点开关归实例访问篇，避免双篇重复。VPC 内网开通后，本篇 [内网 DNS 解析](#内网-dns-解析) 配置私有域名。

> VPC 内网开通是异步操作，DNS 生效需等待。用 `DescribeInternalEndpoints` 轮询直到出现接入记录。

### 步骤 3：公网白名单 — 增强

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
  --Permissions '[{"NamespaceName":"prod","Access":"readwrite"}]' \
  --Duration 2592000
# expected: exit 0, 返回服务账号凭证
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

# 白名单列表
tccli tcr DescribeSecurityPolicies --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "SecurityPolicySet[].{cidr:CidrBlock,desc:Description}"
# expected: 含刚添加的 IP

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
# 关闭 VPC 内网（端点开关归实例访问篇）
tccli tcr ManageInternalEndpoint --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --Operation Close --VpcId "<VPC_ID>"  # 见 instances/manage-access.md
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
| `LimitExceeded` | `DescribeSecurityPolicies` 看数量 | 白名单达上限（basic=5） | 删除闲置白名单或升规格 |
| `FailedOperation` | `DescribeInstanceStatus` 看状态 | 实例非 Running | 等实例 Running |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| VPC 内网开通但 DNS 不解析 | `DescribeInternalEndpoints` | DNS 生成延迟 | 等 1-2 分钟；查 PrivateDNS |
| 白名单添加但 docker push 仍 `denied` | `DescribeExternalEndpointStatus` | 公网端点未开启 | 先 `ManageExternalEndpoint --Operation Open` |
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

> `ModifyServiceAccountPassword` 的 `Random=true` 随机生成密码（返回新密码），`false` 用 `--Password` 指定。`Duration` 是有效期秒数，`ExpiresAt` 是过期时间戳。

## 下一步

- [推送拉取镜像](../images/push-pull.md) — Token 获取与 docker login
- [访问管理](../instances/manage-access.md) — 实例级访问配置
- [创建实例](../instances/create.md) — 实例生命周期
- [故障排查](../troubleshooting.md) — 访问失败诊断

## 控制台替代方案

[容器镜像服务控制台 - 访问控制](https://console.cloud.tencent.com/tcr/access)
