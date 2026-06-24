# Cilium-Overlay 模式介绍（tccli）

> 对照官方：[Cilium-Overlay 模式介绍](https://cloud.tencent.com/document/product/457/77964) · page_id `77964`

## 概述

Cilium-Overlay 是 TKE 基于 Cilium VxLAN 实现的容器网络插件，专为**注册节点（第三方 IDC 节点加入 TKE 集群）**场景设计。云上节点和第三方节点共享一个容器 CIDR，通过 Cilium VxLAN 隧道协议构建 Overlay 网络，实现混合云统一容器网络。

> **核心限制**：此模式**仅支持分布式云中注册节点场景**，不支持纯云上节点集群。VxLAN 封装带来约 10% 性能损耗。纯云上场景推荐 VPC-CNI（时延敏感）或 GR（简单场景）。

### Cilium-Overlay vs GR vs VPC-CNI

| 维度 | Cilium-Overlay | Global Router | VPC-CNI |
|------|:--:|:--:|:--:|
| 适用场景 | 注册节点（混合云） | 简单固定规模 | 时延敏感、迁移 |
| 云上节点支持 | 不支持（仅注册节点） | 支持 | 推荐 |
| 封装方式 | Cilium VxLAN | 无 | 无 |
| 性能损耗 | ~10% | 无 | 无 |
| Pod IP 外部可达 | 否（需 Service/LB 暴露） | 需额外配置 | 是（VPC 内直接路由） |
| 固定 Pod IP | 不支持 | 不支持 | 支持 |
| CIDR 扩展 | 不支持（创建后固定） | 支持 | 支持 |
| 超级节点兼容 | 不兼容 | -- | 兼容 |
| NetworkPolicy | 降级（与 kube-proxy 共存） | 不适用 | 不适用 |

### 核心限制清单

1. **性能**：VxLAN 隧道封装 ~10% 性能损耗
2. **Pod IP 不可从集群外直接访问**：需通过 Service/LB 暴露
3. **超级节点不兼容**：无法与 CiliumOverlay Pod IP 通信
4. **CIDR 不可扩展**：容器网段在创建时固定
5. **需 2 个 LB IP**：从指定子网预留 2 个 IP 创建内网 CLB（IDC 节点访问 APIServer 和云上公共服务）
6. **集群网络与容器网络 CIDR 不可重叠**
7. **不支持固定 Pod IP**
8. **NetworkPolicy 降级**：Cilium 与 kube-proxy 共存时受限，需显式放行 Service CIDR
9. **OS 限制**：云上普通节点仅支持 TencentOS 3.1；注册节点按专线约束

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters
#    验证
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看 Cilium-Overlay 集群 | `DescribeClusters`（`CiliumMode` 非空） | 是 |
| 查看容器 CIDR | `DescribeClusters` → `ClusterNetworkSettings.ClusterCIDR` | 是 |
| 创建 Cilium-Overlay 集群 | `CreateCluster`（`NetworkType="CiliumBGP"`） | 否 |

## 关键字段说明

`DescribeClusters` 中 Cilium-Overlay 相关字段：

| 字段 | 类型 | 含义 | Cilium-Overlay 预期值 |
|------|------|------|----------------------|
| `CiliumMode` | String | Cilium 网络模式 | `"CiliumBGP"` |
| `ClusterCIDR` | String | 容器 CIDR（云上与 IDC 共享） | 不与 VPC CIDR 重叠的私网段 |
| `ServiceCIDR` | String | Service CIDR | 取自 ClusterCIDR 最后一段 |
| `VpcId` | String | 所属 VPC | 云上 VPC 实例 ID |
| `EnableExternalNode` | Boolean | 注册节点开关 | 须为 `true` |
| `NetworkType` | String | 网络类型（Property 内） | `"CiliumBGP"` |
| `DataPlaneV2` | Boolean | Dataplane V2 开关 | `false` |
| `Cni` | Boolean | VPC-CNI 开关 | `false` |

### IP 分配机制

- 云上节点和第三方节点共享一个容器 CIDR
- 每个节点从容器 CIDR 中分配一个固定大小的子段
- Service CIDR 取自容器 CIDR 最后一段
- 节点释放时，其容器 CIDR 子段归还 IP 池
- 扩缩容时，节点轮询选取可用 IP 段

## 操作步骤

### 查看 Cilium-Overlay 集群

```bash
# 查找 Cilium-Overlay 集群
tccli tke DescribeClusters --region <Region> \
    | jq '.Clusters[] | select(.ClusterNetworkSettings.CiliumMode != "") | {
        ClusterId,
        ClusterName,
        CiliumMode: .ClusterNetworkSettings.CiliumMode,
        ClusterCIDR: .ClusterNetworkSettings.ClusterCIDR,
        EnableExternalNode
    }'
# expected: CiliumMode="CiliumBGP" 的集群列表
```

**预期输出**：

```json
{
  "ClusterId": "cls-example",
  "ClusterName": "cluster-hybrid",
  "CiliumMode": "CiliumBGP",
  "ClusterCIDR": "10.200.0.0/16",
  "EnableExternalNode": true
}
```

### 查看单个集群完整网络配置

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: CiliumMode="CiliumBGP", EnableExternalNode=true
```

**预期输出**：

```json
{
    "ClusterId": "cls-example",
    "ClusterName": "cluster-hybrid",
    "ClusterNetworkSettings": {
        "ClusterCIDR": "10.200.0.0/16",
        "ServiceCIDR": "10.200.252.0/22",
        "VpcId": "vpc-example",
        "Cni": false,
        "CiliumMode": "CiliumBGP",
        "DataPlaneV2": false
    },
    "EnableExternalNode": true,
    "Property": "{\"NetworkType\":\"CiliumBGP\"}"
}
```

### 验证容器 CIDR 与 VPC CIDR 不重叠

```bash
# 获取集群容器 CIDR
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '{ClusterCIDR: .Clusters[0].ClusterNetworkSettings.ClusterCIDR, VpcId: .Clusters[0].ClusterNetworkSettings.VpcId}'

# 对比 VPC CIDR
tccli vpc DescribeVpcs --region <Region> \
    --VpcIds '["VPC_ID"]' \
    | jq '.VpcSet[0].CidrBlock'
# expected: ClusterCIDR 与 VPC CIDR 不重叠
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

### 方案选择决策

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 纯云上节点，需 Pod 固定 IP | VPC-CNI | 支持固定 IP，VPC 内直接路由 |
| 纯云上节点，简单场景 | Global Router | 无封装损耗，CIDR 可扩展 |
| 混合云（云上 + IDC 注册节点） | **Cilium-Overlay** | 唯一支持注册节点统一容器网络的方案 |
| 时延敏感（如游戏、交易） | VPC-CNI | 无隧道封装，延迟最低 |
| 需要 NetworkPolicy | VPC-CNI 或独立 Cilium | Cilium-Overlay 降级 |

## 验证

### Cilium-Overlay 特征验证

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '{
        ClusterId: .Clusters[0].ClusterId,
        CiliumMode: .Clusters[0].ClusterNetworkSettings.CiliumMode,
        ClusterCIDR: .Clusters[0].ClusterNetworkSettings.ClusterCIDR,
        ServiceCIDR: .Clusters[0].ClusterNetworkSettings.ServiceCIDR,
        EnableExternalNode: .Clusters[0].EnableExternalNode,
        NetworkType: (.Clusters[0].Property | fromjson | .NetworkType)
    }'
