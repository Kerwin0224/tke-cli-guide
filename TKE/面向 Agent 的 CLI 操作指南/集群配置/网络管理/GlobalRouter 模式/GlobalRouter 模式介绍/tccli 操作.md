# GlobalRouter 模式介绍（tccli）

> 对照官方：[GlobalRouter 模式介绍](https://cloud.tencent.com/document/product/457/50354) · page_id `50354`

## 概述

GlobalRouter（GR）模式基于腾讯云 VPC 的全局路由能力实现容器网络。Pod CIDR 独立于 VPC CIDR，不占用 VPC 地址空间。每个节点从集群 Pod CIDR 中预分配一个子段，跨节点 Pod 通信通过注入 VPC 的全局路由转发，无需隧道封装。

核心机制：
- 每个节点按 `ClusterCIDR / MaxNodePodNum` 计算得到固定大小的 Pod CIDR 子段
- Service CIDR 取自集群 Pod CIDR 的最后一段
- 节点释放时，其 Pod CIDR 子段归还 IP 池
- 容器路由优先于 VPC 路由（当两者重叠时）

参考集群 `cls-xxxxxxxx`（kerwinwjyan-rewrite-s1）：ap-guangzhou，K8s 1.30.0，MANAGED_CLUSTER，GR 模式。VPC `vpc-of73262z`（`172.16.0.0/12`），ClusterCIDR `10.200.0.0/16`，ServiceCIDR `10.201.0.0/20`，Ipvs `false`，DataPlaneV2 `false`。`DescribeClusterRouteTables` 返回 `TotalCount=7`（跨集群的全局路由表）。

### GR 与 VPC-CNI / Cilium-Overlay 的对比

| 维度 | GlobalRouter | VPC-CNI | Cilium-Overlay |
|------|-------------|---------|----------------|
| `Cni` 字段 | `false` | `true` | `false` |
| `Property.NetworkType` | `"GR"` | `"VPC-CNI"` | `"CiliumOverlay"` |
| Pod IP 来源 | 独立 ClusterCIDR | VPC 子网 ENI IP | 独立 Pod CIDR |
| 固定 Pod IP | 不支持 | 支持（共享网卡模式） | 不支持 |
| IPVS 选项 | 可通过 `Ipvs` 字段启用 | 不适用（已用 VPC-CNI 转发） | 不适用（Cilium 自带 eBPF） |
| 路由可见性 | `DescribeClusterRouteTables` 可查路由表 | 不产生全局路由条目 | 不产生全局路由条目 |
| 网络策略 | 需额外组件 | 需额外组件 | 原生支持（Cilium NetworkPolicy） |
| 数据面 | iptables / IPVS | iptables / IPVS | eBPF |
| 适用场景 | 通用场景，需独立 Pod CIDR | 需 Pod IP 直通 VPC 或固定 IP | 需 eBPF 转发或原生 NetworkPolicy |

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
#    tke:DescribeClusters, tke:DescribeClusterRouteTables
#    验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表（含 cls-xxxxxxxx）
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
| 查看 GR 集群网络配置 | `DescribeClusters`（`Cni: false` 的集群） | 是 |
| 查看集群路由表 | `DescribeClusterRouteTables` | 是 |
| 查看集群路由 | `DescribeClusterRoutes` | 是 |
| 新建 GR 集群 | `CreateCluster`（不设 Subnets、设 ClusterCIDR、不启用 VPC-CNI） | 否 |

### 控制台概念与 API 枚举映射

| 控制台概念 | API 枚举（`Property.NetworkType`） | `Cni` 字段 | `CiliumMode` 字段 |
|-----------|----------------------------------|-----------|------------------|
| GR 模式 | `"GR"` | `false` | 空 |
| VPC-CNI 模式 | `"VPC-CNI"` | `true` | 空 |
| Cilium-Overlay 模式 | `"CiliumOverlay"` | `false` | `"tke-route-eni"` 或 `"tke-direct-eni"` |

## 关键字段说明

`DescribeClusters` 中 GR 模式的核心字段：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `Cni` | Boolean | 否（输出） | GR 模式必须为 `false` | — |
| `ClusterCIDR` | String | 是（创建时） | Pod IP 网段，独立于 VPC CIDR。不能与 VPC CIDR 或其他集群 CIDR 重叠 | CIDR 冲突 → `InvalidParameter.ClusterCIDRSettings` |
| `ServiceCIDR` | String | 是（创建时） | Service IP 网段，掩码 17-27。取自 ClusterCIDR 最后一段 | 掩码越界 → `InvalidParameter.CidrMaskSizeOutOfRange` |
| `MaxNodePodNum` | Integer | 否 | 每节点最大 Pod 数，决定每个节点预分配的 CIDR 子段大小 | 过大 → ClusterCIDR 内子段数不足 |
| `MaxClusterServiceNum` | Integer | 否 | 最大 Service 数，须与 ServiceCIDR 掩码对应 | 不匹配 → `InvalidParameter.MaxClusterServiceNum` |
| `Ipvs` | Boolean | 否 | `true` 启用 IPVS 替代 iptables。创建后可读不可改 | 忘设 → 只能用 iptables（不可事后修改） |
| `Property.NetworkType` | String（JSON 内） | 仅输出 | `"GR"` 表示 GR 模式 | — |
| `DataPlaneV2` | Boolean | 否 | `true` 用 eBPF 替代 kube-proxy。与 Ipvs 互斥 | Dataplane V2 与 IPVS 同时生效 → 按 Dataplane V2 为准 |

## 操作步骤

### 查看存量 GR 集群

选择依据：GR 集群的 `Cni` 为 `false`，`Property.NetworkType` 为 `"GR"`。通过过滤 `Cni` 字段可快速找出所有 GR 集群。

```bash
# 列出所有 Cni=false 的 GR 集群
tccli tke DescribeClusters --region ap-guangzhou \
    | jq '.Clusters[] | select(.ClusterNetworkSettings.Cni == false) | {ClusterId, ClusterName, ClusterCIDR: .ClusterNetworkSettings.ClusterCIDR, ServiceCIDR: .ClusterNetworkSettings.ServiceCIDR, Ipvs: .ClusterNetworkSettings.Ipvs}'
# expected: 返回所有 GR 集群的 CIDR 配置，含 cls-xxxxxxxx
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

### 查看单个 GR 集群完整网络配置

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]'
# expected: Cni=false，ClusterCIDR 非空，Subnets 为空或 null
```

预期输出（`cls-xxxxxxxx` 集群）：

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "ClusterNetworkSettings": {
        "ClusterCIDR": "10.200.0.0/16",
        "IgnoreClusterCIDRConflict": false,
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 4096,
        "Ipvs": false,
        "VpcId": "vpc-of73262z",
        "Cni": false,
        "KubeProxyMode": "",
        "ServiceCIDR": "10.201.0.0/20",
        "Subnets": null,
        "IgnoreServiceCIDRConflict": false,
        "IsDualStack": false,
        "Ipv6ServiceCIDR": "",
        "CiliumMode": "",
        "SubnetId": "",
        "DataPlaneV2": false
    },
    "Property": "{\"NetworkType\":\"GR\"}"
}
```

### 查看 GR 集群路由表

```bash
tccli tke DescribeClusterRouteTables --region ap-guangzhou
# expected: 返回 RouteTableSet，TotalCount 为存量 GR 集群数
```

预期输出：

```json
{
    "TotalCount": 7,
    "RouteTableSet": [
        {
            "RouteTableName": "cls-xxxxxxxx",
            "RouteTableCidrBlock": "10.200.0.0/16",
            "VpcId": "vpc-of73262z"
        }
    ]
}
```

> `TotalCount=7` 表示当前账号下共有 7 个 GR 集群的全局路由表。`RouteTableName` 对应集群 ID，`RouteTableCidrBlock` 为该集群的 Pod CIDR，`VpcId` 为所属 VPC。每个 GR 集群对应一条路由表。

### 查看指定路由表的路由条目

```bash
# 获取 RouteTableName 后查询具体路由
tccli tke DescribeClusterRoutes --region ap-guangzhou \
    --RouteTableName "cls-xxxxxxxx"
