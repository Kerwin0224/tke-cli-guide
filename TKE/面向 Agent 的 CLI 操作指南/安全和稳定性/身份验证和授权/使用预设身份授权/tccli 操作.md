# 使用预设身份授权（tccli）

> 对照官方：[使用预设身份授权](https://cloud.tencent.com/document/product/457/46105) · page_id `46105`

## 概述

TKE 新版授权模式下，通过 `GrantFullClusterAccess` API 将 CAM 子账号关联到 RBAC 预设身份（ClusterRole），实现快速授权。预设身份覆盖管理员、运维、开发、只读四种典型场景，每个子账号获得独立的 x509 客户端证书。

控制台授权管理页面对集群主账号和集群创建者默认授予管理员权限，他们可为其他子账号配置权限。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达
- 已安装 tccli 并完成 `tccli configure`
- 当前账号为主账号或具备 `tke:admin` 权限（所需 Action：`tke:GrantFullClusterAccess`、`tke:DescribeUserPermissions`、`tke:DeleteUserPermissions`、`tke:DescribeClusterKubeconfig`、`tke:DescribeClusters`）
- 目标子账号 UIN `100012345678` 已存在且已获得 TKE 集群 CAM 权限（至少 `DescribeCluster`）
- 集群已升级到新版授权模式（RBAC）

### 环境检查

```bash
# 1. 确认集群存在且处于 Running 状态
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
# 2. 确认目标子账号存在
tccli cam ListUsers --region ap-guangzhou --output json
```

```json
{
    "Data": [
        {
            "Uin": 100012345678,
            "Name": "dev-user",
            "Uid": 100012345678
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 授予子账号预设身份 | `tccli tke GrantFullClusterAccess --ClusterId cls-xxxxxxxx --SubUin 100012345678 --region ap-guangzhou` | 是 |
| 查看已授权用户列表 | `tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 撤销子账号权限 | `tccli tke DeleteUserPermissions --ClusterId cls-xxxxxxxx --SubUin 100012345678 --region ap-guangzhou` | 否 |
| 获取子账号 kubeconfig | `tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |

## 操作步骤

### 预设身份一览

**全局范围（所有 Namespace）**：

| 身份 | ClusterRole | 权限说明 |
|------|-------------|----------|
| Admin（管理员） | `tke:admin` | 所有 Namespace 资源读写；集群节点、PV、Namespace、配额读写；可配置子账号权限 |
| Ops（运维） | `tke:ops` | 控制台可见资源全 Namespace 读写；集群节点、PV、Namespace、配额读写 |
| Dev（开发者） | `tke:dev` | 控制台可见资源全 Namespace 读写 |
| Read-Only（只读） | `tke:ro` | 控制台可见资源全 Namespace 只读 |

**指定 Namespace 范围**：

| 身份 | ClusterRole | 权限说明 |
|------|-------------|----------|
| Dev（命名空间开发者） | `tke:ns:dev` | 选定 Namespace 内控制台可见资源读写 |
| Read-Only（命名空间只读） | `tke:ns:ro` | 选定 Namespace 内控制台可见资源只读 |

### 步骤 1：授予预设身份

`GrantFullClusterAccess` 默认授予 `tke:admin` 权限。对于其他预设身份（`tke:ops`、`tke:dev`、`tke:ro`），需在控制台「RBAC 策略生成器」中选择身份类型。

```bash
tccli tke GrantFullClusterAccess \
    --ClusterId cls-xxxxxxxx \
    --SubUin 100012345678 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 步骤 2：获取子账号 kubeconfig

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

保存到本地：

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d > ~/.kube/config-cls-xxxxxxxx
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 验证

```bash
# 控制面：确认子账号已授权
tccli tke DescribeUs

```json
{
  "Permissions": [],
  "ClusterId": "<ClusterId>",
  "RoleName": "<RoleName>",
  "RoleType": "<RoleType>",
  "IsCustom": true,
  "Namespace": "<Namespace>",
  "TargetUin": "<TargetUin>",
  "RequestId": "<RequestId>"
}
```erPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
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
    "RequestId": "e5f6a7b8-c9d0-1234-efab-345678901234"
}
```

数据面（需 VPN/IOA 内网环境）：

```bash
# 子账号验证自身权限
kubectl auth can-i get nodes
# Admin: yes，Read-Only: yes（只读），Dev: no

kubectl auth can-i create deployments -n default
# Admin: yes，Dev: yes，Read-Only: no
```

## 清理

> **警告**：撤销子账号权限后，该子账号的 x509 证书将无法通过 API Server 认证，kubectl 操作返回 `Unauthorized`。不影响 CAM 层面的云 API 权限。

```bash
tccli tke DeleteUserPermissions \
    --ClusterId cls-xxxxxxxx \
    --SubUin 100012345678 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `GrantFullClusterAccess` 报 `InvalidParameter.SubUin` | `tccli cam ListUsers --region ap-guangzhou` 检查 UIN | 子账号 UIN 不存在或格式错误 | 使用 `cam ListUsers` 获取正确的数字 UIN |
| `GrantFullClusterAccess` 报权限不足 | `tccli cam ListAttachedUserPolicies --TargetUin <当前UIN> --region ap-guangzhou` | 当前账号不是集群主账号或 `tke:admin` 持有者 | 使用主账号或具备 `tke:admin` 权限的子账号操作 |
| `DescribeUserPermissions` 返回空 | 检查集群是否处于新授权模式 | 集群可能仍在旧授权模式（未升级 RBAC） | 在控制台「授权管理」页升级授权模式 |
| 子账号获取 kubeconfig 后 kubectl 报 `Unauthorized` | `wc -c ~/.kube/config-cls-xxxxxxxx` 检查文件完整性 | 证书缓存问题或 kubeconfig 文件保存不完整 | 删除 `~/.kube/cache` 后重新执行 `DescribeClusterKubeconfig` |
| 子账号只能看到集群但无法操作节点 | `kubectl describe clusterrolebinding 100012345678-ClusterRoleBinding` 检查身份级别 | `tke:dev` 和 `tke:ro` 没有节点管理权限 | 升级身份为 `tke:admin` 或 `tke:ops` |

## 下一步

- [自定义策略授权](../自定义策略授权/tccli%20操作.md) — page_id `46106`（Kubernetes 对象级 RBAC）
- [授权模式对比](../授权模式对比/tccli%20操作.md) — page_id `46107`
- [更新子账号的 TKE 集群访问凭证](../更新子账号的%20TKE%20集群访问凭证/tccli%20操作.md) — page_id `46108`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 `cls-xxxxxxxx` → **授权管理** → **ClusterRoleBinding** → 点击 **RBAC 策略生成器** → 勾选目标子账号 → **下一步** → 在「集群 RBAC 设置」中选择身份（Admin/Ops/Dev/Read-Only）→ 指定 Namespace 范围 → **完成**。
