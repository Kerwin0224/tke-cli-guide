# 容器集群网络方案选型（tccli）

> 对照官方：[容器集群网络方案选型](https://cloud.tencent.com/document/product/457/106561) · page_id `106561`

## 概述

TKE 提供三种网络方案：**Global Router（GR）**、**VPC-CNI** 和 **Cilium-Overlay**。创建集群时必须通过 `ClusterCIDRSettings` 或子网配置选择其一，该选择不可逆（集群创建后无法切换网络方案类型）。本文提供 tccli 层面的方案选择决策依据，涵盖 `NetworkType` 参数选择决策树以及各方案的 API 特征对照。

参考集群 `cls-xxxxxxxx` 选择 **GR 模式**：GR 是 TKE 默认网络模式，容器 IP 通过 VPC 路由表转发，与 VPC 原生网络互通，适合大多数场景。VPC-CNI 提供 Pod 直通 VPC 能力但需要额外子网。Cilium-Overlay 提供 eBPF 数据面而无需 CRD 子网。

### NetworkType 参数选择决策树

选择网络方案时，核心决策链路如下：

```
是否需要 Pod 安全组 / 固定 Pod IP / EIP 绑定 / IPv4/IPv6 双栈？
├── 是 → 选择 VPC-CNI（需 EnableVpcCniNetworkType 或 CreateCluster 时指定 Subnets）
└── 否 → 是否为注册节点（混合云 / 分布式云）场景？
    ├── 是 → 选择 Cilium-Overlay（NetworkType = CiliumBGP）
    └── 否 → 是否为简单固定规模、无特殊 IP 需求？
        ├── 是 → 选择 Global Router（NetworkType = GR，设置 ClusterCIDR）← cls-xxxxxxxx
        └── 否 → 选择 VPC-CNI（推荐，纯云上节点最佳实践）
```

### NetworkType 与 API 对照

| NetworkType | API 参数写法 | 创建集群时的关键参数 |
|-------------|-------------|---------------------|
| `GR` | `"GR"` | `ClusterCIDR` 非空 + `Subnets` 为空或 null |
| `VPC-CNI` | `"VPC-CNI"` | `Subnets` 非空（需先 EnableVpcCniNetworkType 或在创建时传入） |
| `CiliumBGP` | `"CiliumBGP"` | `CiliumMode: "CiliumBGP"`＋非 VPC 子网 CIDR |

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
| 查看集群网络方案类型 | `DescribeClusters`（查看 `Property.NetworkType`） | 是 |
| 查看 GR 集群 | `DescribeClusters`（筛选 `Cni == false`） | 是 |
| 查看 VPC-CNI 集群 | `DescribeClusters`（筛选 `Cni == true`） | 是 |
| 查看 Cilium-Overlay 集群 | `DescribeClusters`（筛选 `CiliumMode == "CiliumBGP"`） | 是 |
| 新建 GR 集群 | `CreateCluster`（设置 `ClusterCIDR`，不设 `Subnets`） | 否 |
| 新建 VPC-CNI 集群 | `CreateCluster`（需 `EnableVpcCniNetworkType` 后指定 `Subnets`） | 否 |
| 为已有集群开启 VPC-CNI | `EnableVpcCniNetworkType`（`VpcCniType` + `Subnets`） | 否 |

### 概念与 API 枚举映射

三种方案在 `DescribeClusters` 返回结构中的特征字段对照：

| 维度 | GR（`NetworkType = "GR"`） | VPC-CNI（`NetworkType = "VPC-CNI"`） | Cilium-Overlay（`NetworkType = "CiliumBGP"`） |
|------|---------------------------|--------------------------------------|----------------------------------------------|
| `Cni` | `false` | `true` | 视配置 |
| `ClusterCIDR` | 必填（独立于 VPC） | 可选 | 必填 |
| `Subnets` | `null` 或空 | 非空（Pod 子网列表） | 非空（节点子网） |
| `CiliumMode` | `""` | `""` | `"CiliumBGP"` |
| `EnableVpcCniNetworkType` 调用 | 不需要 | 需要（预先调用） | 不需要 |
| 方案互斥 | 不能同时为 VPC-CNI | 不能同时为 GR | 独立方案 |

## 操作步骤

### 按 NetworkType 列出集群

```bash
# 列出所有 GR 集群
tccli tke DescribeClusters --region ap-guangzhou \
    | jq '.Clusters[] | select(.ClusterNetworkSettings.Cni == false) | {ClusterId, ClusterName, NetworkType: "GR"}'
# expected: 返回所有 Cni=false 的集群，包含 cls-xxxxxxxx

# 列出所有 VPC-CNI 集群
tccli tke DescribeClusters --region ap-guangzhou \
    | jq '.Clusters[] | select(.ClusterNetworkSettings.Cni == true) | {ClusterId, ClusterName, NetworkType: "VPC-CNI", Subnets: .ClusterNetworkSettings.Subnets}'
# expected: 返回所有 Cni=true 的集群及其容器子网

# 列出所有 Cilium-Overlay 集群
tccli tke DescribeClusters --region ap-guangzhou \
    | jq '.Clusters[] | select(.ClusterNetworkSettings.CiliumMode == "CiliumBGP") | {ClusterId, ClusterName, NetworkType: "CiliumBGP"}'
# expected: 返回所有 CiliumMode=CiliumBGP 的集群
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

### 查看单个集群的 NetworkType 及方案特征

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]' \
    | jq '{
        ClusterId: .Clusters[0].ClusterId,
        NetworkType: (.Clusters[0].Property | fromjson | .NetworkType),
        Cni: .Clusters[0].ClusterNetworkSettings.Cni,
        ClusterCIDR: .Clusters[0].ClusterNetworkSettings.ClusterCIDR,
        ServiceCIDR: .Clusters[0].ClusterNetworkSettings.ServiceCIDR,
        Subnets: .Clusters[0].ClusterNetworkSettings.Subnets,
        CiliumMode: .Clusters[0].ClusterNetworkSettings.CiliumMode,
        DataPlaneV2: .Clusters[0].ClusterNetworkSettings.DataPlaneV2
    }'
# expected: NetworkType = "GR"，Cni = false
```

预期输出（`cls-xxxxxxxx` 集群）：

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "NetworkType": "GR",
    "Cni": false,
    "ClusterCIDR": "10.200.0.0/16",
    "ServiceCIDR": "10.201.0.0/20",
    "Subnets": null,
    "CiliumMode": "",
    "DataPlaneV2": false
}
```

### VPC-CNI 的 EnableVpcCniNetworkType 调用（概念说明）

VPC-CNI 方案在创建集群时，需先调用 `EnableVpcCniNetworkType` 开启 VPC-CNI 网络类型，再在 `CreateCluster` 中指定容器子网。该 API 的参数选择直接决定 VPC-CNI 子模式：

| 参数 | 取值 | 子模式 |
|------|------|--------|
| `VpcCniType = "tke-route-eni"` | 多 Pod 共享网卡 | 共享网卡模式（推荐） |
| `VpcCniType = "tke-direct-eni"` | Pod 独占网卡 | 独占网卡模式（需提交工单开通） |
| `EnableStaticIp = true` | 启用固定 Pod IP | 仅共享网卡模式支持 |

```bash
# 为已有集群开启 VPC-CNI
tccli tke EnableVpcCniNetworkType --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --VpcCniType tke-route-eni \
    --Subnets '["subnet-xxxxxxxx"]'
