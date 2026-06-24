# TKE Kubernetes 对象级权限控制概述（tccli）

> 对照官方：[概述（Kubernetes 对象级权限控制）](https://cloud.tencent.com/document/product/457/46104) · page_id `46104`

## 概述

TKE 集成 Kubernetes 原生 RBAC（Role-Based Access Control），为子账号提供细粒度的集群内资源访问控制。核心机制：TKE 为每个子账号签发独立的 x509 客户端证书（而非共享 admin token），通过 Role/ClusterRole/RoleBinding/ClusterRoleBinding 四类 RBAC 对象控制子账号对 Namespace、API Group、资源类型和操作动词的权限。

TKE 控制台提供 RBAC 策略生成器，将 CAM 子账号映射到预设 ClusterRole（`tke:admin`、`tke:ops`、`tke:dev`、`tke:ro`）。CLI 侧通过 `GrantFullClusterAccess` API 完成等效操作。

自定义 Role/ClusterRole 通过 kubectl 管理（需端点可达），可配合 RoleBinding/ClusterRoleBinding 实现命名空间级别甚至资源级别的精细化权限控制。

本页为概念概述页。演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou）的 APIServer 端点不可达，kubectl 操作无法在本页执行；本页仅用 tccli 完成控制面只读验证。

## 前置条件

- 已完成 [环境准备](../../../../环境准备.md)，tccli 已安装并配置凭据。
- 演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou），kubectl 不可达——本页只做控制面只读查询，无需 kubectl。涉及 kubectl 的内容仅作概念说明，实操见子页。
- CAM 调用账号具备以下 Action 权限：`tke:DescribeClusters`、`tke:DescribeClusterSecurity`、`tke:DescribeClusterKubeconfig`。

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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群认证信息 | `DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 获取子账号 kubeconfig | `DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看 ClusterRole 列表 | `kubectl get clusterroles`（需端点可达） | 是 |
| 查看 ClusterRoleBinding 列表 | `kubectl get clusterrolebindings`（需端点可达） | 是 |
| 查看 Role 列表（命名空间级） | `kubectl get roles -n NAMESPACE`（需端点可达） | 是 |
| 授予子账号集群内权限 | `GrantFullClusterAccess`（预设身份）或 `kubectl create rolebinding`（自定义） | — |

## 操作步骤

### RBAC 核心对象

Kubernetes RBAC 通过 `rbac.authorization.k8s.io` API Group 实现，包含四类核心对象：

| 对象 | 作用域 | 说明 |
|------|--------|------|
| Role | 命名空间级 | 在特定 Namespace 内定义一组权限（resources + verbs） |
| ClusterRole | 集群级 | 在整个集群范围定义一组权限，也可用于非命名空间资源（Node、PV、Namespace 等） |
| RoleBinding | 命名空间级 | 将 Role 中的权限授予一个或多个主体（User/Group/ServiceAccount） |
| ClusterRoleBinding | 集群级 | 将 ClusterRole 中的权限授予一个或多个主体 |

### TKE x509 证书认证

TKE 选择 x509 证书认证而非 Token/OIDC 等方式，每个子账号拥有独立的客户端证书用于访问 Kubernetes APIServer。查看当前集群的认证配置：

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
    "JnsGwEndpoint": "",
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

### TKE 预设 ClusterRole

TKE 预设四类 ClusterRole（`tke:admin`、`tke:ops`、`tke:dev`、`tke:ro`），用户通过 RoleBinding/ClusterRoleBinding 将子账号关联到预设角色即可完成授权。查看预设 ClusterRole 需端点可达：

```bash
# 需端点可达，本演示集群 cls-xxxxxxxx 端点不可达，仅作示例
kubectl get clusterrole -l cloud.tencent.com/tke-rbac-generated=true
# expected: 返回预设 ClusterRole 列表
```

**预期输出**：

```text
NAME        CREATED AT
tke:admin   2025-01-01T00:00:00Z
tke:dev     2025-01-01T00:00:00Z
tke:ops     2025-01-01T00:00:00Z
tke:ro      2025-01-01T00:00:00Z
```

### RBAC 细粒度权限控制

- **只读访问**：授予子账号对整个集群的只读权限
- **命名空间读写**：授予子账号仅对某个命名空间的读写权限

具体配置步骤参见后续页面。

## 验证

### 控制面（tccli）

```bash
# 验证集群认证配置
tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou
# expected: exit 0，返回集群安全信息，含 UserName、Domain、PgwEndpoint
```

```json
{
    "UserName": "admin",
    "Domain": "cls-xxxxxxxx.ccs.tencent-cloud.com",
    "PgwEndpoint": "https://cls-xxxxxxxx.ccs.tencent-cloud.com",
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 数据面（kubectl）

> 演示集群 `cls-xxxxxxxx` 端点不可达，以下命令需在端点可达的集群执行：

```bash
# 验证 RBAC 配置（需端点可达）
kubectl auth can-i list pods --all-namespaces
# expected: yes（tke:admin）或 no（受限账号）

kubectl get clusterrolebindings -l cloud.tencent.com/tke-account
# expected: 显示已授权的子账号绑定列表
```

```text
NAME                              ROLE                  AGE
100012345678-ClusterRoleBinding   ClusterRole/tke:admin   1h
100012345679-ClusterRoleBinding   ClusterRole/tke:ops     30m
```

## 清理

无需清理。本页为概念概述与只读查询，不涉及资源创建。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kubectl 返回 `Unauthorized` | 检查证书和权限：`kubectl --kubeconfig ~/.kube/config config view --raw -o json \| jq '.users[0].user'` | kubeconfig 中的证书已过期或子账号未获授权 | 重新获取 kubeconfig：`tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou` |
| `kubectl get clusterroles` 看不到 TKE 预设角色 | 检查预设标签：`kubectl get clusterrole -l cloud.tencent.com/tke-rbac-generated=true` | 集群可能仍在旧授权模式 | 参见 [授权模式对比](../授权模式对比/tccli%20操作.md)，在控制台升级到新模式 |
| `DescribeClusterSecurity` 返回空或缺少关键字段 | 检查集群 ID 和 region 是否正确：`tccli tke DescribeClusters --region ap-guangzhou` | 集群不存在或 region 不匹配 | 确认 `--ClusterId cls-xxxxxxxx` 和 `--region ap-guangzhou` 正确；保留 RequestId 备用 |
| kubectl 提示无法连接 APIServer | `tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou` 检查 PgwEndpoint | 集群 APIServer 端点不可达（如演示集群 cls-xxxxxxxx）| 在端点可达的集群执行 kubectl 操作；或通过控制台内嵌 Cloud Shell |

### 权限配置异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 子账号在控制台看不到集群内资源 | 检查 RBAC 绑定：`kubectl get clusterrolebindings -l cloud.tencent.com/tke-account` | 未通过 `GrantFullClusterAccess` 或 RBAC 策略生成器授予集群内权限 | 使用 `GrantFullClusterAccess` 或在控制台 RBAC 策略生成器授权 |
| 证书过期导致 kubectl 不可用 | 检查证书有效期：`kubectl --kubeconfig ~/.kube/config config view --raw -o json \| jq '.users[0].user."client-certificate-data"' \| base64 -d \| openssl x509 -noout -dates` | x509 证书已过期 | 重新调用 `DescribeClusterKubeconfig` 获取新证书 |
| 证书 CN 与预期子账号 UIN 不一致 | 解码证书 CN：`kubectl --kubeconfig ~/.kube/config config view --raw -o json \| jq -r '.users[0].user."client-certificate-data"' \| base64 -d \| openssl x509 -noout -subject` | 使用了错误的 kubeconfig 文件 | 确认使用的 kubeconfig 对应当前子账号，调用 `DescribeClusterKubeconfig` 用当前身份重新获取 |

## 下一步

- [授权模式对比](../授权模式对比/tccli%20操作.md) — 新旧授权模式差异与检测方法
- [使用预设身份授权](../使用预设身份授权/tccli%20操作.md) — GrantFullClusterAccess 快速授权
- [自定义策略授权](../自定义策略授权/tccli%20操作.md) — kubectl 创建自定义 Role/ClusterRole
- [更新子账号的 TKE 集群访问凭证](../更新子账号的%20TKE%20集群访问凭证/tccli%20操作.md) — 凭证轮换操作

## 控制台替代

[TKE 控制台 → 集群详情 → 授权管理](https://console.cloud.tencent.com/tke2/cluster)：查看 ClusterRole/ClusterRoleBinding 列表，点击 **RBAC 策略生成器** 为新子账号配置权限。
