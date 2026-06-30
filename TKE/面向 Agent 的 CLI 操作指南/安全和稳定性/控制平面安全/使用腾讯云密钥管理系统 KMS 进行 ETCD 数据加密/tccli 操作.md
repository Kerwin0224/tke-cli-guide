# 使用腾讯云密钥管理系统 KMS 进行 ETCD 数据加密（tccli）

> 对照官方：[使用腾讯云密钥管理系统 KMS 进行 ETCD 数据加密](https://cloud.tencent.com/document/product/457/45594) · page_id `45594`

## 概述

TKE 支持通过腾讯云密钥管理系统（KMS）对 etcd 中存储的 Secret 数据进行信封加密（Envelope Encryption）。启用后，Secret 数据先由随机生成的数据加密密钥（DEK）加密，DEK 再由 KMS 主密钥（CMK）加密后一并存储于 etcd。

启用加密后，仅**新建和更新**的 Secret 会被加密，已有 Secret 需通过更新触发重新加密。

> **关键风险**：一旦用于加密的 KMS 密钥被禁用或删除，集群 API Server 将无法解密 etcd 中的 Secret，导致 API Server 不可用。生产环境启用后切勿删除或禁用对应的 KMS 密钥。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，本页 kubectl 命令需在端点可达的环境中执行
- 完成 [环境准备](../../../环境准备.md)
- 当前账号具备 `tke:EnableEncryptionProtection`、`tke:DescribeEncryptionStatus`、`tke:DisableEncryptionProtection`、`tke:DescribeClusters`、`kms:ListKeys`、`kms:CreateKey`、`kms:DescribeKey`、`kms:DisableKey` 权限
- 集群 Kubernetes 版本 >= 1.18

### 环境检查

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 2. 查询当前 etcd 加密状态
tccli tke DescribeEncryptionStatus --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "Status": "Closed",
    "RequestId": "b1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

`Status` 为 `Closed`（未启用）或 `Opened`（已启用）。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看 KMS 密钥列表 | `tccli kms ListKeys --region ap-guangzhou` | 是 |
| 查看密钥详情 | `tccli kms DescribeKey --KeyId <KeyId> --region ap-guangzhou` | 是 |
| 创建 KMS 密钥 | `tccli kms CreateKey --Alias <name> --KeyUsage ENCRYPT_DECRYPT --region ap-guangzhou` | 否 |
| 启用 etcd 加密 | `tccli tke EnableEncryptionProtection --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 否 |
| 查看加密状态 | `tccli tke DescribeEncryptionStatus --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 关闭 etcd 加密 | `tccli tke DisableEncryptionProtection --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 否 |
| Secret 读写验证 | `kubectl create secret` / `kubectl get secret`（需端点可达） | — |

### 关键字段说明

`EnableEncryptionProtection` 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 集群不存在或状态非 Running → `FailedOperation.ClusterStateNotRunning` |
| `KMSConfiguration.KeyId` | String | 是 | KMS 密钥 ID，`KeyUsage` 须为 `ENCRYPT_DECRYPT` | 密钥不存在 → `InvalidParameter.KMSKeyNotExists`；密钥已禁用 → `InvalidParameter.KMSKeyStateError` |
| `KMSConfiguration.Region` | String | 是 | 密钥所在地域，须与集群 Region 一致 | Region 不匹配 → `FailedOperation.KmsOperationFailed` |

## 操作步骤

### 步骤 1：列出可用的 KMS 密钥

```bash
tccli kms ListKeys --region ap-guangzhou --output json
```

```json
{
    "Keys": [
        {"KeyId": "kms-myau6jjq"},
        {"KeyId": "kms-example-002"}
    ],
    "TotalCount": 2,
    "RequestId": "c1c2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 步骤 2：查看目标密钥详情

确认密钥状态为 `Enabled`，`KeyUsage` 为 `ENCRYPT_DECRYPT`：

```bash
tccli kms DescribeKey --KeyId kms-myau6jjq --region ap-guangzhou --output json
```

```json
{
    "KeyMetadata": {
        "KeyId": "kms-myau6jjq",
        "Alias": "tke-etcd-encryption-key",
        "KeyState": "Enabled",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "Type": 1,
        "CreateTime": 1700000000,
        "Description": "Key for TKE etcd encryption"
    },
    "RequestId": "d1d2d3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 步骤 3：（可选）创建新的 KMS 密钥

若无可用密钥，需新建。`KeyUsage` 必须选 `ENCRYPT_DECRYPT`；KMS 密钥地域必须与 TKE 集群所在地域一致。

```bash
tccli kms CreateKey --region ap-guangzhou \
    --Alias "tke-etcd-encryption-key" \
    --Description "Key for TKE etcd Secret encryption" \
    --KeyUsage "ENCRYPT_DECRYPT" --output json
```

```json
{
    "KeyId": "kms-myau6jjq",
    "Alias": "tke-etcd-encryption-key",
    "CreateTime": 1700000000,
    "RequestId": "e1e2e3e4-f5f6-7890-abcd-ef1234567890"
}
```

### 步骤 4：对集群启用 etcd 加密

`KMSConfiguration` 必须同时指定 `KeyId` 和 `Region`，Region 须与密钥实际所在地域一致。启用加密后仅新建和更新的 Secret 被加密，已有 Secret 不回溯加密。

`enable-kms-minimal.json`：

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "KMSConfiguration": {
        "KeyId": "kms-myau6jjq",
        "Region": "ap-guangzhou"
    }
}
```

```bash
cat > enable-kms-minimal.json << 'EOF'
{
    "ClusterId": "cls-xxxxxxxx",
    "KMSConfiguration": {
        "KeyId": "kms-myau6jjq",
        "Region": "ap-guangzhou"
    }
}
EOF

tccli tke EnableEncryptionProtection --region ap-guangzhou \
    --cli-input-json file://enable-kms-minimal.json --output json
```

```json
{
    "RequestId": "f1f2f3f4-a5b6-7890-abcd-ef1234567890"
}
```

### 步骤 5：轮询确认加密启用

etcd 加密启用是异步操作。轮询直到状态为 `Opened`，最长等待约 5 分钟。

```bash
tccli tke DescribeEncryptionStatus --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "Status": "Opened",
    "KMSConfiguration": {
        "KeyId": "kms-myau6jjq",
        "Region": "ap-guangzhou"
    },
    "RequestId": "a1a2a3a4-b5b6-7890-abcd-ef1234567890"
}
```

验证维度：

| 维度 | 检查方法 | 预期 |
|------|---------|------|
| 加密状态 | `DescribeEncryptionStatus` 的 `Status` 字段 | `Opened` |
| 密钥一致性 | 对比 `KMSConfiguration.KeyId` 与创建时指定的密钥 ID | 一致 |
| 地域一致性 | 对比 `KMSConfiguration.Region` 与密钥所在地域 | 一致 |
| 集群可访问 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | `ClusterStatus` 仍为 `Running` |

### 步骤 6：验证数据面加密效果（需端点可达）

```bash
# 创建测试 Secret
kubectl create secret generic test-kms-secret \
    --from-literal=secret-key=secret-value

# 确认 Secret 可正常读取（透明解密验证）
kubectl get secret test-kms-secret -o yaml

# 读取并解码 secret 值
kubectl get secret test-kms-secret -o jsonpath='{.data.secret-key}' \
    | base64 -d
```

```text
NAME  STATUS  AGE
...
```

```output
secret-value
```

## 验证

### 控制面（tccli）

```bash
# 确认加密状态保持 Opened
tccli tke DescribeEncryptionStatus --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "Status": "Opened",
    "KMSConfiguration": {
        "KeyId": "kms-myau6jjq",
        "Region": "ap-guangzhou"
    },
    "RequestId": "b1b2b3b4-c5c6-7890-abcd-ef1234567890"
}
```

```bash
# 确认密钥状态正常
tccli kms DescribeKey --KeyId kms-myau6jjq --region ap-guangzhou --output json
```

```json
{
    "KeyMetadata": {
        "KeyId": "kms-myau6jjq",
        "Alias": "tke-etcd-encryption-key",
        "KeyState": "Enabled",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "Type": 1,
        "CreateTime": 1700000000,
        "Description": "Key for TKE etcd encryption"
    },
    "RequestId": "c1c2c3c4-d5d6-7890-abcd-ef1234567890"
}
```

```bash
# 确认集群仍可正常访问
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "d1d2d3d4-e5e6-7890-abcd-ef1234567890"
}
```

### 数据面（kubectl，需端点可达）

```bash
# Secret 读写正常，表明加密层透明
kubectl get secret test-kms-secret

# 删除再重建 Secret，确认加密对新 Secret 生效
kubectl delete secret test-kms-secret
kubectl create secret generic test-kms-secret \
    --from-literal=verify-key=verify-value
kubectl get secret test-kms-secret -o jsonpath='{.data.verify-key}' \
    | base64 -d
```

```text
NAME  STATUS  AGE
...
```

```output
verify-value
```

## 清理

> **致命警告**：禁用或删除用于 etcd 加密的 KMS 密钥将导致集群 API Server 立即不可用。以下清理步骤仅适用于测试环境，且仅在确认不再需要加密功能时执行。生产环境切勿执行本节的清理操作。

### Pre-cleanup state check

```bash
# 确认加密状态
tccli tke DescribeEncryptionStatus --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "Status": "Opened",
    "KMSConfiguration": {
        "KeyId": "kms-myau6jjq",
        "Region": "ap-guangzhou"
    },
    "RequestId": "e1e2e3e4-f5f6-7890-abcd-ef1234567890"
}
```

### 数据面清理（需端点可达）

```bash
kubectl delete secret test-kms-secret --ignore-not-found
```

### 控制面清理

```bash
# 关闭 etcd 加密（仅测试环境）
tccli tke DisableEncryptionProtection --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "RequestId": "f1f2f3f4-a5b6-7890-abcd-ef1234567890"
}
```

关闭后，新创建的 Secret 不再加密，已加密的 Secret 仍保持加密状态直到被更新或重建。

```bash
# 验证加密已关闭
tccli tke DescribeEncryptionStatus --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "Status": "Closed",
    "RequestId": "a1a2a3a4-b5b6-7890-abcd-ef1234567890"
}
```

### 禁用 KMS 密钥（仅测试环境）

> 仅在确认 `DisableEncryptionProtection` 已完成（加密已关闭）后执行。

```bash
tccli kms DisableKey --KeyId kms-myau6jjq --region ap-guangzhou --output json
```

```json
{
    "RequestId": "b1b2b3b4-c5c6-7890-abcd-ef1234567890"
}
```

```bash
# 验证密钥已禁用
tccli kms DescribeKey --KeyId kms-myau6jjq --region ap-guangzhou --output json
```

```json
{
    "KeyMetadata": {
        "KeyId": "kms-myau6jjq",
        "Alias": "tke-etcd-encryption-key",
        "KeyState": "Disabled",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "Type": 1,
        "CreateTime": 1700000000,
        "Description": "Key for TKE etcd encryption"
    },
    "RequestId": "c1c2c3c4-d5d6-7890-abcd-ef1234567890"
}
```

```bash
# 清理 JSON 文件
rm -f enable-kms-minimal.json
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableEncryptionProtection` 返回 `InvalidParameter.KMSKeyNotExists` | `tccli kms DescribeKey --KeyId kms-myau6jjq --region ap-guangzhou` 确认密钥存在 | 指定的 KMS 密钥不存在或不在当前 Region | 用 `tccli kms ListKeys --region ap-guangzhou` 列出正确地域的密钥 |
| `EnableEncryptionProtection` 返回 `InvalidParameter.KMSKeyStateError` | 查看 `KeyState` 字段 | 密钥状态非 `Enabled`（已禁用或待删除） | 通过 `tccli kms EnableKey --KeyId kms-myau6jjq --region ap-guangzhou` 启用密钥 |
| `EnableEncryptionProtection` 返回 `FailedOperation.KmsOperationFailed` | 检查 `KMSConfiguration.Region` 与密钥实际 Region 是否一致 | KMSConfiguration.Region 与密钥所在地域不匹配 | 将 KMSConfiguration.Region 改为密钥实际所在地域 |
| `EnableEncryptionProtection` 返回 `FailedOperation.ClusterStateNotRunning` | 查看 `ClusterStatus` 字段 | 集群未处于 Running 状态 | 等待集群状态变为 Running 后重试 |
| 启用加密后 API Server 不可用 | `tccli kms DescribeKey --KeyId kms-myau6jjq --region ap-guangzhou` 检查 `KeyState` | KMS 密钥被误禁用或删除 | **紧急**：在 KMS 控制台重新启用密钥（`kms EnableKey`）。若密钥被删除，[提交工单](https://console.cloud.tencent.com/workorder) 申请恢复 |
| `DescribeEncryptionStatus` 长时间显示非 `Opened` | 等待后重试；检查集群状态 | 加密启用为异步操作，可能因集群并发操作冲突 | 等待集群空闲，超过 10 分钟仍未变化则登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看详细状态 |
| 启用加密后 Secret 读取失败 | `kubectl get secret test-kms-secret -o yaml` 检查错误；确认密钥状态 | KMS 密钥状态异常或密钥权限不足 | 确认 KeyState 为 `Enabled`；检查 CAM 策略中是否有 `kms:Decrypt` 权限 |

## 下一步

- [KMS 密钥管理官方文档](https://cloud.tencent.com/document/product/573) — KMS 密钥生命周期、轮换、审计
- [策略管理（OPA/Gatekeeper）](../../应用安全/策略管理/tccli%20操作.md) — 基于 Gatekeeper 的应用安全策略
- [容器服务安全组设置](../../容器服务安全组设置/tccli%20操作.md) — 集群网络层安全
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853) — 完整 API 列表

## 控制台替代

[TKE 控制台 → 集群详情 → 基本信息](https://console.cloud.tencent.com/tke2/cluster) → **etcd 加密**：开启/关闭加密，选择或创建 KMS 密钥。控制台操作前会弹出二次确认弹窗提示风险。
