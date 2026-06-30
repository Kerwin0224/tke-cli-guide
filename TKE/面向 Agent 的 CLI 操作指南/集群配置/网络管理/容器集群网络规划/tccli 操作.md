# 容器集群网络规划（tccli）

> 对照官方：[容器集群网络规划](https://cloud.tencent.com/document/product/457/106706) · page_id `106706`

## 概述

创建 TKE 集群前必须规划三类网段：**Node 子网**（节点 IP）、**Pod 子网**（容器 IP）和 **Service 子网**（ClusterIP）。三类网段之间不可重叠，且不能与 VPC CIDR 冲突。本文提供 tccli 层面的 CIDR 规划原则、约束验证命令和冲突检测方法。

参考集群 `cls-xxxxxxxx` 的 CIDR 规划实例：VPC `vpc-of73262z`（`172.16.0.0/12`）、ClusterCIDR `10.200.0.0/16`、ServiceCIDR `10.201.0.0/20`，三网段互不重叠，符合规划原则。ClusterCIDR 与 VPC CIDR 分离在不同的大网段。

### 三类网段定义

| 网段类型 | VPC-CNI 模式 | GR 模式 | 当前集群（GR） |
|---------|-------------|---------|---------------|
| Node 子网 | 节点所在 VPC 子网 | 节点所在 VPC 子网 | `subnet-rdmcho9m`（`172.24.0.0/20`，gz-3） |
| Pod 子网 | `Subnets`（VPC 容器子网） | `ClusterCIDR`（独立于 VPC） | `10.200.0.0/16` |
| Service 子网 | `ServiceCIDR` | `ServiceCIDR` | `10.201.0.0/20` |

### CIDR 规划核心原则

1. **三类网段互不重叠**：Node 子网、Pod 子网、Service 子网 CIDR 之间不可有任何交集
2. **不与 VPC CIDR 重叠**（GR 模式）：`ClusterCIDR` 须完全独立于 VPC CIDR。例如 VPC 为 `172.16.0.0/12`，则 ClusterCIDR `10.200.0.0/16` 不落在该范围，两者分离在不同的大网段
3. **ServiceCIDR 掩码约束**：掩码须在 **17-27** 之间（即 CIDR 块大小在 /17 ~ /27），过大将无法创建足够的 Service，过小会浪费 IP 空间
4. **多集群不重叠**：同一 VPC 内多个集群的 Pod/Node 子网不可重叠
5. **可用区对齐**（VPC-CNI）：Pod 子网须与节点子网在同一可用区

### ServiceCIDR 掩码与最大 Service 数对照

| 掩码 | CIDR 示例 | 可用 IP 数 | 最大 Service 数 |
|------|----------|-----------|----------------|
| `/27` | `10.201.0.0/27` | 32 | ~32 |
| `/24` | `10.201.0.0/24` | 256 | ~256 |
| `/20` | `10.201.0.0/20` | 4,096 | ~4,096 |
| `/17` | `10.200.0.0/17` | 32,768 | ~32,768 |

> 掩码超出 17-27 范围时，API 返回 `InvalidParameter` 错误。当前集群使用 `/20` 掩码，属于推荐值。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters
#    vpc:DescribeVpcs, vpc:DescribeSubnets
#    验证：执行 DescribeClusters、DescribeVpcs 确认权限
tccli tke DescribeClusters --region ap-guangzhou
tccli vpc DescribeVpcs --region ap-guangzhou
# expected: exit 0，返回集群列表和 VPC 列表（含 vpc-of73262z）
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
| 查看集群三类网段 | `DescribeClusters`（查看 `ClusterCIDR`、`Subnets`、`ServiceCIDR`） | 是 |
| 查看 VPC CIDR | `vpc DescribeVpcs`（查看 `CidrBlock`） | 是 |
| 查看子网 CIDR | `vpc DescribeSubnets`（查看 `CidrBlock`） | 是 |
| 验证 CIDR 不重叠 | 本页提供的 `jq` 命令 | 是 |
| 新建集群时配置网段 | `CreateCluster`（设置 `ClusterCIDRSettings`） | 否 |

### 概念与 API 枚举映射

`ClusterCIDRSettings` 参数（创建集群时使用）与 `ClusterNetworkSettings`（查询返回）：

| 字段 | 类型 | 含义 | 规划约束 |
|------|------|------|---------|
| `ClusterCIDR` | String | Pod IP 网段（GR 模式必填） | 不能与 VPC CIDR / 其他集群 CIDR 重叠 |
| `ServiceCIDR` | String | Service IP 网段 | 掩码 17-27；不能与 ClusterCIDR/Node 子网重叠 |
| `MaxNodePodNum` | Integer | 每节点最大 Pod 数 | 影响节点级 CIDR 子段划分；GR 模式下 `ClusterCIDR` / `MaxNodePodNum` 决定可容纳节点数 |
| `MaxClusterServiceNum` | Integer | 最大 Service 数 | 须与 ServiceCIDR 掩码对应 |
| `IgnoreClusterCIDRConflict` | Boolean | 是否忽略 ClusterCIDR 冲突 | 慎用；仅在确认无实际冲突时设为 `true` |
| `IgnoreServiceCIDRConflict` | Boolean | 是否忽略 ServiceCIDR 冲突 | 慎用；仅在确认无实际冲突时设为 `true` |

## 操作步骤

### 查看 VPC CIDR（基础参照）

```bash
tccli vpc DescribeVpcs --region ap-guangzhou \
    | jq '.VpcSet[] | {VpcId, VpcName, CidrBlock}'
# expected: 返回当前地域所有 VPC 及其 CIDR，含 vpc-of73262z
```

预期输出：

```json
{
    "VpcId": "vpc-of73262z",
    "VpcName": "default-vpc",
    "CidrBlock": "172.16.0.0/12"
}
```

### 查看集群三类网段

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]' \
    | jq '{
        ClusterId: .Clusters[0].ClusterId,
        ClusterCIDR: .Clusters[0].ClusterNetworkSettings.ClusterCIDR,
        ServiceCIDR: .Clusters[0].ClusterNetworkSettings.ServiceCIDR,
        Subnets: .Clusters[0].ClusterNetworkSettings.Subnets,
        VpcId: .Clusters[0].ClusterNetworkSettings.VpcId,
        Cni: .Clusters[0].ClusterNetworkSettings.Cni
    }'
