# 配置子账号对 TKE 服务全读写或只读权限（tccli）

> 对照官方：[配置子账号对 TKE 服务全读写或只读权限](https://cloud.tencent.com/document/product/457/31557) · page_id `31557`

## 概述

为子账号授予 TKE 服务的全局访问权限。需要同时关联 TKE 策略和容器镜像服务（CCR/TCR）策略，因为 TKE 集群操作（创建、拉取镜像等）依赖镜像仓库权限。两条策略缺一不可。

| 权限级别 | TKE 策略 | CCR 策略 |
|----------|----------|----------|
| 全读写 | `QcloudTKEFullAccess` | `QcloudCCRFullAccess` |
| 只读 | `QcloudTKEReadOnlyAccess` | `QcloudCCRReadOnlyAccess` |

> 上述 CAM 预设策略仅控制 API 级别权限。如需子账号操作集群内部 K8s 资源（Pod、Service 等），还需调用 `tke:GrantUserPermissions` 在集群侧绑定 RBAC 角色。

## 前置条件

- 已完成 [环境准备](../../../环境准备.md)：安装 tccli 并配置凭据与默认地域。
- 演示集群 `cls-xxxxxxxx`（状态 Running，版本 v1.30.0，地域 ap-guangzhou）。本页所有命令以该集群和子账号 UIN `100012345678` 为例。
- 演示集群未暴露 kube-apiserver 公网入口，kubectl 不可达；权限验证只能通过 tccli API 完成。

环境检查：

```bash
tccli --version
tccli configure list
```

确认当前账号具备以下 CAM/TKE Action 权限：`cam:ListPolicies`、`cam:AttachUserPolicy`、`cam:DetachUserPolicy`、`cam:ListAttachedUserPolicies`、`tke:DescribeUserPermissions`。

资源检查：

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
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查询 TKE 预设策略 | `cam ListPolicies --Keyword QcloudTKE --Scope QCS` | 是 |
| 查询 CCR 预设策略 | `cam ListPolicies --Keyword QcloudCCR --Scope QCS` | 是 |
| 绑定 TKE 全读写 | `cam AttachUserPolicy --AttachUin 100012345678 --PolicyId 300001` | 否 |
| 绑定 CCR 全读写 | `cam AttachUserPolicy --AttachUin 100012345678 --PolicyId 400001` | 否 |
| 解除绑定 | `cam DetachUserPolicy --DetachUin 100012345678 --PolicyId POLICY_ID` | 否 |
| 验证集群权限 | `tke DescribeUserPermissions --TargetUin 100012345678` | 是 |

## 操作步骤

### 步骤 1：查询所需策略 ID

```bash
tccli cam ListPolicies --Keyword "QcloudTKE" --Scope "QCS" --region ap-guangzhou --output json
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
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

```bash
tccli cam ListPolicies --Keyword "QcloudCCR" --Scope "QCS" --region ap-guangzhou --output json
```

```json
{
    "List": [
        {
            "PolicyId": 400001,
            "PolicyName": "QcloudCCRFullAccess",
            "Description": "容器镜像服务（CCR）全读写访问权限",
            "Type": 2,
            "CreateMode": 2
        },
        {
            "PolicyId": 400002,
            "PolicyName": "QcloudCCRReadOnlyAccess",
            "Description": "容器镜像服务（CCR）只读访问权限",
            "Type": 2,
            "CreateMode": 2
        }
    ],
    "TotalNum": 2,
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

记录所需策略的 `PolicyId`。TKE 策略和 CCR 策略的 PolicyId 为不同数字，需分别记录。

### 步骤 2：授予全读写权限

同时关联 TKE 和 CCR 全读写策略。仅关联 TKE 策略 → 可操作集群但无法推送镜像；仅关联 CCR 策略 → 可推送镜像但无法操作集群。两条缺一不可。

```bash
# TKE 全读写
tccli cam AttachUserPolicy \
    --AttachUin 100012345678 \
    --PolicyId 300001 \
    --region ap-guangzhou \
    --output json

# CCR 全读写
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

### 步骤 3：授予只读权限

```bash
# TKE 只读
tccli cam AttachUserPolicy \
    --AttachUin 100012345678 \
    --PolicyId 300002 \
    --region ap-guangzhou \
    --output json

# CCR 只读
tccli cam AttachUserPolicy \
    --AttachUin 100012345678 \
    --PolicyId 400002 \
    --region ap-guangzhou \
    --output json
```

```json
{
    "RequestId": "e5f6a7b8-c9d0-1234-efab-345678901234"
}
```

## 验证

确认策略均已关联：

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
        },
        {
            "PolicyId": 400001,
            "PolicyName": "QcloudCCRFullAccess",
            "AddTime": "2025-01-01 00:01:00",
            "CreateMode": 2,
            "PolicyType": "QCS"
        }
    ],
    "TotalNum": 2,
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

子账号验证权限（需切换到子账号凭据执行）：

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
    "RequestId": "a7b8c9d0-e1f2-3456-abcd-567890123456"
}
```

## 清理

> 解除 TKE 全读写策略后，子账号将失去所有 TKE 操作权限。解除 CCR 策略后，子账号无法拉取/推送镜像。生产环境确认不影响业务后再操作。

清理前状态检查：

```bash
tccli cam ListAttachedUserPolicies --TargetUin 100012345678 --region ap-guangzhou --output json
```

解除关联：

```bash
tccli cam DetachUserPolicy \
    --DetachUin 100012345678 \
    --PolicyId 300001 \
    --region ap-guangzhou \
    --output json

tccli cam DetachUserPolicy \
    --DetachUin 100012345678 \
    --PolicyId 400001 \
    --region ap-guangzhou \
    --output json
```

验证已解除：

```bash
tccli cam ListAttachedUserPolicies --TargetUin 100012345678 --region ap-guangzhou --output json
# expected: 目标策略不在列表中
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ListPolicies` 找不到 QcloudCCR 策略 | 扩大搜索：`--Keyword "CCR"` | Keyword 不精确匹配 | 尝试 `--Keyword "CCR"` 或 `--Keyword "镜像"` |
| `AttachUserPolicy` 报 `InvalidParameter.PolicyId` | 检查 PolicyId 格式 | PolicyId 应为数字而非策略名称 | 使用 `cam ListPolicies` 查询后取数字 PolicyId |
| 关联策略报 `InvalidParameter.Uin` | 检查 UIN | 传入了昵称/邮箱而非数字 UIN | 从 `cam ListUsers` 获取数字 UIN |
| 子账号可访问 TKE 但拉取镜像失败 | 检查关联的策略 | 只关联了 TKE 策略，缺少 CCR 策略 | 补充关联 `QcloudCCRFullAccess` 或 `QcloudCCRReadOnlyAccess` |
| 子账号无法在控制台创建集群 | 检查策略名称 | 使用了 `QcloudTKEInnerFullAccess`（覆盖不全） | 换为 `QcloudTKEFullAccess`（含 CVM、VPC 等依赖权限） |
| 子账号可拉取企业版（TCR）镜像但无法拉取个人版（CCR） | 检查是否同时关联了 TCR 和 CCR 策略 | TCR 和 CCR 的权限策略是独立的 | 同时关联 `QcloudCCRFullAccess` 和对应的 TCR 策略 |

## 下一步

- [使用自定义策略授权](../使用自定义策略授权/tccli%20操作.md)
- [配置子账号对单个 TKE 集群的管理权限](../配置子账号对单个%20TKE%20集群的管理权限/tccli%20操作.md)
- [通过标签为子账号配置批量集群的全读写权限](../通过标签为子账号配置批量集群的全读写权限/tccli%20操作.md)

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam) → **策略** → 在 `QcloudTKEFullAccess` 行点击 **关联用户/组/角色** → 勾选子账号 → **确定**。然后在 `QcloudCCRFullAccess` 行重复相同操作。
