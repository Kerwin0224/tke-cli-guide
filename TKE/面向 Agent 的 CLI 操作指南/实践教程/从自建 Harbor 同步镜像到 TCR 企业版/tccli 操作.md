# 从自建 Harbor 同步镜像到 TCR 企业版（tccli）

> 对照官方：[从自建 Harbor 同步镜像到 TCR 企业版](https://cloud.tencent.com/document/product/1141/44970) · page_id `44970`

## 概述

将自建 Harbor（v1.8.0+）中的容器镜像和 Helm Chart 同步至腾讯云容器镜像服务（TCR）企业版。Harbor 侧配置复制目标为 TCR 实例，触发同步后镜像自动推送到 TCR，实现云上托管。

**同步方向：** Harbor → TCR（Push-based）

**图：Harbor 同步到 TCR 架构**

```
 ┌──────────────────────┐         ┌────────────────────────┐
 │   自建 Harbor         │  Push   │   TCR 企业版             │
 │   (IDC / 云上 CVM)    │ ──────► │   域名: xxx.tencentcloudcr.com │
 │                       │  Docker │   VPC 内网 / 公网        │
 │   Harbor v1.8.0+      │  pull→  │                        │
 │                       │  tag→   │   镜像仓库 + Helm Chart  │
 │   ┌─────────────────┐ │  push   │                        │
 │   │ 复制规则        │ │         │                        │
 │   │ (Tencent TCR    │ │         │                        │
 │   │  提供者)        │ │         │                        │
 │   └─────────────────┘ │         │                        │
 └──────────────────────┘         └────────────────────────┘
```

**使用限制：**
- 仅支持 Harbor v1.8.0 及以上版本。
- Harbor 版本 < 2.1.2 时，提供者选择 "Docker Registry"（使用 TCR 长期访问凭证），不支持在 TCR 侧自动新建命名空间。
- Harbor 版本 >= 2.1.2 时，提供者选择 "Tencent TCR"（使用 CAM SecretId/SecretKey），支持 Pull-based 模式。
- 若自建 Harbor 无法通过专线或私有网络访问 TCR，可使用公网同步（会产生公网流量费用）。

## 前置条件

- 已[购买企业版实例](../../操作指南/创建企业版实例/tccli%20操作.md)。
- 已安装 `tccli` 并完成 [环境准备](../../环境准备.md)。
- 自建 Harbor 可通过专线、公网或私有网络访问 TCR 实例域名。
- 若使用子账号操作，建议将 TCR 全读写权限授予子账号（参考[企业版授权方案示例](https://cloud.tencent.com/document/product/1141/41417)）。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看实例信息 | `tccli tcr DescribeInstances` | 是 |
| 配置内网访问 | `tccli tcr ManageInternalEndpoint --Operation Create` | 否 |
| 配置公网访问 | `tccli tcr ManageExternalEndpoint --Operation Create` | 否 |
| 查看实例访问凭证 | `tccli tcr DescribeInstanceToken` | 是 |
| 创建实例访问凭证 | `tccli tcr CreateInstanceToken` | 否 |
| 查看复制策略 | `tccli tcr DescribeReplicationPolicies` | 是 |
| 查看复制实例 | `tccli tcr DescribeReplicationInstances` | 是 |
| 配置 Docker 客户端 | `docker login` / `docker push` / `docker pull` | 是 |
| 配置 Harbor 复制规则 | Harbor Web UI（控制台操作，无 TCR API） | — |

## 操作步骤

### 步骤1：确认 TCR 实例可访问

查询实例信息，获取访问域名与状态：

```bash
tccli tcr DescribeInstances --Registryids '["<RegistryId>"]' --region <Region> --output json
```

```json
{
    "TotalCount": 1,
    "Registries": [
        {
            "RegistryId": "tcr-example",
            "RegistryName": "harbor-sync",
            "RegistryType": "basic",
            "Status": "Running",
            "PublicDomain": "harbor-sync.tencentcloudcr.com",
            "InternalEndpoint": "https://harbor-sync.tencentcloudcr.com",
            "TagSpecification": {
                "ResourceType": "instance"
            },
            "CreatedAt": "2026-01-01T00:00:00+08:00"
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

记录 `PublicDomain`（域名须以 `.tencentcloudcr.com` 结尾），后续 Harbor 侧配置使用完整 URL：`https://<PublicDomain>`。

### 步骤2：配置 Harbor 到 TCR 的网络访问

根据自建 Harbor 的网络情况，选择内网或公网方案。

#### 方案 A：通过腾讯云私有网络访问（推荐）

若自建 Harbor 部署在腾讯云 VPC 内或已通过专线/VPN 打通至腾讯云 VPC，使用内网访问可提升同步速度并节省公网流量费用。

配置内网访问链路，使 TCR 实例在 VPC 内可被解析：

```bash
tccli tcr ManageInternalEndpoint --cli-input-json '{
    "RegistryId": "<RegistryId>",
    "Operation": "Create",
    "VpcId": "<VpcId>",
    "SubnetId": "<SubnetId>",
    "RegionId": <RegionId>,
    "RegionName": "<RegionName>"
}' --region <Region> --output json
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `RegistryId` | String | **是** | TCR 实例 ID |
| `Operation` | String | **是** | 操作类型：`"Create"` 新建 / `"Delete"` 删除 |
| `VpcId` | String | **是** | 自建 Harbor 所在的 VPC ID |
| `SubnetId` | String | **是** | VPC 下用于分配内网 IP 的子网 ID |
| `RegionId` | Integer | 否 | 地域 ID |
| `RegionName` | String | 否 | 地域名称 |

Output：

```json
{
    "RegistryId": "tcr-example",
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

开启自动解析后，VPC 内即可通过 `harbor-sync.tencentcloudcr.com` 内网访问 TCR 实例。如在 CVM 上手动配置 Host：

```bash
echo <内网访问IP> harbor-sync.tencentcloudcr.com >> /etc/hosts
```

#### 方案 B：通过公网访问

若自建 Harbor 未部署在腾讯云 VPC 内且无法通过专线打通，使用公网访问。

1. 开启公网访问入口：

```bash
tccli tcr ManageExternalEndpoint --cli-input-json '{
    "RegistryId": "<RegistryId>",
    "Operation": "Create"
}' --region <Region> --output json
```

2. 添加公网白名单（允许 Harbor 出口 IP 访问）：

```bash
tccli tcr ManageExternalEndpoint --cli-input-json '{
    "RegistryId": "<RegistryId>",
    "Operation": "Create",
    "Policy": [
        {
            "CidrBlock": "<Harbor出口公网IP/32>",
            "Description": "自建 Harbor 公网访问"
        }
    ]
}' --region <Region> --output json
```

> 如果无法确认 Harbor 的公网出口 IP，可临时配置 `CidrBlock: "0.0.0.0/0"` 以放通全部来源，完成同步后尽快删除该白名单。

查看公网访问状态：

```bash
tccli tcr DescribeExternalEndpointStatus --RegistryId <RegistryId> --region <Region> --output json
```

```json
{
  "Status": "<Status>",
  "Reason": "<Reason>",
  "RequestId": "<RequestId>"
}
```

### 步骤3：获取 TCR 访问凭证

Harbor 配置复制目标时需要 TCR 的访问凭证。根据 Harbor 版本选择不同凭证类型：

#### Harbor >= 2.1.2 — 使用 CAM API 密钥（推荐）

前往 [API 密钥管理](https://console.cloud.tencent.com/cam/capi) 获取 **SecretId** 和 **SecretKey**：
- `访问 ID`（Harbor 侧字段）→ SecretId
- `访问密码`（Harbor 侧字段）→ SecretKey

> 此方式支持 Pull-based 同步（TCR → Harbor），且可在 TCR 侧自动新建命名空间。

#### Harbor < 2.1.2 — 使用 TCR 长期访问凭证

创建专用的长期访问凭证：

```bash
tccli tcr CreateInstanceToken --cli-input-json '{
    "RegistryId": "<RegistryId>",
    "TokenType": "tempermanent",
    "Desc": "自建 Harbor 数据同步专用"
}' --region <Region> --output json
```

```json
{
    "Username": "100012345678",
    "Token": "eyJhbGciOi...",
    "ExpTime": 1735689600,
    "TokenId": "token-xxxxxxxx",
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

> 记录 `Token` 字段，**仅此一次保存机会**。在 Harbor 中填写 `访问 ID` = Username，`访问密码` = Token。

查看已有凭证：

```bash

```json
{
  "TotalCount": 0,
  "Tokens": [],
  "Id": "<Id>",
  "Desc": "<Desc>",
  "RegistryId": "<RegistryId>",
  "Enabled": true,
  "CreatedAt": "<CreatedAt>",
  "ExpiredAt": 0
}
```
tccli tcr DescribeInstanceToken --RegistryId <RegistryId> --region <Region> --output json
```

```json
{
  "RequestId": "..."
}
```

### 步骤4：在 Harbor 侧配置复制目标

使用管理员账号登录 Harbor Web UI，进入 **系统管理 → 仓库管理 → 新建目标**。

**Harbor >= 2.1.2 配置（Tencent TCR 提供者）：**

| 字段 | 值 |
|------|-----|
| 提供者 | `Tencent TCR` |
| 目标名 | 自定义，如 `tencent-tcr` |
| 目标 URL | `https://harbor-sync.tencentcloudcr.com` |
| 访问 ID | CAM SecretId |
| 访问密码 | CAM SecretKey |
| 验证远程证书 | 保持默认 |

**Harbor < 2.1.2 配置（Docker Registry 提供者）：**

| 字段 | 值 |
|------|-----|
| 提供者 | `Docker Registry` |
| 目标名 | 自定义 |
| 目标 URL | `https://harbor-sync.tencentcloudcr.com` |
| 访问 ID | `CreateInstanceToken` 返回的 `Username` |
| 访问密码 | `CreateInstanceToken` 返回的 `Token` |

单击 **测试连接** 验证连通性。若失败，检查步骤2 网络配置。

### 步骤5：在 Harbor 侧创建复制规则

进入 **系统管理 → 复制管理 → 新建规则**：

| 字段 | 值 |
|------|-----|
| 名称 | 自定义，如 `sync-images-to-tcr` |
| 复制模式 | `Push-based`（Harbor 推送至 TCR） |
| 源资源过滤器 | 留空则同步全部；支持按仓库名/Tag 过滤 |
| 目的 Registry | 选择步骤4 创建的目标仓库 |
| 目的 Namespace | 留空则默认同名命名空间 |
| 触发模式 | `手动触发` 或 `事件驱动`（推送时自动同步） |
| 覆盖 | 建议勾选（覆盖同名资源） |

### 步骤6：推送镜像触发同步

从 Harbor 拉取镜像，打上 TCR 域名标签后推送：

```bash
# 登录 Harbor
docker login <Harbor域名> -u <用户名> -p <密码>

# 拉取镜像
docker pull <Harbor域名>/<项目>/<仓库>:<tag>

# 登录 TCR
docker login <TCR域名> --username=100012345678 --password=<Token>

# 打标签
docker tag <Harbor域名>/<项目>/<仓库>:<tag> <TCR域名>/<命名空间>/<仓库>:<tag>

# 推送至 TCR（若规则为事件驱动，此推送会触发 Harbor 侧自动同步）
docker push <TCR域名>/<命名空间>/<仓库>:<tag>
```

> **注意：** 这里是手动推送至 TCR。Harbor 的事件驱动会自动将 Harbor 内的新镜像同步至 TCR；需要在 Harbor 侧将镜像推送到 Harbor 仓库内方可触发。

推送到 Harbor 内的流程：

```bash
docker pull nginx:latest
docker tag nginx:latest <Harbor域名>/tcr-sync/nginx:latest
docker push <Harbor域名>/tcr-s

```json
{
  "ReplicationPolicyInfoList": [],
  "ID": 0,
  "Name": "<Name>",
  "Description": "<Description>",
  "Filters": [],
  "Type": "<Type>",
  "Value": "<Value>",
  "Override": true
}
```ync/nginx:latest
```

若 Harbor 复制规则配置为事件驱动，推送后自动触发同步至 TCR；若为手动触发，进入 Harbor → **复制管理** → 选择规则 → **复制**。

### 步骤7：从 TCR 侧查看同步结果

Harbor 侧配置并触发同步后，TCR 侧可查看同步策略与实例。

查看复制策略列表：

```bash
tccli tcr DescribeReplicationPolicies --RegistryId <RegistryId> --region <Region> --output json
```

```json
{
    "ReplicationPolicyInfoList": [
        {
            "ID": 1,
            "Name": "sync-images-to-tcr",
            "Description": "从 Harbor 同步镜像至 TCR",
            "Filters": [
                {"Type": "name", "Value": "**"},
                {"Type": "tag", "Value": ""},
                {"Type": "resource", "Value": "image"}
            ],
            "Override": true,
            "Enabled": true,
            "SrcResource": "<HarborDomain>/tcr-sync/**",
            "DestResource": "<TcrDomain>/tcr-sync/**",
            "CreationTime": "2026-06-04T10:00:00+08:00",
            "UpdateTime": "2026-06-04T10:00:00+08:00"
        }
    ],
    "TotalCount": 1,
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

查看复制实例：

```bash
tccli tcr DescribeReplicationInstances --RegistryId <RegistryId> --region <Region> --output json
```

```json
{
    "TotalCount": 1,
    "ReplicationRegistries": [
        {
            "RegistryId": "tcr-example",
            "ReplicationRegistryId": "harbor-sync-xxxxxxxx",
            "ReplicationRegionId": 1,
            "ReplicationRegionName": "ap-guangzhou",
            "Status": "Running",
            "CreatedAt": "2026-06-04T10:00:00+08:00"
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

查看同步任务进度：

```bash
tccli tcr DescribeReplicationInstanceCreateTasks \
    --ReplicationRegistryId <ReplicationRegistryId> \
    --ReplicationRegionId <RegionId> \
    --region <Region> --output json
```

```json
{
    "Status": "SUCCESS",
    "TaskDetail": [
        {
            "TaskName": "SyncTask",
            "TaskUUID": "tcr-task-xxxxxxxx",
            "TaskStatus": "SUCCESS",
            "TaskMessage": "",
            "CreatedTime": "2026-06-04T10:00:00+08:00",
            "FinishedTime": "2026-06-04T10:01:00+08:00"
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

### 步骤8：从 TCR 管理同步规则（可选）

如果后续需要从 TCR 侧管理同步规则（而非 Harbor 侧），可使用 `ManageReplication`。

> `ManageReplication` 用于 TCR 实例间的跨实例同步规则（参见[跨实例（账号）同步镜像](../../操作指南/镜像分发/跨实例（账号）同步镜像/tccli%20操作.md)）。**Harbor → TCR 的同步规则由 Harbor 自身维护**，不建议通过 TCR API 修改 Harbor 创建的规则。

如需将已有的 Harbor → TCR 同步迁移为纯 TCR 实例间的同步，参考[跨实例（账号）同步镜像](../../操作指南/镜像分发/跨实例（账号）同步镜像/tccli%20操作.md)中的 `ManageReplication` 用法。

## 验证

### Control plane (tccli)

```bash
# 确认实例状态
tccli tcr DescribeInstances --Registryids '["<RegistryId>"]' --region <Region> --output json
```

```bash
# 确认 Harbor 侧的同步策略已被 TCR 识别
tccli tcr DescribeReplicationPolicies --RegistryId <RegistryId> --region <Region> --output json
```

### Data plane (Docker)

```bash
# 确认镜像已同步至 TCR
docker pull <TCR域名>/<命名空间>/<仓库>:<tag>
```

## 清理

同步完成后删除不再需要的凭证（避免权限泄露）：

```bash
tccli tcr DeleteInstanceToken --RegistryId <RegistryId> --TokenId <TokenId> --region <Region> --output json
```

删除公网白名单（如配置了 `0.0.0.0/0`）：

```bash
tccli tcr ManageExternalEndpoint --cli-input-json '{
    "RegistryId": "<RegistryId>",
    "Operation": "Delete"
}' --region <Region> --output json
```

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| Harbor 测试连接失败 | 网络不通或凭证错误 | 确认 VPC/公网访问已配置；检查域名是否正确 |
| Harbor 提供者中无 "Tencent TCR" | Harbor 版本 < 2.1.2 | 选择 "Docker Registry" 提供者，使用 TCR 长期访问凭证 |
| `CreateInstanceToken` 返回凭证后丢失 | `Token` 仅返回一次 | 重新创建新凭证 |
| `DescribeReplicationPolicies` 返回空 | Harbor 侧尚未创建复制规则 | 确认 Harbor 复制规则已创建并已触发同步 |
| 同命名空间下新镜像未同步 | 规则触发模式为手动 | 进入 Harbor → 复制管理 → 手动触发复制 |
| 公网同步慢或耗时长 | 公网带宽限制或镜像体积大 | 建议通过专线/内网同步 |
| `ManageInternalEndpoint` 报 `FailedOperation` | VPC 或子网不存在/不可用 | 确认 VpcId/SubnetId 正确，子网在 TCR 实例同地域 |
| 同步完成后 TCR 命名空间不存在 | Dock Registry 模式不支持自动创建命名空间 | 先在 TCR 侧手动创建命名空间：`tccli tcr CreateNamespace` |

## 下一步

- [跨实例（账号）同步镜像](../../操作指南/镜像分发/跨实例（账号）同步镜像/tccli%20操作.md)（page_id `41945`） — TCR 实例间的镜像同步规则
- [同实例多地域复制镜像](../../操作指南/镜像分发/同实例多地域复制镜像/tccli%20操作.md)（page_id `52095`） — 高级版实例多地域从实例复制
- [创建企业版实例](../../操作指南/创建企业版实例/tccli%20操作.md) — 购买 TCR 企业版实例
- [配置公网访问控制](../../操作指南/访问配置/访问网络控制/配置公网访问控制/tccli%20操作.md) — 管理公网白名单
- [配置内网访问控制](../../操作指南/访问配置/访问网络控制/配置内网访问控制/tccli%20操作.md) — VPC 内网链路管理
- [管理镜像仓库](../../操作指南/镜像创建/管理镜像仓库/tccli%20操作.md) — 创建命名空间与镜像仓库

## 控制台替代

[容器镜像服务 → 同步复制 → 实例复制](https://console.cloud.tencent.com/tcr/replication)：登录自建 Harbor → 系统管理 → 仓库管理 → 新建目标（Tencent TCR 或 Docker Registry）→ 复制管理 → 新建规则 → 触发同步。
