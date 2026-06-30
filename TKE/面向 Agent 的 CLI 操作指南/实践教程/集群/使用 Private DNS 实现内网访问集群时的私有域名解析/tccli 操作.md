# 使用 Private DNS 实现内网访问集群时的私有域名解析（tccli）

> 对照官方：[使用 Private DNS 实现内网访问集群时的私有域名解析](https://cloud.tencent.com/document/product/457/55348) · page_id `55348`

## 概述

本页介绍如何通过 Private DNS 为 TKE 集群配置私有域名解析，使集群内网访问端点能够通过自定义域名在 VPC 内被解析。适用于需要通过内网访问 TKE 集群 API Server 的场景。

> **注意**：本节涉及的部分操作（如使用 `kubectl` 访问集群内部资源进行验证）需要 `kubectl` 能够连通集群 API Server。如果你的环境当前无法通过 `kubectl` 连接到集群（例如未配置 VPN/IOA 或集群未开启内网访问），请先完成本章节的 Private DNS 配置建立网络连通性，或参考相关网络文档配置访问通道后再执行涉及 `kubectl` 的验证步骤。

### 适用场景

- 企业内部通过自定义私有域名（替代默认的集群内网访问域名）访问 TKE 集群 API Server
- 跨 VPC 场景下，通过 Private DNS 统一解析集群内网端点
- 简化开发与运维对集群访问地址的记忆和管理

### 涉及的服务与 API

| 服务 | API 版本 | CLI 命令空间 | 说明 |
|------|----------|-------------|------|
| 私有域解析 Private DNS | 2020-10-28 | `tccli privatedns` | 管理私有域、解析记录、VPC 关联 |
| 容器服务 TKE | 2022-05-01 | `tccli tke` | 查询集群信息（VPC、内网端点等） |

## 控制台与 CLI 参数映射

| 控制台参数 | CLI 参数 | 类型 | 必填 | 说明 | 幂等性 |
|-----------|---------|------|------|------|--------|
| 地域 | `--region` | String | 是 | 腾讯云地域，如 `ap-guangzhou` | 位置参数，不影响幂等 |
| 集群 ID | `--ClusterIds` | Array | 是 | 目标 TKE 集群 ID | 只读，幂等 |
| 私有域 | `--Domain` | String | 是 | 私有域名，如 `cluster-id.region.tke.private` | 同名域名创建会报错，更新时用 ZoneId 定位 |
| 所属 VPC | `--VpcSet` | Array | 是 | 关联的 VPC 列表，含 Region 与 UniqVpcId | VPC 重复关联会报冲突 |
| 记录类型 | `--RecordType` | String | 是 | DNS 记录类型，一般为 `A` | 同 SubDomain + RecordType 重复创建会报错 |
| 记录值 | `--RecordValue` | String | 是 | 解析目标 IP 地址 | 可重复创建不同记录值 |
| 子域名 | `--SubDomain` | String | 是 | 记录前缀，如 `api`、`*` | 同 SubDomain + RecordType 重复创建会报错 |
| Zone ID | `--ZoneId` | String | 是（更新/删除时） | 私有域的唯一 ID，创建后由系统返回 | 天然幂等，唯一标识 |
| 备注 | `--Remark` | String | 否 | 私有域备注信息 | 不影响幂等 |

## 前置条件

- 已完成[环境准备](../../环境准备.md)
- tccli 已配置凭据和地域

## 操作步骤

### #### 选择依据

根据你的需求选择对应的配置方案：

- **最小配置**：仅配置基本的私有域名解析，使集群 API Server 内网端点可通过自定义域名访问 —— 适用于单一 VPC 内简单解析场景
- **增强配置**：在最小配置基础上，增加多子域名解析记录、多 VPC 关联等 —— 适用于跨 VPC 访问或需要解析多个集群端点（如内网 CLB、节点 IP）的场景

### #### 最小配置

**目标**：为集群 API Server 内网访问端点创建一个私有域并配置一条 A 记录，使同一个 VPC 内的资源可通过自定义域名访问集群。

#### 步骤 1：获取集群信息（控制面）

首先查询目标集群的 VPC ID 和内网访问端点地址。

```bash
tccli tke DescribeClusters \
  --region <Region> \
  --ClusterIds '["CLUSTER_ID"]'
```

**输出示例**：

```json
{
  "Clusters": [
    {
      "ClusterId": "CLUSTER_ID",
      "ClusterName": "example-cluster",
      "ClusterNetworkSettings": {
        "VpcId": "VPC_ID",
        "ClusterCIDR": "10.0.0.0/16"
      },
      "ClusterExternalEndpoint": "https://CLUSTER_ID.ccs.tencent-cloud.com",
      "ClusterInternalEndpoint": "https://INTERNAL_IP:6443"
    }
  ],
  "TotalCount": 1,
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

从输出中提取以下关键信息：

- `VpcId`（`ClusterNetworkSettings.VpcId`）：集群所在 VPC ID，用于后续私有域关联
- `ClusterInternalEndpoint`：集群内网访问地址，从中解析出内网 IP 用于 DNS A 记录

#### 步骤 2：创建私有域（控制面）

使用 `privatedns CreatePrivateZone` 创建私有解析域。由于参数较多，使用 `--cli-input-json` 方式传入。

`create-private-zone.json`：

```json
{
  "Domain": "CLUSTER_ID.REGION.tke.private",
  "Remark": "TKE cluster CLUSTER_ID private DNS zone",
  "VpcSet": [
    {
      "Region": "REGION",
      "UniqVpcId": "VPC_ID"
    }
  ]
}
```

```bash
tccli privatedns CreatePrivateZone \
  --cli-input-json file://create-private-zone.json \
  --region <Region>
```

**输出示例**：

```json
{
  "ZoneId": "zone-xxxxxxxx",
  "Domain": "CLUSTER_ID.REGION.tke.private",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

记录返回的 `ZoneId`（如 `zone-xxxxxxxx`），后续添加解析记录和关联 VPC 都需要用到。

> **注意**：创建 Private DNS 私有域会产生少量按量计费。详见 [Private DNS 计费说明](https://cloud.tencent.com/document/product/1338/50529)。

#### 步骤 3：添加解析记录（控制面）

为私有域添加 A 记录，将 `api` 子域名解析到集群内网端点 IP。

从步骤 1 中获取的 `ClusterInternalEndpoint`（如 `https://INTERNAL_IP:6443`）中提取 IP 地址 `INTERNAL_IP`（去掉协议前缀和端口号）。

```bash
tccli privatedns CreatePrivateZoneRecord \
  --region <Region> \
  --ZoneId ZONE_ID \
  --RecordType A \
  --RecordValue INTERNAL_IP \
  --SubDomain api \
  --TTL 600
```

**输出示例**：

```json
{
  "RecordId": "record-xxxxxxxx",
  "ZoneId": "ZONE_ID",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 步骤 4：验证解析（数据面）

在 VPC 内的资源（CVM、容器等）上验证私有域名解析是否生效。注意：此步骤需要能够访问该 VPC 内网（如通过 VPN、专线或 IOA）。

```bash
nslookup api.CLUSTER_ID.REGION.tke.private
```

**预期输出**：

```
Server:         10.0.0.2
Address:        10.0.0.2#53

Name:   api.CLUSTER_ID.REGION.tke.private
Address: INTERNAL_IP
```

也可使用 `dig` 命令验证：

```bash
dig api.CLUSTER_ID.REGION.tke.private
```

**预期输出**（关键部分）：

```
;; ANSWER SECTION:
api.CLUSTER_ID.REGION.tke.private. 600 IN A   INTERNAL_IP
```

### #### 增强配置

**目标**：在最小配置基础上，添加多条解析记录或关联多个 VPC，满足更复杂的网络场景。

#### 步骤 3（增强）：添加多条解析记录

除 `api` 子域名外，可以添加其他子域名解析到集群的不同端点地址。例如添加通配符记录或解析到其他集群内部服务。

`create-multiple-records.json`：

```json
{
  "ZoneId": "ZONE_ID",
  "Records": [
    {
      "RecordType": "A",
      "RecordValue": "INTERNAL_IP",
      "SubDomain": "api",
      "TTL": 600
    },
    {
      "RecordType": "A",
      "RecordValue": "INTERNAL_IP",
      "SubDomain": "apiserver",
      "TTL": 600
    }
  ]
}
```

```bash
# 逐条创建（通过循环调用 CreatePrivateZoneRecord），上述模板仅为参数参考
# 实际调用仍使用 CreatePrivateZoneRecord 逐条创建
tccli privatedns CreatePrivateZoneRecord \
  --region <Region> \
  --ZoneId ZONE_ID \
  --RecordType A \
  --RecordValue INTERNAL_IP \
  --SubDomain apiserver \
  --TTL 600
```

#### 步骤 4（增强）：关联额外的 VPC

如果创建私有域时未关联目标 VPC，或需要关联额外的 VPC，使用 `ModifyPrivateZoneVpc`。

```bash
tccli privatedns ModifyPrivateZoneVpc \
  --region <Region> \
  --ZoneId ZONE_ID \
  --VpcSet '[{"Region":"REGION","UniqVpcId":"VPC_ID"}]'
```

**输出示例**：

```json
{
  "ZoneId": "ZONE_ID",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **注意**：`ModifyPrivateZoneVpc` 中的 `VpcSet` 会**替换**（而非追加）已有的 VPC 关联。如需保留已有关联并新增 VPC，需在 `VpcSet` 中包含所有目标 VPC。

### 控制面验证

通过查询 Private DNS 服务确认私有域和解析记录已正确创建。

查询私有域列表：

```bash
tccli privatedns DescribePrivateZoneList \
  --region <Region> \
  --Filters '[{"Name":"Domain","Values":["CLUSTER_ID.REGION.tke.private"]}]'
```

**输出示例**：

```json
{
  "PrivateZoneSet": [
    {
      "ZoneId": "ZONE_ID",
      "Domain": "CLUSTER_ID.REGION.tke.private",
      "RecordCount": 1,
      "VpcSet": [
        {
          "Region": "REGION",
          "UniqVpcId": "VPC_ID"
        }
      ]
    }
  ],
  "TotalCount": 1,
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

查询解析记录列表：

```bash
tccli privatedns DescribePrivateZoneRecordList \
  --region <Region> \
  --ZoneId ZONE_ID
```

**输出示例**：

```json
{
  "RecordSet": [
    {
      "RecordId": "record-xxxxxxxx",
      "SubDomain": "api",
      "RecordType": "A",
      "RecordValue": "INTERNAL_IP",
      "TTL": 600,
      "Enabled": 1
    }
  ],
  "TotalCount": 1,
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

## 验证

### 控制面验证

参见上节「控制面验证」中的查询命令，确认私有域状态为正常、记录数量与预期一致、VPC 关联正确。

可通过以下方式进一步确认：

```bash
# 检查私有域状态
tccli privatedns DescribePrivateZone --region <Region> --ZoneId ZONE_ID
```

**输出示例**：

```json
{
  "PrivateZone": {
    "ZoneId": "ZONE_ID",
    "Domain": "CLUSTER_ID.REGION.tke.private",
    "Status": "ENABLED",
    "RecordCount": 1,
    "VpcSet": [
      {
        "Region": "REGION",
        "UniqVpcId": "VPC_ID"
      }
    ],
    "CreatedOn": "YYYY-MM-DDThh:mm:ssZ"
  },
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 数据面验证

在 VPC 内的资源上执行以下测试命令，验证 DNS 解析结果：

```bash
# 使用 nslookup
nslookup api.CLUSTER_ID.REGION.tke.private

# 使用 dig
dig api.CLUSTER_ID.REGION.tke.private +short

# 使用 ping 测试连通性（如果安全组规则允许 ICMP）
ping -c 3 api.CLUSTER_ID.REGION.tke.private
```

**预期结果**：

- `nslookup` / `dig` 返回的内网 IP 与集群 `ClusterInternalEndpoint` 中的 IP 一致
- `ping` 能够成功收到回复（前提：安全组规则允许 ICMP，且目标 IP 可 ping 通）

## 清理

> **注意**：Private DNS 私有域会产生按量计费。如果不再需要使用该私有域，建议及时清理以避免持续产生费用。详见 [Private DNS 计费概述](https://cloud.tencent.com/document/product/1338/50529)。

### 步骤 1：删除解析记录

删除私有域下的所有解析记录（需逐条删除）。

```bash
tccli privatedns DeletePrivateZoneRecord \
  --region <Region> \
  --ZoneId ZONE_ID \
  --RecordId record-xxxxxxxx
```

**输出示例**：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 2：删除私有域

确认所有解析记录已删除后，删除私有域。

```bash
tccli privatedns DeletePrivateZone \
  --region <Region> \
  --ZoneId ZONE_ID
```

**输出示例**：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 3：确认清理完成

验证私有域已被删除。

```bash
tccli privatedns DescribePrivateZoneList \
  --region <Region> \
  --Filters '[{"Name":"Domain","Values":["CLUSTER_ID.REGION.tke.private"]}]'
```

**预期输出**：`TotalCount` 为 `0`。

## 排障

| 问题 | 可能原因 | 排查方法 | 解决方案 |
|------|---------|---------|---------|
| DNS 解析失败，`nslookup` 返回 `NXDOMAIN` 或超时 | VPC 未关联到私有域；私有域未启用；私有域或记录被误删 | `tccli privatedns DescribePrivateZone --region <Region> --ZoneId ZONE_ID` 检查私有域状态；`tccli privatedns DescribePrivateZoneRecordList --region <Region> --ZoneId ZONE_ID` 检查记录是否存在 | 确认 VPC 已关联（`VpcSet` 中包含目标 VPC）；确认私有域 `Status` 为 `ENABLED`；如记录被误删，重新创建 |
| 私有域创建失败，返回 `InvalidParameter.Domain` 或类似错误 | 域名格式不合法；域名已被占用 | 检查错误信息中的具体提示；`tccli privatedns DescribePrivateZoneList --region <Region> --Filters '[{"Name":"Domain","Values":["your.domain"]}]'` 确认是否已存在 | 修改域名格式（仅支持英文、数字、连字符和下划线）；使用其他未被占用的域名 |
| 解析记录创建失败，返回 `ResourceInUse.RecordExists` | 同子域名、同记录类型已存在一条记录 | `tccli privatedns DescribePrivateZoneRecordList --region <Region> --ZoneId ZONE_ID` 查看已有记录 | 删除已有记录后重新创建，或修改现有记录值：`tccli privatedns ModifyPrivateZoneRecord --region <Region> --ZoneId ZONE_ID --RecordId record-xxx --RecordValue NEW_IP` |
| 在 VPC 内 DNS 解析不生效 | VPC 未关联到私有域；VPC 内 DNS 服务器未使用 Private DNS 提供的解析地址 | `tccli privatedns DescribePrivateZone --region <Region> --ZoneId ZONE_ID` 检查 `VpcSet` 是否包含目标 VPC | 使用 `tccli privatedns ModifyPrivateZoneVpc --region <Region> --ZoneId ZONE_ID --VpcSet '[{"Region":"REGION","UniqVpcId":"VPC_ID"}]'` 关联目标 VPC；确认 VPC DNS 配置是否正确 |

## 控制台替代

[TKE 控制台](https://console.cloud.tencent.com/tke2)

## 下一步

完成私有域名解析配置后，你可以继续：

- [使用 TCR 内网免密拉取容器镜像]() —— 配合 Private DNS 实现完整的内网镜像拉取链路
- [VPC-CNI 模式下 Pod 固定 IP 配置]() —— 为集群内 Pod 配置固定 IP 以配合 DNS 解析
- [Private DNS 官方文档 - 私有域管理](https://cloud.tencent.com/document/product/1338/50534) —— 了解更多 Private DNS 功能（转发规则、PTR 记录等）