# expected: 返回路由条目列表，每条含网关 IP 和目的 CIDR
```

预期输出（新集群无路由条目，或含已有节点路由）：

```json
{
    "TotalCount": 0,
    "RouteSet": []
}
```

### 查看 IPVS 状态

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]' \
    | jq '.Clusters[0].ClusterNetworkSettings.Ipvs'
# expected: false（默认 iptables）
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

### 验证 GR 模式关键特征

```bash
# 确认 Cni=false、ClusterCIDR 非空、NetworkType=GR
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]' \
    | jq '{
        ClusterId: .Clusters[0].ClusterId,
        Cni: .Clusters[0].ClusterNetworkSettings.Cni,
        ClusterCIDR: .Clusters[0].ClusterNetworkSettings.ClusterCIDR,
        ServiceCIDR: .Clusters[0].ClusterNetworkSettings.ServiceCIDR,
        Ipvs: .Clusters[0].ClusterNetworkSettings.Ipvs,
        NetworkType: (.Clusters[0].Property | fromjson | .NetworkType),
        Subnets: .Clusters[0].ClusterNetworkSettings.Subnets
    }'
# expected: Cni=false, ClusterCIDR=10.200.0.0/16, Subnets=null, NetworkType="GR"
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

### GR 模式验证维度总览

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| `Cni` | `DescribeClusters → .Cni` | `false` |
| `ClusterCIDR` | `DescribeClusters → .ClusterCIDR` | 非空（`10.200.0.0/16`） |
| `Subnets` | `DescribeClusters → .Subnets` | `null` 或空 |
| `NetworkType` | `Property.fromjson.NetworkType` | `"GR"` |
| 路由表 | `DescribeClusterRouteTables` | 存在对应 `RouteTableName = "cls-xxxxxxxx"` |
| 非 VPC-CNI | `DescribeEnableVpcCniProgress` | 返回 `not vpc-cni cluster` |