# expected: 返回集群的三类网段和所属 VPC
```

预期输出（`cls-xxxxxxxx` 集群）：

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "ClusterCIDR": "10.200.0.0/16",
    "ServiceCIDR": "10.201.0.0/20",
    "Subnets": null,
    "VpcId": "vpc-of73262z",
    "Cni": false
}
```

> 验证：VPC `172.16.0.0/12`、ClusterCIDR `10.200.0.0/16`、ServiceCIDR `10.201.0.0/20` 三网段互不重叠。

### 列出同 VPC 下所有集群的 CIDR（避免多集群冲突）

```bash
# 获取 VPC ID 后，列出该 VPC 下所有集群的 CIDR
tccli tke DescribeClusters --region ap-guangzhou \
    | jq '[.Clusters[] | select(.ClusterNetworkSettings.VpcId == "vpc-of73262z") | {ClusterId, ClusterCIDR: .ClusterNetworkSettings.ClusterCIDR, ServiceCIDR: .ClusterNetworkSettings.ServiceCIDR}]'
# expected: 返回同 VPC 下所有集群的 CIDR，用于人工校验是否重叠
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

### 查看 VPC-CNI 模式容器子网详情

```bash
# 获取子网 ID 列表后查询子网 CIDR 和可用区
tccli vpc DescribeSubnets --region ap-guangzhou \
    --SubnetIds '["subnet-rdmcho9m"]' \
    | jq '.SubnetSet[] | {SubnetId, CidrBlock, Zone, AvailableIpAddressCount}'
# expected: CidrBlock = "172.24.0.0/20", Zone = "ap-guangzhou-3"
```

### ServiceCIDR 掩码验证

```bash
# 检查 ServiceCIDR 掩码是否在 17-27 范围内
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]' \
    | jq '.Clusters[0].ClusterNetworkSettings.ServiceCIDR' \
    | xargs -I {} sh -c 'echo {} | awk -F/ "{if (\$2>=17 && \$2<=27) print \"PASS: mask \"\$2\" in range [17,27]\"; else print \"FAIL: mask \"\$2\" outside range [17,27]\""}'
# expected: PASS: mask 20 in range [17,27]
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

## 验证

### 验证 CIDR 三层不重叠（人工校验清单）

