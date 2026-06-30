# 为容器实例绑定角色

> 对照官方：[为容器实例绑定角色](https://cloud.tencent.com/document/product/457/57438) · page_id `57438`

## 概述

创建容器实例时通过 `CamRoleName` 参数关联 CAM 角色，使容器内程序可通过 metadata 服务获取该角色的临时凭证（`TmpSecretId`、`TmpSecretKey`、`Token`），安全访问腾讯云 API。此方式无需在容器内硬编码 SecretId/SecretKey。

**工作原理**：容器启动后，实例内部可通过 `http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/<RoleName>` 端点获取临时凭证。SDK 可自动配置此端点。

## 前置条件

- [环境准备](../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:CreateEKSContainerInstances, tke:DescribeEKSContainerInstances
#    tke:DeleteEKSContainerInstances
#    cam:GetRole, cam:DescribeRoleList
# 验证：执行 DescribeEKSContainerInstances 确认 TKE 权限
tccli tke DescribeEKSContainerInstances --region <Region>
# expected: exit 0，返回 TotalCount（可为 0）

# 验证：列出 CAM 角色确认 CAM 权限
tccli cam GetRole --RoleName TKE_QCSRole
# expected: exit 0 或 RoleNotFound（说明权限可用）
```

```json
{
  "TotalCount": "<TotalCount>",
  "EksCis": "<EksCis>",
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Containers": "<Containers>",
  "Image": "<Image>"
}
```

> **说明**：示例中 `<Region>` 替换为 `ap-guangzhou`。

### 资源检查

```bash
# 4. 查询已有 CAM 角色
tccli cam DescribeRoleList --RoleType system
# expected: 返回角色列表，确认目标角色已存在

# 5. 查询可用 VPC 和子网
tccli vpc DescribeVpcs --region <Region>
# expected: 至少返回 1 个 VPC

tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: 至少返回 1 个子网，AvailableIpCount >= 1
```

## 关键字段说明

`CreateEKSContainerInstances` 中与 CAM 角色相关的参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `CamRoleName` | String | 是 | CAM 角色名，需预先在 [CAM 控制台](https://console.cloud.tencent.com/cam/role) 创建，且**载体类型**选「云服务器（CVM）」 | 角色不存在 → `InvalidParameterValue.CamRoleName`；载体选错 → 容器内无法获取凭证 |
| `EksCiName` | String | 是 | 实例名称，长度 1-60 字符 | 格式不符 → `InvalidParameterValue.EksCiName` |
| `VpcId` | String | 是 | VPC ID，格式 `vpc-xxxxxxxx` | VPC 不存在 → `InvalidParameterValue.VpcId` |
| `SubnetId` | String | 是 | 子网 ID，需与 VPC 同可用区 | 子网不可用 → 实例长时间 `Pending` |
| `SecurityGroupIds` | Array | 是 | 安全组 ID 列表 | 需放通 metadata 端点 `169.254.0.23` |
| `Containers` | Array | 是 | 容器定义，含 `Name`、`Image`、`Commands` | 镜像不存在 → `ErrImagePull` |


## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 |
|-----------|----------|
| 参见官方文档 | `tccli tke DescribeClusters` |

## 操作步骤

### 步骤 1：创建带 CAM 角色的容器实例

#### 选择依据

- **CAM 角色名**：`TKE_QCSRole` 是 TKE 预设角色，已包含基础权限。生产环境建议创建自定义角色并按最小权限原则配置策略。
- **载体类型**：必须选「云服务器（CVM）」，EKS CI 通过 CVM 载体获取角色凭证。选错载体会导致容器内 metadata 端点返回空。
- **安全组**：需放通 metadata 端点 `169.254.0.23`，否则容器内无法获取临时凭证。

#### 最小创建（仅含必填字段 + CamRoleName）

`create-role-minimal.json`：

```json
{
    "EksCiName": "eksci-role-example",
    "VpcId": "<VpcId>",
    "SubnetId": "<SubnetId>",
    "SecurityGroupIds": ["<SecurityGroupId>"],
    "Cpu": 0.25,
    "Memory": 0.5,
    "CamRoleName": "TKE_QCSRole",
    "Containers": [
        {
            "Name": "app",
            "Image": "centos:7",
            "Commands": ["/bin/bash", "-c", "sleep 3600"]
        }
    ],
    "RestartPolicy": "Never"
}
```

```bash
tccli tke CreateEKSContainerInstances --region <Region> \
    --cli-input-json file://create-role-minimal.json
# expected: exit 0，返回 EksCiIds
```

**预期输出**：

```json
{
    "EksCiIds": ["eksci-example"],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<VpcId>` | VPC 实例 ID | `tccli vpc DescribeVpcs --region <Region>` |
| `<SubnetId>` | 子网 ID | `tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` |
| `<SecurityGroupId>` | 安全组 ID | `tccli vpc DescribeSecurityGroups --region <Region>` |
| `<Region>` | 地域 | `tccli configure list` |

#### 增强配置（含 EIP，方便从外部验证 metadata）

`create-role-enhanced.json`：

```json
{
    "EksCiName": "eksci-role-enhanced",
    "VpcId": "<VpcId>",
    "SubnetId": "<SubnetId>",
    "SecurityGroupIds": ["<SecurityGroupId>"],
    "Cpu": 0.5,
    "Memory": 1.0,
    "CamRoleName": "TKE_QCSRole",
    "AutoCreateEip": true,
    "AutoCreateEipAttribute": {
        "DeletePolicy": "Delete",
        "InternetServiceProvider": "BGP",
        "InternetMaxBandwidthOut": 10
    },
    "Containers": [
        {
            "Name": "app",
            "Image": "centos:7",
            "Commands": [
                "/bin/bash", "-c",
                "curl -s http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/TKE_QCSRole; sleep 3600"
            ]
        }
    ],
    "RestartPolicy": "Never"
}
```

### 步骤 2：轮询直到实例 Running

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: Status: "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "EksCis": "<EksCis>",
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Containers": "<Containers>",
  "Image": "<Image>"
}
```

### 步骤 3：验证角色绑定成功

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: CamRoleName 字段非空
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "EksCis": [
        {
            "EksCiId": "eksci-example",
            "EksCiName": "eksci-role-example",
            "Status": "Running",
            "CamRoleName": "TKE_QCSRole",
            "PrivateIp": "10.0.x.x"
        }
    ],
    "RequestId": "..."
}
```

### 步骤 4：查看容器日志验证 metadata 可访问

```bash
tccli tke DescribeEksContainerInstanceLog --region <Region> \
    --EksCiId <EksCiId> \
    --ContainerName app
# expected: LogContent 含 TmpSecretId、TmpSecretKey、Token
```

```json
{
  "ContainerName": "<ContainerName>",
  "LogContent": "<LogContent>",
  "RequestId": "<RequestId>"
}
```

## 验证

```bash
# 确认角色已绑定
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: CamRoleName: "TKE_QCSRole"
```

```json
{
  "TotalCount": "<TotalCount>",
  "EksCis": "<EksCis>",
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Containers": "<Containers>",
  "Image": "<Image>"
}
```

| 验证维度 | 命令 | 预期 |
|------|------|------|
| 角色绑定 | `DescribeEKSContainerInstances --EksCiIds '["<EksCiId>"]'` | `CamRoleName: "TKE_QCSRole"` |
| 实例状态 | 同上 | `Status: "Running"` |
| 容器运行 | 同上，检查 `Containers[].CurrentState.State` | `"running"` |
| metadata 可访问 | `DescribeEksContainerInstanceLog --EksCiId <EksCiId> --ContainerName app` | `LogContent` 含凭证信息 |

## 清理

> **警告**：删除实例会释放 CAM 角色的绑定关系。角色本身不会被删除，可继续用于其他实例。

### 1. 清理前状态检查

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# 确认是待删除的目标实例
```

```json
{
  "TotalCount": 0,
  "EksCis": [],
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Containers": [],
  "Image": "<Image>",
  "Name": "<Name>",
  "Args": []
}
```

### 2. 删除实例

```bash
tccli tke DeleteEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: exit 0，返回 RequestId
```

### 3. 验证已删除

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: TotalCount: 0
```

```json
{
  "TotalCount": "<TotalCount>",
  "EksCis": "<EksCis>",
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Containers": "<Containers>",
  "Image": "<Image>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEKSContainerInstances` 返回 `InvalidParameterValue.CamRoleName` | `tccli cam GetRole --RoleName <角色名>` 检查角色是否存在 | 角色不存在或名称拼写有误（大小写不一致） | 在 [CAM 控制台](https://console.cloud.tencent.com/cam/role) 确认角色名，注意大小写精确匹配 |
| `CreateEKSContainerInstances` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:CreateEKSContainerInstances` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或 `tke:CreateEKSContainerInstances` |
| `cam GetRole` 返回 `ResourceNotFound.RoleNotFound` | `tccli cam DescribeRoleList --RoleType system` 确认角色列表 | 角色未创建或名称错误 | 在 CAM 控制台创建角色，载体选「云服务器（CVM）」 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| metadata 端点返回空 | `tccli tke DescribeEKSContainerInstanceLog --region <Region> --EksCiId <EksCiId> --ContainerName app` 查看容器日志 | 安全组未放通 metadata 端点 `169.254.0.23`，或角色载体未选 CVM | 检查安全组规则：`tccli vpc DescribeSecurityGroupPolicies --region <Region> --SecurityGroupId <SecurityGroupId>`；确认角色载体为 CVM |
| 获取到凭证但调用 API 返回 403 | 查看日志中的凭证信息（TmpSecretId、Token） | 角色策略不包含目标 API 的 Action 权限 | 在 [CAM 控制台](https://console.cloud.tencent.com/cam/role) 中为该角色添加所需 API 策略 |
| 实例进入 `Failed` 状态 | `tccli tke DescribeEKSContainerInstanceEvent --region <Region> --EksCiId <EksCiId>` 查看事件 | 容器启动命令执行失败或镜像拉取失败 | 检查 Commands 参数是否有效，镜像是否可拉取 |
| 日志中看不到凭证信息 | `tccli tke DescribeEksContainerInstanceLog --region <Region> --EksCiId <EksCiId> --ContainerName app --Tail 50` | 容器在 curl metadata 前已退出，或 curl 命令被跳过 | 增大 `--Tail` 参数，或修改 Commands 确保 curl 在容器持续运行前执行 |

## 下一步

- [容器实例操作手册](../容器实例操作手册/tccli%20操作.md) — 完整 CRUD 操作
- [通过绑定弹性公网 IP 访问外网](../通过绑定弹性公网%20IP%20访问外网/tccli%20操作.md) — 为实例配置公网访问
- [CAM 角色文档](https://cloud.tencent.com/document/product/598/19420) — CAM 角色概念与使用指南

## 控制台替代

[TKE 控制台 → 弹性容器 → 容器实例 → 新建 → 高级设置 → 绑定 CAM 角色](https://console.cloud.tencent.com/tke2/eks/ci)：可视化选择 CAM 角色绑定。
