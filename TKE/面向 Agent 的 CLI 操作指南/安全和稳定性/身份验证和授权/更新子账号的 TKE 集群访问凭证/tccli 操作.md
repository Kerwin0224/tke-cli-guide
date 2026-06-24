# 更新子账号的 TKE 集群访问凭证（tccli）

> 对照官方：[更新子账号的 TKE 集群访问凭证](https://cloud.tencent.com/document/product/457/46108) · page_id `46108`

## 概述

在 TKE 的新版授权模型下，每个子账号拥有独立的 x509 客户端证书作为集群访问凭证。当子账号证书存在泄露风险或需要定期轮换时，可通过撤销当前权限并重新授权来更新访问凭证。主账号和有 `tke:admin` 权限的管理员可执行此操作。

控制台「更新」按钮本质上是撤销旧权限 + 重新授权，CLI 中通过 `DeleteUserPermissions` + `GrantFullClusterAccess` 两步实现等效效果。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达
- 已安装 tccli 并完成 `tccli configure`
- 当前账号为主账号或具备 `tke:admin` 权限（所需 Action：`tke:GrantFullClusterAccess`、`tke:DeleteUserPermissions`、`tke:DescribeClusterKubeconfig`、`tke:DescribeUserPermissions`）
- 目标子账号 UIN `100012345678` 已通过 `GrantFullClusterAccess` 获得集群访问权限

### 环境检查

```bash
# 1. 确认目标集群存在且处于新授权模式
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
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 2. 确认目标子账号已授权
tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "Permissions": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "SubUin": 100012345678,
            "Role": "tke:admin",
            "PolicyType": "preset"
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取当前 kubeconfig | `tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看子账号权限列表 | `tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 撤销子账号权限 | `tccli tke DeleteUserPermissions --ClusterId cls-xxxxxxxx --SubUin 100012345678 --region ap-guangzhou` | 否 |
| 重新授权 | `tccli tke GrantFullClusterAccess --ClusterId cls-xxxxxxxx --SubUin 100012345678 --region ap-guangzhou` | 是 |
| 凭证更新（控制台按钮） | 控制台独有，CLI 通过 撤销+重新授权 实现等效效果 | — |

## 操作步骤

### 步骤 1：获取当前子账号 kubeconfig

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 步骤 2：撤销子账号权限（使当前凭证失效）

撤销权限后，该子账号的 x509 证书将无法通过 API Server 认证。

```bash
tccli tke DeleteUserPermissions \
    --ClusterId cls-xxxxxxxx \
    --SubUin 100012345678 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

### 步骤 3：重新授权子账号（生成新凭证）

重新授权后，TKE 将为该子账号生成新的 x509 客户端证书。

```bash
tccli tke GrantFullClusterAccess \
    --ClusterId cls-xxxxxxxx \
    --SubUin 100012345678 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "e5f6a7b8-c9d0-1234-efab-345678901234"
}
```

### 步骤 4：获取新 kubeconfig

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d > ~/.kube/config-cls-xxxxxxxx
# expected: 文件保存成功，非空
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 验证

数据面（需 VPN/IOA 内网环境）：

```bash
# 1. 确认撤销后旧凭证不可用
kubectl --kubeconfig ~/.kube/config-old get nodes
# expected: error: You must be logged in to the server (Unauthorized)

# 2. 获取新 kubeconfig 后验证
kubectl --kubeconfig ~/.kube/config-cls-xxxxxxxx get nodes
# expected: 返回节点列表（正常）

# 3. 验证新证书与旧证书不同
diff <(kubectl --kubeconfig ~/.kube/config-old config view --raw -o json \
    | jq '.users[0].user."client-certificate-data"') \
    <(kubectl --kubeconfig ~/.kube/config-cls-xxxxxxxx config view --raw -o json \
    | jq '.users[0].user."client-certificate-data"')
# expected: 两行不同（证书已更新）
```

## 清理

> **警告**：此操作仅撤销指定子账号对集群的访问权限（使其 x509 证书失效），不删除子账号本身。如需恢复访问，需重新执行 `GrantFullClusterAccess`。

```bash
# 测试完成后撤销测试用户权限
tccli tke DeleteUserPermissions \
    --ClusterId cls-xxxxxxxx \
    --SubUin 100012345678 \
    --region ap-guangzhou \
    --output json
```

清理旧 kubeconfig：

```bash
# 删除旧 kubeconfig 文件（避免混淆）
rm -f ~/.kube/config-old
# expected: 文件已删除
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterKubeconfig` 返回空 kubeconfig | `tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` 检查子账号权限 | 该子账号尚未授予集群访问权限 | 先执行 `GrantFullClusterAccess` 授权 |
| `DeleteUserPermissions` 报权限不足 | `tccli cam ListAttachedUserPolicies --TargetUin <当前UIN> --region ap-guangzhou` 检查权限 | 需主账号或拥有 `tke:admin` 权限的子账号 | 使用主账号或 `tke:admin` 身份操作 |
| `GrantFullClusterAccess` 报 `InvalidParameter.SubUin` | `tccli cam ListUsers --region ap-guangzhou` 检查 UIN | SUB_UIN 格式错误或子账号不存在 | 使用 `cam ListUsers` 获取正确的数字 UIN |
| 撤销后子账号仍能访问集群 | 检查 kubeconfig 文件路径是否正确 | 权限生效有短暂延迟（秒级），或使用了缓存的旧凭证 | 等待 10-30 秒后重试；确认使用了正确的 kubeconfig 文件 |
| 重新授权后 kubectl 仍报 `Unauthorized` | 确认是否重新获取了 kubeconfig | 仍使用了旧 kubeconfig（证书已随 DeleteUserPermissions 失效） | 重新执行 `DescribeClusterKubeconfig` 获取新凭证 |

## 下一步

- [使用预设身份授权](../使用预设身份授权/tccli%20操作.md) — page_id `46105`
- [自定义策略授权](../自定义策略授权/tccli%20操作.md) — page_id `46106`（Kubernetes 对象级）
- [授权模式对比](../授权模式对比/tccli%20操作.md) — page_id `46107`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 `cls-xxxxxxxx` → **基本信息** → **集群凭证** → 点击「更新」按钮完成凭证轮换。控制台「更新」操作本质上是撤销旧权限 + 重新授权，CLI 中通过 `DeleteUserPermissions` + `GrantFullClusterAccess` 两步实现等效效果。