| 检查项 | 命令 | 通过条件 |
|--------|------|---------|
| VPC CIDR | `vpc DescribeVpcs → CidrBlock` | `172.16.0.0/12` |
| ClusterCIDR vs VPC CIDR | 比对 `ClusterCIDR` 与 `CidrBlock` | `10.200.0.0/16` 不在 `172.16.0.0/12` 范围内 |
| ClusterCIDR vs ServiceCIDR | 比对 `ClusterCIDR` 与 `ServiceCIDR` | `10.200.0.0/16` 与 `10.201.0.0/20` 不重叠 |
| 多集群 ClusterCIDR 互不重叠 | `DescribeClusters` 筛选同 VpcId 的集群 | 各集群 ClusterCIDR 互不重叠 |
| ServiceCIDR 掩码 | 解析 `/xx` 后缀 | `20`，在 17-27 范围内 |
| VPC-CNI Pod 子网与节点同 AZ | `vpc DescribeSubnets` 查看 Zone | 与节点所在子网 Zone 一致 |

### 批量验证脚本

```bash
# 查看所有集群的 CIDR 配置
tccli tke DescribeClusters --region ap-guangzhou \
    | jq '.Clusters[] | {
        id: .ClusterId,
        vpc: .ClusterNetworkSettings.VpcId,
        pod_cidr: .ClusterNetworkSettings.ClusterCIDR,
        svc_cidr: .ClusterNetworkSettings.ServiceCIDR,
        subnets: .ClusterNetworkSettings.Subnets
    }'
# expected: 列出所有集群，人工核查 CIDR 不重叠
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

## 清理

无需清理。本页为概念说明与只读查询，无资源创建。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusters` 权限 | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| `vpc DescribeVpcs` 返回空列表 | `tccli configure list` 检查 region | 当前地域无 VPC | 在正确地域创建 VPC 后重新查询 |
| `vpc DescribeSubnets` 返回 `InvalidParameter.SubnetId` | 核对子网 ID 格式 `subnet-xxxxxxxx` | 子网 ID 格式错误或不属于当前地域 | `tccli vpc DescribeSubnets --region ap-guangzhou` 列出全部子网核对 |
| `CreateCluster` 返回 `InvalidParameter.ClusterCIDRSettings` | 检查 `ClusterCIDR` 是否与 VPC CIDR 或已有集群冲突 | ClusterCIDR 与已有 CIDR 重叠 | 创建前用 `DescribeClusters` + `DescribeVpcs` 检查所有现有 CIDR；选取完全独立的大网段 |
| `CreateCluster` 返回 `InvalidParameter.CidrMaskSizeOutOfRange` | 检查 `ServiceCIDR` 掩码 | ServiceCIDR 掩码写成 `/16`，不在 17-27 范围 | 掩码必须 17-27，推荐 `/20`（当前集群 `10.201.0.0/20`） |
| 新集群创建后 Pod/Service 网络不通 | `DescribeClusters` 列出当前所有集群的 CIDR | 新集群 CIDR 与其他集群或 VPC 内路由冲突 | 删除新集群，重新规划不重叠的 CIDR 后重建 |
| VPC-CNI Pod 无法调度到某可用区 | `vpc DescribeSubnets --SubnetIds '["SUBNET_ID"]'` 查看 Zone | Pod 子网与节点子网不在同一可用区 | 为该可用区新增 Pod 子网，或调整节点分布 |
| VPC-CNI 子网 IP 耗尽 | `vpc DescribeSubnets → AvailableIpAddressCount` 为 0 或过低 | Pod 子网 IP 地址池已耗尽 | 添加新的容器子网（`AddVpcCniSubnets`）或扩大现有子网 |

## 下一步

- [容器网络概述](../容器网络概述/tccli%20操作.md) -- 三种方案原理与对比
- [容器集群网络方案选型](../容器集群网络方案选型/tccli%20操作.md) -- 决策流程与方案对比矩阵
- [GlobalRouter 模式介绍](../GlobalRouter%20模式/GlobalRouter%20模式介绍/tccli%20操作.md) -- GR 模式 CIDR 规划详解
- [VPC-CNI 模式介绍](../VPC-CNI%20模式/VPC-CNI%20模式介绍/tccli%20操作.md) -- VPC-CNI 子网与 AZ 规划
- [创建集群](../../集群管理/创建集群/tccli%20操作.md) -- 使用规划好的 CIDR 创建集群

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster)：创建集群时「集群信息」步骤填写 ClusterCIDR、ServiceCIDR，并在「选择网络模式」步骤配置 Pod 子网。存量集群基本信息页展示当前网段。
