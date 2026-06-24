# Dataplane V2 介绍

> 对照官方：[Dataplane V2 介绍](https://cloud.tencent.com/document/product/457/113227) · page_id `113227`

## 概述

Dataplane V2 是 TKE 推出的下一代 Kubernetes 网络数据面，基于 eBPF（通过 Cilium）实现东西向 Service 流量转发（ClusterIP 和 NodePort）和 NetworkPolicy，完全替代 kube-proxy。同时支持部署 Cilium Hubble 实现网络可观测性。

Dataplane V2 将 `cilium-agent` 集成到 TKE 网络插件 `tke-eni-agent`，将 `cilium-operator` 合并到 TKE 网络控制器 `tke-eni-ipamd`，简化组件管理。

Dataplane V2 需在创建集群时通过 `ClusterAdvancedSettings.DataPlaneV2=true` 开启。当前集群 `DataPlaneV2` 状态可通过 `DescribeClusters` 查询。

### 性能优势对比

| 维度 | Dataplane V2 (eBPF) | kube-proxy IPVS | kube-proxy iptables |
|------|:--:|:--:|:--:|
| 数据面转发性能 | 最优（比 IPVS 提升 15-20%） | 良好 | 随 Service 数量增长而下降 |
| 控制面性能（Service 变更） | 基本无感知 | 频繁变更时占 >1 核 CPU（持续数十秒） | Service 达 5000 时性能急剧下降 |
| Service 规模上限 | 10,000+ 无显著影响 | 中等 | 约 5,000 即不可用 |
| 转发性能与 Service 数关系 | 基本无关 | 基本无关 | 相关（Service 越多越慢） |
| 定期对账开销 | 无需对账 | 无需对账 | 需要，消耗 CPU |
| NetworkPolicy | 原生支持（无需额外组件） | 需额外组件 | 需额外组件 |
| 可观测性 | 支持 Hubble 可视化 | 有限 | 有限 |

### 兼容性

- 支持 Service 类型：ClusterIP、NodePort
- 与 GR、VPC-CNI 网络方案均可配合使用
- 需要集群创建时指定 `ClusterAdvancedSettings.DataPlaneV2=true`

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
#    tke:DescribeClusters
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看是否启用 Dataplane V2 | `DescribeClusters`（`DataPlaneV2` 字段） | 是 |
| 查看 kube-proxy 模式 | `DescribeClusters`（`KubeProxyMode` 字段） | 是 |
| 新建集群开启 Dataplane V2 | `CreateCluster`（`ClusterAdvancedSettings.DataPlaneV2: true`） | 否 |

### 控制台概念与 API 参数映射

| 控制台概念 | API 参数 | 取值 | 说明 |
|-----------|---------|------|------|
| 集群网络 → Dataplane V2 开关 | `ClusterAdvancedSettings.DataPlaneV2`（创建时） | `true` / `false` | 创建时可设 |
| 基本信息页 → Dataplane V2 字段 | `ClusterNetworkSettings.DataPlaneV2`（查询时） | `true` / `false` | 创建后只读 |

## 关键字段说明

`DescribeClusters` 中 Dataplane V2 核心字段，以及 `CreateCluster` 时的创建参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterAdvancedSettings.DataPlaneV2`（创建时） | Boolean | 否 | `true` 启用 Dataplane V2。仅创建时可设 | 不传默认不使用 |
| `ClusterNetworkSettings.DataPlaneV2`（查询时） | Boolean | 仅输出 | `true`（已启用）/ `false`（未启用） | — |
| `ClusterNetworkSettings.KubeProxyMode` | String | 仅输出 | Dataplane V2 启用时为空（kube-proxy 已替代）；否则为 `iptables` 或 `ipvs` | — |
| `ClusterNetworkSettings.Ipvs` | Boolean | 仅输出 | Dataplane V2 与 IPVS 互斥。`DataPlaneV2: true` 时 `Ipvs` 为 `false` | 同时启用两者 → IPVS 无效（Dataplane V2 优先） |

## 操作步骤

### 查看 Dataplane V2 启用状态

选择依据：`DataPlaneV2` 是 `ClusterNetworkSettings` 下的布尔字段。`true` 表示已启用 eBPF 数据面，此时 `KubeProxyMode` 通常为空（kube-proxy 已被替代），`Ipvs` 为 `false`。

```bash
# 列出所有集群的 Dataplane V2 状态
tccli tke DescribeClusters --region <Region> \
    | jq '.Clusters[] | {ClusterId, ClusterName, DataPlaneV2: .ClusterNetworkSettings.DataPlaneV2, KubeProxyMode: .ClusterNetworkSettings.KubeProxyMode, Ipvs: .ClusterNetworkSettings.Ipvs}'
# expected: 每集群一行，DataPlaneV2=true 或 false
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

### 查看单个集群详情

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: 返回 DataPlaneV2 和 KubeProxyMode 字段
```

预期输出（已启用 Dataplane V2 的集群）：

```json
{
    "ClusterId": "cls-example",
    "ClusterNetworkSettings": {
        "ClusterCIDR": "10.200.0.0/16",
        "VpcId": "vpc-example",
        "Cni": true,
        "Ipvs": false,
        "KubeProxyMode": "",
        "DataPlaneV2": true,
        "ServiceCIDR": "10.201.0.0/20"
    }
}
```

> 当 `DataPlaneV2: true` 时，`KubeProxyMode` 通常为空（kube-proxy 已被 eBPF 替代），`Ipvs` 为 `false`（互斥）。

预期输出（未启用 Dataplane V2 的 GR 集群）：

```json
{
    "ClusterId": "cls-example",
    "ClusterNetworkSettings": {
        "ClusterCIDR": "10.200.0.0/16",
        "VpcId": "vpc-example",
        "Cni": false,
        "Ipvs": false,
        "KubeProxyMode": "",
        "DataPlaneV2": false,
        "ServiceCIDR": "10.201.0.0/20"
    }
}
```

### 筛选已启用 Dataplane V2 的集群

```bash
tccli tke DescribeClusters --region <Region> \
    | jq '.Clusters[] | select(.ClusterNetworkSettings.DataPlaneV2 == true) | {ClusterId, ClusterName, NetworkType: (.Property | fromjson | .NetworkType)}'
# expected: 列出所有已启用 Dataplane V2 的集群
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

### 验证 Dataplane V2 关键特征

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '{
        ClusterId: .Clusters[0].ClusterId,
        DataPlaneV2: .Clusters[0].ClusterNetworkSettings.DataPlaneV2,
        KubeProxyMode: .Clusters[0].ClusterNetworkSettings.KubeProxyMode,
        Ipvs: .Clusters[0].ClusterNetworkSettings.Ipvs,
        Cni: .Clusters[0].ClusterNetworkSettings.Cni
    }'