# expected: CiliumMode="CiliumBGP", EnableExternalNode=true, NetworkType="CiliumBGP"
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

### 验证维度

| 维度 | 命令 | 预期 |
|------|------|------|
| `CiliumMode` | `DescribeClusters` → `.CiliumMode` | `"CiliumBGP"` |
| `NetworkType` | `Property.fromjson.NetworkType` | `"CiliumBGP"` |
| `EnableExternalNode` | `DescribeClusters` → `.EnableExternalNode` | `true` |
| CIDR 不重叠 | 对比 VPC CIDR 与 ClusterCIDR | 两者不重叠 |
| `Cni=false` | `DescribeClusters` → `.Cni` | `false`（非 VPC-CNI） |

## 清理

> 本页为概念说明与只读查询，无资源创建。如需删除 Cilium-Overlay 集群，使用 `DeleteCluster`（参见 [创建集群](../../../集群管理/创建集群/tccli 操作.md) 的清理节）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusters` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| `jq` 解析 `CiliumMode` 为空 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.Clusters[0].ClusterNetworkSettings.CiliumMode'` | 该集群非 Cilium-Overlay 模式（GR 或 VPC-CNI），`CiliumMode` 为空字符串是正常现象 | 确认集群网络方案：GR（`Cni=false, CiliumMode=""`）或 VPC-CNI（`Cni=true, CiliumMode=""`） |

### Cilium-Overlay 常见问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 纯云上集群想用 Cilium-Overlay | -- | Cilium-Overlay 仅支持注册节点场景，不支持纯云上节点集群 | 改用 VPC-CNI（推荐，时延敏感）或 GR（简单固定规模） |
| NetworkPolicy 配置后 Service 访问失败 | 检查是否配置了允许 Service CIDR 的规则 | Cilium 与 kube-proxy 共存导致 NetworkPolicy 降级 | 在 NetworkPolicy 中显式添加 `ipBlock` 放行 Service CIDR |
| IDC 节点无法访问 APIServer | 检查是否从指定子网创建了内网 CLB | 缺少 2 个预留 IP 的内网 CLB | 从指定子网预留 2 个 IP 创建内网 CLB，供 IDC 节点访问 APIServer 和云上公共服务 |
| Pod IP 无法从集群外直接访问 | -- | Cilium-Overlay Pod IP 使用 VxLAN 封装，不可从集群外直接路由 | 通过 Service/LB 暴露服务；如需 Pod IP 直接可达，改用 VPC-CNI |
| 容器 CIDR 不足无法扩展 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.ClusterNetworkSettings.ClusterCIDR'` | CIDR 创建后固定，不可扩展 | 规划时预留足够的 CIDR 容量；存量不足需新建集群 |
| 超级节点与 Pod 无法通信 | -- | Cilium-Overlay 与超级节点不兼容 | 注册节点场景不使用超级节点 |

## 下一步

- [容器网络概述](../../容器网络概述/tccli 操作.md) -- 三种网络方案总览
- [GlobalRouter 模式介绍](../../GlobalRouter%20模式/GlobalRouter%20模式介绍/tccli%20操作.md) -- GR 模式
- [VPC-CNI 模式介绍](../../VPC-CNI%20模式/VPC-CNI%20模式介绍/tccli%20操作.md) -- VPC-CNI 模式
- [Dataplane V2 介绍](../../Dataplane%20V2%20转发模式/Dataplane%20V2%20介绍/tccli%20操作.md) -- eBPF 转发模式

## 控制台替代

[TKE 控制台 → 新建集群](https://console.cloud.tencent.com/tke2/cluster/create) → 网络配置 → 选择「Cilium-Overlay」（需已注册第三方节点）。
