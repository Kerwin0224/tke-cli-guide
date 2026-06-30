# 使用预设身份授权（tccli）

> 对照官方：[使用预设身份授权](https://cloud.tencent.com/document/product/457/46105) · page_id `46105`

## 概述

TKE 提供 RBAC 预设身份（ClusterRole），通过 `GrantFullClusterAccess` API 将 CAM 子账号关联到预设角色，实现快速授权。预设身份覆盖管理员（`tke:admin`）、运维（`tke:ops`）、开发（`tke:dev`）、只读（`tke:ro`）四种典型场景。

授权后子账号可通过 `DescribeClusterKubeconfig` 获取包含独立 x509 证书的 kubeconfig，用 kubectl 访问集群资源（需端点可达）。

本页授权操作全部通过 tccli 控制面完成。演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou）的 APIServer 端点不可达，验证小节中涉及 kubectl 的数据面命令仅作预期说明，无法在本集群执行。演示子账号 UIN `100012345679`。

## 前置条件

- 已完成 [环境准备](../../../../环境准备.md)，tccli 已安装并配置凭据。
- 演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou），kubectl 不可达——授权动作通过 tccli 完成，kubect 验证需在端点可达的集群执行。
- 主账号 UIN `100012345678`，目标子账号 UIN `100012345679`（已存在于 CAM）。
- CAM 调用账号具备以下 Action 权限：`tke:DescribeClusters`、`tke:GrantFullClusterAccess`、`tke:DescribeUserPermissions`、`tke:DeleteUserPermissions`、`tke:DescribeClusterKubeconfig`、`cam:ListUsers`。

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
# 3. 确认目标集群存在且运行中
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
# 4. 确认子账号存在
tccli cam ListUsers --region ap-guangzhou
# expected: 返回用户列表，确认目标子账号 UIN 100012345679
```

```bash
# 5. 确认集群处于新授权模式
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d \
    | grep -c "client-certificate-data"
# expected: 1（新授权模式）
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-34567890abcd"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 授予子账号预设身份 | `GrantFullClusterAccess --ClusterId cls-xxxxxxxx --SubUin 100012345679 --region ap-guangzhou` | 是 |
| 查看已授权用户 | `DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 撤销子账号权限 | `DeleteUserPermissions --ClusterId cls-xxxxxxxx --SubUin 100012345679 --region ap-guangzhou` | 否 |
| 获取子账号 kubeconfig | `DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看 ClusterRoleBinding 列表 | `kubectl get clusterrolebindings -l cloud.tencent.com/tke-account`（需端点可达） | 是 |

## 操作步骤

### 步骤 1：预设身份一览

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

所有预设 ClusterRole 均携带标签 `cloud.tencent.com/tke-rbac-generated: "true"`。所有预设 ClusterRoleBinding 均携带注解 `cloud.tencent.com/tke-account-nickname` 和标签 `cloud.tencent.com/tke-account`。

### 步骤 2：授予预设身份

#### 选择依据

- **`GrantFullClusterAccess` 默认授予 `tke:admin`**。如需其他预设身份（`tke:ops`、`tke:dev`、`tke:ro`），需在控制台「RBAC 策略生成器」中选择身份类型（API 不支持直接指定预设角色名）。
- **最小权限原则**：开发人员用 `tke:dev`，只读人员用 `tke:ro`，运维用 `tke:ops`。避免对所有子账号授予 `tke:admin`。
- **幂等性**：对已授权的子账号重复调用 `GrantFullClusterAccess` 不会报错，操作安全。

```bash
tccli tke GrantFullClusterAccess \
    --ClusterId cls-xxxxxxxx \
    --SubUin 100012345679 \
    --region ap-guangzhou
# expected: exit 0，返回 RequestId
```

```json
{
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

### 步骤 3：获取子账号 kubeconfig

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou
# expected: exit 0，返回 base64 编码的 Kubeconfig
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

保存到本地文件：

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d \
    > ~/.kube/config-cls-xxxxxxxx
# expected: 文件非空，约 5-7 KB
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### 步骤 4：查看已授权用户（需端点可达）

> 演示集群 `cls-xxxxxxxx` 端点不可达，以下 kubectl 命令需在端点可达的集群执行：

```bash
# 列出所有 TKE 管理的 ClusterRoleBinding
kubectl get clusterrolebindings -l cloud.tencent.com/tke-account
# expected: 返回已授权的子账号绑定列表
```

```text
NAME                              ROLE                  AGE
100012345678-ClusterRoleBinding   ClusterRole/tke:admin   5m
100012345679-ClusterRoleBinding   ClusterRole/tke:admin   1m
```

## 验证

### 控制面（tccli）

```bash
# 验证 TKE 侧权限记录
tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou
# expected: exit 0，返回已授权用户列表，目标 100012345679 在其中
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
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

```bash
# 验证 kubeconfig 文件非空且包含证书
wc -c ~/.kube/config-cls-xxxxxxxx
# expected: 文件大小约 5000-7000 字节

grep -c "client-certificate-data" ~/.kube/config-cls-xxxxxxxx
# expected: 1
```

### 数据面（kubectl）

> 需端点可达，演示集群 `cls-xxxxxxxx` 不可达，以下为预期说明：

```bash
# 子账号验证自身权限（需端点可达）
kubectl --kubeconfig ~/.kube/config-cls-xxxxxxxx auth can-i get nodes
# expected: Admin: yes，Ops: yes，Dev: no，Read-Only: yes

kubectl --kubeconfig ~/.kube/config-cls-xxxxxxxx auth can-i create deployments -n default
# expected: Admin: yes，Ops: yes，Dev: yes，Read-Only: no

kubectl --kubeconfig ~/.kube/config-cls-xxxxxxxx auth can-i delete nodes
# expected: Admin: yes，Ops: yes，Dev: no，Read-Only: no
```

## 清理

> **警告**：撤销子账号权限后，该子账号的 x509 证书将无法通过 API Server 认证，kubectl 操作返回 `Unauthorized`。不影响 CAM 层面的云 API 权限。

### 清理前状态检查

```bash
# 确认当前已授权用户列表
tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou
# expected: 确认目标 100012345679 在列表中
```

```json
{
  "Permissions": "<Permissions>",
  "ClusterId": "<ClusterId>",
  "RoleName": "<RoleName>",
  "RoleType": "<RoleType>",
  "IsCustom": "<IsCustom>",
  "Namespace": "<Namespace>"
}
```

### 控制面（tccli）

```bash
tccli tke DeleteUserPermissions \
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

### 数据面

```bash
# 清理本地 kubeconfig 文件
rm -f ~/.kube/config-cls-xxxxxxxx
# expected: 文件已删除
```

### 验证已撤销

```bash
# 控制面验证
tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou
# expected: 目标 100012345679 不再出现在列表中
```

```json
{
    "Permissions": [],
    "TotalCount": 0,
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-34567890abcd"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `GrantFullClusterAccess` 返回 `InvalidParameter.SubUin` | 检查子账号 UIN：`tccli cam ListUsers --region ap-guangzhou` | 子账号 UIN 不存在或格式错误 | 使用 `cam ListUsers` 获取正确的数字 UIN |
| `GrantFullClusterAccess` 返回 `UnauthorizedOperation.CamNoAuth` | 检查当前账号权限：`tccli cam ListAttachedUserPolicies --TargetUin 100012345678 --region ap-guangzhou` | 当前账号不是集群主账号或 `tke:admin` 持有者（此为环境限制，非命令错误） | 使用主账号或具备 `tke:admin` 权限的子账号操作 |
| `DescribeUserPermissions` 返回空列表 | 检查集群是否处于新授权模式 | 集群可能仍在旧授权模式 | 在控制台「授权管理」页升级授权模式 |
| `DescribeClusterKubeconfig` 返回空 kubeconfig | 检查子账号权限：`tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 该子账号尚未被授权 | 先执行 `GrantFullClusterAccess` 授权 |

### 授权后不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 子账号获取 kubeconfig 后 kubectl 仍返回 `Unauthorized` | 检查 kubeconfig 文件完整性：`wc -c ~/.kube/config-cls-xxxxxxxx` 和 `grep "client-certificate-data" ~/.kube/config-cls-xxxxxxxx` | 证书缓存或文件保存不完整 | 删除 `~/.kube/cache` 后重试；重新执行 `DescribeClusterKubeconfig` 获取新文件 |
| 子账号只能查看集群但无法操作节点 | 检查绑定的角色：`kubectl describe clusterrolebinding 100012345679-ClusterRoleBinding` | `tke:dev` 和 `tke:ro` 没有节点管理权限 | 若确实需要节点管理权限，使用控制台 RBAC 策略生成器改为 `tke:admin` 或 `tke:ops` |
| kubectl 无法连接 APIServer | `tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou` 检查 PgwEndpoint | 集群 APIServer 端点不可达（如演示集群 cls-xxxxxxxx） | 在端点可达的集群执行 kubectl；或通过控制台内嵌 Cloud Shell |

## 下一步

- [自定义策略授权](../自定义策略授权/tccli%20操作.md) — kubectl 创建自定义 Role/ClusterRole
- [授权模式对比](../授权模式对比/tccli%20操作.md) — 新旧授权模式差异
- [更新子账号的 TKE 集群访问凭证](../更新子账号的%20TKE%20集群访问凭证/tccli%20操作.md) — 凭证轮换操作

## 控制台替代

[TKE 控制台 → 集群详情 → 授权管理](https://console.cloud.tencent.com/tke2/cluster) → **ClusterRoleBinding** → 点击 **RBAC 策略生成器** → 勾选目标子账号 → **下一步** → 在「集群 RBAC 设置」中选择身份（Admin/Ops/Dev/Read-Only）→ 指定 Namespace 范围 → **完成**。
