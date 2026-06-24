# 更新子账号的 TKE 集群访问凭证（tccli）

> 对照官方：[更新子账号的 TKE 集群访问凭证](https://cloud.tencent.com/document/product/457/46108) · page_id `46108`

## 概述

在 TKE 新授权模式下，每个子账号拥有独立的 x509 客户端证书作为集群访问凭证。当证书存在泄露风险或需定期轮换时，可通过 **撤销当前权限（`DeleteUserPermissions`）→ 重新授权（`GrantFullClusterAccess`）→ 获取新 kubeconfig（`DescribeClusterKubeconfig`）** 三步流程更新凭证。

主账号和持有 `tke:admin` 的管理员可为其他子账号执行凭证更新。子账号自身无法直接更新自己的凭证。

本页的撤销/重新授权/获取 kubeconfig 全部通过 tccli 控制面完成。演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou）端点不可达，验证小节中涉及 kubectl 的数据面命令仅作预期说明，无法在本集群执行。演示子账号 UIN `100012345679`。

## 前置条件

- 已完成 [环境准备](../../../../../../环境准备.md)，tccli 已安装并配置凭据。
- 演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou），kubectl 不可达——撤销/重新授权/获取 kubeconfig 通过 tccli 完成；kubect 数据面验证需在端点可达的集群执行。
- 主账号 UIN `100012345678`（凭证更新执行者），目标子账号 UIN `100012345679`（已授权）。
- CAM 调用账号具备以下 Action 权限：`tke:DescribeClusters`、`tke:GrantFullClusterAccess`、`tke:DeleteUserPermissions`、`tke:DescribeClusterKubeconfig`、`tke:DescribeUserPermissions`。

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置
```

### 资源检查

```bash
# 3. 确认目标集群存在
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"
```

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.30.0"
        }
    ],
    "TotalCount": 1,
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 4. 确认目标子账号已授权
tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou
# expected: 返回已授权用户列表，含目标 100012345679
```

```json
{
    "Permissions": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "SubAccountUin": "100012345679",
            "PolicyType": "admin"
        }
    ],
    "TotalCount": 1,
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

```bash
# 5. 确认集群处于新授权模式
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d \
    | grep -c "client-certificate-data"
# expected: 1
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 获取当前 kubeconfig | `DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看子账号权限列表 | `DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 撤销子账号权限 | `DeleteUserPermissions --ClusterId cls-xxxxxxxx --SubUin 100012345679 --region ap-guangzhou` | 否 |
| 重新授权 | `GrantFullClusterAccess --ClusterId cls-xxxxxxxx --SubUin 100012345679 --region ap-guangzhou` | 是 |
| 获取新 kubeconfig | `DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 凭证更新（控制台按钮） | 控制台独有；CLI 通过 撤销+重新授权 三步实现等效效果 | — |

## 操作步骤

### 步骤 1：获取当前 kubeconfig（旧凭证）

保存旧凭证用于后续对比验证：

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d \
    > ~/.kube/config-cls-xxxxxxxx-old
# expected: 文件保存成功，非空

# 查看旧证书特征
grep "client-certificate-data" ~/.kube/config-cls-xxxxxxxx-old
# expected: 包含 client-certificate-data 字段
```

**预期输出**（DescribeClusterKubeconfig）：

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 步骤 2：撤销子账号权限（使当前凭证失效）

#### 选择依据

- **撤销即失效**：`DeleteUserPermissions` 执行后，该子账号的 x509 证书立即无法通过 API Server 认证，任何使用旧证书的 kubectl 操作将返回 `Unauthorized`。
- **不可逆**：撤销不会自动备份当前权限配置。确认记录子账号当前的角色信息后再执行。

```bash
tccli tke DeleteUserPermissions \
    --ClusterId cls-xxxxxxxx \
    --SubUin 100012345679 \
    --region ap-guangzhou
# expected: exit 0，返回 RequestId
```

```json
{
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

验证撤销生效（需端点可达）：

```bash
# 演示集群 cls-xxxxxxxx 端点不可达，请在端点可达的集群执行
kubectl --kubeconfig ~/.kube/config-cls-xxxxxxxx-old get nodes
# expected: error: You must be logged in to the server (Unauthorized)
```

### 步骤 3：重新授权子账号（生成新凭证）

`GrantFullClusterAccess` 将子账号重新绑定到 `tke:admin` ClusterRole，TKE 为该子账号签发新的 x509 客户端证书。

```bash
tccli tke GrantFullClusterAccess \
    --ClusterId cls-xxxxxxxx \
    --SubUin 100012345679 \
    --region ap-guangzhou
# expected: exit 0，返回 RequestId
```

```json
{
    "RequestId": "e5f6a7b8-c9d0-1234-efab-234567890abc"
}
```

### 步骤 4：获取新 kubeconfig

重新授权后，调用 `DescribeClusterKubeconfig` 获取包含新证书的 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d \
    > ~/.kube/config-cls-xxxxxxxx
# expected: 文件保存成功，非空，约 5-7 KB
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

### 步骤 5：新旧证书对比

```bash
# 提取旧证书的 client-certificate-data
OLD_CERT=$(grep "client-certificate-data" ~/.kube/config-cls-xxxxxxxx-old \
    | awk '{print $2}')
# 提取新证书的 client-certificate-data
NEW_CERT=$(grep "client-certificate-data" ~/.kube/config-cls-xxxxxxxx \
    | awk '{print $2}')
# 对比
[ "$OLD_CERT" != "$NEW_CERT" ] && echo "CERTIFICATES DIFFER — rotation successful" \
    || echo "WARNING: certificates are identical"
# expected: CERTIFICATES DIFFER — rotation successful
```

## 验证

### 控制面（tccli）

```bash
# 1. 确认撤销后子账号从权限列表移除（步骤 2 后）；重新出现在列表中（步骤 3 后）
tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou
# expected: 步骤 3 后，100012345679 重新出现在列表中
```

```json
{
    "Permissions": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "SubAccountUin": "100012345679",
            "PolicyType": "admin"
        }
    ],
    "TotalCount": 1,
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-34567890abcd"
}
```

```bash
# 2. 确认新 kubeconfig 文件大小正常
wc -c ~/.kube/config-cls-xxxxxxxx
# expected: 约 5000-7000 字节
```

### 数据面（kubectl）

> 需端点可达，演示集群 `cls-xxxxxxxx` 不可达，以下为预期说明：

```bash
# 3. 旧凭证不可用（需端点可达）
kubectl --kubeconfig ~/.kube/config-cls-xxxxxxxx-old get nodes
# expected: error: Unauthorized

