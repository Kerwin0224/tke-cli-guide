# 混合云下的多平台镜像数据同步复制（tccli）

> 对照官方：[混合云下的多平台镜像数据同步复制](https://cloud.tencent.com/document/product/1141/60740) · page_id `60740`

## 概述

混合云场景下，容器镜像需要在跨主账号、跨地域、跨国、跨平台的多个容器镜像仓库之间同步复制。TCR 提供了**实例同步**（按需同步）、**实例复制**（全量实时复制）、**镜像迁移工具 image-transfer** 及**自定义域名**等多项能力，覆盖以下三类典型场景：

| 场景 | 说明 | 核心能力 |
|------|------|---------|
| 跨地域 TCR 实例复制 | 国内/跨国多地域间同步镜像，实现就近拉取 | 实例复制 + 实例同步 |
| 跨平台镜像迁移或同步 | 公有云/自建仓库/多家云平台间的镜像流转 | image-transfer + Harbor 同步 |
| DevOps 镜像流转 | 开发 → 测试 → 生产环境间的镜像自动传递 | 实例同步 + 交付流水线 |

> **使用限制：**
> - 实例同步功能需要实例规格为**标准版或高级版**。
> - 实例复制功能需要实例规格为**高级版**。
> - 暂不支持跨国实例复制（需使用实例同步替代）。

## 前置条件

- [环境准备](../../环境准备.md)
- 已成功[购买企业版实例](https://cloud.tencent.com/document/product/1141/51110)（参见[企业版快速入门](../../快速入门/企业版快速入门/tccli%20操作.md)），根据场景选择标准版或高级版。
- 如果使用子账号操作，需提前授予对应实例的操作权限（参见[企业版授权方案示例](https://cloud.tencent.com/document/product/1141/41417)）。
- **跨主账号同步**：目标侧需提供实例 ID、主账号 UIN 及[用户级账号](https://cloud.tencent.com/document/product/1141/41829)长期访问凭证。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查看 TCR 实例列表 | `DescribeInstances` | 是 |
| 查看实例复制列表 | `DescribeReplicationInstances` | 是 |
| 创建从实例（实例复制） | `CreateReplicationInstance` | 否（重复创建同一地域报错） |
| 查看从实例创建进度 | `DescribeReplicationInstanceCreateTasks` | 是 |
| 删除从实例 | `DeleteReplicationInstance` | 是 |
| 查看同步策略列表 | `DescribeReplicationPolicies` | 是 |
| 创建/管理同步规则（实例同步） | `ManageReplication` | 否（覆盖同名规则） |
| 查看同步状态/日志 | `DescribeReplicationInstanceSyncStatus` | 是 |
| 删除同步实例 | `DeleteReplicationInstance` | 是 |
| 删除同步规则 | `DeleteReplicationRule` | 是 |
| 查看自定义域名 | `DescribeInstanceCustomizedDomain` | 是 |

## 操作步骤

### 场景1：跨地域 TCR 实例复制

#### 1.1 国内跨地域实例复制

当业务部署在国内多个地域时，使用**实例复制**功能实现单地域上传、多地域高速实时同步、就近内网拉取。

**步骤1：确认主实例为高级版**

```bash
tccli tcr DescribeInstances --Registryids '["<RegistryId>"]' --region ap-guangzhou --output json
```

```json
{
  "TotalCount": 0,
  "Registries": [],
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>",
  "PublicDomain": "<PublicDomain>",
  "CreatedAt": "<CreatedAt>"
}
```

确认 `RegistryType: "premium"` 且 `Status: "Running"`。

**步骤2：创建从实例（复制实例）**

选择需要复制到的目标地域，创建从实例。参见[同实例多地域复制镜像](../../操作指南/镜像分发/同实例多地域复制镜像/tccli%20操作.md)：

```bash
tccli tcr CreateReplicationInstance --cli-input-json '{
    "RegistryId": "<RegistryId>",
    "ReplicationRegionId": 4,
    "ReplicationRegionName": "ap-shanghai",
    "SyncTag": true
}' --region ap-guangzhou --output json
```

记录返回的 `ReplicationRegistryId`（格式为 `<Re

```json
{
  "TaskDetail": [],
  "TaskName": "<TaskName>",
  "TaskUUID": "<TaskUUID>",
  "TaskStatus": "<TaskStatus>",
  "TaskMessage": "<TaskMessage>",
  "CreatedTime": "<CreatedTime>",
  "FinishedTime": "<FinishedTime>",
  "Status": "<Status>"
}
```gistryId>-<RegionId>`）。

**步骤3：等待复制实例就绪**

```bash
tccli tcr DescribeReplicationInstanceCreateTasks \
    --ReplicationRegistryId <ReplicationRegistryId> \
    --ReplicationRegionId <ReplicationRegionId> \
    --region ap-guangzhou --output json
```

```json
{
  "RequestId": "..."
}
```

当 `TaskDetail` 中所有子任务 `TaskStatus` 均为 `"SUCCESS"` 时，复制实例就绪。后续可用 `DescribeReplicationInstances` 查看复制实例列表，确认 `Status: "Running"`。

**步骤4：将复制地域内的 VPC 接入复制实例**

复制实例创建后，需将目标地域内需拉取镜像的 VPC 接入该实例。参见[配置内网访问控制](../../操作指南/访问配置/访问网络控制/配置内网访问控制/tccli%20操作.md)：

```bash
tccli tcr ManageInternalEndpoint \
    --RegistryId <RegistryId> \
    --Operation Create \
    --VpcId <VpcId> \
    --SubnetId <SubnetId> \
    --region ap-guangzhou \
    --output json
```

> **注意：** 为实现就近内网拉取，需手动将每个复制地域内的 VPC 接入该实例。

#### 1.2 跨国跨地域实例同步复制

跨国场景下，暂不支持实例复制。需结合**实例同步** + **本地实例复制**实现。

**步骤1：国内外各创建一个高级版实例**

国内地域及国外地域分别创建。

**步骤2：配置实例同步规则，实现跨国按需同步**

```bash
tccli tcr ManageReplication \
    --SourceRegistryId <SourceRegistryId> \
    --DestinationRegistryId <DestinationRegistryId> \
    --region <Region> \
    --Rule '{
        "Name": "<RuleName>",
        "DestNamespace": "<DestNamespace>",
        "Override": false,
        "Filters": [
            {"Type": "name", "Value": "prod/**"},
            {"Type": "tag", "Value": ""},
            {"Type": "resource", "Value": "image"}
        ]
    }' \
    --Description "<描述>" \
    --output json
```

`Rule` 对象字段参见[跨实例（账号）同步镜像](../../操作指南/镜像分发/跨实例（账号）同步镜像/tccli%20操作.md)。

**步骤3：在两实例内配置本地从实例**

分别在国内和国外实例上执行 `CreateReplicationInstance`，覆盖各自所需的地域。最终实现：
- 国内：单点推送 → 实例复制到各国内地域
- 国外：通过同步规则接收 → 实例复制到各国外地域

### 场景2：跨平台镜像迁移或同步

#### 2.1 跨平台镜像迁移（image-transfer）

[image-transfer](https://github.com/tkestack/image-transfer) 是腾讯云开源镜像迁移工具，支持 Docker Registry V2 标准的多种镜像仓库（TCR、Docker Hub、Quay、ACR、Harbor 等）间的批量迁移。

**通用模式：**

在多仓库间迁移镜像，需编写认证鉴权文

```json
{
  "DomainInfoList": [],
  "RegistryId": "<RegistryId>",
  "CertId": "<CertId>",
  "DomainName": "<DomainName>",
  "Status": "<Status>",
  "TotalCount": 0,
  "RequestId": "<RequestId>"
}
```件和迁移规则文件后运行工具。参见 [image-transfer README](https://github.com/tkestack/image-transfer/blob/main/README.md)。

**一键迁移模式：**

从 TCR 个人版（CCR）迁移至 TCR 企业版。参见[个人版迁移至企业版完全指南](https://cloud.tencent.com/document/product/1141/52292)。

> **说明：** image-transfer 运行在本地终端/服务器上，不属于 tccli 范围。

#### 2.2 使用自定义域名保证服务连续性

从其他镜像仓库迁移至 TCR 时，可为 TCR 实例配置自定义域名，沿用原有域名，保持发布配置不变：

**查看已有自定义域名：**

```bash
tccli tcr DescribeInstanceCustomizedDomain \
    --RegistryId <RegistryId> \
    --region ap-guangzhou \
    --output json
```

```json
{
  "RequestId": "..."
}
```

**配置自定义域名：**

参见[配置自定义域名](../../操作指南/访问配置/访问域名配置/配置自定义域名/tccli%20操作.md)。

#### 2.3 通过自建 Harbor 实现跨平台实时同步

若需在第三方平台间实时同步镜像，可使用自建 Harbor 作为中转：

1. 在 Harbor 配置 Pull-based 复制策略，将源平台（如阿里云 ACR）的镜像拉取至 Harbor。
2. 在 Harbor 配置 Push-based 复制策略，将 Harbor 内的镜像推送至目标平台（如腾讯云 TCR）。

此方式可实现阿里云 ACR → TCR、Docker Hub → TCR 等跨平台实时同步，无需 tccli 操作。

> **支持的镜像仓库服务：** Docker Hub、Docker Registry、AWS ECR、Azure ACR、阿里云 ACR、GCR、华为 SWR、Artifact Hub、GitLab、Quay、JFrog Artifactory、TCR。

#### 2.4 从自建 Harbor 同步至 TCR

参见[从自建 Harbor 同步镜像到 TCR 企业版](https://cloud.tencent.com/document/product/1141/44970) — 在 Harbor 侧配置同步规则，无需 TCR tccli 操作。

### 场景3：DevOps 镜像流转

利用 TCR 实例同步功能，实现开发 → 测试 → 预发布 → 生产环境之间的镜像自动流转。

**步骤1：在各环境创建 TCR 实例（或共享实例的不同命名空间）**

**步骤2：配置实例同步规则**

在开发环境实例上配置同步规则，将指定 Tag 的镜像同步至下游实例：

```bash
tccli tcr ManageReplication \
    --SourceRegistryId <DevRegistryId> \
    --DestinationRegistryId <TestRegistryId> \
    --region ap-guangzhou \
    --Rule '{
        "Name": "dev-to-test",
        "DestNamespace": "test",
        "Override": false,
        "Filters": [
            {"Type": "name", "Value": "*/**"},
            {"Type": "tag", "Value": "dev-*"},
            {"Type": "resource", "Value": "image"}
        ]
    }' \
    --Description "开发环境 → 测试环境同步" \
    --output json
```

多个环境之间依次配置同步规则即可形成镜像流水线。

**步骤3（可选）：跨主账号同步**

若不同环境分属不同的腾讯云主账号，在配置同步规则时启用跨主账号：

```bash
tccli tcr ManageReplication \
    --SourceRegistryId <DevRegistryId> \
    --DestinationRegistryId <TestRegistryId> \
    --region ap-guangzhou \
    --Rule '{
        "Name": "dev-to-test-cross-account",
        "DestNamespace": "test",
        "Override": false,
        "Filters": [{"Type": "name", "Value": "*/**"}]
    }' \
    --PeerReplicationOption '{
        "PeerRegistryUin": "<目标主账号UIN>",
        "PeerRegistryToken": "<目标实例长期访问凭证>",
        "EnablePeerReplication": true
    }' \
    --Description "跨主账号同步" \
    --output json
```

#### 交付流水线（CODING DevOps）

对于更复杂的 DevOps 需求，可使用腾讯云 [CODING DevOps](https://console.cloud.tencent.com/coding) 一站式平台。TCR 的交付流水线功能依赖于 CODING DevOps 的 CI/CD 能力，支持：
- [推送代码自动触发镜像构建和应用部署](https://cloud.tencent.com/document/product/1141/48186#scene1)
- [本地推送镜像后自动触发部署](https://cloud.tencent.com/document/product/1141/48186#scene2)

> 交付流水线非 tccli 范围，详见[使用交付流水线实现容器 DevOps](https://cloud.tencent.com/document/product/1141/48186)。

## 验证

### Control plane (tccli)

**实例复制验证：**

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 从实例已创建 | `DescribeReplicationInstances` | `ReplicationRegistryId` 非空，`Status: "Running"` |
| 创建任务完成 | `DescribeReplicationInstanceCreateTasks` | `TaskStatus: "SUCCESS"` |

**实例同步验证：**

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 同步规则已存在 | `DescribeReplicationPolicies` | 目标规则在列表中，`Enabled: true` |
| 同步状态正常 | `DescribeReplicationInstanceSyncStatus` | `ReplicationStatus: "Succeed"` |

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 处理 |
|------|------|
| `CreateReplicationInstance` 报 `UnsupportedOperation` | 实例非高级版；确认 `RegistryType: "premium"` |
| 跨国实例复制不可用 | 使用实例同步（`ManageReplication`）替代 |
| 实例同步一直处于 `InProgress` | 待同步镜像较多，查看同步日志中的 `Percentage` 进度 |
| 跨主账号同步报权限错误 | 确认 `PeerRegistryToken` 为[用户级账号](https://cloud.tencent.com/document/product/1141/41829)密码，非服务级账号 |
| 删除实例时报需先删除复制实例/同步规则 | 先执行 `DeleteReplicationInstance` 删除所有从实例，再删除对应同步规则 `DeleteReplicationRule` |
| `ManageReplication` 返回 `InvalidParameter` | 检查 `Rule.Name` 命名规范：小写字母、数字及 `-._`，以字母或数字开头 |
| 同步速度慢 | 受限于跨地域云联网带宽，多租户共享；如需特殊保障，[在线咨询](https://cloud.tencent.com/online-service?from=doc_1141) |
| image-transfer 迁移失败 | 参考 [image-transfer README](https://github.com/tkestack/image-transfer/blob/main/README.md) 排查，确认认证鉴权文件和迁移规则文件格式正确 |

## 控制台替代

- **实例复制：** [容器镜像服务控制台 → 同步复制 → 实例复制](https://console.cloud.tencent.com/tcr/replication)
- **实例同步：** [容器镜像服务控制台 → 同步复制 → 实例同步](https://console.cloud.tencent.com/tcr/replication)
- **自定义域名：** [容器镜像服务控制台 → 访问控制 → 域名管理](https://console.cloud.tencent.com/tcr)
- **交付流水线：** [容器镜像服务控制台 → 交付流水线](https://console.cloud.tencent.com/tcr/delivery)
