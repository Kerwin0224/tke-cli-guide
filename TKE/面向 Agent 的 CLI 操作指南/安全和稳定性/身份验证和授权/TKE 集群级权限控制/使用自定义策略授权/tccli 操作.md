# 使用自定义策略授权（tccli）

> 对照官方：[使用自定义策略授权](https://cloud.tencent.com/document/product/457/45337) · page_id `45337`

## 概述

当 TKE 预设策略无法满足精细化权限管理需求时，可通过 CAM 自定义策略按操作（action）、资源（resource）、条件（condition）三个维度定义权限边界。本页面列出 TKE 各功能模块所需的 API Action，并提供常见策略示例。自定义策略创建后还需通过 `tke:GrantUserPermissions` 在集群侧完成 RBAC 绑定，撤销权限则使用 `tke:DeleteUserPermissions`。

> TKE 部分 API 操作不支持资源级权限。对于这些 API，必须在策略中指定 `"resource": "*"`，无法指定具体集群。

## 前置条件

- 已完成 [环境准备](../../../../环境准备.md)：安装 tccli 并配置凭据与默认地域。
- 演示集群 `cls-xxxxxxxx`（状态 Running，版本 v1.30.0，地域 ap-guangzhou）。本页所有命令以该集群和子账号 UIN `100012345678`（Uid `1001408`）为例。
- 演示集群未暴露 kube-apiserver 公网入口，kubectl 不可达；权限验证只能通过 tccli API 完成。

环境检查：

```bash
tccli --version
tccli configure list
```

确认当前账号具备以下 CAM/TKE Action 权限：`cam:CreatePolicy`、`cam:GetPolicy`、`cam:UpdatePolicy`、`cam:DeletePolicy`、`cam:ListPolicies`、`cam:AttachUserPolicy`、`tke:GrantUserPermissions`、`tke:DeleteUserPermissions`、`tke:DescribeUserPermissions`。

资源检查：

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterType": "MANAGED_CLUSTER"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
tccli cam ListUsers --region ap-guangzhou --output json
```

```json
{
    "Data": [
        {
            "Uin": 100012345678,
            "Name": "dev-subaccount",
            "Uid": 1001408
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建自定义策略 | `cam CreatePolicy --PolicyName NAME --PolicyDocument 'JSON'` | 否 |
| 查看策略内容 | `cam GetPolicy --PolicyId POLICY_ID` | 是 |
| 关联策略到子账号 | `cam AttachUserPolicy --AttachUin 100012345678 --PolicyId POLICY_ID` | 否 |
| 授予集群级 RBAC 角色 | `tke GrantUserPermissions --TargetUin 100012345678 --Permissions.0.ClusterId cls-xxxxxxxx ...` | 否 |
| 验证集群级权限 | `tke DescribeUserPermissions --TargetUin 100012345678` | 是 |
| 解除策略关联 | `cam DetachUserPolicy --DetachUin 100012345678 --PolicyId POLICY_ID` | 否 |
| 删除策略 | `cam DeletePolicy --cli-unfold-argument --PolicyId POLICY_ID` | 否 |

## 操作步骤

### 步骤 1：策略语法结构

CAM 自定义策略 JSON 基本结构：

```json
{
    "version": "2.0",
    "statement": [
        {
            "effect": "allow",
            "action": ["SERVICE:ACTION"],
            "resource": ["QCS_ARN"],
            "condition": {}
        }
    ]
}
```

| 字段 | 必填 | 取值与约束 | 错误后果 |
|------|:--:|------|------|
| `version` | 是 | 必须为 `"2.0"` | 填其他版本 → `CreatePolicy` 返回策略语法错误 |
| `statement[].effect` | 是 | `"allow"` 或 `"deny"`，`deny` 优先级高于 `allow` | 填错 → 策略不生效或被拒绝访问 |
| `statement[].action` | 是 | 格式 `"service:action"`，如 `"tke:DescribeClusters"`、`"tke:*"`。支持通配符 | 不精确 → 权限不足或过大 |
| `statement[].resource` | 是 | QCS ARN 格式或 `"*"`。部分 TKE API 不支持资源级权限，必须用 `"*"` | 格式错误 → 策略不匹配目标资源 |
| `statement[].condition` | 否 | 条件键，如 `{"ip_equal":{"qcs:ip":"10.0.0.0/8"}}` | 条件不满足 → 策略不生效 |

### 步骤 2：QCS ARN 格式

TKE 资源的 QCS ARN 格式：

```
qcs::tke:REGION:uin/UIN:cluster/CLUSTER_ID
```

| 段 | 说明 | 示例 |
|----|------|------|
| `qcs` | Qcloud Service 固定前缀 | `qcs` |
| `tke` | 服务名（容器服务） | `tke` |
| `REGION` | 地域代码，支持完整名或短代码 | `ap-guangzhou`、`gz` |
| `uin/UIN` | 主账号 UIN，可省略为 `::` | `uin/100012345678` |
| `cluster/CLUSTER_ID` | 资源类型和资源 ID | `cluster/cls-xxxxxxxx` |

支持资源级权限的 TKE API（可指定 `resource` 为具体 QCS ARN）：`DescribeClusterSecurity`、`DescribeClusters`、`DescribeClusterInstances`、`DeleteClusterInstances`、`AddExistedInstances`、`CreateClusterInstances`。

不支持资源级权限的 TKE API（`resource` 必须为 `"*"`）：`CreateCluster`、`DeleteCluster`、`DescribeClusterStatus`、`CreateClusterNodePool`、`DeleteClusterNodePool`、`DescribeClusterKubeconfig`、`DescribeClusterEndpointStatus`。

### 步骤 3：TKE API 权限分类

| 操作 | 直接 API | 间接依赖 API |
|------|----------|-------------|
| 创建空集群 | `tke:CreateCluster` | `cam:GetRole`、`tag:GetTagKeys`、`vpc:DescribeVpcEx`、`cvm:DescribeImages` |
| 查询集群列表 | `tke:DescribeClusters` | — |
| 查看集群凭证 | `tke:DescribeClusterSecurity` | — |
| 删除集群 | `tke:DeleteCluster` | `tke:DescribeClusterInstances`、`tke:DescribeInstancesVersion`、`tke:DescribeClusterStatus` |
| 添加已有节点 | `tke:AddExistedInstances` | `cvm:DescribeInstances`、`vpc:DescribeSubnetEx`、`cvm:DescribeSecurityGroups`、`cvm:ResetInstance` |
| 新建节点 | `tke:CreateClusterInstances` | `cvm:RunInstances`、`vpc:DescribeSubnetEx`、`cvm:DescribeImages` |
| 移出节点 | `tke:DeleteClusterInstances` | `cvm:TerminateInstances` |

### 步骤 4：示例一 —— 单个集群管理员（含扩缩容）

通过 `resource` 限定到演示集群，CVM/VPC/CLB 等依赖服务设为 `"*"`。适合将单个生产集群授权给特定运维团队。

`single-cluster-admin-policy.json`：

```json
{
    "PolicyName": "TKE-SingleCluster-Admin",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tke:*\"],\"resource\":[\"qcs::tke:ap-guangzhou::cluster/cls-xxxxxxxx\",\"qcs::cvm:ap-guangzhou::instance/*\"]},{\"effect\":\"allow\",\"action\":[\"cvm:*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"vpc:*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"clb:*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"monitor:*\",\"cam:ListUsersForGroup\",\"cam:ListGroups\",\"cam:GetGroup\",\"cam:GetRole\"],\"resource\":\"*\"}]}",
    "Description": "单个 TKE 集群的全读写管理权限（含节点扩缩容）"
}
```

```bash
tccli cam CreatePolicy --cli-input-json file://single-cluster-admin-policy.json --region ap-guangzhou --output json
```

```json
{
    "PolicyId": 400001,
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

> 如需子账号执行集群扩缩容，还需额外配置支付权限。

### 步骤 5：示例二 —— 集群创建者策略

仅授予创建集群所需的 API，不含已有集群的管理权限。创建集群时无法预知集群 ID，`resource` 设为 `"*"`。

`cluster-creator-policy.json`：

```json
{
    "PolicyName": "TKE-ClusterCreator",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tke:CreateCluster\",\"tke:DescribeClusters\",\"tke:DescribeClusterSecurity\",\"tke:DescribeClusterKubeconfig\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"cvm:DescribeSecurityGroups\",\"cvm:DescribeKeyPairs\",\"cvm:RunInstances\",\"vpc:DescribeSubnetEx\",\"vpc:DescribeVpcEx\",\"cvm:DescribeImages\",\"cam:GetRole\",\"tag:GetTagKeys\"],\"resource\":\"*\"}]}",
    "Description": "允许创建集群并查看集群信息"
}
```

```bash
tccli cam CreatePolicy --cli-input-json file://cluster-creator-policy.json --region ap-guangzhou --output json
```

### 步骤 6：示例三 —— 只读策略

全局只读：`tke:Describe*` + `tke:Check*` 覆盖所有读操作。适合审计、巡检角色。

`readonly-policy.json`：

```json
{
    "PolicyName": "TKE-ReadOnly",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tke:Describe*\",\"tke:Check*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"cvm:Describe*\",\"cvm:Inquiry*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"vpc:Describe*\",\"vpc:Inquiry*\",\"vpc:Get*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"clb:Describe*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"monitor:*\",\"cam:ListUsersForGroup\",\"cam:ListGroups\",\"cam:GetGroup\",\"cam:GetRole\"],\"resource\":\"*\"}]}",
    "Description": "TKE 全局只读权限"
}
```

```bash
tccli cam CreatePolicy --cli-input-json file://readonly-policy.json --region ap-guangzhou --output json
```

### 步骤 7：关联策略并授予集群级 RBAC 角色

CAM 策略绑定后，还需调用 `GrantUserPermissions` 在集群侧将子账号与 RBAC 角色绑定。仅绑定 CAM 策略不授予集群内部操作权限。

```bash
# 关联 CAM 策略
tccli cam AttachUserPolicy \
    --AttachUin 100012345678 \
    --PolicyId 400001 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

```bash
# 授予集群级管理员角色
tccli tke GrantUserPermissions --cli-unfold-argument \
    --TargetUin 100012345678 \
    --Permissions.0.ClusterId cls-xxxxxxxx \
    --Permissions.0.RoleName tke:admin \
    --Permissions.0.RoleType cluster \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "e5f6a7b8-c9d0-1234-efab-345678901234"
}
```

> `GrantUserPermissions` 采用声明式语义：传入的 `Permissions` 列表代表子账号最终应拥有的全部权限。`RoleName` 可选 `tke:admin`、`tke:ops`、`tke:dev`、`tke:ro`。

## 验证

查看策略内容：

```bash
tccli cam GetPolicy --PolicyId 400001 --region ap-guangzhou --output json
```

```json
{
    "PolicyName": "TKE-SingleCluster-Admin",
    "Description": "单个 TKE 集群的全读写管理权限（含节点扩缩容）",
    "Type": 2,
    "AddTime": "2025-01-01 00:00:00",
    "UpdateTime": "2025-01-01 00:00:00",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[...]}",
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

验证集群级权限已生效：

```bash
tccli tke DescribeUserPermissions --TargetUin 100012345678 --region ap-guangzhou --output json
```

```json
{
    "Permissions": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "RoleName": "tke:admin",
            "RoleType": "cluster",
            "IsCustom": false,
            "Namespace": ""
        }
    ],
    "TargetUin": "100012345678",
    "RequestId": "a7b8c9d0-e1f2-3456-abcd-567890123456"
}
```

## 清理

> 删除自定义策略前必须先解除所有 CAM 关联，并撤销集群侧的权限绑定（`DeleteUserPermissions`）。仅删除 CAM 策略不会自动移除集群内部的 RBAC 权限。

清理前状态检查：

```bash
tccli cam GetPolicy --PolicyId 400001 --region ap-guangzhou --output json
tccli tke DescribeUserPermissions --TargetUin 100012345678 --region ap-guangzhou --output json
```

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
```

撤销集群侧权限：

```bash
tccli tke DeleteUserPermissions --cli-unfold-argument \
    --TargetUin 100012345678 \
    --Permissions.0.ClusterId cls-xxxxxxxx \
    --Permissions.0.RoleName tke:admin \
    --Permissions.0.RoleType cluster \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "b8c9d0e1-f2a3-4567-bcde-678901234567"
}
```

解除 CAM 关联并删除策略：

```bash
tccli cam DetachUserPolicy \
    --DetachUin 100012345678 \
    --PolicyId 400001 \
    --region ap-guangzhou \
    --output json

tccli cam DeletePolicy --cli-unfold-argument \
    --PolicyId 400001 \
    --region ap-guangzhou \
    --output json
```

验证已删：

```bash
tccli cam GetPolicy --PolicyId 400001 --region ap-guangzhou
# expected: ResourceNotFound
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePolicy` 报策略语法错误 | 用 `jq` 验证 JSON | `PolicyDocument` 内部 `"` 未转义 | `cat policy.json \| jq '.PolicyDocument \| fromjson'` 验证结构 |
| `DeletePolicy` 报 `PolicyInUse` | 查看关联 | 策略仍关联着用户/组 | 先 `DetachUserPolicy` 解除所有关联 |
| `CreatePolicy` 报 `PolicyNameInUse` | 搜索同名策略 | 同名策略已存在 | 修改 PolicyName 或删除旧策略 |
| 指定 `resource` 后仍报 `Unauthorized` | 将对应 statement 的 `resource` 改为 `"*"` | 该 API 不支持资源级权限 | 将该 statement 的 `resource` 改为 `"*"` |
| CAM 策略已绑定但无法操作集群 | 检查集群侧 RBAC | 未调用 `GrantUserPermissions` | 执行 `GrantUserPermissions` 绑定 RBAC 角色 |
| 创建集群操作失败（权限不足） | 检查错误信息中的 Action 名 | 缺少 `cvm:RunInstances` 等依赖权限 | 参考 API 权限分类表补充缺失的依赖 Action |

## 下一步

- [通过标签为子账号配置批量集群的全读写权限](../使用示例/通过标签为子账号配置批量集群的全读写权限/tccli%20操作.md)
- [配置子账号对单个 TKE 集群的管理权限](../使用示例/配置子账号对单个%20TKE%20集群的管理权限/tccli%20操作.md)
- [使用 TKE 预设策略授权](../使用%20TKE%20预设策略授权/tccli%20操作.md)

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam) → **策略** → **新建自定义策略** → **按策略语法创建** → 选择空白模板 → 输入 JSON 策略内容 → **完成**。然后在 [TKE 控制台](https://console.cloud.tencent.com/tke2) → 集群详情 → **授权管理** → 添加子账号并选择角色。
