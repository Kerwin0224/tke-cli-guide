# 集群启用 IPVS（tccli）

> 对照官方：[集群启用 IPVS](https://cloud.tencent.com/document/product/457/32193) · page_id `32193`

## 概述

TKE 默认使用 iptables 作为 kube-proxy 转发模式。对于大规模集群（Service 数量超过 1000），iptrules 规则链线性匹配会导致性能下降。IPVS 基于内核哈希表实现负载均衡，在大量 Service 场景下性能显著优于 iptables。

**核心约束：IPVS 仅在创建集群时通过 `ClusterAdvancedSettings.IPVS=true` 启用，创建后不可切换、不可关闭。**

| 维度 | iptables（默认） | IPVS |
|------|-----------------|------|
| `ClusterAdvancedSettings.IPVS` | `false` 或不设置 | `true` |
| `ClusterNetworkSettings.Ipvs`（读） | `false` | `true` |
| `KubeProxyMode`（读） | `""`（空串，等价 iptables） | `"ipvs"` |
| 转发机制 | 规则链线性匹配 O(n) | 内核哈希表 O(1) |
| 大规模 Service 性能 | 随 Service 数量线性下降 | 保持稳定 |
| 切换方式 | — | 仅创建时可选，创建后不可修改 |
| 可逆性 | — | 不可关闭 |
| K8s 版本要求 | 无 | ≥ 1.10 |

> **注意：** 若创建时同时启用了 [DataPlane V2](../使用%20Dataplane%20V2/tccli%20操作.md)（`ClusterAdvancedSettings.DataPlaneV2=true`），kube-proxy 不会安装，IPVS 选项无效。

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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建集群时启用 IPVS | `CreateCluster` → `ClusterAdvancedSettings.IPVS=true` | 否 |
| 查看集群 IPVS 状态 | `DescribeClusters` → `ClusterNetworkSettings.Ipvs` | 是 |
| 查看 kube-proxy 模式 | `DescribeClusters` → `ClusterNetworkSettings.KubeProxyMode` | 是 |

## 关键字段说明

### CreateCluster 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `IPVS` | Boolean | 否 | `true` / `false`。默认 `false`（iptables）。仅创建时可设，创建后不可修改 | 对存量集群设此字段无效 |
| `KubeProxyMode` | String | 否 | `""`（iptables）、`"ipvs"`。与 `IPVS` 对应，通常只设 `IPVS` 即可 | 与 `IPVS` 不一致时行为未定义 |
| `DataPlaneV2` | Boolean | 否 | 若为 `true`，kube-proxy 不安装，IPVS 设置失效 | 同时启用时 IPVS 不生效 |

### DescribeClusters 响应字段

| 字段 | 类型 | 含义 | 取值 |
|------|------|------|------|
| `Ipvs` | Boolean | 是否启用 IPVS | `true` / `false` |
| `KubeProxyMode` | String | kube-proxy 模式 | `""`（iptables）、`"ipvs"` |
| `DataPlaneV2` | Boolean | 是否启用 DataPlane V2 | `true` / `false` |

> 以上字段位于响应的 `ClusterNetworkSettings` 对象内。

## 操作步骤

### 操作场景

默认 kube-proxy 使用 iptables 模式。IPVS 适用于大规模集群（Service 数量较多），基于哈希表的内核态转发在查询效率上优于 iptables 的线性规则匹配。

### 注意事项

- **仅新建集群时可启用**，存量集群不支持修改 `IPVS` 字段。
- **全集群生效**，不可混合使用 IPVS 与 iptables。
- **开启后不可关闭**，此为不可逆配置。
- 仅 Kubernetes **1.10 及以上版本** 生效。
- 若同时启用 [DataPlane V2](../使用%20Dataplane%20V2/tccli%20操作.md)，kube-proxy 不会被安装，IPVS 设置不生效。

### 新建集群时启用 IPVS

#### 选择依据

- **IPVS vs iptables**：集群预期运行超过 1000 个 Service 时选 IPVS，性能优势明显。中小规模集群用默认 iptables 即可。
- **不可逆**：一旦启用 IPVS 便不可关闭。确认集群规模需求后再做选择。
- **版本要求**：K8s ≥ 1.10。`tccli tke DescribeVersions --region <Region>` 查询可用版本。

在 `CreateCluster` 的 `ClusterAdvancedSettings` 中设置 `"IPVS": true`。完整集群创建 JSON 见 [创建集群](../../../集群管理/创建集群/tccli%20操作.md)。

关键字段（最小示例）：

```json
{
  "ClusterType": "MANAGED_CLUSTER",
  "ClusterBasicSettings": {
    "ClusterName": "CLUSTER_NAME",
    "ClusterVersion": "1.32.2",
    "VpcId": "VPC_ID",
    "SubnetId": "SUBNET_ID"
  },
  "ClusterCIDRSettings": {
    "ClusterCIDR": "10.168.0.0/16",
    "ServiceCIDR": "10.168.252.0/22"
  },
  "ClusterAdvancedSettings": {
    "IPVS": true
  }
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_NAME` | 集群名称 | 长度 1-60 字符 | 自定义 |
| `VPC_ID` | VPC 实例 ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs --region <Region>` |
| `SUBNET_ID` | 子网 ID | 格式 `subnet-xxxxxxxx` | `tccli vpc DescribeSubnets --region <Region>` |

```bash
tccli tke CreateCluster --region <Region> --cli-input-json file://cluster-ipvs.json
# expected: exit 0，返回 ClusterId
```

创建成功后，集群的 kube-proxy 将以 IPVS 模式运行。

### 核对存量集群 IPVS 状态

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，返回集群信息
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.32.2",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "10.0.0.0/16",
        "ServiceCIDR": "10.1.0.0/20",
        "Ipvs": false,
        "VpcId": "vpc-example",
        "Cni": true,
        "KubeProxyMode": "",
        "DataPlaneV2": false
      }
    }
  ]
}
```

- `Ipvs: true` -- 已启用 IPVS 模式。
- `Ipvs: false` 且 `KubeProxyMode: ""` -- 使用默认 iptables 模式。
- `Ipvs: true` 且 `KubeProxyMode: "ipvs"` -- IPVS 已生效。

### 控制台步骤对照

1. 控制台 → 集群 → [新建](https://console.cloud.tencent.com/tke2/cluster/create)。
2. 选择 **1.10 及以上** Kubernetes 版本。
3. 网络配置 → **高级设置** → kube-proxy 转发模式选择 **ipvs**。
4. 上述操作由 `ClusterAdvancedSettings.IPVS=true` 单一参数控制。

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '{
        ClusterId: .Clusters[0].ClusterId,
        Ipvs: .Clusters[0].ClusterNetworkSettings.Ipvs,
        KubeProxyMode: .Clusters[0].ClusterNetworkSettings.KubeProxyMode,
        ClusterVersion: .Clusters[0].ClusterVersion,
        NetworkType: (.Clusters[0].Property | fromjson | .NetworkType)
    }'
# expected: Ipvs=true, KubeProxyMode="ipvs", ClusterVersion ≥ 1.10
```

```json
{
  "ClusterId": "cls-example",
  "Ipvs": false,
  "KubeProxyMode": "",
  "ClusterVersion": "1.32.2",
  "NetworkType": "GR"
}
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| IPVS 开关 | `DescribeClusters → .ClusterNetworkSettings.Ipvs` | `true` |
| kube-proxy 模式 | `DescribeClusters → .ClusterNetworkSettings.KubeProxyMode` | `"ipvs"` |
| K8s 版本 | `DescribeClusters → .ClusterVersion` | ≥ 1.10 |
| 网络类型 | `DescribeClusters → .Property.NetworkType` | `"GR"` |

### 数据面（kubectl）

**前置要求**：集群需有公网端点（`ClusterExternalEndpoint` 非空）或当前环境已接入 VPC 内网，否则 kubectl 不可达。

```bash
kubectl get cm kube-proxy -n kube-system -o yaml | grep mode
# expected: 输出含 "mode: ipvs"
```

```text
mode: ipvs
```

> **可达性要求**：kubectl 命令需在集群端点可达的环境下执行。若当前环境仅能通过 tccli API 操作集群（如仅有内网端点），可通过 `DescribeClusters` 的 `ClusterNetworkSettings.Ipvs` 字段判断 IPVS 状态，数据面验证暂不可用。

## 清理

本页为说明与只读查询，无额外资源。IPVS 为不可逆配置，无需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ClusterId` 不存在 | `tccli tke DescribeClusters --region <Region>` 列出所有集群 | 集群 ID 拼写错误或 region 不匹配 | 确认 region 和集群 ID。用 `DescribeClusters` 不带 `--ClusterIds` 列出该地域所有集群 |
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeClusters` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| JSON 数组参数在 PowerShell 中被剥引号 | 改用 `--filter` 筛选 | PowerShell 对 `'[...]'` 处理与 bash 不同 | 改用 `--filter` 方式，或将 JSON 写入文件后传参 |

### IPVS 状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Ipvs: false` 但控制台显示为 IPVS | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 确认 | 可能存在字段同步延迟，以 API 返回的 `Ipvs` 字段为准 | 等待数分钟再查；仍不一致则保留 RequestId → 提工单 |
| 存量集群想改为 IPVS 模式 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 查看 `CreateTime` | TKE 不支持存量集群切换 kube-proxy 模式 | 新建集群并设置 `ClusterAdvancedSettings.IPVS=true`，迁移工作负载到新集群 |
| 启用 DataPlane V2 后 IPVS 不生效 | 检查 `DataPlaneV2` 字段 | DataPlane V2 不安装 kube-proxy，IPVS 转发由 eBPF 程序接管 | 如需 IPVS，创建集群时不启用 DataPlane V2 |
| `KubeProxyMode` 为空但 `Ipvs: true` | 查看集群创建时间 | 集群刚创建，kube-proxy 组件尚未完全就绪 | 等待集群状态为 `Running` 后重新查询 |

## 下一步

- [创建集群](../../../集群管理/创建集群/tccli%20操作.md) -- 完整创建流程与 JSON 参数
- [连接集群](../../../集群管理/连接集群/tccli%20操作.md) -- 下载 kubeconfig 并验证连通性
- [GlobalRouter 模式介绍](../GlobalRouter%20模式介绍/tccli%20操作.md) -- GR 模式原理与字段说明
- [使用 DataPlane V2](../使用%20Dataplane%20V2/tccli%20操作.md) -- eBPF 转发模式（与 IPVS 互斥）

## 控制台替代

[控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) 新建 → 网络配置 → 高级设置 → kube-proxy 转发模式选择「ipvs」。
