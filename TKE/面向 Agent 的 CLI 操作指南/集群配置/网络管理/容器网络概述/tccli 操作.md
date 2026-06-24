# 容器网络概述（tccli）

> 对照官方：[容器网络概述](https://cloud.tencent.com/document/product/457/50353) · page_id `50353`

## 概述

TKE 提供三种 CNI 网络插件方案：**Global Router（GR）**、**VPC-CNI** 和 **Cilium-Overlay**。三者均满足「每个 Pod 拥有独立 IP、跨节点 Pod 无需 NAT 即可互通」的基线约束，但在 IP 来源、网络平面、性能特性和适用场景上存在显著差异。

参考集群 `cls-xxxxxxxx`（kerwinwjyan-rewrite-s1）当前使用 **GR 模式**：VPC `vpc-of73262z`（`172.16.0.0/12`），ClusterCIDR `10.200.0.0/16`，ServiceCIDR `10.201.0.0/20`，Ipvs `false`，DataPlaneV2 `false`。GR 是 TKE 默认网络模式，容器 IP 通过 VPC 路由表转发，与 VPC 原生网络互通，适合大多数场景。

### 方案对比表

| 维度 | Global Router | VPC-CNI | Cilium-Overlay |
|------|--------------|---------|----------------|
| IP 来源 | 独立于 VPC 的 Pod CIDR（预分配至节点） | VPC 子网弹性网卡 IP | 独立于 VPC 的容器 CIDR（Overlay） |
| 网络平面 | 独立平面，路由注入 VPC | 与 VPC 同一平面 | Overlay（VxLAN）叠加于底层网络 |
| 隧道封装 | 无 | 无 | Cilium VxLAN |
| 性能 | 良好（无隧道开销） | 最优（比网桥方案提升约 10%） | 约 10% 封装性能损耗 |
| 固定 Pod IP | 不支持 | 支持（共享网卡模式） | 不支持 |
| IPv4/IPv6 双栈 | 不支持 | 支持 | 不支持 |
| Pod 安全组 | 不支持 | 支持 | 不支持 |
| Pod 绑定 EIP | 不支持 | 支持 | 不支持 |
| CLB 直通 Pod | 支持（需白名单） | 支持 | 不支持 |
| Pod IP 集群外可达 | 需额外配置（专线/对等/云联网） | 是 | 否 |
| 适用场景 | 简单固定规模、无特殊 IP 需求 | 时延敏感、传统架构迁移、安全策略 | 注册节点（混合云/分布式云） |
| 纯云上节点 | 支持 | 支持（推荐） | 不支持（仅注册节点场景） |

### NetworkType 枚举对照

`DescribeClusters` 返回的 `Property` 字段中的 `NetworkType` 值与方案对应关系：

| NetworkType 值 | 对应方案 |
|----------------|---------|
| `GR` | Global Router 模式（当前集群） |
| `VPC-CNI` | VPC-CNI 模式 |
| `CiliumBGP` | Cilium-Overlay 模式 |

**建议：公有云场景推荐 VPC-CNI；注册节点场景使用 Cilium-Overlay；简单固定规模场景可用 GR。**

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
| 查看集群网络类型 | `DescribeClusters`（查看 `Property.NetworkType`） | 是 |
| 查看集群网络配置 | `DescribeClusters`（查看 `ClusterNetworkSettings`） | 是 |
| 新建集群时选择网络方案 | `CreateCluster`（通过 `ClusterCIDRSettings` 与子网配置决定） | 否 |

### 概念与 API 枚举映射

`ClusterNetworkSettings` 中与网络方案相关的核心字段（以 `cls-xxxxxxxx` 为例）：

| 字段 | 类型 | 含义 | 当前集群值 |
|------|------|------|---------|
| `Cni` | Boolean | 是否启用 VPC-CNI | `false`（GR 模式） |
| `ClusterCIDR` | String | Pod IP 网段 | `10.200.0.0/16` |
| `ServiceCIDR` | String | Service IP 网段 | `10.201.0.0/20` |
| `VpcId` | String | 所属 VPC | `vpc-of73262z` |
| `Ipvs` | Boolean | 是否启用 IPVS | `false` |
| `DataPlaneV2` | Boolean | 是否启用 Dataplane V2 | `false` |
| `Subnets` | Array | VPC-CNI 容器子网列表 | `null`（GR 模式） |
| `CiliumMode` | String | Cilium 模式 | `""` |

`Property` 字段（JSON 字符串）中的 `NetworkType` 枚举：

| NetworkType | 含义 | 对应 `DescribeClusters` 特征 |
|-------------|------|------------------------------|
| `GR` | Global Router | `Cni: false`，`ClusterCIDR` 非空 |
| `VPC-CNI` | VPC-CNI | `Cni: true`，`Subnets` 非空 |
| `CiliumBGP` | Cilium-Overlay | `CiliumMode: "CiliumBGP"` |

## 操作步骤

### 查看集群当前网络方案

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]'
# expected: exit 0，返回集群完整网络配置
```

预期输出（GR 集群）：

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "ClusterNetworkSettings": {
        "ClusterCIDR": "10.200.0.0/16",
        "VpcId": "vpc-of73262z",
        "Cni": false,
        "Ipvs": false,
        "ServiceCIDR": "10.201.0.0/20",
        "Subnets": null,
        "DataPlaneV2": false,
        "CiliumMode": ""
    },
    "Property": "{\"NetworkType\":\"GR\"}"
}
```