# expected: exit 0, VPC-CNI 组件开始部署
```

> 注意：更改网络模式可能影响已在运行的 Pod 网络。GR 切换至 VPC-CNI 是不可逆操作，请谨慎评估。

### 不同方案的子网规划差异

| 方案 | Pod 子网要求 | 与 VPC CIDR 关系 |
|------|------------|-----------------|
| GR | 不需要 Pod 子网（ClusterCIDR 独立） | ClusterCIDR 不与 VPC CIDR 或其他集群 CIDR 重叠即可。当前集群 VPC `172.16.0.0/12`，ClusterCIDR `10.200.0.0/16`，两者不重叠 |
| VPC-CNI | 必须指定 Pod 子网（Subnets） | Pod 子网属于 VPC CIDR 范围内，须与节点子网分离 |
| Cilium-Overlay | 需要节点子网 | 容器 CIDR 独立于 VPC |

## 验证

### 验证 NetworkType 判决结果

```bash
# 对照决策树验证集群方案是否正确
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]' \
    | jq '{
        ClusterId: .Clusters[0].ClusterId,
        NetworkType: (.Clusters[0].Property | fromjson | .NetworkType),
        Cni: .Clusters[0].ClusterNetworkSettings.Cni,
        ClusterCIDR: .Clusters[0].ClusterNetworkSettings.ClusterCIDR,
        Subnets: .Clusters[0].ClusterNetworkSettings.Subnets,
        CiliumMode: .Clusters[0].ClusterNetworkSettings.CiliumMode
    }'
