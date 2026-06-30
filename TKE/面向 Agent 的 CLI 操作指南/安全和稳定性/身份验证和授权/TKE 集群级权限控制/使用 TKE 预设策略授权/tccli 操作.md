# 使用 TKE 预设策略授权（tccli）

> 对照官方：[使用 TKE 预设策略授权](https://cloud.tencent.com/document/product/457/46033) · page_id `46033`

## 概述

TKE 提供多个 CAM 预设策略，可直接关联给子账号，无需创建自定义策略。涵盖全读写（`QcloudTKEFullAccess`）、只读（`QcloudTKEReadOnlyAccess`）等典型场景。此外还有服务角色专属策略（如 `QcloudAccessForTKERole`），仅供 TKE 服务自身使用，不应关联给子账号。

关联方式分直接关联（策略绑定到子账号）和随组关联（子账号加入用户组后继承组策略）。CAM 策略仅控制 API 级别权限；若需子账号操作集群内部 K8s 资源，还需调用 `tke:GrantUserPermissions` 在集群侧绑定 RBAC 角色。授权后用 `tke:DescribeUserPermissions` 验证集群级权限是否生效。

## 前置条件

- 已完成 [环境准备](../../../../环境准备.md)：安装 tccli 并配置凭据与默认地域。
- 演示集群 `cls-xxxxxxxx`（状态 Running，版本 v1.30.0，地域 ap-guangzhou）。本页所有命令以该集群和子账号 UIN `100012345678` 为例。
- 演示集群未暴露 kube-apiserver 公网入口，kubectl 不可达；权限验证只能通过 tccli API 完成。

环境检查：

```bash
tccli --version
tccli configure list
```

确认当前账号具备以下 CAM/TKE Action 权限：`cam:ListPolicies`、`cam:AttachUserPolicy`、`cam:DetachUserPolicy`、`cam:CreateGroup`、`cam:AddUserToGroup`、`cam:AttachGroupPolicy`、`tke:DescribeUserPermissions`、`tke:GrantUserPermissions`。

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
            "Uid": 1001408,
            "NickName": "开发子账号"
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查询所有 TKE 预设策略 | `cam ListPolicies --Keyword QcloudTKE --Scope QCS` | 是 |
| 直接关联策略到子账号 | `cam AttachUserPolicy --AttachUin 100012345678 --PolicyId 300001` | 否 |
| 解除直接关联 | `cam DetachUserPolicy --DetachUin 100012345678 --PolicyId 300001` | 否 |
| 创建用户组 | `cam CreateGroup --GroupName TKE-Admins` | 否 |
| 将子账号加入用户组 | `cam AddUserToGroup --Info.0.Uid 1001408 --Info.0.GroupId 50001` | 否 |
| 将策略关联到用户组 | `cam AttachGroupPolicy --PolicyId 300001 --AttachGroupId 50001` | 否 |
| 授予集群级 RBAC 角色 | `tke GrantUserPermissions --TargetUin 100012345678 --Permissions.0.ClusterId cls-xxxxxxxx ...` | 否 |
| 验证集群级权限 | `tke DescribeUserPermissions --TargetUin 100012345678` | 是 |

## 操作步骤

### 步骤 1：区分预设策略类型

子账号授权用（关联给子账号或用户组）：

| 策略名称 | 权限范围 | 推荐使用 |
|----------|----------|----------|
| `QcloudTKEFullAccess` | 全读写：管理所有 TKE 资源及关联的 CVM、CLB、VPC、监控等 | 推荐（覆盖最全） |
| `QcloudTKEReadOnlyAccess` | 只读：查看所有 TKE 资源 | 推荐用于只读场景 |
| `QcloudTKEInnerFullAccess` | TKE 全部权限 | 不推荐（覆盖不如 QcloudTKEFullAccess 完整） |

服务角色专属（仅供 TKE 服务角色 `TKE_QCSRole` 使用，不应关联给子账号——关联后子账号仍无法正常调用 TKE API，因为这类策略的作用主体是服务角色）：

| 策略名称 | 说明 |
|----------|------|
| `QcloudAccessForTKERole` | TKE 服务操作 CVM、标签、CLB、日志等云资源 |
| `QcloudAccessForTKERoleInOpsManagement` | TKE 运维管理（含日志服务 CLS） |
| `QcloudAccessForIPAMDofTKERole` | TKE IPAMD 弹性网卡权限 |

### 步骤 2：查询预设策略 ID

```bash
tccli cam ListPolicies --Keyword "QcloudTKE" --Scope "QCS" --region ap-guangzhou --output json
```

```json
{
    "List": [
        {
            "PolicyId": 300001,
            "PolicyName": "QcloudTKEFullAccess",
            "AddTime": "2024-06-01 00:00:00",
            "Type": 2,
            "Description": "容器服务（TKE）全读写访问权限",
            "CreateMode": 2,
            "Attachments": 3,
            "ServiceType": "tke"
        },
        {
            "PolicyId": 300002,
            "PolicyName": "QcloudTKEReadOnlyAccess",
            "AddTime": "2024-06-01 00:00:00",
            "Type": 2,
            "Description": "容器服务（TKE）只读访问权限",
            "CreateMode": 2,
            "Attachments": 1,
            "ServiceType": "tke"
        }
    ],
    "TotalNum": 2,
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

记录所需策略的 `PolicyId`。`Type: 2` 为预设策略，`CreateMode: 2` 表示腾讯云预设（而非用户自定义 `CreateMode: 1`）。

### 步骤 3：方式一 —— 直接关联（策略绑定到子账号）

适合子账号数量少的场景。授予全读写权限：

```bash
tccli cam AttachUserPolicy \
    --AttachUin 100012345678 \
    --PolicyId 300001 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

### 步骤 4：方式二 —— 随组关联（策略绑定到用户组）

适合批量授权。先创建用户组并关联策略，再将多个子账号加入该组，子账号自动继承组策略；后续新增子账号只需加入用户组。

```bash
tccli cam CreateGroup --GroupName "TKE-Admins" --Remark "TKE管理员组" --region ap-guangzhou --output json
```

```json
{
    "GroupId": 50001,
    "RequestId": "e5f6a7b8-c9d0-1234-efab-345678901234"
}
```

将子账号加入用户组（`Uid` 从 `ListUsers` 输出获取）：

```bash
tccli cam AddUserToGroup --cli-unfold-argument \
    --Info.0.Uid 1001408 \
    --Info.0.GroupId 50001 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

将策略关联到用户组：

```bash
tccli cam AttachGroupPolicy \
    --PolicyId 300001 \
    --AttachGroupId 50001 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "a7b8c9d0-e1f2-3456-abcd-567890123456"
}
```

### 步骤 5：授予集群级 RBAC 角色（可选）

CAM 策略仅控制 API 级别权限。若需子账号操作集群内部资源（Pod、Service 等），调用 `GrantUserPermissions` 在集群侧绑定 RBAC 角色：

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
    "RequestId": "b8c9d0e1-f2a3-4567-bcde-678901234567"
}
```

