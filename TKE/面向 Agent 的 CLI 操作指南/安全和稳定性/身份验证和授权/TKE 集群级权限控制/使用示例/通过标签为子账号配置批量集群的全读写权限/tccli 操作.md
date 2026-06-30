# 通过标签为子账号配置批量集群的全读写权限（tccli）

> 对照官方：[通过标签为子账号配置批量集群的全读写权限](https://cloud.tencent.com/document/product/457/46034) · page_id `46034`

## 概述

通过 CAM 策略的 `qcs:resource_tag` 条件键，按集群标签（Tag）批量控制子账号的管理权限范围。为项目相关集群打上统一标签后（如 `env=production`），子账号自动获得所有带该标签集群的管理权限。新增打上该标签的集群也将自动纳入权限范围，无需逐一修改策略。

标签条件策略可与 `tke:GrantUserPermissions` 配合使用——CAM 标签策略控制 API 级别权限（哪些集群可见），`GrantUserPermissions` 在集群侧绑定 RBAC 角色（对可见集群的操作权限级别）。

## 前置条件

- 已完成 [环境准备](../../../../../环境准备.md)：安装 tccli 并配置凭据与默认地域。
- 演示集群 `cls-xxxxxxxx`（状态 Running，版本 v1.30.0，地域 ap-guangzhou）。本页所有命令以该集群和子账号 UIN `100012345678` 为例。
- 演示集群未暴露 kube-apiserver 公网入口，kubectl 不可达；权限验证只能通过 tccli API 完成。

环境检查：

```bash
tccli --version
tccli configure list
```

确认当前账号具备以下 CAM/TKE Action 权限：`cam:CreatePolicy`、`cam:AttachUserPolicy`、`cam:DetachUserPolicy`、`cam:DeletePolicy`、`cam:GetPolicy`、`cam:ListPolicies`、`tke:ModifyClusterTags`、`tke:DescribeClusters`、`tke:GrantUserPermissions`、`tke:DescribeUserPermissions`。

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
| 为集群打标签 | `tke ModifyClusterTags --ClusterId cls-xxxxxxxx --Tags '[{"Key":"env","Value":"production"}]'` | 否（覆盖式） |
| 创建标签条件策略 | `cam CreatePolicy --PolicyName NAME --PolicyDocument 'JSON'` | 否 |
| 关联策略到子账号 | `cam AttachUserPolicy --AttachUin 100012345678 --PolicyId POLICY_ID` | 否 |
| 授予集群级 RBAC 角色 | `tke GrantUserPermissions --TargetUin 100012345678 --Permissions.0.ClusterId cls-xxxxxxxx ...` | 否 |
| 查看策略内容 | `cam GetPolicy --PolicyId POLICY_ID` | 是 |
| 验证可见集群 | `tke DescribeClusters`（子账号身份） | 是 |
| 解除策略关联 | `cam DetachUserPolicy --DetachUin 100012345678 --PolicyId POLICY_ID` | 否 |
| 删除策略 | `cam DeletePolicy --cli-unfold-argument --PolicyId POLICY_ID` | 否 |

## 操作步骤

### 步骤 1：为集群打标签

使用项目维度标签（如 `env=production`），确保在所有需批量授权的集群间统一。`ModifyClusterTags` 会覆盖已有标签（非追加），先 `DescribeClusters` 查当前标签再修改，避免误删其他用途的标签。

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
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "TagSpecification": [
                {
                    "ResourceType": "cluster",
                    "Tags": [
                        {"Key": "env", "Value": "production"}
                    ]
                }
            ]
        }
    ],
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

### 步骤 2：创建标签条件策略

