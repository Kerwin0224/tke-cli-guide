# VPC-CNI 模式 Pod 数量限制

> 对照官方：[VPC-CNI 模式 Pod 数量限制](https://cloud.tencent.com/document/product/457/64917) · page_id `64917`

## 概述

VPC-CNI 模式下 Pod IP 来自 VPC 子网 ENI（弹性网卡），因此每个节点的 Pod 数量受限于该节点实例类型所能绑定的 ENI 数量和每个 ENI 可分配的辅助 IP 数量。不同实例类型（机型）的上限可通过 `DescribeVpcCniPodLimits` API 精确查询。

Pod 类型说明：

| Pod 类型 | API 字段 | 含义 |
|---------|---------|------|
| TKERouteENI（非固定 IP） | `TKERouteENINonStaticIP` | 共享网卡多 IP 模式，非固定 IP |
| TKERouteENI（固定 IP） | `TKERouteENIStaticIP` | 共享网卡多 IP 模式，固定 IP |
| TKEDirectENI（独占网卡） | `TKEDirectENI` | 独占网卡模式，每个 Pod 独占一个 ENI |
| TKESubENI（子网卡） | `TKESubENI` | 子网卡模式，在 ENI 上创建子接口 |

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

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeVpcCniPodLimits, tke:DescribeClusters
#    验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 4. 确认目标集群存在且为 VPC-CNI 模式
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterNetworkSettings.Cni 为 true，VpcCniType 非空
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查询实例类型的 Pod 上限 | `DescribeVpcCniPodLimits` | 是 |
| 查看集群网络模式 | `DescribeClusters`（`Cni`、`VpcCniType` 字段） | 是 |
| 查看非 VPC-CNI 集群提示 | `DescribeEnableVpcCniProgress` | 是 |

## 关键字段说明

`DescribeVpcCniPodLimits` 的主要参数和返回：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `Zone` | String | 否 | 可用区，如 `ap-guangzhou-3`。不传返回所有可用区 | — |
| `InstanceFamily` | String | 否 | 实例机型系列，如 `S5`。不传返回所有系列 | 大小写错误 → 返回空 |
| `InstanceType` | String | 否 | 实例规格，如 `S5.MEDIUM2`。须精确匹配（含型号后缀） | 格式错误 → `InvalidParameter` |
| `TKERouteENINonStaticIP` | Integer | 仅输出 | 共享网卡非固定 IP 模式的 Pod 上限 | — |
| `TKERouteENIStaticIP` | Integer | 仅输出 | 共享网卡固定 IP 模式的 Pod 上限 | — |
| `TKEDirectENI` | Integer | 仅输出 | 独占网卡模式的 Pod 上限 | — |
| `TKESubENI` | Integer | 仅输出 | 子网卡模式的 Pod 上限 | — |

## 操作步骤

### 步骤 1：查询指定实例规格的 Pod 上限

选择依据：不同实例规格（如 S5.MEDIUM2）的 ENI 数量不同，因此 Pod 上限也不同。查询时 `InstanceType` 必须精确匹配完整规格名（含型号后缀，如 `S5.MEDIUM2` 而非 `S5`）。可用区 `Zone` 为可选参数。

```bash
# 最小查询：仅指定实例规格
tccli tke DescribeVpcCniPodLimits --region <Region> \
    --InstanceType "INSTANCE_TYPE"
# expected: exit 0，返回 PodLimitsInstanceSet
```

预期输出：

```json
{
    "TotalCount": 1,
    "PodLimitsInstanceSet": [
        {
            "Zone": "ap-guangzhou-3",
            "InstanceFamily": "S5",
            "InstanceType": "S5.MEDIUM2",
            "PodLimits": {
                "TKERouteENINonStaticIP": 27,
                "TKERouteENIStaticIP": 27,
                "TKEDirectENI": 9,
                "TKESubENI": 100
            }
        }
    ]
}
```

增强查询：同时指定实例系列和可用区，精确查询：

```bash
tccli tke DescribeVpcCniPodLimits --region <Region> \
    --Zone "ZONE" \
    --InstanceFamily "INSTANCE_FAMILY" \
    --InstanceType "INSTANCE_TYPE"
# expected: exit 0，返回指定可用区、指定系列的 Pod 上限
```

```json
{
  "TotalCount": "<TotalCount>",
  "PodLimitsInstanceSet": "<PodLimitsInstanceSet>",
  "Zone": "<Zone>",
  "InstanceFamily": "<InstanceFamily>",
  "InstanceType": "<InstanceType>",
  "PodLimits": "<PodLimits>"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `INSTANCE_TYPE` | 实例规格 | 必须精确匹配，如 `S5.MEDIUM2` | CVM 控制台或 `tccli cvm DescribeInstanceTypeConfigs` |
| `INSTANCE_FAMILY` | 实例机型系列 | 如 `S5`、`SA2` | 同上 |
| `ZONE` | 可用区 | 如 `ap-guangzhou-3` | `tccli cvm DescribeZones` |

### 步骤 2：按实例系列统计 Pod 上限

```bash
# 查询某系列所有规格的 Pod 上限（不指定 InstanceType）
tccli tke DescribeVpcCniPodLimits --region <Region> \
    --InstanceFamily "INSTANCE_FAMILY"
# expected: 返回该系列下所有实例规格的 PodLimitsInstanceSet
```

```json
{
  "TotalCount": "<TotalCount>",
  "PodLimitsInstanceSet": "<PodLimitsInstanceSet>",
  "Zone": "<Zone>",
  "InstanceFamily": "<InstanceFamily>",
  "InstanceType": "<InstanceType>",
  "PodLimits": "<PodLimits>"
}
```

### 步骤 3：确认集群为 VPC-CNI 模式

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings'
# expected: Cni 为 true，VpcCniType 非空
```

预期输出（VPC-CNI 集群）：

```json
{
    "ClusterId": "cls-example",
    "ClusterNetworkSettings": {
        "VpcId": "vpc-example",
        "Cni": true,
        "VpcCniType": "tke-route-eni",
        "Subnets": ["subnet-example"],
        "EnableStaticIp": false
    }
}
```

- `VpcCniType: "tke-route-eni"` -- 共享网卡多 IP 模式（TKERouteENI）
- `VpcCniType: "tke-direct-eni"` -- 独占网卡模式（TKEDirectENI）

> 如果集群非 VPC-CNI，`DescribeEnableVpcCniProgress` 会返回错误 `"cluster CLUSTER_ID is not vpc-cni cluster"`。

## 验证

### 验证 Pod 上限查询结果

```bash
# 查询 S5.MEDIUM2 的 Pod 上限
tccli tke DescribeVpcCniPodLimits --region <Region> \
    --Zone "ap-guangzhou-3" \
    --InstanceFamily "S5" \
    --InstanceType "S5.MEDIUM2" \
    | jq '.PodLimitsInstanceSet[0].PodLimits'
# expected: TKERouteENINonStaticIP=27, TKERouteENIStaticIP=27, TKEDirectENI=9, TKESubENI=100
```

```json
{
  "TotalCount": "<TotalCount>",
  "PodLimitsInstanceSet": "<PodLimitsInstanceSet>",
  "Zone": "<Zone>",
  "InstanceFamily": "<InstanceFamily>",
  "InstanceType": "<InstanceType>",
  "PodLimits": "<PodLimits>"
}
```

### VPC-CNI 验证维度总览

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| `Cni` | `DescribeClusters → .Cni` | `true` |
| `VpcCniType` | `DescribeClusters → .VpcCniType` | `tke-route-eni` 或 `tke-direct-eni` |
| 非 VPC-CNI | `DescribeEnableVpcCniProgress` | 报错 `not vpc-cni cluster` |
| Pod 上限 | `DescribeVpcCniPodLimits` | 返回 `PodLimits` 含四种类型上限 |

## 清理

本页为说明与只读查询，无额外资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeVpcCniPodLimits` 返回 `InvalidParameter` | 检查 `--InstanceType` 值，确认格式为 `FAMILY.SIZE` | `InstanceType` 不完整或拼写错误（如填 `S5` 而非 `S5.MEDIUM2`） | 用精确规格名，从 `tccli cvm DescribeInstanceTypeConfigs --region <Region>` 查询可用规格 |
| `DescribeVpcCniPodLimits` 返回空列表 | 检查 `Zone` 和 `InstanceFamily` 匹配 | 该可用区不支持该实例系列 | 尝试不指定 `Zone`，或更换可用区 |
| `DescribeEnableVpcCniProgress` 返回 `not vpc-cni cluster` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.ClusterNetworkSettings.Cni'` | 集群为 GR 模式，非 VPC-CNI（此为正常行为，非错误） | 确认集群模式后使用对应方案的功能 |
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeClusters` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |

### Pod 数量超限或调度异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 无法调度（ENI 资源不足） | 查看节点可用 IP：`kubectl describe node NODE_NAME \| grep tke.cloud.tencent.com` | 该节点已达 `TKERouteENINonStaticIP` 上限或 VPC 子网 IP 耗尽 | 扩容节点，或更换更高规格实例类型以获得更多 ENI |
| `TKEDirectENI` 为 0 | `tccli tke DescribeVpcCniPodLimits --region <Region> --InstanceType "INSTANCE_TYPE"` | 该实例类型不支持独占网卡模式 | 使用支持独占网卡的机型（如 S5、SA2、IT5 等） |
| `TKESubENI` 返回但不知含义 | 对照本文「概述」中 Pod 类型说明表 | 子网卡模式仅在特定网络场景下使用 | 通常以 `TKERouteENINonStaticIP` 或 `TKEDirectENI` 为主要参考 |

## 下一步

- [VPC-CNI 模式介绍](../VPC-CNI%20模式介绍/tccli%20操作.md) -- VPC-CNI 原理与选型
- [多 Pod 共享网卡模式](../多%20Pod%20共享网卡模式/tccli%20操作.md) -- TKERouteENI 使用指南
- [Pod 间独占网卡模式](../Pod%20间独占网卡模式/tccli%20操作.md) -- TKEDirectENI 使用指南
- [全局网卡使用限制（ENI 限制官方文档）](https://cloud.tencent.com/document/product/215/128412)

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) → 基本信息 → 节点与网络信息：查看 VPC-CNI 模式类型。Pod 数量限制以 API `DescribeVpcCniPodLimits` 返回值为准。