`RoleName` 与权限级别对应：

| RoleName | 权限级别 | 适用场景 |
|----------|----------|----------|
| `tke:admin` | 管理员（全读写） | 全读写策略 `tke:*` |
| `tke:ops` | 运维人员（读 + 部分写） | 日常运维、查看日志 |
| `tke:dev` | 开发人员（只读 + 命名空间操作） | 应用部署和调试 |
| `tke:ro` | 只读用户 | 只读策略 `tke:Describe*` |

> `GrantUserPermissions` 采用声明式语义：传入的 `Permissions` 列表代表子账号最终应拥有的全部权限，系统自动计算差异并执行创建/删除。重复传入相同列表不会产生副作用。

## 验证

确认策略已关联到子账号：

```bash
tccli cam ListAttachedUserPolicies --TargetUin 100012345678 --region ap-guangzhou --output json
```

```json
{
    "List": [
        {
            "PolicyId": 300001,
            "PolicyName": "QcloudTKEFullAccess",
            "AddTime": "2025-01-01 00:00:00",
            "CreateMode": 2,
            "PolicyType": "QCS"
        }
    ],
    "TotalNum": 1,
    "RequestId": "c9d0e1f2-a3b4-5678-cdef-789012345678"
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
    "RequestId": "d0e1f2a3-b4c5-6789-defa-890123456789"
}
```