# 4. 新凭证可用（需端点可达）
kubectl --kubeconfig ~/.kube/config-cls-xxxxxxxx get nodes
# expected: 返回节点列表（正常）
```

## 清理

> **注意**：凭证更新完成后，旧 kubeconfig 中的证书已永久失效。删除旧文件避免混淆。

### 清理前确认

```bash
# 确认新 kubeconfig 可用（需端点可达，演示集群不可达）
kubectl --kubeconfig ~/.kube/config-cls-xxxxxxxx cluster-info
# expected: Kubernetes control plane is running

# 确认新旧证书不同
grep "client-certificate-data" ~/.kube/config-cls-xxxxxxxx-old
grep "client-certificate-data" ~/.kube/config-cls-xxxxxxxx
# expected: 两行的 base64 值不同
```

### 删除旧 kubeconfig

```bash
rm -f ~/.kube/config-cls-xxxxxxxx-old
# expected: 文件已删除
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterKubeconfig` 返回空 kubeconfig | 检查子账号权限：`tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 该子账号尚未授予集群访问权限 | 先执行 `GrantFullClusterAccess` 授权 |
| `DeleteUserPermissions` 返回 `UnauthorizedOperation.CamNoAuth` | 检查当前账号权限：`tccli cam ListAttachedUserPolicies --TargetUin 100012345678 --region ap-guangzhou` | 当前账号不是主账号或 `tke:admin` 持有者（此为环境限制，非命令错误） | 使用主账号或具备 `tke:admin` 权限的子账号执行 |
| `GrantFullClusterAccess` 返回 `InvalidParameter.SubUin` | 检查 SUB_UIN：`tccli cam ListUsers --region ap-guangzhou` | SUB_UIN 不存在或格式错误 | 使用 `cam ListUsers` 获取正确的数字 UIN |
| `GrantFullClusterAccess` 在撤销后立即调用失败 | 等待 5 秒后重试 | 权限系统有短暂的状态同步延迟 | 在 `DeleteUserPermissions` 返回成功后等待 5-10 秒再调用 `GrantFullClusterAccess` |
| kubectl 无法连接 APIServer | `tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou` 检查 PgwEndpoint | 集群 APIServer 端点不可达（如演示集群 cls-xxxxxxxx） | 在端点可达的集群执行 kubectl；或通过控制台内嵌 Cloud Shell |

### 凭证更新后异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 撤销后子账号仍能访问集群 | 检查 kubeconfig 文件路径：确认 `--kubeconfig` 参数指向旧文件 | 使用了缓存的旧凭证或错误的 kubeconfig 文件 | 等待 10-30 秒后重试；确认使用了正确的 kubeconfig 路径 |
| 重新授权后 kubectl 仍返回 `Unauthorized` | 确认是否获取了新 kubeconfig：`ls -la ~/.kube/config-cls-xxxxxxxx` | 仍在用旧 kubeconfig（证书已随 DeleteUserPermissions 失效） | 重新执行 `DescribeClusterKubeconfig` 获取新凭证文件 |
| 新旧证书 base64 值完全一致 | 检查操作顺序：是否在 `DeleteUserPermissions` 和 `GrantFullClusterAccess` 之间调用了 `DescribeClusterKubeconfig` | 证书在重新授权后才更新，撤销后未重新授权前获取的仍是旧证书 | 确认已执行完整的 撤销 → 重新授权 → 获取 三步流程 |

## 下一步

- [使用预设身份授权](../使用预设身份授权/tccli%20操作.md) — GrantFullClusterAccess 详解
- [自定义策略授权](../自定义策略授权/tccli%20操作.md) — kubectl 自定义 Role/ClusterRole
- [授权模式对比](../授权模式对比/tccli%20操作.md) — 新旧授权模式差异

## 控制台替代

[TKE 控制台 → 集群详情 → 基本信息](https://console.cloud.tencent.com/tke2/cluster) → **集群凭证** → 点击 **更新** 按钮完成凭证轮换。控制台「更新」操作本质上是撤销旧权限 + 重新授权 + 返回新凭证的组合。