策略核心：通过 `qcs:resource_tag` 条件键限制子账号只能操作带有 `env=production` 标签的集群。`for_any_value:string_equal` 对数组中任一标签匹配即生效（OR 逻辑）。

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
    "RequestId": "e5f6a7b8-c9d0-1234-efab-345678901234"
}
```

| 字段 | 必填 | 取值与约束 | 错误后果 |
|------|:--:|------|------|
| `qcs:resource_tag` | 是 | 格式 `["KEY&VALUE"]`，`&` 连接标签键与值，键值区分大小写 | 格式错误 → 条件不生效，子账号看不到目标集群 |
| `for_any_value:string_equal` | 是 | 条件运算符，数组中任一标签匹配即生效（OR 逻辑） | 换用 `for_all_values:string_equal` → AND 逻辑，匹配要求更严格 |
| `version` | 是 | 必须为 `"2.0"` | 策略语法版本错误 → `CreatePolicy` 报错 |

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

### 步骤 4：多标签条件（高级）

支持多个标签组合（`for_any_value` = OR 逻辑）：任一标签匹配即生效。例如 `env=production` 或 `team=platform` 的集群均受策略控制：

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

如需 AND 逻辑（所有标签必须同时存在），改用 `for_all_values:string_equal`：

```json
"condition": {
    "for_all_values:string_equal": {
        "qcs:resource_tag": [
            "env&production",
            "team&platform"
        ]
    }
}
```

### 步骤 5：授予集群内部访问权限（可选）

标签条件策略控制子账号可见哪些集群。如需子账号具体操作集群内部资源（Pod、Service 等），还需在集群侧绑定 RBAC 角色：

```bash
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
    "RequestId": "a7b8c9d0-e1f2-3456-abcd-567890123456"
}
```

> `GrantUserPermissions` 需对每个匹配标签的集群逐一执行，不可批量。新增打上标签的集群也需要单独执行此步骤。

## 验证

查看策略内容确认条件键：

```bash
tccli cam GetPolicy --PolicyId 500001 --region ap-guangzhou --output json
```

```json
{
    "PolicyName": "TKE-TagBased-FullAccess",
    "Description": "按标签 env=production 批量授权集群全读写",
    "Type": 2,
    "AddTime": "2025-01-01 00:00:00",
    "UpdateTime": "2025-01-01 00:00:00",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":\"tke:*\",\"resource\":\"*\",\"condition\":{\"for_any_value:string_equal\":{\"qcs:resource_tag\":[\"env&production\"]}}}]}",
    "RequestId": "b8c9d0e1-f2a3-4567-bcde-678901234567"
}
```

子账号验证权限范围（需切换到子账号凭据执行）：

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0"
        }
    ],
    "RequestId": "c9d0e1f2-a3b4-5678-cdef-789012345678"
}
```

验证集群内部权限（如已执行 `GrantUserPermissions`）：

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
    "RequestId": "d0e1f2a3-b4c5-6789-defa-890123456789"
}
```

## 清理

> 删除标签条件策略后，子账号将失去对应标签集群的 TKE 操作权限。确认无其他子账号依赖该策略后再删除。如需撤销单个集群的访问权限，先执行 `DeleteUserPermissions`。

清理前状态检查：

```bash
tccli cam GetPolicy --PolicyId 500001 --region ap-guangzhou --output json
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

撤销集群侧权限（如有）并解除策略关联：

```bash
# 撤销集群侧权限
tccli tke DeleteUserPermissions --cli-unfold-argument \
    --TargetUin 100012345678 \
    --Permissions.0.ClusterId cls-xxxxxxxx \
    --Permissions.0.RoleName tke:admin \
    --Permissions.0.RoleType cluster \
    --region ap-guangzhou \
    --output json

# 解除策略关联
tccli cam DetachUserPolicy \
    --DetachUin 100012345678 \
    --PolicyId 500001 \
    --region ap-guangzhou \
    --output json

# 删除策略
tccli cam DeletePolicy --cli-unfold-argument \
    --PolicyId 500001 \
    --region ap-guangzhou \
    --output json
```

验证已删：

```bash
tccli cam GetPolicy --PolicyId 500001 --region ap-guangzhou
# expected: ResourceNotFound
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterTags` 报 `InvalidParameter` | 检查 Tags 格式 | 写成 Object 而非 Array，或 Value 类型不对 | 确保格式为 `[{"Key":"env","Value":"production"}]`，Key/Value 均为 String |
| `CreatePolicy` 报条件键语法错误 | 检查 `condition` 运算符拼写 | 运算符名拼写错误，或 `qcs:resource_tag` 值非数组 | 确认格式 `{"for_any_value:string_equal":{"qcs:resource_tag":["env&production"]}}` |
| `CreatePolicy` 报 `PolicyNameInUse` | 搜索同名策略 | 同名策略已存在 | 修改 PolicyName 或删除旧策略 |
| `ModifyClusterTags` 覆盖了原有标签 | 执行前查询现有标签 | `ModifyClusterTags` 的 Tags 参数完全覆盖已有标签（非追加） | 将需要保留的已有标签与新增标签合并后一并传入 Tags 参数 |
| 子账号看不到已打标签的集群 | 对比集群标签与策略条件键 | 集群标签 Key/Value 与策略条件键不完全一致（区分大小写） | 确保标签 Key/Value 字符级一致（含大小写） |
| 修改集群标签后权限未立即更新 | 等待后重试 | CAM 条件策略生效有延迟（秒级） | 等待 10-30 秒后重试 |
| 需要同时限制标签和地域 | 检查 condition | 当前策略仅含标签条件 | 在 condition 中追加 `"string_equal": {"tke:region": "ap-guangzhou"}` |

## 下一步

- [使用自定义策略授权](../../使用自定义策略授权/tccli%20操作.md)
- [配置子账号对单个 TKE 集群的管理权限](../配置子账号对单个%20TKE%20集群的管理权限/tccli%20操作.md)
- [配置子账号对 TKE 服务全读写或只读权限](../配置子账号对%20TKE%20服务全读写或只读权限/tccli%20操作.md)

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam) → **策略** → **新建自定义策略** → **按标签授权** → 选择服务「容器服务（tke）」→ 勾选操作 → 选择标签 Key/Value → 关联用户/组 → **完成**。
