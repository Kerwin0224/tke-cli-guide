# TCR 镜像仓库资源级权限设置（tccli）

> 对照官方：[TCR 镜像仓库资源级权限设置](https://cloud.tencent.com/document/product/457/11527) · page_id `11527`

## 概述

TCR 企业版支持 CAM 资源级权限管理。通过 CAM 自定义策略的 `resource` 字段，可精确限制子账号只能访问特定的 TCR 实例、命名空间或镜像仓库。策略使用 QCS ARN 格式指定资源范围。

### 资源级权限层级

TCR 资源级权限支持三个粒度层级，从上到下逐级细化：

| 层级 | QCS ARN 格式 | 可访问范围 |
|------|-------------|-----------|
| 实例级 | `qcs::tcr:ap-guangzhou:uin/100012345678:instance/tcr-d7hd5b6q` | 整个 TCR 实例（所有命名空间和仓库）|
| 命名空间级 | `qcs::tcr:ap-guangzhou:uin/100012345678:repository/tcr-d7hd5b6q/team-a/*` | 指定命名空间下的所有仓库 |
| 仓库级 | `qcs::tcr:ap-guangzhou:uin/100012345678:repository/tcr-d7hd5b6q/team-a/app-frontend` | 单个指定的镜像仓库 |

> **注意**：`qcs::tcr:ap-guangzhou::instance/tcr-d7hd5b6q` 是省略 uin 的简写形式（等价于 `qcs::tcr:ap-guangzhou:uin/100012345678:instance/tcr-d7hd5b6q`），适用于同一主账号下的资源授权。

本页聚焦 TKE 场景下的 TCR 权限配置。本页为纯 CAM 操作，不涉及集群数据面，演示集群 `cls-xxxxxxxx` 仅作为 TKE 侧的引用上下文。

## 前置条件

- 已完成 [环境准备](../../../环境准备.md)，tccli 已安装并配置凭据。
- 主账号 UIN `100012345678`，目标子账号 UIN `100012345679`（已存在于 CAM）。
- 已开通 TCR 企业版，目标实例 `tcr-d7hd5b6q`（ap-guangzhou）。
- 演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou），kubectl 不可达——本页不依赖集群数据面，无需 kubectl。
- CAM 调用账号具备以下 Action 权限：`cam:CreatePolicy`、`cam:AttachUserPolicy`、`cam:DetachUserPolicy`、`cam:DeletePolicy`、`cam:GetPolicy`、`cam:ListPolicies`、`cam:ListAttachedUserPolicies`、`tcr:DescribeInstances`、`tcr:DescribeNamespaces`、`tcr:DescribeRepositories`。

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
# 3. 确认目标 TCR 企业版实例存在
tccli tcr DescribeInstances --region ap-guangzhou
# expected: 至少返回 1 个 TCR 实例，记录其 RegistryId
```

```json
{
    "Registries": [
        {
            "RegistryId": "tcr-d7hd5b6q",
            "RegistryName": "tcr-prod",
            "Status": "RUNNING",
            "Region": "ap-guangzhou"
        }
    ],
    "TotalCount": 1,
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 4. 确认目标子账号存在
tccli cam ListUsers --region ap-guangzhou
# expected: 返回子账号列表，确认目标子账号 UIN 100012345679
```

```bash
# 5. 确认目标命名空间和仓库存在（如需仓库级授权）
tccli tcr DescribeNamespaces --RegistryId tcr-d7hd5b6q --region ap-guangzhou
# expected: 返回命名空间列表
```

```json
{
    "Data": [
        {
            "Namespace": "team-a",
            "RegistryId": "tcr-d7hd5b6q",
            "CreationTime": "2025-01-01T00:00:00+00:00"
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

```bash
tccli tcr DescribeRepositories --RegistryId tcr-d7hd5b6q --NamespaceName team-a --region ap-guangzhou
# expected: 返回仓库列表
```

```json
{
    "Data": [
        {
            "Name": "app-frontend",
            "Namespace": "team-a",
            "RegistryId": "tcr-d7hd5b6q",
            "CreationTime": "2025-01-02T00:00:00+00:00"
        }
    ],
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看已有策略 | `cam ListPolicies --Scope Local --region ap-guangzhou` | 是 |
| 查看预设 TCR 策略 | `cam ListPolicies --Scope QCS --Keyword TCR --region ap-guangzhou` | 是 |
| 查看 TCR 实例列表 | `tcr DescribeInstances --region ap-guangzhou` | 是 |
| 查看命名空间列表 | `tcr DescribeNamespaces --RegistryId tcr-d7hd5b6q --region ap-guangzhou` | 是 |
| 查看仓库列表 | `tcr DescribeRepositories --RegistryId tcr-d7hd5b6q --region ap-guangzhou` | 是 |
| 创建自定义策略 | `cam CreatePolicy --cli-input-json file://policy.json --region ap-guangzhou` | 否 |
| 关联策略到子账号 | `cam AttachUserPolicy --cli-input-json file://attach.json --region ap-guangzhou` | 否 |
| 验证权限 | `cam ListAttachedUserPolicies --TargetUin 100012345679 --region ap-guangzhou` | 是 |

## 操作步骤

### 步骤 1：查看已有 TCR 预设策略

```bash
tccli cam ListPolicies --Scope "QCS" --Keyword "TCR" --region ap-guangzhou --output json
# expected: exit 0，返回 TCR 相关预设策略
```

```json
{
    "List": [
        {
            "PolicyId": 151001,
            "PolicyName": "QcloudTCRFullAccess",
            "Description": "容器镜像服务（TCR）全读写访问权限",
            "Type": 2,
            "CreateMode": 2
        }
    ],
    "TotalNum": 1,
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

预设策略（如 `QcloudTCRFullAccess`、`QcloudTCRReadOnlyAccess`）作用于账号下全部 TCR 实例。如需限制到特定实例/命名空间/仓库，需使用自定义策略。

### 步骤 2：创建自定义资源级策略

#### 选择依据

- **实例级授权**：通过 QCS ARN `qcs::tcr:ap-guangzhou:uin/100012345678:instance/tcr-d7hd5b6q` 精确指定到实例。适用于只允许子账号使用特定 TCR 实例的场景。
- **命名空间级授权**：通过 QCS ARN `qcs::tcr:ap-guangzhou:uin/100012345678:repository/tcr-d7hd5b6q/team-a/*` 精确指定到命名空间。适用于按团队/项目划分命名空间的场景。
- **仓库级授权**：通过 QCS ARN `qcs::tcr:ap-guangzhou:uin/100012345678:repository/tcr-d7hd5b6q/team-a/app-frontend` 精确指定到单个仓库。适用于按最小权限原则只授权特定仓库的场景。
- **action 选择**：仅拉取 `tcr:PullRepository` vs 全读写 `tcr:*`。根据最小权限原则选择。

#### 示例：仓库级只读策略（最小权限）

`tcr-readonly-policy.json`：

```json
{
    "PolicyName": "TCR-RepoReadOnly",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tcr:PullRepository\"],\"resource\":[\"qcs::tcr:ap-guangzhou:uin/100012345678:repository/tcr-d7hd5b6q/team-a/app-frontend\"]}]}",
    "Description": "TCR 特定仓库只读权限"
}
```

#### 示例：命名空间级读写策略

`tcr-ns-rw-policy.json`：

```json
{
    "PolicyName": "TCR-NAMESPACE-ReadWrite",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tcr:*\"],\"resource\":[\"qcs::tcr:ap-guangzhou:uin/100012345678:repository/tcr-d7hd5b6q/team-a/*\"]}]}",
    "Description": "TCR 指定命名空间全读写权限"
}
```

#### 示例：实例级只读策略

`tcr-instance-ro-policy.json`：

```json
{
    "PolicyName": "TCR-Instance-ReadOnly",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tcr:PullRepository\"],\"resource\":[\"qcs::tcr:ap-guangzhou::instance/tcr-d7hd5b6q\"]}]}",
    "Description": "TCR 指定实例只读权限"
}
```

```bash
tccli cam CreatePolicy --cli-input-json file://tcr-readonly-policy.json --region ap-guangzhou --output json
# expected: exit 0，返回 PolicyId
```

```json
{
    "PolicyId": 151101,
    "PolicyName": "TCR-RepoReadOnly",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"action\":[\"tcr:PullRepository\"],\"resource\":[\"qcs::tcr:ap-guangzhou:uin/100012345678:repository/tcr-d7hd5b6q/team-a/app-frontend\"]}]}",
    "Description": "TCR 特定仓库只读权限",
    "RequestId": "e5f6a7b8-c9d0-1234-efab-234567890abc"
}
```

### 步骤 3：绑定策略到子账号

`attach-policy.json`：

```json
{
    "AttachUin": "100012345679",
    "PolicyId": "151101"
}
```

```bash
tccli cam AttachUserPolicy --cli-input-json file://attach-policy.json --region ap-guangzhou --output json
# expected: exit 0，返回 RequestId
```

```json
{
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-34567890abcd"
}
```

## 验证

```bash
# 确认策略已关联到子账号
tccli cam ListAttachedUserPolicies --TargetUin 100012345679 --region ap-guangzhou --output json
# expected: 返回已关联策略列表，含目标策略
```

```json
{
    "List": [
        {
            "PolicyId": 151101,
            "PolicyName": "TCR-RepoReadOnly",
            "AddTime": "2026-06-18 00:00:00",
            "CreateMode": 1
        }
    ],
    "TotalNum": 1,
    "RequestId": "a7b8c9d0-e1f2-3456-abcd-4567890abcdef"
}
```

## 清理

本页为概念+查询演示，如实际创建了测试策略，按以下顺序清理（先解除关联再删除策略，否则报 `PolicyInUse`）。

```bash
# 1. 解除子账号策略关联
tccli cam DetachUserPolicy --DetachUin 100012345679 --PolicyId 151101 --region ap-guangzhou
# expected: exit 0
```

```bash
# 2. 删除自定义策略
tccli cam DeletePolicy --PolicyId 151101 --region ap-guangzhou
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePolicy` 报策略语法错误 | 检查 PolicyDocument 中 QCS ARN 格式 | QCS ARN 格式错误：某段缺失或格式不对（如 uin 未用 `uin/` 前缀）| 确认 ARN 各部分完整：Region 代码、UIN（数字）、RegistryId（`tcr-d7hd5b6q`）、Namespace、RepoName |
| `AttachUserPolicy` 报 `InvalidParameter.PolicyId` | 检查 PolicyId 是否为数字：`tccli cam ListPolicies --region ap-guangzhou` | PolicyId 应为数字而非策略名称 | 使用 `cam ListPolicies` 查询策略 ID，使用数字 PolicyId |
| `DeletePolicy` 报 `PolicyInUse` | 查看策略关联：`tccli cam ListEntitiesForPolicy --PolicyId 151101 --region ap-guangzhou` | 策略仍关联着用户/组/角色 | 先用 `DetachUserPolicy` 解除所有关联，再删除策略 |
| `ListPolicies` 搜索不到 TCR 相关策略 | 尝试扩大搜索范围：`tccli cam ListPolicies --Scope QCS --Keyword "镜像" --region ap-guangzhou` | Keyword 不匹配（实际策略名可能用 "TCR"、"CCR" 或中文）| 用 `--Keyword "镜像"` 或 `--Keyword "CCR"` 扩大搜索 |

### 权限不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 子账号关联策略后仍无法拉取镜像 | 对比策略中的 RegistryId/Namespace/RepoName 与实际目标 | 策略 QCS ARN 中的实例/命名空间/仓库名与实际不匹配 | 使用 `tccli tcr DescribeRepositories --RegistryId tcr-d7hd5b6q --region ap-guangzhou` 确认准确名称，修正策略 ARN |
| 策略绑定后权限未立即生效 | 等待数秒后重试验证：`tccli cam ListAttachedUserPolicies --TargetUin 100012345679 --region ap-guangzhou` | CAM 策略生效有短暂延迟（秒级）| 等待 10-30 秒后重试 |

## 下一步

- [服务授权相关角色权限说明](../服务授权相关角色权限说明/tccli%20操作.md) — TKE 服务角色与 TCR 权限边界
- [配置子账号对单个 TKE 集群的管理权限](../配置子账号对单个%20TKE%20集群的管理权限/tccli%20操作.md)
- [使用 TKE 预设策略授权](../使用%20TKE%20预设策略授权/tccli%20操作.md)

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam) → **策略** → **新建自定义策略** → **按资源授权** → 选择服务"容器镜像服务（tcr）"→ 选择操作（如 `tcr:PullRepository`）→ 指定资源（实例/命名空间/仓库）→ 关联用户/组 → **完成**。
