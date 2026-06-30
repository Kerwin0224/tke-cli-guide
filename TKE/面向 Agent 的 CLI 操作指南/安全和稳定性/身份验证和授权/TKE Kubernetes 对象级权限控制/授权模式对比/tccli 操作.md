# 授权模式对比（tccli）

> 对照官方：[授权模式对比](https://cloud.tencent.com/document/product/457/46107) · page_id `46107`

## 概述

TKE 集群存在新旧两种授权模式。旧模式使用共享 admin token，所有子账号共享同一凭证，无法细粒度控制权限。新模式为每个子账号分配独立 x509 证书，集成 Kubernetes RBAC 实现资源级权限控制。

> **重要警告**：不要直接修改或删除 TKE 生成的 ClusterRoleBinding（标签为 `cloud.tencent.com/tke-account`）。权限变更应通过控制台「授权管理」页面或 TKE API（`GrantFullClusterAccess` / `DeleteUserPermissions`）进行，绕过将导致权限状态不一致。

本页为概念对比页。演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou）的 APIServer 端点不可达，kubectl 操作无法在本页执行；本页仅通过 tccli 控制面调用判断当前授权模式。

## 前置条件

- 已完成 [环境准备](../../../../环境准备.md)，tccli 已安装并配置凭据。
- 演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou），kubectl 不可达——本页只做控制面只读判断，无需 kubectl。涉及 kubectl 的对比说明仅作概念展示。
- CAM 调用账号具备以下 Action 权限：`tke:DescribeClusters`、`tke:DescribeClusterSecurity`、`tke:DescribeClusterKubeconfig`、`tke:DescribeUserPermissions`。

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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群认证模式 | `DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 获取当前 kubeconfig | `DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看 ClusterRole 列表 | `kubectl get clusterroles`（需端点可达） | 是 |
| 查看 ClusterRoleBinding 列表 | `kubectl get clusterrolebindings`（需端点可达） | 是 |
| 升级授权模式 | 控制台独有操作（无对应 tccli API） | — |

## 操作步骤

### 步骤 1：判断当前集群授权模式

#### 检测方法一：kubeconfig 文件大小

旧模式使用 admin token（kubeconfig 约 1-2 KB），新模式包含 x509 证书数据（kubeconfig 约 5-7 KB）：

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d | wc -c
# expected: 新模式返回约 5725 字节，旧模式返回约 1500 字节
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

#### 检测方法二：证书数据字段

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d \
    | grep -c "client-certificate-data"
# expected: 新模式返回 1，旧模式返回 0
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

#### 旧模式特征 — DescribeClusterSecurity 输出

```bash
tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou
# expected: exit 0，返回集群安全配置
```

```json
{
    "UserName": "admin",
    "Password": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "CertificationAuthority": "",
    "ClusterExternalEndpoint": "https://cls-xxxxxxxx.ccs.tencent-cloud.com",
    "Domain": "cls-xxxxxxxx.ccs.tencent-cloud.com",
    "PgwEndpoint": "https://cls-xxxxxxxx.ccs.tencent-cloud.com",
    "SecurityPolicy": ["0.0.0.0/0"],
    "Kubeconfig": "...",
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

旧模式下 `Kubeconfig` 包含 admin token（32 字符），而非用户独立 x509 证书。`DescribeClusterSecurity` 无论新旧模式调用者身份均返回一致结果。

#### 新模式特征 — 子账号调用 DescribeClusterKubeconfig 返回独立证书

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

新模式下 kubeconfig 包含 `client-certificate-data` 和 `client-key-data` 字段。注意：主账号调用 `DescribeClusterKubeconfig` 返回的是 admin token（保持兼容），需用子账号身份调用才能获取到 x509 证书版本。

### 步骤 2：新旧模式对比

| 对比项 | 旧模式 | 新模式 |
|--------|--------|--------|
| Kubeconfig 凭证 | 共享 admin token | 每个子账号独立 x509 证书 |
| 控制台权限控制 | 无细粒度权限，子账号默认全读写 | 集成 Kubernetes RBAC 资源级控制 |
| 证书安全 | Token 泄露影响所有用户 | 单个子账号证书泄露可单独轮换 |
| 预设角色 | 无 | `tke:admin`、`tke:ops`、`tke:dev`、`tke:ro` |
| 集群创建者权限 | 无区分 | 集群创建者 + 主账号始终拥有 `tke:admin` 权限 |
| kubeconfig 大小 | 约 1-2 KB | 约 5-7 KB（含证书和密钥） |

### 步骤 3：kubeconfig 字段差异详解

| 字段 | 旧模式（admin token） | 新模式（x509 证书） |
|------|----------------------|-------------------|
| `users[].user.token` | 32 字符 token | 无（或不使用） |
| `users[].user.client-certificate-data` | 无 | base64 编码 x509 证书 |
| `users[].user.client-key-data` | 无 | base64 编码私钥 |

### 步骤 4：升级授权模式

升级为控制台独有操作。升级过程中 TKE 自动执行：

1. 创建默认预设 admin ClusterRole：`tke:admin`
2. 拉取子账号列表
3. 为每个子账号生成 x509 客户端证书
4. 将 `tke:admin` 绑定到所有子账号（保持向后兼容）
5. 升级完成

升级后所有子账号默认拥有 `tke:admin` 权限。集群管理员需手动撤销非必要权限。

### 步骤 5：新模式下的权限管理（需端点可达）

> 演示集群 `cls-xxxxxxxx` 端点不可达，以下 kubectl 命令需在端点可达的集群执行：

```bash
# 列出所有 TKE 管理的 ClusterRoleBinding
kubectl get clusterrolebindings -l cloud.tencent.com/tke-account
# expected: 返回已授权的子账号绑定列表
```

```text
NAME                              ROLE                  AGE
100012345678-ClusterRoleBinding   ClusterRole/tke:admin   1h
100012345679-ClusterRoleBinding   ClusterRole/tke:ops     30m
```

每条绑定的标签和注解：

| 键 | 值 | 说明 |
|---|-----|------|
| Label: `cloud.tencent.com/tke-account` | 子账号 UIN | 标识该绑定归属的子账号 |
| Annotation: `cloud.tencent.com/tke-account-nickname` | 子账号昵称 | 便于人工识别 |

## 验证

以下操作分别从控制面和数据面维度验证当前集群的授权模式：

### 控制面（tccli）

```bash
# 1. 检测 kubeconfig 大小（判断模式）
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d | wc -c
# expected: 新模式 ≈ 5700 字节，旧模式 ≈ 1500 字节
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

```bash
# 2. 检测证书字段
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d \
    | grep -c "client-certificate-data"
# expected: 新模式 1，旧模式 0
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "e5f6a7b8-c9d0-1234-efab-234567890abc"
}
```

### 数据面（kubectl）

> 需端点可达，演示集群 `cls-xxxxxxxx` 不可达：

```bash
# 3. 验证预设 ClusterRole 存在（需端点可达）
kubectl get clusterrole tke:admin tke:ops tke:dev tke:ro
# expected: 新模式返回 4 个预设角色，旧模式返回 NotFound
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需清理。本页为概念对比与只读查询，不涉及资源创建。

> **警告**：不要直接删除 TKE 生成的 ClusterRoleBinding（标签为 `cloud.tencent.com/tke-account`），这会绕过 TKE 权限管理系统导致状态不一致。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get clusterroles` 看不到 TKE 预设角色 | 检测模式：`tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json \| jq -r '.Kubeconfig' \| base64 -d \| grep "client-certificate-data"` | 集群仍在旧授权模式 | 在控制台「授权管理」页点击 **RBAC 策略生成器** 触发升级 |
| `DescribeClusterKubeconfig` 返回空或极小 kubeconfig | 检查子账号是否已被授权：`tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 该子账号未获得集群访问权限（新模式）或 API 调用者身份不正确 | 先通过 `GrantFullClusterAccess` 授权 |
| kubectl 无法连接 APIServer | `tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou` 检查 PgwEndpoint | 集群 APIServer 端点不可达（如演示集群 cls-xxxxxxxx） | 在端点可达的集群执行 kubectl；或通过控制台内嵌 Cloud Shell |

### 升级后异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 升级后子账号权限过大（预期 `tke:dev` 却获 `tke:admin`） | 检查绑定列表：`kubectl get clusterrolebindings -l cloud.tencent.com/tke-account` | 升级流程默认将所有子账号绑定到 `tke:admin` | 管理员手动撤销多余权限：`tccli tke DeleteUserPermissions --ClusterId cls-xxxxxxxx --SubUin SUB_UIN --region ap-guangzhou`，再按需用 `GrantFullClusterAccess` 重新授权 |
| 删除 CAM 子账号后 ClusterRoleBinding 残留 | 检查绑定列表：`kubectl get clusterrolebindings -l cloud.tencent.com/tke-account` | CAM 删除子账号不会自动清理 K8s RBAC 绑定 | 在控制台「授权管理」页 → ClusterRoleBinding → 点击 **清理无效账号** |
| 直接修改 ClusterRoleBinding 后出现权限异常 | 对比 TKE 控制台显示与实际绑定：`kubectl get clusterrolebinding BINDING_NAME -o yaml` | 直接修改绕过了 TKE 权限管理系统 | 通过控制台或 TKE API（`DeleteUserPermissions` + `GrantFullClusterAccess`）重新操作，覆盖手动修改 |
| kubeconfig 含 `client-certificate-data` 但 kubectl 返回 `Unauthorized` | 检查证书有效期：`kubectl --kubeconfig ~/.kube/config config view --raw -o json \| jq '.users[0].user."client-certificate-data"' \| base64 -d \| openssl x509 -noout -dates` | x509 证书已过期（新模式）或子账号权限被撤销 | 若证书已过期 → 执行凭证更新流程（撤销 + 重新授权）；若证书未过期 → 检查 `DescribeUserPermissions` 确认子账号是否仍被授权 |

## 下一步

- [使用预设身份授权](../使用预设身份授权/tccli%20操作.md) — GrantFullClusterAccess 快速授权
- [自定义策略授权](../自定义策略授权/tccli%20操作.md) — kubectl 自定义 Role/ClusterRole
- [更新子账号的 TKE 集群访问凭证](../更新子账号的%20TKE%20集群访问凭证/tccli%20操作.md) — 凭证轮换操作

## 控制台替代

[TKE 控制台 → 集群详情 → 授权管理](https://console.cloud.tencent.com/tke2/cluster)：查看 ClusterRole/ClusterRoleBinding 列表。若为旧模式，点击 **RBAC 策略生成器** 触发升级，在弹出的「切换权限管理模式」窗口中点击 **切换权限管理模式**。
