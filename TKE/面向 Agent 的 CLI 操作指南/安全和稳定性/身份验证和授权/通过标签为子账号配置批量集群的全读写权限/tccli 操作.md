# 通过标签为子账号配置批量集群的全读写权限（tccli）

> 对照官方：[通过标签为子账号配置批量集群的全读写权限](https://cloud.tencent.com/document/product/457/46034) · page_id `46034`

## 概述

通过 CAM 策略的 `qcs:resource_tag` 条件键，按集群标签（Tag）批量控制子账号的管理权限范围。为项目相关集群打上统一标签后（如 `env=production`），子账号自动获得所有带该标签集群的管理权限。新增打上该标签的集群也将自动纳入权限范围，无需逐一修改策略。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达
- 已安装 tccli 并完成 `tccli configure`
- 当前账号具备 CAM 和 TKE 管理权限（`cam:CreatePolicy`、`cam:AttachUserPolicy`、`cam:DetachUserPolicy`、`cam:DeletePolicy`、`cam:GetPolicy`、`cam:ListPolicies`、`tke:ModifyClusterTags`、`tke:DescribeClusters`）
- 目标子账号 UIN `100012345678` 已存在

### 环境检查

```bash
# 1. 确认目标集群已存在
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
| 为集群打标签 | `tccli tke ModifyClusterTags --ClusterId cls-xxxxxxxx --Tags '[{"Key":"env","Value":"production"}]' --region ap-guangzhou` | 否 |
| 查看集群标签 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 创建标签条件策略 | `tccli cam CreatePolicy --cli-input-json file://tag-based-policy.json --region ap-guangzhou` | 否 |
| 关联策略到子账号 | `tccli cam AttachUserPolicy --AttachUin 100012345678 --PolicyId 500001 --region ap-guangzhou` | 否 |
| 查看策略内容 | `tccli cam GetPolicy --PolicyId 500001 --region ap-guangzhou` | 是 |
| 解除策略关联 | `tccli cam DetachUserPolicy --DetachUin 100012345678 --PolicyId 500001 --region ap-guangzhou` | 否 |
| 删除策略 | `tccli cam DeletePolicy --PolicyId 500001 --region ap-guangzhou` | 否 |

## 操作步骤

### 步骤 1：为集群打标签

> **选择依据**：使用项目维度的标签（如 `env=production`、`team=platform`），而非技术细节标签。确保标签在所有需要批量授权的集群间统一。`ModifyClusterTags` 会覆盖已有标签（非追加），先 Describe 查当前标签再修改。

```bash
tccli tke ModifyClusterTags \
    --ClusterId cls-xxxxxxxx \
    --Tags '[{"Key":"env","Value":"production"}]' \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

验证标签已生效：

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterStatus": "Running",
            "TagSpecification": [
                {"ResourceType": "cluster", "Tags": [{"Key": "env", "Value": "production"}]}
            ]
        }
    ],
    "TotalCount": 1,
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

### 步骤 2：创建标签条件策略

策略核心：通过 `qcs:resource_tag` 条件键限制子账号只能操作带有 `env=production` 标签的集群。服务设为 `tke`，操作设为 `tke:*`。

`tag-based-policy.json`：

```json
{
    "PolicyName": "TKE-TagBased-FullAccess",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":\"tke:*\",\"resource\":\"*\",\"condition\":{\"for_any_value:string_equal\":{\"qcs:resource_tag\":[\"env&production\"]}}}]}",
    "Description": "按标签 env=production 批量授权集群全读写"
}
```

```bash
tccli cam CreatePolicy --cli-input-json file://tag-based-policy.json --region ap-guangzhou --output json
```

```json
{
    "PolicyId": 500001,
    "PolicyName": "TKE-TagBased-FullAccess",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[...]}",
    "Description": "按标签 env=production 批量授权集群全读写",
    "RequestId": "e5f6a7b8-c9d0-1234-efab-345678901234"
}
```

### 步骤 3：关联策略到子账号

```bash
tccli cam AttachUserPolicy \
    --AttachUin 100012345678 \
    --PolicyId 500001 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

授权完成后，子账号仅能看到和操作带有 `env=production` 标签的集群。

### 多标签条件（高级）

支持多个标签组合（`for_any_value` = OR 逻辑）：

```json
"condition": {
    "for_any_value:string_equal": {
        "qcs:resource_tag": [
            "env&production",
            "team&platform"
        ]
    }
}
```

该条件表示：标签满足 `env=production` **或** `team=platform` 的集群均匹配。

## 验证

```bash
# 查看策略内容确认条件键
tccli cam GetPolicy --PolicyId 500001 --region ap-guangzhou --output json
```

```json
{
    "PolicyName": "TKE-TagBased-FullAccess",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":\"tke:*\",\"resource\":\"*\",\"condition\":{\"for_any_value:string_equal\":{\"qcs:resource_tag\":[\"env&production\"]}}}]}",
    "Description": "按标签 env=production 批量授权集群全读写",
    "RequestId": "a7b8c9d0-e1f2-3456-0123-456789012345"
}
```

子账号验证权限范围：

```bash
# 子账号执行——应仅返回带 env=production 标签的集群
tccli tke DescribeClusters --region ap-guangzhou --output json
# expected: 仅返回带指定标签的集群
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 清理

> **警告**：删除标签条件策略后，子账号将失去对应标签集群的 TKE 操作权限。删除策略前确认无其他子账号依赖该策略。

```bash
# 1. 解除策略关联
tccli cam DetachUserPolicy --DetachUin 100012345678 --PolicyId 500001 --region ap-guangzhou --output json

# 2. 删除策略
tccli cam DeletePolicy --PolicyId 500001 --region ap-guangzhou --output json
```

验证已删：

```bash
tccli cam GetPolicy --PolicyId 500001 --region ap-guangzhou
# expected: ResourceNotFound
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterTags` 报 `InvalidParameter` | 检查 Tags 格式 | Tags 格式错误（如写成 Object 而非 Array） | 确保格式为 `[{"Key":"env","Value":"production"}]`，Key/Value 均为字符串 |
| `CreatePolicy` 报条件键语法错误 | 检查 `condition` 中条件运算符和标签格式 | 条件键格式错误 | 确认格式为 `{"for_any_value:string_equal":{"qcs:resource_tag":["env&production"]}}` |
| 删除策略前忘记解除关联 | `tccli cam ListEntitiesForPolicy --PolicyId 500001 --region ap-guangzhou` 查看关联 | `DeletePolicy` 要求先解除所有关联 | 先执行 `DetachUserPolicy` 再删除 |
| 子账号看不到已打标签的集群 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou \| jq '.Clusters[].TagSpecification'` 对比标签 | 集群标签 Key/Value 与策略条件键不完全一致（区分大小写） | 确保标签 Key/Value 字符级一致（含大小写） |
| 修改集群标签后权限未立即更新 | 等待 10-30 秒后重试 | CAM 条件策略生效有短暂延迟（秒级） | 等待后重试验证 |

## 下一步

- [使用自定义策略授权](../使用自定义策略授权/tccli%20操作.md) — page_id `45337`
- [使用 TKE 预设策略授权](../使用%20TKE%20预设策略授权/tccli%20操作.md) — page_id `46033`
- [授权模式对比](../授权模式对比/tccli%20操作.md) — page_id `46107`

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam) → **策略** → **新建自定义策略** → **按标签授权** → 选择服务「容器服务（tke）」→ 勾选操作 → 选择标签 Key/Value → 关联用户/组 → **完成**。