## 清理

> 解除 `QcloudTKEFullAccess` 绑定后，子账号将失去所有 TKE 操作权限（包括查看集群）。生产环境确认不影响业务后再操作。

清理前状态检查：

```bash
tccli cam ListAttachedUserPolicies --TargetUin 100012345678 --region ap-guangzhou --output json
```

解除关联：

```bash
# 直接关联：解除策略绑定
tccli cam DetachUserPolicy \
    --DetachUin 100012345678 \
    --PolicyId 300001 \
    --region ap-guangzhou \
    --output json

# 随组关联：从用户组移除子账号
tccli cam RemoveUserFromGroup --cli-unfold-argument \
    --Info.0.Uid 1001408 \
    --Info.0.GroupId 50001 \
    --region ap-guangzhou \
    --output json
```

验证已解除：

```bash
tccli cam ListAttachedUserPolicies --TargetUin 100012345678 --region ap-guangzhou --output json
# expected: QcloudTKEFullAccess 不在列表中
```

可选：删除不再需要的用户组：

```bash
tccli cam DeleteGroup --GroupId 50001 --region ap-guangzhou --output json
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `AttachUserPolicy` 报 `InvalidParameter.Uin` | 检查 UIN 是否为数字 | 传入了昵称或邮箱而非数字 UIN | 从 `cam ListUsers` 输出的 `Uin` 字段获取数字 ID |
| `ListPolicies` 搜不到 TKE 策略 | 确认参数 | `Keyword` 大小写不匹配或 `Scope` 非 `QCS` | 确保 `--Keyword "QcloudTKE"` 且 `--Scope "QCS"` |
| `CreateGroup` 报 `GroupNameInUse` | 确认组名 | 用户组名称重复 | 使用不同的 GroupName |
| `DescribeUserPermissions` 返回空 | 检查是否已授予 RBAC | 仅关联了 CAM 策略，未调用 `GrantUserPermissions` | 执行 `GrantUserPermissions` 绑定集群角色，等待 10-30 秒后重试 |
| 子账号关联策略后仍无法调用 TKE API | 检查策略名称 | 关联了服务角色专属策略（如 `QcloudAccessForTKERole`） | 换为 `QcloudTKEFullAccess` 或 `QcloudTKEReadOnlyAccess` |
| `DescribeUserPermissions` 返回 `tke:ro` 但需管理员 | 对比策略与所需操作 | 关联的是 `QcloudTKEReadOnlyAccess` | 换为 `QcloudTKEFullAccess` 并重新授予 `tke:admin` 角色 |

## 下一步

- [使用自定义策略授权](../使用自定义策略授权/tccli%20操作.md)
- [配置子账号对 TKE 服务全读写或只读权限](../使用示例/配置子账号对%20TKE%20服务全读写或只读权限/tccli%20操作.md)
- [配置子账号对单个 TKE 集群的管理权限](../使用示例/配置子账号对单个%20TKE%20集群的管理权限/tccli%20操作.md)

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam) → **用户** → **用户列表** → 选择子账号 → **关联策略** → 搜索「TKE」→ 勾选目标策略 → **确定**。或：**用户组** → 创建用户组 → 关联策略 → 将子账号加入用户组。
