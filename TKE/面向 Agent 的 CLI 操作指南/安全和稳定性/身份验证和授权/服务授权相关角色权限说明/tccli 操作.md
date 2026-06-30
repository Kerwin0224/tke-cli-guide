# 服务授权相关角色权限说明（tccli）

> 对照官方：[服务授权相关角色权限说明](https://cloud.tencent.com/document/product/457/43416) · page_id `43416`

## 概述

TKE 通过 CAM 服务角色（Service Role）获取操作其他云服务（CVM、VPC、CLB、CLS、CFS、CBS 等）的权限。主要涉及两个角色：

| 角色 | 说明 |
|------|------|
| `TKE_QCSRole` | 容器服务主角色，首次使用 TKE 时自动创建 |
| `IPAMDofTKE_QCSRole` | VPC-CNI 网络插件角色，首次创建 VPC-CNI 集群时触发授权 |

这些角色由 TKE 服务自身使用，不应删除或修改其授权策略。角色关联的预设策略在特定场景触发授权时才会出现在已授权策略列表中。

> **注意**：服务角色供 TKE 服务自身调用云 API 使用，不能直接分配给子账号。服务角色不包含容器镜像仓库（TCR/CCR）的授权策略。镜像仓库权限需单独配置，参见 [TCR 镜像仓库资源级权限设置](../TCR%20镜像仓库资源级权限设置/tccli%20操作.md)。

本页为纯 CAM 查询操作，不涉及集群数据面。演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou，kubectl 不可达）仅作为 TKE 服务已开通的验证上下文。

## 前置条件

- 已完成 [环境准备](../../../环境准备.md)，tccli 已安装并配置凭据。
- 主账号 UIN `100012345678`，演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou），kubectl 不可达——本页仅用集群确认 TKE 服务已开通，无需 kubectl。
- CAM 调用账号具备以下 Action 权限：`cam:GetRole`、`cam:ListAttachedRolePolicies`、`cam:GetPolicy`、`tke:DescribeClusters`。

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
# 3. 确认已开通 TKE 服务
tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表（可为空，表示服务已开通）
```

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.30.0"
        }
    ],
    "TotalCount": 1,
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看 TKE_QCSRole 详情 | `cam GetRole --RoleName TKE_QCSRole --region ap-guangzhou` | 是 |
| 查看角色已关联策略 | `cam ListAttachedRolePolicies --RoleName TKE_QCSRole --region ap-guangzhou` | 是 |
| 查看 IPAMDofTKE_QCSRole | `cam GetRole --RoleName IPAMDofTKE_QCSRole --region ap-guangzhou` | 是 |
| 按策略 ID 查看策略详情 | `cam GetPolicy --PolicyId POLICY_ID --region ap-guangzhou` | 是 |

## 操作步骤

### TKE_QCSRole（容器服务主角色）

TKE_QCSRole 在首次使用 TKE 时自动创建，关联多组预设策略，不同场景需单独触发授权。

#### 默认关联策略

| 策略 | 触发场景 | 说明 |
|------|----------|------|
| `QcloudAccessForTKERole` | 首次登录 TKE 控制台，在"服务授权"弹窗中点击"同意授权" | 授权 TKE 操作 CVM、CLB、CBS、VPC 等云资源 |
| `QcloudAccessForTKERoleInOpsManagement` | 随 QcloudAccessForTKERole 同时授权，无需额外操作 | 授权 TKE 运维管理（含 CLS 日志服务等） |

```bash
# 查看 TKE_QCSRole 基本信息
tccli cam GetRole --RoleName "TKE_QCSRole" --region ap-guangzhou --output json
# expected: exit 0，返回角色基本信息
```

```json
{
    "RoleId": "4611686018427387904",
    "RoleName": "TKE_QCSRole",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"action\":\"name/sts:AssumeRole\",\"effect\":\"allow\",\"principal\":{\"service\":\"tke.cloud.tencent.com\"}}]}",
    "Description": "容器服务（TKE）服务角色，用于 TKE 访问其他云服务资源",
    "ConsoleLogin": 0,
    "RoleType": "service",
    "SessionDuration": 7200,
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

#### 可选关联策略

| 策略 | 触发场景 | 说明 |
|------|----------|------|
| `QcloudAccessForTKERoleInCreatingCFSStorageclass` | 首次在集群中安装 CFS 扩展组件 | 授权 TKE 创建/删除/查询 CFS 文件系统和挂载点 |
| `QcloudCVMFinanceAccess` | 使用包年包月云硬盘创建 PVC 时，需手动在 CAM 控制台将策略关联到 TKE_QCSRole | CVM 财务/支付权限 |

```bash
# 查看角色已关联的策略列表
tccli cam ListAttachedRolePolicies --RoleName "TKE_QCSRole" --region ap-guangzhou --output json
# expected: exit 0，返回已关联策略列表
```

```json
{
    "List": [
        {
            "PolicyId": 9087631,
            "PolicyName": "QcloudAccessForTKERole",
            "CreateMode": 2
        },
        {
            "PolicyId": 34488263,
            "PolicyName": "QcloudAccessForTKERoleInOpsManagement",
            "CreateMode": 2
        }
    ],
    "TotalNum": 2,
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### IPAMDofTKE_QCSRole（VPC-CNI 网络插件角色）

此角色在首次创建 VPC-CNI 模式的集群时触发授权，关联以下策略：

| 策略 | 触发场景 | 说明 |
|------|----------|------|
| `QcloudAccessForIPAMDofTKERole` | 集群创建页面选择 VPC-CNI 网络模式时，在"服务授权"弹窗中点击"同意授权" | 授权 TKE IPAMD 管理弹性网卡等云资源 |

```bash
# 查看 IPAMDofTKE_QCSRole 基本信息
tccli cam GetRole --RoleName "IPAMDofTKE_QCSRole" --region ap-guangzhou --output json
# expected: exit 0（已创建）或 RoleNotExist（未创建过 VPC-CNI 集群）
```

```json
{
    "RoleId": "4611686018427387905",
    "RoleName": "IPAMDofTKE_QCSRole",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"action\":\"name/sts:AssumeRole\",\"effect\":\"allow\",\"principal\":{\"service\":\"tke.cloud.tencent.com\"}}]}",
    "Description": "TKE IPAMD 支持服务角色",
    "RoleType": "service",
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

### 授权触发场景速查

| 角色 | 策略 | 触发时机 |
|------|------|----------|
| TKE_QCSRole | QcloudAccessForTKERole | 首次登录 TKE 控制台 |
| TKE_QCSRole | QcloudAccessForTKERoleInOpsManagement | 自动随上者授权 |
| TKE_QCSRole | QcloudAccessForTKERoleInCreatingCFSStorageclass | 首次安装 CFS 组件 |
| TKE_QCSRole | QcloudCVMFinanceAccess | 手动关联——包年包月 CBS 场景 |
| IPAMDofTKE_QCSRole | QcloudAccessForIPAMDofTKERole | 首次创建 VPC-CNI 集群 |

## 验证

```bash
# 验证 TKE_QCSRole 已创建
tccli cam GetRole --RoleName "TKE_QCSRole" --region ap-guangzhou --output json | jq '.RoleName'
# expected: "TKE_QCSRole"
```

```bash
# 验证 IPAMDofTKE_QCSRole（仅 VPC-CNI 场景）
tccli cam GetRole --RoleName "IPAMDofTKE_QCSRole" --region ap-guangzhou --output json | jq '.RoleName'
# expected: "IPAMDofTKE_QCSRole"（若未创建过 VPC-CNI 集群，可能返回 RoleNotExist）
```

```bash
# 查看 TKE_QCSRole 已关联的全部策略
tccli cam ListAttachedRolePolicies --RoleName "TKE_QCSRole" --region ap-guangzhou --output json | jq '.List[].PolicyName'
# expected: 列出已关联策略名
```

## 清理

无需清理。本页为概念概述与只读查询，不涉及资源创建。

> **警告**：服务角色由 TKE 自动管理，不应手动删除或修改。删除 TKE_QCSRole 或解除其关联策略将导致集群创建、节点管理、日志采集等功能异常。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `GetRole` 返回 `RoleNotExist` | 确认角色名称拼写：`tccli cam GetRole --RoleName "TKE_QCSRole" --region ap-guangzhou` | 角色尚未创建。首次使用 TKE 时需完成服务授权 | 登录 TKE 控制台，完成"服务授权"弹窗中的授权操作 |
| `ListAttachedRolePolicies` 返回空列表 | 检查角色是否已触发过授权流程 | 角色已创建但未触发策略关联场景（如未安装 CFS 组件，则 CFS 策略不会出现）| 正常现象——策略按需关联，未被触发的策略不会显示 |
| `GetRole` 返回 `AccessDenied` | 检查调用者 CAM 权限：`tccli cam GetRole --RoleName "TKE_QCSRole" --region ap-guangzhou` | 当前账号缺少 `cam:GetRole` 权限 | 联系主账号管理员授予 `cam:GetRole` 和 `cam:ListAttachedRolePolicies` 权限 |
| `GetPolicy` 返回 `ResourceNotFound` | 检查 PolicyId 是否为数字且在正确范围 | 策略 ID 填写错误，或策略为服务预设策略 | 通过 `tccli cam ListPolicies --Scope QCS --region ap-guangzhou` 确认策略 ID |

### 创建/操作异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 创建集群失败，提示服务授权未完成 | 检查角色状态：`tccli cam GetRole --RoleName "TKE_QCSRole" --region ap-guangzhou` | 服务授权弹窗未完成，或授权已过期 | 登录 TKE 控制台，在集群列表页右上角查找授权入口并完成授权 |
| CFS 组件安装失败 | 检查 CFS 策略是否已关联：`tccli cam ListAttachedRolePolicies --RoleName "TKE_QCSRole" --region ap-guangzhou \| jq '.List[].PolicyName'` | TKE_QCSRole 未关联 CFS 策略 | 在组件管理页点击"服务授权"触发 CFS 策略关联 |
| 包年包月 PVC 创建失败 | 检查财务策略是否已关联 | 未手动关联 `QcloudCVMFinanceAccess` 策略 | 在 CAM 控制台手动将 `QcloudCVMFinanceAccess` 关联到 TKE_QCSRole |
| VPC-CNI 集群创建提示 IPAMD 未授权 | 检查 IPAMD 角色：`tccli cam GetRole --RoleName "IPAMDofTKE_QCSRole" --region ap-guangzhou` | IPAMDofTKE_QCSRole 未创建或未授权 | 在集群创建页选择 VPC-CNI 网络模式后，点击"服务授权"完成 IPAMD 授权 |

## 下一步

- [TCR 镜像仓库资源级权限设置](../TCR%20镜像仓库资源级权限设置/tccli%20操作.md) — 镜像仓库资源级授权
- [TKE Kubernetes 对象级权限控制 概述](../TKE%20Kubernetes%20对象级权限控制/概述/tccli%20操作.md) — 集群内 RBAC 概览
- [使用 TKE 预设策略授权](../使用%20TKE%20预设策略授权/tccli%20操作.md)

## 控制台替代

登录 [访问管理控制台](https://console.cloud.tencent.com/cam/role) → **角色** → 搜索 `TKE_QCSRole` → 点击角色名查看详情和已关联策略列表。也可搜索 `IPAMDofTKE_QCSRole` 查看 VPC-CNI 网络角色。