# expected: DataPlaneV2=true, KubeProxyMode="", Ipvs=false（启用时）
# expected: DataPlaneV2=false（未启用时）
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

### Dataplane V2 验证维度总览

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| `DataPlaneV2` | `DescribeClusters → .DataPlaneV2` | `true`（启用）/ `false`（未启用） |
| `KubeProxyMode` | `DescribeClusters → .KubeProxyMode` | 启用 Dataplane V2 时为空；否则为 `iptables` 或 `ipvs` |
| `Ipvs` | `DescribeClusters → .Ipvs` | Dataplane V2 启用时为 `false`（互斥） |
| `Cni` | `DescribeClusters → .Cni` | 可为 `true` 或 `false`（与 GR/VPC-CNI 均可配合） |
| kube-proxy 存在性 | `kubectl get cm kube-proxy -n kube-system` | 启用 Dataplane V2 时不存在 |

## 清理

无需清理。本页为概念说明与只读查询，无资源创建。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeClusters` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| `jq` 解析 `DataPlaneV2` 为 `null` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.Clusters[0].ClusterNetworkSettings.DataPlaneV2'` | 老集群可能不存在此字段（创建时 Dataplane V2 尚未发布） | 判断为 `false`（未启用）；该集群创建于 Dataplane V2 功能发布前 |
| `CreateCluster` 中 `DataPlaneV2` 参数不生效 | 检查 JSON 中参数位置和大小写 | 创建时参数须在 `ClusterAdvancedSettings` 下，而非 `ClusterNetworkSettings` | 改为 `ClusterAdvancedSettings.DataPlaneV2: true` |

### Dataplane V2 常见问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 存量集群想启用 Dataplane V2 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.ClusterNetworkSettings.DataPlaneV2'` | Dataplane V2 通常在新建集群时设置 | 查看 TKE 控制台是否支持为存量集群开启；或新建集群时指定 `DataPlaneV2: true` |
| Service 数量多时 iptables 性能差 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.ClusterNetworkSettings.{DataPlaneV2, KubeProxyMode, Ipvs}'` | 当前使用 kube-proxy iptables 模式，Service 规模达数千时性能下降 | 若集群支持，启用 Dataplane V2；或新建集群时启用 IPVS 模式 |
| NetworkPolicy 未生效 | 确认 `DataPlaneV2` 是否已启用 | Dataplane V2 原生支持 NetworkPolicy；iptables/IPVS 模式需额外组件 | 启用 Dataplane V2 以获得原生 NetworkPolicy 支持 |
| 同时设置了 `DataPlaneV2` 和 `IPVS` | 查看 `KubeProxyMode` 是否为空 | Dataplane V2 优先级更高，kube-proxy 被跳过 | 二选一；Dataplane V2 启用后 IPVS 不生效 |

## 下一步

- [容器网络概述](../../容器网络概述/tccli%20操作.md) -- 三种网络方案对比
- [使用 Dataplane V2](../使用%20Dataplane%20V2/tccli%20操作.md) -- 实际操作步骤
- [GlobalRouter 模式介绍](../../GlobalRouter%20%E6%A8%A1%E5%BC%8F/GlobalRouter%20%E6%A8%A1%E5%BC%8F%E4%BB%8B%E7%BB%8D/tccli%20%E6%93%8D%E4%BD%9C.md) -- GR 模式下的转发
- [集群启用 IPVS](../../集群启用%20IPVS/tccli%20操作.md) -- 传统 IPVS 模式

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster)：集群基本信息页「集群网络」→「Dataplane V2」字段展示启用状态。
