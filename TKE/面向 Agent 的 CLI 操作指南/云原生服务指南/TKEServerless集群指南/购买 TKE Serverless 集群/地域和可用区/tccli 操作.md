# 地域和可用区

> 对照官方：[地域和可用区](https://cloud.tencent.com/document/product/457/58172) · page_id `58172`

## 概述

TKE Serverless 集群在特定地域和可用区内创建和运行。选择合适的地域和可用区对延迟、可用性和合规性至关重要。

- **地域（Region）**：物理数据中心的所在地，如广州、上海、北京等。不同地域间网络隔离，不可互通（除非通过云联网等跨地域互联产品）。
- **可用区（Zone）**：同一地域内物理隔离的数据中心，通过低延迟内网互联。TKE Serverless 集群的 Pod 调度到底层资源池，底层资源池覆盖指定可用区。建议多可用区部署以提高高可用性。

TKE Serverless 集群当前支持的地域与标准 TKE 基本一致，可通过 `DescribeRegions` 查询。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看支持的地域列表 | `DescribeRegions` | 是 |
| 查看地域下可用区 | `DescribeZones`（底层 CVM API） | 是 |
| 按地域过滤 Serverless 集群 | `DescribeEKSClusters --region <Region>` | 是 |
| 创建集群时选择地域 | `CreateEKSCluster --region <Region>` | 否 |
| 查看全球地域和可用区 | 无单一 API，逐地域查询 | — |

## 操作步骤

### 步骤 1：查询 TKE 支持的全部地域

```bash
tccli tke DescribeRegions
# expected: exit 0，返回所有 TKE 支持的地域列表
```

**预期输出**：

```json
{
    "Regions": [
        {
            "RegionName": "ap-guangzhou",
            "RegionRemark": "华南地区（广州）"
        },
        {
            "RegionName": "ap-shanghai",
            "RegionRemark": "华东地区（上海）"
        },
        {
            "RegionName": "ap-beijing",
            "RegionRemark": "华北地区（北京）"
        },
        {
            "RegionName": "ap-nanjing",
            "RegionRemark": "华东地区（南京）"
        },
        {
            "RegionName": "ap-chengdu",
            "RegionRemark": "西南地区（成都）"
        },
        {
            "RegionName": "ap-hongkong",
            "RegionRemark": "港澳台地区（中国香港）"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 2：查询特定地域下的可用区

TKE Serverless 集群的 Pod 调度到底层资源池，资源池覆盖的可用区可通过查询该地域下已有 Serverless 集群的 `SubnetIds` 推断，或通过 CVM 的 `DescribeZones` 查看地域下可用区：

```bash
tccli cvm DescribeZones --region <Region>
# expected: exit 0，返回该地域下的可用区列表
```

**预期输出**：

```json
{
    "TotalCount": 3,
    "ZoneSet": [
        {
            "Zone": "ap-guangzhou-1",
            "ZoneName": "广州一区",
            "ZoneState": "AVAILABLE"
        },
        {
            "Zone": "ap-guangzhou-3",
            "ZoneName": "广州三区",
            "ZoneState": "AVAILABLE"
        },
        {
            "Zone": "ap-guangzhou-4",
            "ZoneName": "广州四区",
            "ZoneState": "AVAILABLE"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **注意**：`DescribeZones` 返回的是 CVM 可用区列表。TKE Serverless 集群的底层资源池覆盖这些可用区，因此可用区与 CVM 一致。实际可用的底层资源池可能小于此列表。

### 步骤 3：按地域查询已有 Serverless 集群

```bash
tccli tke DescribeEKSClusters --region <Region>
# expected: exit 0，返回该地域下所有 Serverless 集群
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

通过已有集群的分布可确认该地域当前可用。

### 步骤 4：选择地域的最佳实践

| 考量因素 | 建议 |
|---------|------|
| 用户地理位置 | 选择离用户最近的地域以降低网络延迟 |
| 合规要求 | 选择符合数据驻留法规的地域 |
| 其他云服务所在地域 | 确保 TKE Serverless 集群与 TCR、CLS、CDB 等云服务在同一地域，避免跨地域流量费用 |
| 成本 | 不同地域计费单价可能有差异，参考 [[计费概述](../计费概述/tccli%20操作.md)] |
| 多地域容灾 | 在多个地域部署集群以提高容灾能力 |

## 验证

```bash
# 验证所选地域可用
tccli tke DescribeRegions | jq '.Regions[] | select(.RegionName == "REGION")'
# expected: 返回该地域信息

# 验证该地域下可用区可用
tccli cvm DescribeZones --region <Region> | jq '.ZoneSet[] | select(.ZoneState == "AVAILABLE").Zone'
# expected: 返回可用区列表

# 验证该地域已有 Serverless 集群（可选）
tccli tke DescribeEKSClusters --region <Region> | jq '.TotalCount'
# expected: 返回该地域 Serverless 集群总数
```

```json
{
  "TotalCount": "<TotalCount>",
  "Regions": "<Regions>",
  "Alias": "<Alias>",
  "RegionId": "<RegionId>",
  "RegionName": "<RegionName>",
  "Status": "<Status>"
}
```

## 清理

本页面为只读操作，无需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeRegions` 返回 `AuthFailure` | `tccli configure list` 检查 secretId/secretKey | 凭据未配置或已过期 | 执行 `tccli configure set` 重新配置 |
| `DescribeZones` 返回 `InvalidParameter.Region` | 检查 `--region` 值拼写 | 地域名拼写错误或不存在 | 使用 `DescribeRegions` 返回的有效地域名 |
| `DescribeZones` 返回的 `ZoneState` 为 `UNAVAILABLE` | 等待或换其他可用区 | 该可用区当前资源紧张或维护中 | 选择 `ZoneState` 为 `AVAILABLE` 的可用区 |
| `DescribeEKSClusters` 在特定地域返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少该地域 `tke:DescribeEKSClusters` 权限 | 联系主账号为该地域授予相应权限 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 所选地域在 `DescribeRegions` 返回但创建集群失败 | 错误信息中查看具体原因 | 该地域 TKE Serverless 资源池可能暂未开放或已售罄 | 选择其他可用地域，或联系售后确认该地域 Serverless 资源可用性 |
| `DescribeZones` 返回可用区列表但 Serverless 不可用 | TKE Serverless 底层资源池覆盖的可用区可能少于 CVM | 资源池容量限制 | 选择该地域其他可用区，或换地域 |

## 下一步

- [计费概述](../计费概述/tccli%20操作.md) — 了解各区域计费差异
- [购买限制](../购买限制/tccli%20操作.md) — 了解地域级配额和限制
- [创建 Serverless 集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md) — 在选定地域创建集群

## 控制台替代

[TKE 控制台 → Serverless 集群 → 新建](https://console.cloud.tencent.com/tke2/ecluster/create)：在创建集群流程的第一步选择地域和可用区，下拉列表实时反映当前可用项。
