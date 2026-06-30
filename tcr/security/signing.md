---
doc_type: How-to
subtype: 6A
fused: false
---
# 镜像签名

> 配置镜像签名策略，用 KMS 托管密钥对镜像签名，确保镜像不被篡改。仅企业版 premium 支持。

## 概述

签名策略绑定 KMS 密钥与命名空间，push 镜像时自动签名或手动签名。拉取侧可验签，确保镜像完整性与来源可信。

| 能力 | 作用 | 规格 |
|:-----|:-----|:-----|
| 签名 | 用 KMS 密钥对镜像签名 | premium 专属 |
| 验签 | 拉取时验证签名 | TKE 集成 |

> 签名是 premium 专属功能，basic/standard 不支持。需先在 KMS 服务创建密钥。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

# 确认实例是 premium 规格（签名仅 premium 支持）
tccli tcr DescribeInstances --region <REGION> --Registryids '["<REGISTRY_ID>"]' \
  --filter "Registries[0].{type:RegistryType,status:Status}"
# expected: type="premium", status="Running"
```

### 资源检查

```bash
# 确认 KMS 密钥存在
tccli kms ListKeys --region <REGION> --filter "Keys[].{id:KeyId,alias:Alias}" --output text
# expected: 含目标 KMS 密钥 ID
```

## 关键字段

> 来源：`tccli tcr CreateSignaturePolicy --generate-cli-skeleton`（实测）。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx` | `ResourceNotFound` |
| Name | string | 是 | 策略名，实例内唯一 | `InvalidParameter` |
| NamespaceName | string | 是 | 命名空间名 | `ResourceNotFound` |
| KmsId | string | 是 | KMS 密钥 ID | `ResourceNotFound` |
| KmsRegion | string | 是 | KMS 密钥地域 | `InvalidParameterValue` |
| Domain | string | 否 | 自定义签名域名 | `InvalidParameterValue` |
| Disabled | boolean | 否 | 是否禁用策略 | — |

> `KmsId` 须先在 KMS 服务创建非对称签名密钥（如 SM2/RSA）。`KmsRegion` 是 KMS 密钥所在地域，须与实例地域一致或可跨地域。

## 操作步骤

### 步骤 1：决策 — 签名策略

#### 为什么用 KMS 托管密钥

- **KMS 托管（推荐）**: 密钥在 KMS 服务管理，自动轮转，权限可控
- **自带证书**: 手动管理证书，易泄露，不推荐
- **默认推荐**: KMS 托管 SM2（国密）或 RSA 密钥
- **能改吗?**: 策略创建后可 `ModifySignaturePolicy`（如存在）改密钥，已签名镜像不受影响

### 步骤 2：创建 — 最小化

```bash
tccli tcr CreateSignaturePolicy --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --Name "<POLICY_NAME>" \
  --NamespaceName "<NAMESPACE_NAME>" \
  --KmsId "<KMS_KEY_ID>" --KmsRegion "<REGION>"
# expected: exit 0, 返回策略 ID
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<REGISTRY_ID>` | 实例 ID | `tcr-xxxxxxxx`，须 premium | `tccli tcr DescribeInstances` |
| `<KMS_KEY_ID>` | KMS 密钥 ID | 须存在且为签名类型 | `tccli kms ListKeys` |
| `<NAMESPACE_NAME>` | 命名空间名 | 须已存在 | `tccli tcr DescribeNamespaces` |

### 步骤 3：签名镜像

```bash
# 创建签名（对指定镜像版本签名）
tccli tcr CreateSignature --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" \
  --RepositoryName "<REPOSITORY_NAME>" --ImageVersion "<TAG>" \
  --SignaturePolicyName "<POLICY_NAME>"
# expected: exit 0
```

### 步骤 4：验证

> ⚠️ TCR 无 `DescribeSignaturePolicies` 接口查询签名策略——通过控制台或 `CreateSignature` 不报错反证策略存在。

```bash
# 验证策略存在：CreateSignature 成功即反证策略有效
tccli tcr CreateSignature --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" \
  --RepositoryName "<REPOSITORY_NAME>" --ImageVersion "<TAG>" \
  --SignaturePolicyName "<POLICY_NAME>" --DryRun true 2>&1 | /usr/bin/head -3
# expected: 无 ResourceNotFound（策略存在）
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 策略存在 | `CreateSignature --DryRun` 不报策略错 | 无 `ResourceNotFound` |
| 签名成功 | `CreateSignature` | exit 0 |
| 证书未过期 | KMS 密钥状态 `Enabled` | `tccli kms ListKeys` |

## 清理

> **副作用警告**：删除签名策略不影响已签名镜像的签名（签名是镜像的附属数据）。但新镜像无法再按该策略签名。

```bash
# 删除签名策略
tccli tcr DeleteSignaturePolicy --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" --PolicyName "<POLICY_NAME>"
# expected: exit 0
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound` (KmsId) | `tccli kms ListKeys` | KMS 密钥不存在 | 先在 KMS 创建签名密钥 |
| `UnsupportedOperation` | `DescribeInstances` 看规格 | 实例非 premium | 升级到 premium 或换实例 |
| `ResourceNotFound` (Namespace) | `DescribeNamespaces` 核对 | 命名空间不存在 | 先 `CreateNamespace` |
| `InvalidParameterValue.KmsRegion` | 核对地域 | KmsRegion 与密钥实际地域不符 | 用密钥所在地域 |
| `UnauthorizedOperation` | 查 CAM 策略 | 无 KMS 使用权限 | 授予 `kms:SignByAsymmetricKey` |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 策略创建但签名失败 | `CreateSignature` 错误码 | KMS 密钥类型非签名类型 | 用 SM2/RSA 签名密钥 |
| 签名成功但验签失败 | TKE 侧验签配置 | 验签策略未在 TKE 集群配置 | 在 TKE 集群配置镜像验签 |
| 证书过期 | `tccli kms ListKeys` 看密钥状态 | KMS 密钥被禁用或过期 | 启用密钥或换新密钥 |

> 签名涉及 TCR + KMS 跨产品。TCR 无查询签名策略的 API（gap），管理主要靠控制台。

## 下一步

- [访问控制](../access/manage.md) — 签名后的访问权限
- [推送拉取镜像](../images/push-pull.md) — 签名镜像的 push/pull
- [实例概览](../instances/index.md) — premium 规格说明
- [故障排查](../troubleshooting.md) — 签名失败诊断

## 控制台替代方案

[容器镜像服务控制台 - 镜像签名](https://console.cloud.tencent.com/tcr/signature)

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `CreateSignature` | 主操作 | TCR | 对指定镜像版本签名 |
| `CreateSignaturePolicy` | 主操作 | TCR | 创建签名策略（绑定 KMS 密钥） |
| `CreateNamespace` | 主操作 | TCR | 创建命名空间（签名前提） |
| `DescribeNamespaces` | 验证 | TCR | 查询命名空间 |
| `DescribeInstances` | 验证 | TCR | 查询实例（确认 premium 规格） |
| `DeleteSignaturePolicy` | 清理 | TCR | 删除签名策略 |
