# 使用自定义策略授权（tccli）

> 对照官方：[使用自定义策略授权](https://cloud.tencent.com/document/product/457/45337) · page_id `45337`

## 概述

当 TKE 预设策略无法满足精细化权限管理需求时，可通过 CAM 自定义策略按操作（action）、资源（resource）、条件（condition）三个维度定义权限边界。本文档提供 TKE 各功能模块所需的 API Action 参考及常见策略示例。

> **重要提示**：TKE 部分 API 操作不支持资源级权限。对于这些 API，必须在策略中指定 `"resource": "*"`，无法指定具体集群。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达
- 已安装 tccli 并完成 `tccli configure`
- 当前账号具备 CAM 管理权限（`cam:CreatePolicy`、`cam:GetPolicy`、`cam:UpdatePolicy`、`cam:DeletePolicy`、`cam:ListPolicies`、`cam:AttachUserPolicy`）
- 目标子账号 UIN `100012345678` 已存在

### 环境检查

```bash
# 1. 确认 CAM 权限——查询自定义策略列表
tccli cam ListPolicies --Scope "Local" --region ap-guangzhou --output json
```

```json
{
    "List": [],
    "TotalNum": 0,
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 2. 确认目标集群存在（如需指定资源级权限）
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
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建自定义策略 | `tccli cam CreatePolicy --cli-input-json file://policy.json --region ap-guangzhou` | 否 |
| 查看策略内容 | `tccli cam GetPolicy --PolicyId 400001 --region ap-guangzhou` | 是 |
| 更新策略 | `tccli cam UpdatePolicy --PolicyId 400001 --region ap-guangzhou` | 是 |
| 关联策略到子账号 | `tccli cam AttachUserPolicy --AttachUin 100012345678 --PolicyId 400001 --region ap-guangzhou` | 否 |
| 删除策略 | `tccli cam DeletePolicy --PolicyId 400001 --region ap-guangzhou` | 否 |

## 操作步骤

### 策略语法结构

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

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `version` | String | 是 | 必须为 `"2.0"` | 填其他版本 → 策略语法错误 |
| `statement[].effect` | String | 是 | `"allow"` 或 `"deny"` | 填错 → 策略不生效 |
| `statement[].action` | Array | 是 | 如 `"tke:DescribeClusters"`、`"tke:*"` | 不精确 → 权限不足或过大 |
| `statement[].resource` | Array | 是 | QCS ARN 格式，如 `"qcs::tke:ap-guangzhou::cluster/cls-xxxxxxxx"` 或 `"*"` | 格式错误 → 策略不匹配目标资源 |
| `statement[].condition` | Object | 否 | 如 `{"ip_equal":{"qcs:ip":"10.0.0.0/8"}}` | 条件不满足 → 策略不生效 |

### 示例一：单个集群管理员（含扩缩容）

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
    "PolicyName": "TKE-SingleCluster-Admin",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[...]}",
    "Description": "单个 TKE 集群的全读写管理权限（含节点扩缩容）",
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 示例二：集群创建者策略

`cluster-creator-policy.json`：

```json
{
    "PolicyName": "TKE-ClusterCreator",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tke:CreateCluster\",\"tke:DescribeClusters\",\"tke:DescribeClusterSecurity\",\"tke:DescribeClusterKubeconfig\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"cvm:DescribeSecurityGroups\",\"cvm:DescribeKeyPairs\",\"cvm:RunInstances\",\"vpc:DescribeSubnetEx\",\"vpc:DescribeVpcEx\",\"cvm:DescribeImages\",\"cam:GetRole\",\"tag:GetTagKeys\"],\"resource\":\"*\"}]}",
    "Description": "允许创建集群并查看集群信息（含 CVM、VPC 等依赖权限）"
}
```

### 示例三：只读 + 节点操作组合

`readonly-node-ops-policy.json`：

```json
{
    "PolicyName": "TKE-ReadOnly-NodeManager",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tke:Describe*\",\"tke:Check*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"cvm:Describe*\",\"cvm:Inquiry*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"vpc:Describe*\",\"vpc:Inquiry*\",\"vpc:Get*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"clb:Describe*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"monitor:*\",\"cam:ListUsersForGroup\",\"cam:ListGroups\",\"cam:GetGroup\",\"cam:GetRole\"],\"resource\":\"*\"}]}",
    "Description": "TKE 全局只读权限"
}
```

## 验证

```bash
# 查看策略内容
tccli cam GetPolicy --PolicyId 400001 --region ap-guangzhou --output json
```

```json
{
    "PolicyName": "TKE-SingleCluster-Admin",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[...]}",
    "Description": "单个 TKE 集群的全读写管理权限（含节点扩缩容）",
    "Type": 2,
    "CreateMode": 1,
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

## 清理

> **警告**：删除自定义策略前必须先解除所有关联。直接删除已关联的策略将报 `PolicyInUse`。

```bash
# 1. 解除策略关联
tccli cam DetachUserPolicy --DetachUin 100012345678 --PolicyId 400001 --region ap-guangzhou --output json

# 2. 删除策略
tccli cam DeletePolicy --PolicyId 400001 --region ap-guangzhou --output json
```

验证已删：

```bash
tccli cam GetPolicy --PolicyId 400001 --region ap-guangzhou
# expected: ResourceNotFound
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePolicy` 报策略语法错误 | `echo '{"version":"2.0",...}' \| jq .` 验证 JSON 结构 | `PolicyDocument` 字段应为字符串化的 JSON，内部 `"` 需转义为 `\"` | 用 `jq` 验证后重试 |
| `DeletePolicy` 报 `PolicyInUse` | `tccli cam ListAttachedUserPolicies --TargetUin 100012345678 --region ap-guangzhou` 查看关联 | 策略仍有关联的用户/组/角色 | 先用 `DetachUserPolicy` 解除所有关联 |
| `CreatePolicy` 报 `PolicyNameInUse` | `tccli cam ListPolicies --Scope Local --region ap-guangzhou` 搜索同名 | 同名策略已存在 | 修改 PolicyName 或删除旧策略后重试 |
| 创建集群操作失败（权限不足） | 检查错误信息中的 Action 名 | 创建集群不仅需要 `tke:CreateCluster`，还需 `cvm:RunInstances`、`vpc:DescribeVpcEx` 等依赖权限 | 补充缺失的间接依赖 Action |
| 指定 `resource` 后仍报 `Unauthorized` | 在策略中将对应 statement 的 `resource` 改为 `"*"` 后重试 | 该 API 不支持资源级权限 | 将该 API 对应 statement 的 `resource` 改为 `"*"` |

## 下一步

- [通过标签为子账号配置批量集群的全读写权限](../通过标签为子账号配置批量集群的全读写权限/tccli%20操作.md) — page_id `46034`
- [使用 TKE 预设策略授权](../使用%20TKE%20预设策略授权/tccli%20操作.md) — page_id `46033`
- [授权模式对比](../授权模式对比/tccli%20操作.md) — page_id `46107`

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam) → **策略** → **新建自定义策略** → **按策略语法创建** → 选择空白模板 → 输入 JSON 策略内容 → **完成**。
