# 配置子账号对单个 TKE 集群的管理权限（tccli）

> 对照官方：[配置子账号对单个 TKE 集群的管理权限](https://cloud.tencent.com/document/product/457/31556) · page_id `31556`

## 概述

通过 CAM 自定义策略的 `resource` 字段限制子账号只能操作特定集群，实现最小权限原则。提供全读写和只读两种策略模板。

策略通过 QCS 资源描述格式指定目标集群：`qcs::tke:REGION::cluster/CLUSTER_ID`。Region 代码支持完整地域名（如 `ap-guangzhou`）或短代码（如 `gz`）。

> CAM 策略仅控制 API 级别的权限。要操作集群内部资源（Pod、Service 等），还需额外调用 `tke:GrantUserPermissions` 在集群侧绑定 RBAC 角色。两项配置互补，缺一不可。

## 前置条件

- 已完成 [环境准备](../../../../../环境准备.md)：安装 tccli 并配置凭据与默认地域。
- 演示集群 `cls-xxxxxxxx`（状态 Running，版本 v1.30.0，地域 ap-guangzhou）。本页所有命令以该集群和子账号 UIN `100012345678` 为例。
- 演示集群未暴露 kube-apiserver 公网入口，kubectl 不可达；权限验证只能通过 tccli API 完成。

环境检查：

```bash
tccli --version
tccli configure list
```

确认当前账号具备以下 CAM/TKE Action 权限：`cam:CreatePolicy`、`cam:AttachUserPolicy`、`cam:DetachUserPolicy`、`cam:DeletePolicy`、`cam:GetPolicy`、`cam:ListPolicies`、`tke:GrantUserPermissions`、`tke:DeleteUserPermissions`、`tke:DescribeUserPermissions`。

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
| 创建单集群策略 | `cam CreatePolicy --PolicyName NAME --PolicyDocument 'JSON'` | 否 |
| 关联策略到子账号 | `cam AttachUserPolicy --AttachUin 100012345678 --PolicyId POLICY_ID` | 否 |
| 授予集群级 RBAC 角色 | `tke GrantUserPermissions --TargetUin 100012345678 --Permissions.0.ClusterId cls-xxxxxxxx ...` | 否 |
| 查看策略内容 | `cam GetPolicy --PolicyId POLICY_ID` | 是 |
| 验证集群级权限 | `tke DescribeUserPermissions --TargetUin 100012345678` | 是 |
| 解除策略关联 | `cam DetachUserPolicy --DetachUin 100012345678 --PolicyId POLICY_ID` | 否 |
| 删除策略 | `cam DeletePolicy --cli-unfold-argument --PolicyId POLICY_ID` | 否 |

## 操作步骤

### 步骤 1：方案一 —— 单个集群全读写权限

`tke:*` 限定在演示集群，CVM 实例限定到同地域，其他依赖服务使用 `"*"`。适合将单个生产集群授权给特定运维团队。

`single-cluster-full-access.json`：

```json
{
    "PolicyName": "TKE-FullAccess-SingleCluster",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tke:*\"],\"resource\":[\"qcs::tke:ap-guangzhou::cluster/cls-xxxxxxxx\",\"qcs::cvm:ap-guangzhou::instance/*\"]},{\"effect\":\"allow\",\"action\":[\"cvm:*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"vpc:*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"clb:*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"monitor:*\",\"cam:ListUsersForGroup\",\"cam:ListGroups\",\"cam:GetGroup\",\"cam:GetRole\"],\"resource\":\"*\"}]}",
    "Description": "单个 TKE 集群全读写管理权限"
}
```

```bash
tccli cam CreatePolicy --cli-input-json file://single-cluster-full-access.json --region ap-guangzhou --output json
```

```json
{
    "PolicyId": 600001,
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

> 如需子账号执行集群扩缩容，还需额外配置支付权限。

### 步骤 2：方案二 —— 单个集群只读权限

仅授予 `Describe*`、`Check*`、`Inquiry*`、`Get*` 前缀的只读 API。CVM、VPC、CLB 仅授予只读权限。

`single-cluster-readonly.json`：

```json
{
    "PolicyName": "TKE-ReadOnly-SingleCluster",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tke:Describe*\",\"tke:Check*\"],\"resource\":\"qcs::tke:ap-guangzhou::cluster/cls-xxxxxxxx\"},{\"effect\":\"allow\",\"action\":[\"cvm:Describe*\",\"cvm:Inquiry*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"vpc:Describe*\",\"vpc:Inquiry*\",\"vpc:Get*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"clb:Describe*\"],\"resource\":\"*\"},{\"effect\":\"allow\",\"action\":[\"monitor:*\",\"cam:ListUsersForGroup\",\"cam:ListGroups\",\"cam:GetGroup\",\"cam:GetRole\"],\"resource\":\"*\"}]}",
    "Description": "单个 TKE 集群只读权限"
}
```

```bash
tccli cam CreatePolicy --cli-input-json file://single-cluster-readonly.json --region ap-guangzhou --output json
```

```json
{
    "PolicyId": 600002,
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

### 步骤 3：关联策略并授予集群级 RBAC 角色

CAM 策略绑定后，必须调用 `GrantUserPermissions` 在集群侧建立 RBAC 绑定。两项操作互补：

| 步骤 | 命令 | 作用 |
|------|------|------|
| 1. 绑定 CAM 策略 | `cam AttachUserPolicy` | 控制 API 级别权限（能否调用 `tke:*`、`cvm:*` 等接口） |
| 2. 授予集群访问 | `tke GrantUserPermissions` | 控制集群内部权限（能否操作 Pod、Service 等 K8s 资源） |

```bash
# 步骤 1：关联 CAM 策略
tccli cam AttachUserPolicy \
    --AttachUin 100012345678 \
    --PolicyId 600001 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "e5f6a7b8-c9d0-1234-efab-345678901234"
}
```

```bash
# 步骤 2：授予集群级管理员角色
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
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

`RoleName` 与权限级别对应：

| RoleName | 权限级别 | 适用场景 |
|----------|----------|----------|
| `tke:admin` | 管理员（全读写） | 全读写策略 `tke:*` |
| `tke:ops` | 运维人员（读 + 部分写） | 日常运维、查看日志 |
| `tke:dev` | 开发人员（只读 + 命名空间操作） | 应用部署和调试 |
| `tke:ro` | 只读用户 | 只读策略 `tke:Describe*` |

### 步骤 4：地域代码映射

QCS ARN 中支持完整地域名或短地域代码，两种格式等价：

| 地域 | 完整地域名 | 短代码 | 地域 | 完整地域名 | 短代码 |
|------|-----------|--------|------|-----------|--------|
| 广州 | `ap-guangzhou` | `gz` | 上海 | `ap-shanghai` | `sh` |
| 北京 | `ap-beijing` | `bj` | 成都 | `ap-chengdu` | `cd` |
| 重庆 | `ap-chongqing` | `cq` | 南京 | `ap-nanjing` | `nj` |
| 香港 | `ap-hongkong` | `hk` | 新加坡 | `ap-singapore` | `sg` |
| 硅谷 | `na-siliconvalley` | `usw` | 弗吉尼亚 | `na-ashburn` | `use` |
| 法兰克福 | `eu-frankfurt` | `de` | 东京 | `ap-tokyo` | `jp` |

示例：`qcs::tke:ap-guangzhou::cluster/cls-xxxxxxxx` 与 `qcs::tke:gz::cluster/cls-xxxxxxxx` 等价。

## 验证

查看策略内容：

```bash
tccli cam GetPolicy --PolicyId 600001 --region ap-guangzhou --output json
```

```json
{
    "PolicyName": "TKE-FullAccess-SingleCluster",
    "Description": "单个 TKE 集群全读写管理权限",
    "Type": 2,
    "AddTime": "2025-01-01 00:00:00",
    "UpdateTime": "2025-01-01 00:00:00",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[...]}",
    "RequestId": "a7b8c9d0-e1f2-3456-abcd-567890123456"
}
```

验证集群侧权限已生效：

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
    "RequestId": "b8c9d0e1-f2a3-4567-bcde-678901234567"
}
```

## 清理

> 删除策略前必须先解除所有 CAM 关联并撤销集群侧权限（`DeleteUserPermissions`）。仅删除 CAM 策略不会自动移除集群内部 RBAC 权限。

清理前状态检查：

```bash
tccli cam GetPolicy --PolicyId 600001 --region ap-guangzhou --output json
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
    "RequestId": "c9d0e1f2-a3b4-5678-cdef-789012345678"
}
```

解除 CAM 关联并删除策略：

```bash
tccli cam DetachUserPolicy \
    --DetachUin 100012345678 \
    --PolicyId 600001 \
    --region ap-guangzhou \
    --output json

tccli cam DeletePolicy --cli-unfold-argument \
    --PolicyId 600001 \
    --region ap-guangzhou \
    --output json
```

验证已删：

```bash
tccli cam GetPolicy --PolicyId 600001 --region ap-guangzhou
# expected: ResourceNotFound
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePolicy` 返回策略语法错误 | 检查 QCS ARN 格式 | ARN 中 Region、资源类型或资源 ID 格式错误 | 确认格式 `qcs::tke:ap-guangzhou::cluster/cls-xxxxxxxx`，每段用 `::` 分隔 |
| `DeletePolicy` 报 `PolicyInUse` | 查看关联：`cam ListEntitiesForPolicy --PolicyId POLICY_ID` | 策略仍关联着用户/组 | 先 `DetachUserPolicy` 解除所有关联 |
| `CreatePolicy` 报 `PolicyNameInUse` | 搜索同名策略 | 同名策略已存在 | 修改 PolicyName 或删除旧策略 |
| 子账号仍能查看其他集群 | 检查策略 JSON 中 `resource` 值 | `resource` 未正确限定到目标集群 | 确保 TKE 部分 resource 指向正确地域和集群 |
| 只读策略的 `DescribeClusters` 返回空 | 使用不加 ClusterIds 的查询 | CAM 资源级权限下，不加 ClusterIds 过滤时可能返回空（预期行为） | 子账号调用时明确指定 `--ClusterIds '["cls-xxxxxxxx"]'` |
| 地域代码写错导致策略不生效 | 检查 QCS ARN 中 Region 与集群实际地域 | 短代码与集群地域不匹配 | 确保 Region 代码与集群所在地域一致 |
| CAM 策略已绑定但无法操作 K8s 资源 | 检查集群侧 RBAC | 未调用 `GrantUserPermissions` 建立 RBAC 绑定 | 执行 `GrantUserPermissions` 绑定 RBAC 角色 |

## 下一步

- [配置子账号对 TKE 服务全读写或只读权限](../配置子账号对%20TKE%20服务全读写或只读权限/tccli%20操作.md)
- [通过标签为子账号配置批量集群的全读写权限](../通过标签为子账号配置批量集群的全读写权限/tccli%20操作.md)
- [使用自定义策略授权](../../使用自定义策略授权/tccli%20操作.md)

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam) → **策略** → **新建自定义策略** → **按策略语法创建** → 空白模板 → 输入策略 JSON → 关联用户 → **完成**。然后在 [TKE 控制台](https://console.cloud.tencent.com/tke2) → 集群详情 → **授权管理** → 添加子账号并选择角色。