### 确认集群非 VPC-CNI 模式

```bash
tccli tke DescribeEnableVpcCniProgress --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx
# expected: 返回错误 "cluster cls-xxxxxxxx is not vpc-cni cluster"
```

```json
{
  "Status": "<Status>",
  "ErrorMessage": "<ErrorMessage>",
  "RequestId": "<RequestId>"
}
```

## 清理

无需清理。本页为概念说明与只读查询，无资源创建。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusters` 权限 | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| `DescribeClusterRouteTables` 返回空 | `tccli tke DescribeClusterRouteTables --region ap-guangzhou` | 该地域无 GR 集群（VPC-CNI / Cilium-Overlay 集群不产生路由表） | 检查集群是否为 GR 模式：`jq '.Clusters[].ClusterNetworkSettings.Cni'` 须为 `false` |
| `DescribeClusterRoutes` 返回 `InvalidParameter` | 核对 `--RouteTableName` 参数值 | 路由表名称不存在或拼写错误 | 从 `DescribeClusterRouteTables` 获取准确的 `RouteTableName` |
| `CreateClusterRouteTable` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 确认当前凭据 | 账号缺少路由表写入权限 | `CreateClusterRouteTable` 需特定的 CAM 策略授权；检查账号是否绑定了所需标签 |
| GR 集群 Pod 无法访问外部服务 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` 确认 ClusterCIDR | Pod CIDR 与外部目标 CIDR 路由冲突（容器路由优先于 VPC 路由） | 检查 VPC 路由表和容器路由是否重叠；保留 RequestId 以便排查 |
| 新节点加入后 Pod 无法获取 IP | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' \| jq '.ClusterNetworkSettings.{ClusterCIDR, MaxNodePodNum}'` | ClusterCIDR 内可用子段已耗尽 | 增大 ClusterCIDR 或减少 MaxNodePodNum；保留 RequestId 以便排查 |
| `DescribeRouteTableConflicts` 报告 HasConflict 为 true | `tccli tke DescribeRouteTableConflicts --region ap-guangzhou --RouteTableCidrBlock "10.200.0.0/16" --VpcId "vpc-of73262z"` | 目标 CIDR 与已有路由表冲突 | 更换为非冲突的 CIDR；确认 `HasConflict` 为 `false` 后再创建 |
| `CreateCluster` 返回 `InvalidParameter.ClusterCIDRSettings` | 检查 `ClusterCIDR` 与 VPC CIDR（`172.16.0.0/12`） | CIDR 重叠 | 创建前用 `DescribeClusters` + `DescribeVpcs` 检查所有现有 CIDR |
| `CreateCluster` 返回 `InvalidParameter.CidrMaskSizeOutOfRange` | 检查 `ServiceCIDR` 掩码 | 掩码不在 17-27 范围 | 掩码必须 17-27，推荐 `/20` |

## 下一步

- [容器网络概述](../../容器网络概述/tccli%20操作.md) -- 三种方案原理与对比
- [容器集群网络方案选型](../../容器集群网络方案选型/tccli%20操作.md) -- 选型决策
- [容器集群网络规划](../../容器集群网络规划/tccli%20操作.md) -- CIDR 三层不重叠规划
- [同地域及跨地域 GR 集群互通](../同地域及跨地域%20GlobalRouter%20模式集群间互通/tccli%20操作.md)
- [VPC-CNI 模式介绍](../../VPC-CNI%20模式/VPC-CNI%20模式介绍/tccli%20操作.md) -- VPC-CNI 原理详解

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster)：GR 集群 `cls-xxxxxxxx` 基本信息页「集群网络」字段展示 ClusterCIDR `10.200.0.0/16` 和 ServiceCIDR `10.201.0.0/20`。