# expected: NetworkType = "GR"，Cni = false，Subnets = null，CiliumMode = ""
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

### 方案特征验证对照

| 验证维度 | GR 预期 | VPC-CNI 预期 | Cilium-Overlay 预期 |
|---------|--------|-------------|-------------------|
| `Property.NetworkType` | `"GR"` | `"VPC-CNI"` | `"CiliumBGP"` |
| `Cni` | `false` | `true` | `false`（通常） |
| `ClusterCIDR` | 非空 | 可能为空或非空 | 非空 |
| `Subnets` | `null` | 非空 | 非空 |
| `CiliumMode` | `""` | `""` | `"CiliumBGP"` |

## 清理

无需清理。本页为概念说明与只读查询，无资源创建。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusters` 权限 | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| 集群 ID 不存在 | `tccli tke DescribeClusters --region ap-guangzhou` 列出全部集群 | 集群不属于当前地域或账号 | 确认 `tccli configure list` 中的 `region` 与集群所在地域一致 |
| `EnableVpcCniNetworkType` 返回 `FailedOperation` | 检查集群当前网络状态 | 集群可能已启用 VPC-CNI 或正处于其他操作中 | 确认 `Cni: false` 且集群状态为 Running 后再调用 |
| 不确定集群网络方案是否合理 | 用 `jq` 提取 `Property.NetworkType`，对照本页决策树 | 方案选择可能不符合最佳实践 | 纯云上节点推荐 VPC-CNI；注册节点用 Cilium-Overlay；GR 适用于简单场景 |
| 集群创建后发现网络方案不对 | `DescribeClusters` 查看 `NetworkType` | 集群创建后 NetworkType 不可更改 | 需新建集群并迁移工作负载。创建前务必对照本页决策树验证 |
| `CreateCluster` 返回 `InvalidParameter.ClusterCIDRSettings` | 检查 `ClusterCIDR` 与现有 VPC CIDR 及集群 CIDR | CIDR 网段重叠 | 创建前用 `DescribeClusters` + `DescribeVpcs` 检查所有现有 CIDR；选取完全独立的大网段 |

## 下一步

- [容器网络概述](../容器网络概述/tccli%20操作.md) -- 三种方案原理与能力对比
- [容器集群网络规划](../容器集群网络规划/tccli%20操作.md) -- CIDR 三层不重叠规划
- [GlobalRouter 模式介绍](../GlobalRouter%20模式/GlobalRouter%20模式介绍/tccli%20操作.md) -- GR 模式原理详解
- [VPC-CNI 模式介绍](../VPC-CNI%20模式/VPC-CNI%20模式介绍/tccli%20操作.md) -- VPC-CNI 原理与子模式
- [Cilium-Overlay 模式介绍](../Cilium-Overlay%20模式/Cilium-Overlay%20模式介绍/tccli%20操作.md) -- Cilium-Overlay 原理与限制

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster)：创建集群时「选择网络模式」步骤提供 GR / VPC-CNI / Cilium-Overlay 三种选项。存量集群基本信息页「集群网络」字段显示当前方案。
