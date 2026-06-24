# 身份验证和授权概述（tccli）

> 对照官方：[概述（身份验证和授权）](https://cloud.tencent.com/document/product/457/11542) · page_id `11542`

## 概述

TKE 的身份验证和授权体系分为两层：

1. **云 API 层（CAM）**：通过腾讯云访问管理（Cloud Access Management）控制子账号能否调用 TKE 云 API —— 即"谁能进集群大门"。管理员可为不同子账号分配 TKE 全读写、只读或自定义策略。
2. **集群内对象级（RBAC）**：在集群内部通过 Kubernetes RBAC 控制子账号能操作哪些 K8s 资源（Pod、Deployment、Namespace 等）—— 即"进门后能做什么"。TKE 基于 x509 证书认证，每个子账号拥有独立的客户端证书。

当多人共用一个云账号密钥管理 TKE 时，存在密钥泄露风险和无权限管控两大问题。通过 CAM 为不同人员创建子账号并授予最小权限，可实现安全的协作管理。

### 策略类型对比

| 策略类型 | 层级 | 管理平台 | 作用范围 | 示例 |
|---------|------|---------|---------|------|
| CAM 预设策略 | 云 API 层 | CAM 控制台 / tccli | 账号下全部 TKE 集群 | `QcloudTKEFullAccess`（全读写）、`QcloudTKEReadOnlyAccess`（只读）|
| CAM 自定义策略 | 云 API 层 | CAM 控制台 / tccli | 可精确到单个集群、单个 API Action | 仅允许 `DescribeClusters` 操作集群 `cls-xxxxxxxx` |
| Kubernetes RBAC（ClusterRole） | 集群内 | TKE 控制台 / kubectl | 单个集群内全部命名空间 | `tke:admin`、`tke:ops`、`tke:dev`、`tke:ro` |
| Kubernetes RBAC（Role） | 集群内 | TKE 控制台 / kubectl | 单个命名空间 | 仅允许在 `dev` 命名空间操作 Deployment |

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，本页仅通过 tccli 操作控制面
- 完成 [环境准备](../../../环境准备.md)
- 当前账号具备 `cam:ListUsers`、`cam:GetUser`、`cam:ListAttachedUserPolicies`、`cam:ListPolicies`、`tke:DescribeClusters`、`tke:DescribeUserPermissions` 权限

### 环境检查

```bash
# 1. 确认集群状态
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
            "ClusterOs": "tlinux3.1",
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
    "TotalNum": 2,
    "Data": [
        {
            "Uin": 100000000001,
            "Name": "dev-user",
            "Uid": 10001,
            "Remark": "development team"
        },
        {
            "Uin": 100000000002,
            "Name": "ops-user",
            "Uid": 10002,
            "Remark": "ops team"
        }
    ],
    "RequestId": "b1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看子账号列表 | `tccli cam ListUsers --region ap-guangzhou` | 是 |
| 查看子账号详情 | `tccli cam GetUser --Name <name> --region ap-guangzhou` | 是 |
| 查看用户关联策略 | `tccli cam ListAttachedUserPolicies --TargetUin <Uin> --region ap-guangzhou` | 是 |
| 创建子账号 | `tccli cam AddUser --Name <name> --Remark <remark> --ConsoleLogin 1 --region ap-guangzhou` | 否 |
| 为子账号授权 | `tccli cam AttachUserPolicy --AttachUin <Uin> --PolicyId <PolicyId> --region ap-guangzhou` | 否 |
| 查看预设策略 | `tccli cam ListPolicies --Scope QCS --Keyword TKE --region ap-guangzhou` | 是 |
| 查看集群内已授权用户 | `tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |

## 操作步骤

### 步骤 1：查看当前账号的 TKE 云 API 权限

```bash
tccli cam ListAttachedUserPolicies --TargetUin 100000000001 --region ap-guangzhou --output json
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
    "RequestId": "c1c2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 步骤 2：查看所有 TKE 预设策略

```bash
tccli cam ListPolicies --Scope "QCS" --Keyword "TKE" --region ap-guangzhou --output json
```

```json
{
    "List": [
        {
            "PolicyId": 300001,
            "PolicyName": "QcloudTKEFullAccess",
            "Description": "容器服务（TKE）全读写访问权限",
            "Type": 2,
            "CreateMode": 2
        },
        {
            "PolicyId": 300002,
            "PolicyName": "QcloudTKEReadOnlyAccess",
            "Description": "容器服务（TKE）只读访问权限",
            "Type": 2,
            "CreateMode": 2
        }
    ],
    "TotalNum": 2,
    "RequestId": "d1d2d3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 步骤 3：为子账号授予 TKE 全读写权限

```bash
tccli cam AttachUserPolicy --AttachUin 100000000002 \
    --PolicyId 300001 --region ap-guangzhou --output json
```

```json
{
    "RequestId": "e1e2e3e4-f5f6-7890-abcd-ef1234567890"
}
```

### 步骤 4：查看集群内的 RBAC 授权情况

```bash
tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "UserPermissions": [
        {
            "Uin": "100000000001",
            "Nickname": "dev-user",
            "RBACName": "tke:dev",
            "IsAdmin": false
        },
        {
            "Uin": "100000000002",
            "Nickname": "ops-user",
            "RBACName": "tke:ops",
            "IsAdmin": false
        }
    ],
    "RequestId": "f1f2f3f4-a5b6-7890-abcd-ef1234567890"
}
```

## 验证

```bash
# 确认策略已关联到子账号
tccli cam ListAttachedUserPolicies --TargetUin 100000000002 --region ap-guangzhou --output json
```

```json
{
    "List": [
        {
            "PolicyId": 300001,
            "PolicyName": "QcloudTKEFullAccess",
            "AddTime": "2025-06-18 00:00:00",
            "CreateMode": 2,
            "PolicyType": "QCS"
        }
    ],
    "TotalNum": 1,
    "RequestId": "a1a2a3a4-b5b6-7890-abcd-ef1234567890"
}
```

```bash
# 确认集群内 RBAC 授权
tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "UserPermissions": [
        {
            "Uin": "100000000001",
            "Nickname": "dev-user",
            "RBACName": "tke:dev",
            "IsAdmin": false
        },
        {
            "Uin": "100000000002",
            "Nickname": "ops-user",
            "RBACName": "tke:ops",
            "IsAdmin": false
        }
    ],
    "RequestId": "b1b2b3b4-c5c6-7890-abcd-ef1234567890"
}
```

## 清理

无需清理。本页为概念概述，不涉及需回收的资源。若为子账号授予了测试策略，可通过以下命令解绑：

```bash
tccli cam DetachUserPolicy --DetachUin 100000000002 \
    --PolicyId 300001 --region ap-guangzhou --output json
```

```json
{
    "RequestId": "c1c2c3c4-d5d6-7890-abcd-ef1234567890"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ListAttachedUserPolicies` 返回空列表 | `tccli cam ListUsers --region ap-guangzhou` 检查目标 UIN 是否正确 | 子账号未关联任何策略，或 UIN 填写错误 | 使用 `tccli cam AttachUserPolicy` 为子账号关联策略 |
| `ListPolicies` 找不到 TKE 策略 | 检查 `Scope` 和 `Keyword` 参数 | `Scope` 参数错误（应为 `QCS`）或 `Keyword` 大小写不匹配 | 确保 `--Scope "QCS"` 且 `--Keyword "TKE"` |
| `AttachUserPolicy` 返回 `InvalidParameter.Uin` | `tccli cam GetUser --Name <name> --region ap-guangzhou` 检查 UIN 格式 | 传入了昵称或邮箱而非数字 UIN | 使用 `tccli cam ListUsers --region ap-guangzhou` 获取正确的数字 UIN |
| `DescribeUserPermissions` 返回空列表 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` 确认集群存在 | 集群可能处于旧授权模式，不支持 `DescribeUserPermissions` | 在控制台触发「RBAC 策略生成器」升级至新模式 |
| 子账号能看到集群但无法操作 | `tccli cam GetPolicy --PolicyId <PolicyId> --region ap-guangzhou` 检查策略详情 | 策略权限不足，仅含只读操作 | 确认策略包含写操作 Action，或更换更高权限的策略 |
| 部分 API 报 `UnauthorizedOperation` | 在策略 JSON 中检查 `resource` 字段值 | 部分 TKE API 不支持资源级权限，需在策略中指定 `"resource": "*"` | 将对应 statement 的 `resource` 改为 `"*"` |
| 子账号 CAM 策略生效但 kubectl 操作被拒 | `tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` 检查 RBAC 绑定 | CAM 策略通过但集群内 RBAC 角色权限不足 | 在控制台「授权管理」为该子账号提升 RBAC 角色等级 |

## 下一步

- [集群安全增强能力](../../集群安全增强能力/tccli%20操作.md) — 一键开启集群级安全加固
- [策略管理（OPA/Gatekeeper）](../../应用安全/策略管理/tccli%20操作.md) — 准入控制与删除保护
- [使用腾讯云密钥管理系统 KMS 进行 ETCD 数据加密](../../控制平面安全/使用腾讯云密钥管理系统%20KMS%20进行%20ETCD%20数据加密/tccli%20操作.md) — etcd Secret 加密
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853) — 完整 API 列表

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam) → **用户** → **用户列表** → 选择子账号 → **关联策略**，搜索并勾选 TKE 相关策略。登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → **授权管理** → **RBAC 策略生成器** 配置集群内权限。
