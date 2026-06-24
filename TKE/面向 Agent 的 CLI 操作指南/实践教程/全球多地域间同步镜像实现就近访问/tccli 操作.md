# 全球多地域间同步镜像实现就近访问（tccli）

> 对照官方：[全球多地域间同步镜像实现就近访问](https://cloud.tencent.com/document/product/1141/61458) · page_id `61458`

## 概述

企业将容器业务拓展至多个地域时，需实现容器镜像的全球多地域同步与就近拉取，以提高拉取速度、降低跨地域公网流量成本。TCR 企业版提供了两项互补能力：

| 能力 | 说明 | 规格要求 |
|------|------|---------|
| **实例同步** | 多个实例间按需自动同步指定镜像，支持跨主账号 | 标准版 / 高级版 |
| **实例复制** | 单一实例在多个地域部署副本，统一域名，实时流式同步 | 高级版 |

**对比：**

| 维度 | 实例同步 | 实例复制 |
|------|---------|---------|
| 同步范围 | 按需（基于规则精确匹配） | 全量（推送后全部复制） |
| 推送能力 | 每个实例均可推送 | 仅主实例可推送 |
| 域名 | 各实例独立域名 | 统一域名 |
| 跨主账号 | 支持 | 不支持 |
| Helm Chart 筛选 | 支持 | 全量复制 |
| 同步日志 | 支持 | 支持 |
| 跨国 | 支持 | 不支持 |

> **结合使用自定义域名：** TCR 企业版实例支持自定义域名，结合 DNS 服务可实现多个实例共用同一域名，进一步增强灵活性和就近访问能力。参见[配置自定义域名](../../操作指南/访问配置/访问域名配置/配置自定义域名/tccli%20操作.md)。

## 前置条件

- [环境准备](../../环境准备.md)
- 已成功[购买企业版实例](https://cloud.tencent.com/document/product/1141/51110)（参见[企业版快速入门](../../快速入门/企业版快速入门/tccli%20操作.md)）。注意：实例同步需要标准版或高级版，实例复制需要高级版。
- 如果使用子账号操作，需提前授予对应实例的 `tcr:ManageReplication`、`tcr:CreateReplicationInstance` 等权限（参见[企业版授权方案示例](https://cloud.tencent.com/document/product/1141/41417)）。
- 跨主账号同步时，目标侧需提供主账号 UIN 及[用户级账号](https://cloud.tencent.com/document/product/1141/41829)长期访问凭证。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查看 TCR 实例列表 | `DescribeInstances` | 是 |
| 创建同步规则（实例同步） | `ManageReplication` | 否（覆盖同名规则） |
| 查看同步策略列表 | `DescribeReplicationPolicies` | 是 |
| 查看同步状态/日志 | `DescribeReplicationInstanceSyncStatus` | 是 |
| 删除同步实例 | `DeleteReplicationInstance` | 是 |
| 创建从实例（实例复制） | `CreateReplicationInstance` | 否（同一地域重复创建报错） |
| 查看实例复制列表 | `DescribeReplicationInstances` | 是 |
| 查看从实例创建进度 | `DescribeReplicationInstanceCreateTasks` | 是 |
| 删除从实例 | `DeleteReplicationInstance` | 是 |
| 查看自定义域名 | `DescribeInstanceCustomizedDomain` | 是 |
| 配置自定义域名 | 参见[配置自定义域名](../../操作指南/访问配置/访问域名配置/配置自定义域名/tccli%20操作.md) | — |

## 操作步骤

### 场景1：出海业务实现全球多地域就近访问

出海业务需要在全球多地同时部署，需兼顾数据合规（国内外独立管理），同时保持统一配置。方案：国内实例 + 国外实例，通过**实例同步**连接国内外，各自通过**实例复制**覆盖区域内的多个地域。

```
 国内: 北京（主）──实复制──→ 上海 │ 广州 │ 成都
                ┊
         实例同步（按需，出境合规筛选）
                ┊
 国外: 法兰克福（主）──实复制──→ 硅谷 │ 弗吉尼亚 │ 新加坡
```

#### 步骤1：创建国内和国外两个高级版实例

分别在需要的地域创建（如北京 `ap-beijing`、法兰克福 `eu-frankfurt`）：

```bash
tccli tcr CreateInstance --cli-input-json '{
    "RegistryName": "<RegistryName>",
    "RegistryType": "premium",
    "RegistryChargePrepaid": {
        "Period": 1,
        "RenewFlag": 0
    }
}' --region <Region> --output json
```

记录返回的 `RegistryId`。

#### 步骤2：配置实例同步规则（国内 → 国外，按需出境）

在国内实例上创建同步规则，仅同步需出境发布的镜像：

```bash
tccli tcr ManageReplication \
    --SourceRegistryId <国内RegistryId> \
    --DestinationRegistryId <国外RegistryId> \
    --region ap-beijing \
    --Rule '{
        "Name": "cn-to-overseas",
        "DestNamespace": "production",
        "Override": false,
        "Filters": [
            {"Type": "name", "Value": "production/**"},
            {"Type": "tag", "Value": "release-*"},
            {"Type": "resource", "Value": "image"}
        ],
        "Deletion": false
    }' \
    --Description "国内→境外 按需同步（生产镜像 release 版本）" \
    --output json
```

> **注意：** 由于腾讯云国内外控制台有独立的账号体系，跨国同步只能通过实例同步实现。同步规则中通过 `Filters` 精确控制哪些镜像出境。

#### 步骤3：在国内外实例上各自创建从实例（实例复制）

国内实例（北京）创建上海、广州、成都从实例：

```bash
# 上海从实例
tccli tcr CreateReplicationInstance --cli-input-json '{
    "RegistryId": "<国内RegistryId>",
    "ReplicationRegionId": 4,
    "ReplicationRegionName": "ap-shanghai",
    "SyncTag": true
}' --region ap-beijing --output json

# 广州从实例
tccli tcr CreateReplicationInstance --cli-input-json '{
    "RegistryId": "<国内RegistryId>",
    "ReplicationRegionId": 1,
    "ReplicationRegionName": "ap-guangzhou",
    "SyncTag": true
}' --region ap-beijing --output json

# 成都从实例
tccli tcr CreateReplicationInstance --cli-input-json '{
    "RegistryId": "<国内RegistryId>",
    "ReplicationRegionId": 16,
    "ReplicationRegionName": "ap-chengdu",
    "SyncTag": true
}' --region ap-beijing --output json
```

国外实例（法兰克福）创建硅谷、弗吉尼亚、新加坡从实例：

```bash
# 新加坡从实例
tccli tcr CreateReplicationInstance --cli-input-json '{
    "RegistryId": "<国外RegistryId>",
    "ReplicationRegionId": 9,
    "ReplicationRegionName": "ap-singapore",
    "SyncTag": true
}' --region eu-frankfurt --output json

# 硅谷从实例
tccli tcr CreateReplicationInstance --cli-input-json '{
    "RegistryId": "<国外RegistryId>",
    "ReplicationRegionId": 15,
    "ReplicationRegionName": "na-siliconvalley",
    "SyncTag": true
}' --region eu-frankfurt --output json

# 弗吉尼亚从实例
tccli tcr CreateReplicationInstance --cli-input-json '{
    "RegistryId": "<国外RegistryId>",
    "ReplicationRegionId": 22,
    "ReplicationRegionName": "na-ashburn",
    "SyncTag": true
}' --region eu-frankfurt --output json
```

#### 步骤4：验证同步状态

```bash
# 查看实例同步状态
tccli tcr DescribeReplicationInstanceSyncStatus \
    --RegistryId <国内RegistryId> \
    --ReplicationRegistryId <同步RegistryId> \
    --ShowReplicationLog true \
    --region ap-beijing --output json

# 查看各从实例状态
tccli tcr DescribeReplicationInstances \
    --RegistryId <国内RegistryId> \
    --region ap-beijing --output json
```

```json
{
  "ReplicationStatus": "<ReplicationStatus>",
  "ReplicationTime": "<ReplicationTime>",
  "ReplicationLog": "<ReplicationLog>",
  "ResourceType": "<ResourceType>",
  "Source": "<Source>",
  "Destination": "<Destination>",
  "Status": "<Status>",
  "StartTime": "<StartTime>"
}
```

#### 步骤5：将各地域 VPC 接入对应的从实例

每个地域的容器集群或云服务器需通过内网访问其对应地域的从实例，参见[配置内网访问控制](../../操作指南/访问配置/访问网络控制/配置内网访问控制/tccli%20操作.md)。

### 场景2：大型集团内多个子公司及业务间流转镜像

大型集团内多家子公司/BG 使用不同的腾讯云主账号独立管理云资源，通过跨主账号实例同步实现公共基础镜像共享。

#### 步骤1：各子公司/BG 创建各自的 TCR 实例

在各自账号下按需创建。

#### 步骤2：配置跨主账号实例同步（公共基础镜像共享）

在集团基础平台团队的 TCR 实例上，创建指向各子公司 TCR 实例的跨主账号同步规则：

```bash
tccli tcr ManageReplication \
    --SourceRegistryId <基础平台RegistryId> \
    --DestinationRegistryId <子公司RegistryId> \
    --region ap-guangzhou \
    --Rule '{
        "Name": "shared-base-images",
        "DestNamespace": "shared",
        "Override": false,
        "Filters": [
            {"Type": "name", "Value": "base/**"},
            {"Type": "tag", "Value": ""},
            {"Type": "resource", "Value": "image"}
        ]
    }' \
    --PeerReplicationOption '{
        "PeerRegistryUin": "<子公司主账号UIN>",
        "PeerRegistryToken": "<子公司实例长期访问凭证>",
        "EnablePeerReplication": true
    }' \
    --Description "集团公共基础镜像共享" \
    --output json
```

> **注意：** 跨主账号同步需要目标侧提供[用户级账号](https://cloud.tencent.com/document/product/1141/41829)长期访问密码（`PeerRegistryToken`）。不支持服务级账号。同步规则生命周期与目标账号最新添加规则的凭证生命周期保持一致。

#### 步骤3：各业务线间的镜像按需同步

同一子公司内不同业务线之间也可配置实例同步，实现镜像在各生产阶段（开发 → 测试 → 生产）的自动流转：

```bash
# 开发 → 测试
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
        ],
        "Deletion": false
    }' \
    --output json

# 测试 → 生产
tccli tcr ManageReplication \
    --SourceRegistryId <TestRegistryId> \
    --DestinationRegistryId <ProdRegistryId> \
    --region ap-guangzhou \
    --Rule '{
        "Name": "test-to-prod",
        "DestNamespace": "production",
        "Override": false,
        "Filters": [
            {"Type": "name", "Value": "*/**"},
            {"Type": "tag", "Value": "release-*"},
            {"Type": "resource", "Value": "image"}
        ],
        "Deletion": false
    }' \
    --output json
```

#### 步骤4：结合实例复制覆盖各地域

在集团实例或各业务实例上，根据业务地域需求添加从实例，实现单地域推送、多地域就近拉取。参见[场景1](#场景1出海业务实现全球多地域就近访问) 或[同实例多地域复制镜像](../../操作指南/镜像分发/同实例多地域复制镜像/tccl

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
```i%20操作.md)。

### 配置自定义域名

TCR 企业版支持自定义域名，结合 DNS 服务可实现多个实例共用同一域名，做到配置统一 + 就近访问。具体操作参见[配置自定义域名](../../操作指南/访问配置/访问域名配置/配置自定义域名/tccli%20操作.md)。

**查看现有自定义域名：**

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

> **提示：** 国内站可使用 PrivateDNS 产品配置内网域名解析；国际站暂不支持 PrivateDNS，建议使用自建 DNS 服务。

## 验证

### Control plane (tccli)

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 源实例已就绪 | `DescribeInstances --Registryids '["<RegistryId>"]'` | `Status: "Running"` |
| 同步规则已创建 | `DescribeReplicationPolicies --RegistryId <RegistryId>` | 规则在列表中，`Enabled: true` |
| 同步状态正常 | `DescribeReplicationInstanceSyncStatus` | `ReplicationStatus: "Succeed"` |
| 从实例已创建 | `DescribeReplicationInstances --RegistryId <RegistryId>` | 各复制地域的从实例 `Status: "Running"` |
| 从实例创建完成 | `DescribeReplicationInstanceCreateTasks` | `TaskStatus: "SUCCESS"` |
| 自定义域名已配置 | `DescribeInstanceCustomizedDomain` | 目标域名在列表中 |

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| 实例同步失败 | 跨地域网络波动或规则配置错误 | 前往控制台查看同步日志，手动触发同步；仍失败 [在线咨询](https://cloud.tencent.com/online-service?from=doc_1141) |
| 实例复制失败/长时间未完成 | 后端异步复制慢 | 暂不支持查看具体仓库的复制日志；等待 `Status` 更新为 `"Running"`；仍异常 [在线咨询](https://cloud.tencent.com/online-service?from=doc_1141) |
| 不确定指定镜像是否已复制到子实例 | 缺少仓库级同步状态 | 通过 `DescribeReplicationInstanceSyncStatus --ShowReplicationLog true` 查看历史同步任务日志，按仓库/镜像名称查找 |
| `CreateReplicationInstance` 报 `UnsupportedOperation` | 实例非高级版或地域不支持 | 确认 `RegistryType: "premium"` 及目标复制地域可用 |
| 跨国同步不可用 | 暂不支持跨国的实例复制 | 使用实例同步（`ManageReplication`）替代 |
| 跨主账号同步报权限错误 | 凭证类型错误 | 确认 `PeerRegistryToken` 为[用户级账号](https://cloud.tencent.com/document/product/1141/41829)密码，非服务级账号；确认凭证未过期 |
| 同步速度慢 | 跨地域云联网带宽限制 | 暂不支持单独提高同步速度；如需保障 [在线咨询](https://cloud.tencent.com/online-service?from=doc_1141) |
| 实例复制后目标地域无法拉取 | VPC 未接入复制实例 | 手动将复制地域内的 VPC 接入实例（`ManageInternalEndpoint`） |

## 控制台替代

- **实例同步：** [容器镜像服务控制台 → 同步复制 → 实例同步](https://console.cloud.tencent.com/tcr/replication)
- **实例复制：** [容器镜像服务控制台 → 同步复制 → 实例复制](https://console.cloud.tencent.com/tcr/replication)
- **自定义域名：** [容器镜像服务控制台 → 访问控制 → 域名管理](https://console.cloud.tencent.com/tcr)
