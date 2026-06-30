# 授权模式对比（tccli）

> 对照官方：[授权模式对比](https://cloud.tencent.com/document/product/457/46107) · page_id `46107`

## 概述

TKE 集群存在新旧两种授权模式。旧模式使用共享 admin token，无法细粒度控制权限。新版模式为每个子账号分配独立 x509 证书，集成 Kubernetes RBAC 实现资源级权限控制。

> **重要警告**：不要直接修改或删除 `ClusterRoleBinding` 或 `ClusterRole` 资源。权限变更应通过控制台「授权管理」页面或 TKE API（`GrantFullClusterAccess` / `DeleteUserPermissions`）进行。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达
- 已安装 tccli 并完成 `tccli configure`
- 当前账号具备 TKE 查询权限（`tke:DescribeClusterSecurity`、`tke:DescribeClusterKubeconfig`、`tke:DescribeClusters`）

### 环境检查

```bash
# 1. 确认集群存在且可访问
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

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群认证模式（旧模式） | `tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 获取当前 kubeconfig（新模式） | `tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 升级授权模式 | 控制台独有操作（无 CLI） | — |

## 操作步骤

### 步骤 1：判断当前集群授权模式

#### 旧模式特征——使用 admin token 认证

```bash
tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "UserName": "admin",
    "Password": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "CertificationAuthority": "",
    "ClusterExternalEndpoint": "https://cls-example.ccs.tencent-cloud.com",
    "Domain": "cls-example.ccs.tencent-cloud.com",
    "PgwEndpoint": "https://cls-example.ccs.tencent-cloud.com",
    "SecurityPolicy": ["0.0.0.0/0"],
    "Kubeconfig": "...",
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

旧模式下 `Kubeconfig` 包含 admin token（32 字符），而非用户独立 x509 证书。

#### 新模式特征——子账号调用返回独立 x509 证书

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

新模式下 kubeconfig 约 5725 字节，包含 `client-certificate-data` 和 `client-key-data` 字段。

### 步骤 2：新旧模式对比

| 对比项 | 旧模式 | 新模式 |
|--------|--------|--------|
| Kubeconfig 凭证 | 共享 admin token | 每个子账号独立 x509 证书 |
| 控制台权限控制 | 无细粒度权限，子账号默认全读写 | 集成 Kubernetes RBAC 资源级控制 |
| 证书安全 | Token 泄露影响所有用户 | 单个子账号证书泄露可单独轮换 |
| 预设角色 | 无 | `tke:admin`、`tke:ops`、`tke:dev`、`tke:ro` |
| 集群创建者权限 | — | 集群创建者 + 主账号始终拥有 `tke:admin` 权限 |

新旧模式 kubeconfig 字段差异：

| 字段 | 旧模式（admin token） | 新模式（x509 证书） |
|------|----------------------|-------------------|
| `users[].user.token` | 32 字符 token | 无（或不使用） |
| `users[].user.client-certificate-data` | 无 | base64 编码 x509 证书 |
| `users[].user.client-key-data` | 无 | base64 编码私钥 |
| kubeconfig 大小 | 约 1-2 KB | 约 5-7 KB（含证书和密钥） |

### 步骤 3：升级授权模式

升级为控制台独有操作，升级过程中 TKE 自动执行以下步骤：

1. 创建默认预设 admin ClusterRole：`tke:admin`
2. 拉取子账号列表
3. 为每个子账号生成 x509 客户端证书
4. 将 `tke:admin` 角色绑定到所有子账号（保持向后兼容）
5. 升级完成

升级后所有子账号默认拥有 `tke:admin` 权限。集群管理员需手动撤销非必要权限。

## 验证

```bash
# 确认集群认证模式（新模式检测）
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json | jq -r '.Kubeconfig' | base64 -d | grep -c "client-certificate-data"
# expected: 新模式返回 1，旧模式返回 0
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 清理

无需清理。本页为概念对比，不涉及资源创建。

> **警告**：不要直接删除 TKE 生成的 ClusterRoleBinding（标签为 `cloud.tencent.com/tke-account`）。这会绕过 TKE 权限管理系统，可能导致控制台授权页面数据不一致。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get clusterroles` 看不到 TKE 预设角色 | `tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json \| jq -r '.Kubeconfig' \| base64 -d \| grep "client-certificate-data"` 检查模式 | 集群仍在旧授权模式 | 在控制台「授权管理」页点击「RBAC 策略生成器」触发升级 |
| 升级后子账号权限异常（权限过大） | `kubectl get clusterrolebindings -l cloud.tencent.com/tke-account` 查看绑定列表 | 升级流程默认将 `tke:admin` 绑定到所有子账号 | 管理员手动撤销多余权限：`tccli tke DeleteUserPermissions` |
| 删除 CAM 子账号后 ClusterRoleBinding 残留 | `kubectl get clusterrolebindings -l cloud.tencent.com/tke-account` 查看绑定列表 | CAM 删除子账号不会自动清理 K8s RBAC 绑定 | 控制台「授权管理」页 → ClusterRoleBinding → 点击「清理无效账号」按钮 |
| 直接修改 ClusterRoleBinding 后出现异常 | `kubectl get clusterrolebinding <name> -o yaml` 对比 TKE 控制台显示 | 直接修改绕过了 TKE 权限管理系统 | 通过控制台或 TKE API 重新操作，覆盖手动修改 |
| 子账号无法管理自己的权限 | 检查是否有 `tke:admin` 权限 | 控制台不支持自管理（无法给自己授权） | 联系集群管理员（`tke:admin` 持有者）代为操作 |

## 下一步

- [使用预设身份授权](../使用预设身份授权/tccli%20操作.md) — page_id `46105`
- [自定义策略授权](../自定义策略授权/tccli%20操作.md) — page_id `46106`
- [更新子账号的 TKE 集群访问凭证](../更新子账号的%20TKE%20集群访问凭证/tccli%20操作.md) — page_id `46108`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 `cls-xxxxxxxx` → **授权管理** → 查看 **ClusterRole** 和 **ClusterRoleBinding** 列表。若为旧模式，点击 **RBAC 策略生成器** 触发升级，在弹出的「切换权限管理模式」窗口中点击 **切换权限管理模式**。升级后通过 ClusterRoleBinding 页面的 **RBAC 策略生成器** 管理子账号权限。
