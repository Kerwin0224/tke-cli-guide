# 使用 TKE 预设策略授权（tccli）

> 对照官方：[使用 TKE 预设策略授权](https://cloud.tencent.com/document/product/457/46033) · page_id `46033`

## 概述

TKE 提供多个 CAM 预设策略，可直接关联给子账号，无需创建自定义策略。涵盖全读写（`QcloudTKEFullAccess`）、只读（`QcloudTKEReadOnlyAccess`）等典型场景。关联方式分**直接关联**（策略绑定到子账号）和**随组关联**（子账号加入用户组，继承组策略）。

服务角色专属策略（如 `QcloudAccessForTKERole`）仅供 TKE 服务自身使用，不应关联给子账号。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达
- 已安装 tccli 并完成 `tccli configure`（secretId、secretKey、region 均已配置）
- 当前账号具备 CAM 管理权限（`cam:ListPolicies`、`cam:AttachUserPolicy`、`cam:DetachUserPolicy`、`cam:CreateGroup`、`cam:AddUserToGroup`、`cam:AttachGroupPolicy`）
- 目标子账号 UIN `100012345678` 已存在

### 环境检查

```bash
# 1. 确认 tccli 凭据已配置
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 2. 确认 CAM 权限——查询 TKE 预设策略
tccli cam ListPolicies --Scope "QCS" --Keyword "QcloudTKE" --region ap-guangzhou --output json
```

```json
{
    "List": [
        {
            "PolicyId": 300001,
            "PolicyName": "QcloudTKEFullAccess",
            "Description": "容器服务（TKE）全读写访问权限",
            "Type": 2,
            "CreateMode": 2,
            "ServiceType": "tke"
        },
        {
            "PolicyId": 300002,
            "PolicyName": "QcloudTKEReadOnlyAccess",
            "Description": "容器服务（TKE）只读访问权限",
            "Type": 2,
            "CreateMode": 2,
            "ServiceType": "tke"
        }
    ],
    "TotalNum": 2,
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 3. 确认目标子账号存在
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
| 查询所有 TKE 预设策略 | `tccli cam ListPolicies --Keyword QcloudTKE --Scope QCS --region ap-guangzhou` | 是 |
| 直接关联策略到子账号 | `tccli cam AttachUserPolicy --AttachUin 100012345678 --PolicyId 300001 --region ap-guangzhou` | 否 |
| 解除直接关联 | `tccli cam DetachUserPolicy --DetachUin 100012345678 --PolicyId 300001 --region ap-guangzhou` | 否 |
| 创建用户组 | `tccli cam CreateGroup --GroupName TKE-Admins --region ap-guangzhou` | 否 |
| 将子账号加入用户组 | `tccli cam AddUserToGroup --Uid 100012345678 --GroupId 50001 --region ap-guangzhou` | 否 |
| 将策略关联到用户组 | `tccli cam AttachGroupPolicy --PolicyId 300001 --AttachGroupId 50001 --region ap-guangzhou` | 否 |

## 操作步骤

### TKE 预设策略清单

**子账号授权用**（关联给子账号或用户组）：

| 策略名称 | 权限范围 | 推荐使用 |
|----------|----------|----------|
| `QcloudTKEFullAccess` | 全读写：管理所有 TKE 资源及关联的 CVM、CLB、VPC、监控等 | 推荐（覆盖最全） |
| `QcloudTKEReadOnlyAccess` | 只读：查看所有 TKE 资源 | 推荐用于只读场景 |

**服务角色专属**（仅供 TKE 服务角色 `TKE_QCSRole` 使用，**不应关联给子账号**）：

| 策略名称 | 说明 |
|----------|------|
| `QcloudAccessForTKERole` | TKE 服务操作 CVM、标签、CLB、日志等云资源 |
| `QcloudAccessForTKERoleInOpsManagement` | TKE 运维管理（含日志服务 CLS） |

### 方式一：直接关联（策略绑定到子账号）

适合子账号数量少的场景，简单直接。

```bash
# 授予子账号 100012345678 全读写权限（PolicyId 300001 = QcloudTKEFullAccess）
tccli cam AttachUserPolicy \
    --AttachUin 100012345678 \
    --PolicyId 300001 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 方式二：随组关联（策略绑定到用户组）

适合批量授权场景，一次配置覆盖组内所有子账号。

```bash
# 1. 创建用户组
tccli cam CreateGroup --GroupName "TKE-Admins" --Remark "TKE管理员组" --region ap-guangzhou --output json
```

```json
{
    "GroupId": 50001,
    "GroupName": "TKE-Admins",
    "Remark": "TKE管理员组",
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

```bash
# 2. 将子账号加入用户组
tccli cam AddUserToGroup --Uid 100012345678 --GroupId 50001 --region ap-guangzhou --output json
```

```bash
# 3. 将策略关联到用户组（组内成员自动继承）
tccli cam AttachGroupPolicy --PolicyId 300001 --AttachGroupId 50001 --region ap-guangzhou --output json
```

## 验证

```bash
# 确认策略已关联到子账号
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
    "RequestId": "e5f6a7b8-c9d0-1234-efab-345678901234"
}
```

## 清理

> **警告**：解除 `QcloudTKEFullAccess` 策略绑定后，子账号将失去所有 TKE 操作权限（包括查看集群）。生产环境需确认不影响业务后再操作。

```bash
# 直接关联：解除策略绑定
tccli cam DetachUserPolicy --DetachUin 100012345678 --PolicyId 300001 --region ap-guangzhou --output json
```

```bash
# 随组关联：从用户组移除子账号
tccli cam RemoveUserFromGroup --Uid 100012345678 --GroupId 50001 --region ap-guangzhou --output json
```

验证已解除：

```bash
tccli cam ListAttachedUserPolicies --TargetUin 100012345678 --region ap-guangzhou --output json
# expected: QcloudTKEFullAccess 不在列表中
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `AttachUserPolicy` 报 `InvalidParameter.Uin` | `tccli cam ListUsers --region ap-guangzhou` 检查 UIN 格式 | 传入了昵称或邮箱而非数字 UIN | 从 `ListUsers` 输出中获取正确的数字 UIN |
| `ListPolicies` 搜索不到 TKE 策略 | 确认 `--Keyword "QcloudTKE"` 且 `--Scope "QCS"` | Keyword 大小写不匹配，或 Scope 不是 QCS | 确保 `--Keyword "QcloudTKE"` 且 `--Scope "QCS"` |
| `CreateGroup` 报 `GroupNameInUse` | 确认组名是否已存在 | 用户组名称重复 | 使用不同的 GroupName，或复用已有用户组 |
| 子账号关联策略后仍无法调用 TKE API | `tccli cam ListAttachedUserPolicies --TargetUin 100012345678 --region ap-guangzhou` 检查策略名 | 关联了服务角色专属策略（如 `QcloudAccessForTKERole`） | 换为 `QcloudTKEFullAccess` 或 `QcloudTKEReadOnlyAccess` |
| 从用户组移除子账号后权限未立即撤销 | 等待 10-30 秒后重试 | 权限生效有短暂延迟（秒级） | 等待后重试验证 |

## 下一步

- [使用自定义策略授权](../使用自定义策略授权/tccli%20操作.md) — page_id `45337`
- [使用预设身份授权](../使用预设身份授权/tccli%20操作.md) — page_id `46105`
- [授权模式对比](../授权模式对比/tccli%20操作.md) — page_id `46107`

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam) → **用户** → **用户列表** → 选择子账号 → 点击 **关联策略** → 搜索「TKE」→ 勾选目标策略 → **确定**。或：**用户组** → 创建用户组 → 关联策略 → 将子账号加入用户组。
