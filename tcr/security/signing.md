---
doc_type: How-to
subtype: 6A
fused: false
---
# 镜像签名

> 控制台: [容器镜像服务控制台 - 镜像签名](https://console.cloud.tencent.com/tcr/signature)
> 官方文档: [容器镜像签名](https://cloud.tencent.com/document/product/1141/80862)（仅 premium 支持）
> 配置镜像签名策略，用 KMS 托管密钥对镜像签名，确保镜像不被篡改。仅企业版 premium 支持。

## 触发条件

- `tccli tcr DescribeInstances --Registryids '["<ID>"]'` 返回 `RegistryType: premium` 且需镜像完整性校验（合规/安全要求镜像不可篡改）
- `tccli kms ListKeys` 有可用签名密钥但镜像未签名，拉取侧 TKE 验签准入控制器拒绝未签名镜像
- `tccli tcr CreateSignature` 无法发起签名：先确认签名策略已创建，且实例为 premium


## 概述

签名策略绑定 KMS 密钥与命名空间，push 镜像时自动签名或手动签名。拉取侧可验签，确保镜像完整性与来源可信。

| 能力 | 作用 | 规格 |
|:-----|:-----|:-----|
| 手动签名 | 用 KMS 密钥对已存在镜像签名 | premium 专属，tccli `CreateSignature` |
| 签名策略 | 绑定 KMS 密钥+命名空间，push 时自动签名 | premium 专属，tccli `CreateSignaturePolicy` |
| 验签 | 拉取时验证签名 | TKE 集成（非 tccli，见下） |

> **产品边界**：TCR API 落地签名策略与手动签名；**自动签名**由策略触发（push 时 TCR 服务侧执行，无需 tccli 调用）；**验签**在 TKE 侧部署签名准入控制器（K8s 准入 webhook，非 tccli）。TCR 文档覆盖签名侧闭环，验签侧见 TKE 集成文档。

> 签名是 **premium（高级版）** 专属；basic/standard 不支持。
>
> **前提**：
> - KMS 密钥用途须为 **非对称签名验签**，算法仅支持 **RSA_2048**（SM2 及其他算法不可用于本功能）
> - TCR 可读取 KMS 全地域的用户密钥；`KmsRegion` 必须填写密钥的实际地域。密钥可与 TCR 实例跨地域，但为降低跨地域通信开销，建议同地域部署
> - 服务角色 `TCR_QCSRole` 须关联 **QcloudKMSFullAccess**（或等价 KMS 权限），否则签名失败——见下 [服务角色（TCR/KMS）](#服务角色tcrkms)
> - **单个命名空间仅一条**签名策略

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
# 列出 KMS 密钥 ID；ListKeys 不返回别名，需确认单个密钥详情时使用 DescribeKey
tccli kms ListKeys --region <REGION> --filter "Keys[].KeyId" --output text
# expected: 含目标 KMS 密钥 ID
```

### 服务角色（TCR/KMS）

> 官方要求：授权容器镜像服务使用 KMS 时，在 `TCR_QCSRole` 上关联 **QcloudKMSFullAccess**（控制台角色详情页操作）。下列为 CLI 探测与补挂。

```bash
# 探测角色
tccli cam DescribeRoleList --Page 1 --Rp 100 \
  --filter "List[?contains(RoleName,'TCR')].RoleName" --output text
# expected: 含 TCR_QCSRole；空 → 先走控制台 TCR 服务授权或 CreateRole（载体以官方/控制台为准）

# 查是否已挂 KMS 策略
tccli cam ListAttachedRolePolicies --Page 1 --Rp 50 --RoleName TCR_QCSRole \
  --filter "List[].PolicyName" --output text
# expected: 含 QcloudKMSFullAccess（或等价 KMS 策略名）

# 补挂 KMS 全读写（角色已存在时）
tccli cam AttachRolePolicy \
  --AttachRoleName TCR_QCSRole \
  --PolicyName QcloudKMSFullAccess
# expected: RequestId；Role not exist → 先控制台开通 TCR 服务角色
```

> 总表见 [配置凭证 — 服务角色](../../getting-started/credentials.md#服务角色tke--ipamd--as--tcr--可观测)。用户侧还须有 `kms:SignByAsymmetricKey` 等权限时，与服务角色分开处理。

## 关键字段

> 完整入参以 `tccli tcr CreateSignaturePolicy help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 |
|:------|------|:--------:|------------|
| RegistryId | string | 是 | `tcr-xxxxxxxx`，且实例须为 premium |
| Name | string | 是 | 策略名，实例内唯一 |
| NamespaceName | string | 是 | 已存在的命名空间名 |
| KmsId | string | 是 | 已存在、用途及算法符合要求的 KMS 密钥 ID |
| KmsRegion | string | 是 | KMS 密钥实际所属地域 |
| Domain | string | 否 | 自定义签名域名；为空时使用 TCR 实例默认域名 |
| Disabled | boolean | 否 | 是否禁用策略 |

> 当前 API 契约仅明确 `FailedOperation.DependenceError`、`InternalError.ErrorTcrUnauthorized`、`InvalidParameter.ErrorTcrInvalidParameter` 和 `UnsupportedOperation` 等接口级错误边界，未给出字段到错误码的一一映射；排障时应结合响应中的完整错误码和 `RequestId` 定位。

> `KmsId` 须先在 KMS 创建用途为“非对称签名验签”、算法为 `RSA_2048` 的用户密钥；TCR 镜像签名不支持 SM2。`KmsRegion` 填写该密钥的实际地域，不要求与 TCR 实例地域一致；跨地域可用，但会增加跨地域通信开销。

## 操作步骤

### 步骤 1：决策 — 签名策略

#### 为什么用 KMS 托管密钥

- **KMS 托管**: 镜像签名策略使用 KMS 用户密钥，密钥权限由 KMS/CAM 管理
- **算法要求**: KMS 托管的 `RSA_2048` 非对称签名验签密钥；本功能不支持 SM2
- **变更密钥**: 当前 TCR API 未提供 `ModifySignaturePolicy`；需要更换密钥时，先评估删除策略会清除存量签名信息的影响，再重新创建策略

### 步骤 2：创建签名策略

```bash
tccli tcr CreateSignaturePolicy --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --Name "<POLICY_NAME>" \
  --NamespaceName "<NAMESPACE_NAME>" \
  --KmsId "<KMS_KEY_ID>" --KmsRegion "<REGION>"
# expected: exit 0，返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<REGISTRY_ID>` | 实例 ID | `tcr-xxxxxxxx`，须 premium | `tccli tcr DescribeInstances` |
| `<KMS_KEY_ID>` | KMS 密钥 ID | 须存在且为签名类型 | `tccli kms ListKeys` |
| `<NAMESPACE_NAME>` | 命名空间名 | 须已存在 | `tccli tcr DescribeNamespaces` |

### 步骤 3：签名镜像

```bash
# 创建签名（对指定镜像版本签名，须先有签名策略）
tccli tcr CreateSignature --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" \
  --RepositoryName "<REPOSITORY_NAME>" --ImageVersion "<TAG>"
# expected: exit 0
```

> `CreateSignature` 入参仅 4 个（RegistryId/NamespaceName/RepositoryName/ImageVersion，全必填），签名所用密钥由该命名空间绑定的签名策略（步骤 2 `CreateSignaturePolicy`）决定，无需在签名命令指定策略名。

### 步骤 4：验证

> TCR API 无 `DescribeSignaturePolicies` 接口。请在控制台的**命名空间**页面查看是否启用加签策略，并在**镜像仓库 > 版本管理**查看具体镜像的签名状态。`CreateSignature` 的成功响应只表示该请求被服务端成功处理，不能单独证明后续验签一定成功。

```bash
# 对存量镜像发起手动签名；随后在控制台“镜像仓库 > 版本管理”查看签名状态
tccli tcr CreateSignature --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>" \
  --RepositoryName "<REPOSITORY_NAME>" --ImageVersion "<TAG>"
# expected: exit 0；签名状态仍以控制台“镜像仓库 > 版本管理”为准
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 策略已启用 | 控制台**命名空间**页面 | 显示已开启加签策略 |
| KMS 密钥状态 | `tccli kms DescribeKey --KeyId "<KMS_KEY_ID>"` | `KeyMetadata.KeyState=Enabled` |
| TCR 镜像签名状态 | 控制台**镜像仓库 > 版本管理** | 目标镜像显示签名状态 |

## 清理

> **副作用警告**：官方文档明确说明，删除签名策略会同时删除该命名空间内的存量镜像签名信息，可能导致签名验证失败。删除前应确认该命名空间不再依赖这些签名。

```bash
# 删除签名策略（按命名空间，单命名空间仅一条策略；无 --PolicyName 参数）
tccli tcr DeleteSignaturePolicy --region <REGION> \
  --RegistryId "<REGISTRY_ID>" --NamespaceName "<NAMESPACE_NAME>"
# expected: exit 0
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 返回 `InvalidParameter.ErrorTcrInvalidParameter` | 按关键字段说明逐项核对请求 | TCR 请求参数无效；接口未公布具体字段映射 | 修正参数后重试；仍失败时携带 `RequestId` 定位 |
| 返回 `UnsupportedOperation` | `DescribeInstances` 查看规格 | 当前实例或操作不支持镜像签名 | 确认实例为 premium；若仍失败，携带 `RequestId` 定位 |
| 返回 `FailedOperation.DependenceError` | `DescribeKey` 核对密钥，并检查服务角色策略 | KMS 等依赖服务异常 | 确认密钥可用且服务角色具备 KMS 权限后重试 |
| 返回 `InternalError.ErrorTcrUnauthorized` | 查用户 CAM + `ListAttachedRolePolicies --RoleName TCR_QCSRole` | TCR 操作未获授权 | 补齐用户侧权限；服务侧见 [服务角色（TCR/KMS）](#服务角色tcrkms) |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 策略创建但签名失败 | `CreateSignature` 错误码 | KMS 密钥用途或算法不符合要求 | 改用用途为“非对称签名验签”、算法为 `RSA_2048` 的 KMS 用户密钥 |
| 签名成功但验签失败 | TKE 侧验签配置 | 验签策略未在 TKE 集群配置 | 在 TKE 集群配置镜像验签 |
| KMS 密钥未启用 | `tccli kms DescribeKey --KeyId "<KMS_KEY_ID>"` 查看 `KeyMetadata.KeyState` | KMS 用户密钥未处于受支持的启用状态 | 启用受支持的 KMS 用户密钥，或改用处于 `Enabled` 状态且符合签名要求的密钥 |

> 签名涉及 TCR + KMS 跨产品。TCR 无查询签名策略的 API，管理主要靠控制台。

## 收尾确认

汇总确认以下三项：
- 在控制台**命名空间**页面确认加签策略已开启。
- 在控制台**镜像仓库 > 版本管理**确认目标镜像的签名状态；策略创建前已存在的镜像需要手动触发签名。
- `tccli kms DescribeKey` 可确认 KMS 密钥状态，但密钥为 `Enabled` 不能替代 TCR 镜像签名状态或 TKE 验签结果。

```bash
tccli kms DescribeKey --region <KMS_REGION> --KeyId "<KMS_KEY_ID>" \
  --filter "KeyMetadata.{id:KeyId,state:KeyState,usage:KeyUsage}"
# expected: state=Enabled；再按上面的控制台检查确认镜像签名状态
```

---

## 下一步

- [访问控制](../access/manage.md) — 签名后的访问权限
- [推送拉取镜像](../images/push-pull.md) — 签名镜像的 push/pull
- [CVE 白名单](cve-whitelist.md) — 漏洞扫描阻断白名单（与签名互补）
- [实例概览](../instances/index.md) — premium 规格说明
- [故障排查](../troubleshooting.md) — 签名失败诊断