### 获取 NetworkType 枚举值

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]' \
    | jq -r '.Clusters[0].Property' \
    | jq -r '.NetworkType'
# expected: "GR"
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

### 列出所有集群的网络方案

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    | jq -r '.Clusters[] | "\(.ClusterId)\t\(.ClusterNetworkSettings.Cni)\t\(.ClusterNetworkSettings.DataPlaneV2)"'
# expected: 集群列表，含 Cni 和 DataPlaneV2 布尔值
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

### 验证集群网络类型归属

```bash
# 查看完整网络配置，确认 NetworkType
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]' \
    | jq '{ClusterId: .Clusters[0].ClusterId, NetworkType: (.Clusters[0].Property | fromjson | .NetworkType), Cni: .Clusters[0].ClusterNetworkSettings.Cni, DataPlaneV2: .Clusters[0].ClusterNetworkSettings.DataPlaneV2, CiliumMode: .Clusters[0].ClusterNetworkSettings.CiliumMode}'
# expected: NetworkType = "GR", Cni = false, DataPlaneV2 = false, CiliumMode = ""
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

### 验证不同方案的字段特征

| 验证维度 | GR 模式预期 | VPC-CNI 模式预期 | Cilium-Overlay 预期 |
|---------|-----------|----------------|-------------------|
| `Cni` | `false` | `true` | 视配置 |
| `ClusterCIDR` | 非空（如 `10.200.0.0/16`） | 可能为空或非空 | 非空 |
| `Subnets` | `null` 或空 | 非空 | 非空 |
| `CiliumMode` | `""` | `""` | `"CiliumBGP"` |
| `Property.NetworkType` | `"GR"` | `"VPC-CNI"` | `"CiliumBGP"` |

## 清理

无需清理。本页为概念说明与只读查询，无资源创建。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusters` 权限 | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| `jq` 解析 `Property` 字段报错 | `echo '<Property 值>' \| jq .` 检查 JSON 格式 | `Property` 为 JSON 字符串，需 `fromjson` 二次解析 | 使用 `jq '.Clusters[0].Property \| fromjson \| .NetworkType'` 而非直接 `.Property.NetworkType` |
| 集群 ID 不存在 | `tccli tke DescribeClusters --region ap-guangzhou` 列出全部集群 | 集群不属于当前地域或账号 | 确认 `tccli configure list` 中的 `region` 与集群所在地域一致 |
| CIDR 重叠导致创建集群失败 | `DescribeClusters` + `DescribeVpcs` 检查现有 CIDR | ClusterCIDR 与已有 VPC CIDR 或其他集群 CIDR 重叠 | 创建前检查所有现有 CIDR；选取不重叠的新网段 |
| `InvalidParameter.ClusterCIDRSettings` | 检查 `ClusterCIDR` 与 VPC CIDR、其他集群 CIDR | CIDR 网段冲突 | 选择完全独立的大网段（如当前集群 VPC `172.16.0.0/12` 与 ClusterCIDR `10.200.0.0/16` 不重叠） |
| `InvalidParameter.CidrMaskSizeOutOfRange` | 检查 `ServiceCIDR` 掩码 | ServiceCIDR 掩码写成 `/16`，不在 17-27 范围 | 掩码必须 17-27，推荐 `/20`（当前集群使用 `10.201.0.0/20`） |

## 下一步

- [容器集群网络方案选型](../容器集群网络方案选型/tccli%20操作.md) -- 决策流程与方案对比矩阵
- [容器集群网络规划](../容器集群网络规划/tccli%20操作.md) -- CIDR 规划与冲突检测
- [GlobalRouter 模式介绍](../GlobalRouter%20模式/GlobalRouter%20模式介绍/tccli%20操作.md) -- GR 模式原理与配置
- [VPC-CNI 模式介绍](../VPC-CNI%20模式/VPC-CNI%20模式介绍/tccli%20操作.md) -- VPC-CNI 原理与子模式
- [Cilium-Overlay 模式介绍](../Cilium-Overlay%20模式/Cilium-Overlay%20模式介绍/tccli%20操作.md) -- Cilium-Overlay 原理与限制

## 控制台替代

[TKE 控制台 → 集群列表](https://console.cloud.tencent.com/tke2/cluster)：点击集群 ID `cls-xxxxxxxx` → 基本信息页「集群网络」字段展示网络方案 GR。
