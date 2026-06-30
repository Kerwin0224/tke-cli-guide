# 容器镜像签名及验证（tccli）

> 对照官方：[容器镜像签名及验证](https://cloud.tencent.com/document/product/457/80909) · page_id `80909`

## 概述

通过 Cosign 或 Notation 工具对 TCR 企业版容器镜像进行数字签名，并在 Kubernetes 集群中部署准入控制器（Admission Webhook）强制验证签名，确保只有经过可信签名的镜像才能被部署到集群中。

签名工具选择：
- **Cosign**：Sigstore 生态核心工具，支持无密钥签名（Keyless）、密钥对签名、KMS 签名
- **Notation**：CNCF Notary v2 标准实现，与 OCI 分发规范原生集成

本指南涵盖完整的镜像签名、验证策略配置和准入控制部署流程。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId/secretKey 已配置，region: ap-guangzhou

tccli tcr DescribeInstances --region <Region>
# expected: TotalCount >= 1, 实例状态 Running

tccli tcr DescribeNamespaces --region <Region> \
    --RegistryId INSTANCE_ID
# expected: 命名空间列表，含目标 NAMESPACE

cosign version
# expected: cosign version >= 2.0.0

notation version
# expected: Notation CLI 已安装（备选方案）
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

### CAM 权限检查

```bash
tccli cam DescribeRoleList --region <Region> --Page 1 --Rp 20
# expected: exit 0

# 验证 TCR 操作权限
tccli tcr DescribeRepositories --region <Region> \
    --RegistryId INSTANCE_ID \
    --NamespaceName NAMESPACE
# expected: RepositoryList 返回仓库列表
```

```json
{
  "RepositoryList": "<RepositoryList>",
  "Name": "<Name>",
  "Namespace": "<Namespace>",
  "CreationTime": "<CreationTime>",
  "Public": "<Public>",
  "Description": "<Description>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/cosign/kubectl 命令 | 幂等 |
|-----------|-------------------------|:--:|
| 查看 TCR 实例 | `tccli tcr DescribeInstances --region <Region>` | 是 |
| 查看命名空间 | `tccli tcr DescribeNamespaces --region <Region> --RegistryId INSTANCE_ID` | 是 |
| 查看仓库列表 | `tccli tcr DescribeRepositories --region <Region> --RegistryId INSTANCE_ID --NamespaceName NAMESPACE` | 是 |
| 生成 Cosign 密钥对 | `cosign generate-key-pair` | 否 |
| 生成 Notation 证书 | `notation cert generate-test --default "notation-demo"` | 否 |
| 对镜像签名（Cosign） | `cosign sign --key cosign.key TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG` | 是 |
| 对镜像签名（Notation） | `notation sign TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG` | 是 |
| 验证镜像签名（Cosign） | `cosign verify --key cosign.pub TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG` | 是 |
| 验证镜像签名（Notation） | `notation verify TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG` | 是 |
| 部署准入控制器 | `kubectl apply -f cluster-image-policy.yaml`（需 VPN/IOA） | 是 |
| 查看签名记录 | `cosign tree TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG` | 是 |

## 操作步骤

### 步骤 1：准备 TCR 镜像仓库和认证

#### 选择依据

- **Cosign** vs **Notation**：Cosign 社区活跃度更高，与 Sigstore 生态（Rekor 透明日志、Fulcio 证书颁发）原生集成；Notation 遵循 Notary v2 标准，与 OCI 1.1 规范对齐
- **密钥管理策略**：测试环境可使用本地密钥对；生产环境建议使用腾讯云 KMS 或 Keyless 签名模式
- **签名存储**：Cosign 签名默认存储为 OCI 镜像的附加 manifest（tag 为 `sha256-xxx.sig`）；Notation 签名以 OCI 附件形式存储

```bash
# 创建 TCR 长期访问凭证
tccli tcr CreateInstanceToken --region <Region> \
    --RegistryId INSTANCE_ID \
    --TokenType longterm \
    --Desc "cosign-signing-token"
# expected: 返回 Username 和 Token

# 登录 TCR
echo "TOKEN_VALUE" | docker login TCR_INSTANCE_ID.tencentcloudcr.com \
    --username=TOKEN_USERNAME \
    --password-stdin
# expected: Login Succeeded

# 推送待签名镜像
docker tag SOURCE_IMAGE TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG
docker push TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG
# expected: push 成功，digest: sha256:xxx
```

### 步骤 2：生成签名密钥

#### 选择依据

- **本地密钥对**：适用于开发测试，私钥需妥善保管
- **KMS 签名**：适用于生产环境，私钥不离开 KMS（需 Cosign KMS 插件支持）
- **Keyless**：通过 OIDC 身份（如 GitHub Actions）获取短期证书签名，无需管理密钥

```bash
# 方案 A：Cosign 本地密钥对（推荐入门）
cosign generate-key-pair
# expected: 交互式提示输入密码保护私钥
# 生成文件：
#   cosign.key  — 私钥（加密存储）
#   cosign.pub  — 公钥（用于验证）

# 方案 B：Notation 测试证书
notation cert generate-test --default "notation-demo"
# expected: 生成自签名测试证书，设为默认签名证书
```

### 步骤 3：对容器镜像签名

#### 最小创建（Cosign）

```bash
cosign sign --key cosign.key \
    TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG
# expected: tlog entry created with index: XXXXX
# 签名存储为: TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:sha256-xxx.sig
```

#### 增强配置

```bash
# Cosign Keyless 签名（需 OIDC 身份）
cosign sign \
    TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG

# Notation 签名
notation sign \
    TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG \
    --signature-format cose
# expected: Successfully signed

# 查看签名元数据
cosign tree TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG
# expected: 显示签名链和 Rekor 日志条目
```

### 步骤 4：本地验证签名

```bash
# Cosign 验证
cosign verify --key cosign.pub \
    TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG | jq .
# expected: 输出签名验证结果

# Notation 验证
notation verify \
    TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG
# expected: Successfully verified signature
```

预期输出（Cosign）：

```json
{
    "critical": {
        "identity": {
            "docker-reference": "TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME"
        },
        "image": {
            "docker-manifest-digest": "sha256:..."
        },
        "type": "cosign container image signature"
    },
    "optional": {
        "Bundle": {
            "SignedEntryTimestamp": "...",
            "Payload": {
                "body": "..."
            }
        }
    }
}
```

### 步骤 5：部署准入控制器强制签名验证

#### 选择依据

- **Sigstore Policy Controller**：基于 Kyverno 的策略引擎，支持 CUE/Rego 策略语言
- **原生 Cosign Webhook**：轻量级，仅支持 Cosign 签名验证
- **Gatekeeper + Cosign**：通过 OPA Gatekeeper 扩展实现，适合已有 OPA 体系

数据面（需 VPN/IOA）：

```bash
# 安装 Sigstore Policy Controller
helm repo add sigstore https://sigstore.github.io/helm-charts
helm repo update
helm install policy-controller sigstore/policy-controller \
    --namespace cosign-system \
    --create-namespace
# expected: policy-controller deployed
```

#### 创建 ClusterImagePolicy

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-cosign-signature
spec:
  mode: enforce
  images:
  - glob: "TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/**"
  - glob: "TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE2/**"
  authorities:
  - key:
      data: |
        -----BEGIN PUBLIC KEY-----
        PLACEHOLDER_PUBLIC_KEY_DATA
        -----END PUBLIC KEY-----
    ctlog:
      url: https://rekor.sigstore.dev
  - keyless:
      identities:
      - issuer: "https://accounts.google.com"
        subject: "user@example.com"
```

```bash
kubectl apply -f cluster-image-policy.yaml
# expected: clusterimagepolicy.policy.sigstore.dev/require-cosign-signature created

kubectl get clusterimagepolicy
# expected: READY True, MODE enforce
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 验证镜像已签名
cosign verify --key cosign.pub \
    TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG
# expected: Verification successful - valid signature found

# 验证签名条目在 TCR 中存在
tccli tcr DescribeImageManifests --region <Region> \
    --RegistryId INSTANCE_ID \
    --NamespaceName NAMESPACE \
    --RepositoryName REPO_NAME \
    --ImageVersion IMAGE_TAG
# expected: Manifest 含签名相关 layer
```

```json
{
  "Manifest": "<Manifest>",
  "Config": "<Config>",
  "Labels": "<Labels>",
  "Key": "<Key>",
  "Value": "<Value>",
  "Size": "<Size>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 部署已签名镜像（应成功）
kubectl run test-signed --image=TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG \
    -n NAMESPACE --dry-run=server
# expected: pod created (server dry run)

# 尝试部署未签名镜像（应被拒绝）
kubectl run test-unsigned \
    --image=TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/unsigned-app:v1.0 \
    -n NAMESPACE --dry-run=server
# expected: admission webhook "policy.sigstore.dev" denied the request:
#   no valid signatures found

# 查看策略控制器日志
kubectl logs -n cosign-system -l app=policy-controller-webhook --tail=50
# expected: 含 signature validation 日志
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

```bash
# 删除准入策略
kubectl delete clusterimagepolicy require-cosign-signature
# expected: clusterimagepolicy deleted

# 卸载 Policy Controller
helm uninstall policy-controller -n cosign-system
kubectl delete ns cosign-system
# expected: namespace deleted
# ⚠️ 警告：删除 cosign-system 命名空间会清除所有策略配置
```

### 控制面（tccli）

```bash
# 清理本地密钥文件（可选）
rm -f cosign.key cosign.pub

# ⚠️ 警告：删除 cosign.key 后无法对已有镜像追加签名
# ⚠️ 警告：删除公钥后准入控制器将无法验证已有签名

# 删除 TCR 长期 Token
tccli tcr DeleteInstanceToken --region <Region> \
    --RegistryId INSTANCE_ID \
    --TokenId TOKEN_ID
# expected: exit 0
```

```bash
# 验证清理结果
cosign verify --key cosign.pub \
    TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG 2>&1
# expected: Error: no matching signatures (公钥已删除)
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| cosign sign 返回 401 | `cosign sign --key cosign.key IMAGE --verbose 2>&1 \| grep -i "unauthorized"` | TCR 认证失败，Token 无效或过期 | `tccli tcr CreateInstanceToken` 重新生成 Token，`docker login` 重新登录 |
| cosign sign 提示 manifest unknown | `docker manifest inspect TCR_INSTANCE_ID.tencentcloudcr.com/NAMESPACE/REPO_NAME:IMAGE_TAG` | 镜像不存在或 tag 错误 | `docker push` 确认镜像已推送，检查 tag 名称拼写 |
| cosign verify 返回 "no matching signatures" | `cosign tree IMAGE` 查看签名链 | 镜像未被签名、公钥不匹配或使用了不同密钥签名 | 确认使用签名时的对应公钥；若 Keyless 签名则省略 `--key` 参数 |
| 准入控制器未拦截未签名镜像 | `kubectl describe clusterimagepolicy POLICY_NAME` 查看 status | 策略 mode 为 `warn` 而非 `enforce` | 修改 ClusterImagePolicy 中 `spec.mode: enforce` |

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Policy Controller Pod CrashLoopBackOff | `kubectl describe pod -n cosign-system POD_NAME` | webhook TLS 证书未就绪或资源不足 | `kubectl logs -n cosign-system POD_NAME`，确认 cert-manager 已安装或增加资源 limit |
| 合法签名镜像被拒绝 | `kubectl describe clusterimagepolicy POLICY_NAME` 查看匹配规则 | images.glob 未覆盖该镜像 | 在 ClusterImagePolicy.spec.images 中添加对应的 glob 模式 |
| cosign.key 私钥丢失 | — | 私钥文件被删除或未备份 | 无法恢复：重新生成密钥对并重新签名所有镜像；⚠️ 生产环境建议使用 KMS 签名或 Keyless 模式 |
| Notation 签名证书过期 | `notation cert list` 查看证书有效期 | 测试证书有固定有效期 | `notation cert generate-test --default "new-cert"` 重新生成，并对镜像重新签名 |

## 下一步

- [Pod 安全组](../../安全/Pod%20安全组/tccli%20操作.md) -- page_id `80587`
- [在 TKE 中自定义 RBAC 授权](../../运维/在%20TKE%20中自定义%20RBAC%20授权/tccli%20操作.md) -- page_id `51683`
- [使用 cert-manager 签发免费证书](../使用%20cert-manager%20签发免费证书/tccli%20操作.md) -- page_id `76095`

## 控制台替代

[TCR 控制台](https://console.cloud.tencent.com/tcr) -> 镜像仓库 -> 选择仓库 -> 版本管理 -> 镜像安全扫描。
Cosign/Notation 签名为 CLI 原生操作，控制台仅支持查看镜像安全扫描结果，不支持直接签名。
