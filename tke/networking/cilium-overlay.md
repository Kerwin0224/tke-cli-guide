---
doc_type: How-to
subtype: 6B
fused: false
---
# 配置 CiliumOverlay 网络

> 控制台: [容器服务控制台 - 集群网络](https://console.cloud.tencent.com/tke2/cluster)
> 创建集群时选定 CiliumOverlay 网络模型。CiliumOverlay 用 Cilium 的 Overlay 隧道承载 Pod 网络，独立于 VPC 网段。**只能在创建集群时指定，无独立开关 Action，创建后不可切换。**

## 触发条件

- `DescribeClusters` → `ClusterStatus=Running` 但 `Property` 解析出 `NetworkType` 非 `CiliumOverlay`，要新建集群选此模型（创建后不可切换）
- 新建集群时评估三种 Pod 网络模型（Global Router / VPC-CNI / CiliumOverlay），需要 Cilium 数据面且不占 VPC IP
- CiliumOverlay 集群创建后 `AddClusterCIDR` 扩 Pod 网段报错（此模型不支持），看 [限制与故障恢复](#限制与故障恢复)


## 概述

CiliumOverlay 是 TKE 的三种容器网络类型之一（另两种：Global Router / VPC-CNI）。它以 Cilium 作为数据面，Pod 流量经 Overlay 隧道封装，不直接占用 VPC 子网 IP。

| 模型 | Pod IP 来源 | 占用 VPC IP | 固定 IP | 安全组直通 | 后期扩 Pod 网段 |
|:-----|:-----------|:----------:|:------:|:----------|:--------------:|
| Global Router（默认） | 容器网段（独立） | ❌ | ❌ | ❌ | ✅ `AddClusterCIDR` |
| VPC-CNI | VPC 子网 | ✅ | ✅ | ✅ | ❌ |
| CiliumOverlay | Overlay 隧道 | ❌ | ❌ | ❌ | ❌ |

> 三模型对比与选用见 [网络管理](index.md#网络模型对比)。CiliumOverlay 与 VPC-CNI 互斥（均为非 GR 模型），与 Global Router 亦互斥——`NetworkType` 三选一，创建时定型。

> 官方文档：[容器网络概述](https://cloud.tencent.com/document/product/457/50353) · [网络方案选型](https://cloud.tencent.com/document/product/457/106561) · [集群启用 IPVS](https://cloud.tencent.com/document/product/457/32193)
> 配额：集群 CIDR 规划（创建时自定义，暂不支持变更）；控制面子网须预留 ≥2 IP。[配额限制](https://cloud.tencent.com/document/product/457/9087)
> ⚠️ **高危操作**：开启 CiliumOverlay 后不可回退至 GlobalRouter（`NetworkType` 创建时定型不可切换）；DataPlaneV2 与 kube-proxy iptables 模式强绑定，误配致集群不可用。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 决策依据

#### 什么时候选 CiliumOverlay

- **要 Cilium 数据面但不想占 VPC IP**: CiliumOverlay 用隧道封装，Pod 不从 VPC 子网拿 IP（区别于 VPC-CNI）；又以 Cilium 替代传统 kube-proxy/iptables 路径，适合需要 eBPF 可观测性或高性能转发的场景
- **不需要固定 IP / 安全组直通**: 这两项是 VPC-CNI 独有，CiliumOverlay 不支持。若需 Pod 固定 IP 或安全组直通，选 VPC-CNI（见 [配置 VPC-CNI](vpc-cni.md)）
- **不需要后期扩 Pod 网段**: 仅 Global Router 支持 `AddClusterCIDR` 扩容容器网段。CiliumOverlay 不支持（见 [配置集群 — 扩容容器网段](../clusters/configure.md#步骤-5扩容容器网段)）
- **能切换吗?**: 不能。`NetworkType` 创建后不可变，无 `EnableCiliumOverlay`/`Disable` 类 Action（与 VPC-CNI 的 `EnableVpcCniNetworkType`/`DisableVpcCniNetworkType` 不同）。要换网络模型只能重建集群

> 默认推荐仍是 Global Router（`NetworkType=GR`）。仅当明确要 Cilium 数据面且接受"创建时定型、不可扩网段"约束时选 CiliumOverlay。

## 配置项

| 字段 | 所属对象 | 类型 | 必填 | 作用 |
|:------|:---------|------|:--------:|------|
| NetworkType | ClusterAdvancedSettings | string | 否（默认 `GR`） | 容器网络类型枚举：`GR` / `VPC-CNI` / `CiliumOverlay`。选 CiliumOverlay 传 `CiliumOverlay` |
| SubnetId | ClusterBasicSettings | string | **CiliumOverlay 时必填** | 控制面子网。CiliumOverlay 时 TKE 从该子网取 **2 个 IP** 创建内网负载均衡 |
| ClusterOs | ClusterBasicSettings | string | **CiliumOverlay 时受限** | 传 `ubuntu20.04x86_64` 等报 `FailedOperation.Param`（`cluster os … is not supported to use cilium overlay mode`）；可用 **`tlinux3.1x86_64`** |
| CiliumMode | ClusterAdvancedSettings | string | **CiliumOverlay 时禁止传** | 与 `NetworkType=CiliumOverlay` 同传报 `FailedOperation.Param`（`CiliumMode … must not set when use CiliumOverlay`） |
| DataPlaneV2 | ClusterAdvancedSettings | boolean | **CiliumOverlay 时禁止 `true`** | 同传 `true` 报 `NetworkType CiliumOverlay is not supported to use dataplaneV2 mode` |
| IPVS / kube-proxy | ClusterAdvancedSettings | — | **CiliumOverlay 仅 iptables** | 传 `IPVS=true` 报 `cluster of CiliumOverlay only support kubeproxy with mode iptables`；勿传 `IPVS=true` |

> ⚠️ **`SubnetId` 的条件必填**：容器网络插件为 CiliumOverlay 时，TKE 会从该子网获取 2 个 IP 用来创建内网负载均衡，故 `ClusterBasicSettings.SubnetId` 必传，且子网须有可用 IP。该字段 API 层 `required=false`（条件必填不体现在字段级 required 标记），易漏。

## 应用

### 创建 CiliumOverlay 集群

> CiliumOverlay 无独立开关 Action，通过 `CreateCluster` 的 `ClusterAdvancedSettings.NetworkType` 指定。完整创建流程（凭证/配额/VPC 检查）见 [创建集群](../clusters/create.md)，此处仅列 CiliumOverlay 相对于默认 GR 的差异。

```bash
tccli tke CreateCluster --region ap-guangzhou \
  --ClusterType MANAGED_CLUSTER \
  --ClusterBasicSettings '{"ClusterName":"<CLUSTER_NAME>","ClusterOs":"tlinux3.1x86_64","VpcId":"<VPC_ID>","SubnetId":"<SUBNET_ID>"}' \
  --ClusterCIDRSettings '{"ClusterCIDR":"<CLUSTER_CIDR>","ServiceCIDR":"<SERVICE_CIDR>"}' \
  --ClusterAdvancedSettings '{"NetworkType":"CiliumOverlay"}'
# expected: { "ClusterId": "cls-xxxxxxxx", "RequestId": "..." }（tccli 默认剥离 Response 包装层）
```

> `ClusterType` 是 `CreateCluster` 顶层入参（不在 `ClusterBasicSettings`）；`ClusterCIDR`/`ServiceCIDR` 在 `ClusterCIDRSettings` object（非 `ClusterNetworkSettings`，后者是响应层 object）。完整入参结构以 `tccli tke CreateCluster help --detail` 为准。

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_NAME>` | 集群名 | ≤60 字符，字母数字连字符 | 自取 |
| `<VPC_ID>` | VPC ID | 已有 VPC | `tccli vpc DescribeVpcs` |
| `<SUBNET_ID>` | 控制面子网 ID | 须在集群 VPC 内，可用 IP ≥ 2 | `tccli vpc DescribeSubnets --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]'` |
| `<CLUSTER_CIDR>` | 容器网段 | 不得与 VPC CIDR 冲突 | 自取，如 `172.16.0.0/16` |
| `<SERVICE_CIDR>` | 服务网段 | 不与 ClusterCIDR/VPC 冲突 | 自取，如 `10.96.0.0/20` |

> **CiliumOverlay 创建时只传 `NetworkType`**：勿叠 `CiliumMode` / `DataPlaneV2=true` / `IPVS=true`（同传均报 `FailedOperation.Param`）。`ClusterOs` 用 `tlinux3.1x86_64`（ubuntu 系列报错拒绝 CiliumOverlay）。

### 用 --generate-cli-skeleton 取完整入参骨架

> `CreateCluster` 入参多且嵌套，上述命令仅列 CiliumOverlay 差异字段。实际创建前建议取完整骨架核全必填项。

```bash
tccli tke CreateCluster --generate-cli-skeleton
# expected: 完整入参 JSON 骨架（含 ClusterBasicSettings/ClusterCIDRSettings/ClusterAdvancedSettings 全字段）
```

## 验证

```bash
# 创建后轮询集群到 Running
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: Clusters[0].ClusterStatus=Running
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 网络类型 | `DescribeClusters` → `Clusters[].Property`（JSON 字符串）解析 `NetworkType` | `CiliumOverlay` |
| 控制面子网 | `DescribeClusters` → `Clusters[].ClusterNetworkSettings.SubnetId` | CiliumOverlay 时返回控制面子网 ID |
| 集群就绪 | `DescribeClusters` → `ClusterStatus` | `Running` |

> ⚠️ **`NetworkType` 在响应层的落点**：创建时 `NetworkType` 传在 `ClusterAdvancedSettings`（入参 object），但 `DescribeClusters` 响应的 `Cluster` object **无顶层 `NetworkType` 字段**，`ClusterNetworkSettings` 也不含 `NetworkType`——`NetworkType` 藏在 `Cluster.Property` 这个 **JSON 字符串**里（如 `"{\"NodeNameType\":\"lan-ip\",\"NetworkType\":\"GR\"}"`），须二次解析。`CiliumMode`/`DataPlaneV2`/控制面 `SubnetId` 则在 `ClusterNetworkSettings`（正常字段）。确认网络类型时勿找 `ClusterNetworkSettings.NetworkType`（不存在）。

## 回滚

> CiliumOverlay 在集群创建时定型（`ClusterAdvancedSettings.NetworkType`），**创建后不可切换回 GR/VPC-CNI**——只能重建集群。集群创建后可改的属性（名称/等级/项目/高可用/安全模式等）用 `ModifyClusterAttribute`，但其入参不含 `ClusterAdvancedSettings` 的网络字段——`NetworkType` 无事后修改路径，误配只能重建集群。

---

## 限制与故障恢复

### CiliumOverlay 的固有限制

| 限制 | 影响 | 规避 |
|:-----|:-----|:-----|
| 不支持 `AddClusterCIDR` 扩 Pod 网段 | 调用 **exit≠0**：`UnsupportedOperation.ClusterNotSuitAddClusterCIDR`（消息含 `CLUSTER NOT SUIT ADD CLUSTER CIDR` / `failed to get tke-bridge-agent`）；**不会**静默成功 | 容量规划在创建时一次定够；要扩网段只能选 GR 集群（见 [配置集群](../clusters/configure.md#步骤-5扩容容器网段)） |
| 创建后不可切换网络模型 | 要换 GR/VPC-CNI 只能重建集群 | 选型在创建前定 |
| 控制面子网须预留 ≥2 IP | IP 不足时创建失败或控制面异常 | 子网可用 IP ≥ 2，`tccli vpc DescribeSubnets` 核 `AvailableIpAddressCount` |

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound.SubnetId` | `tccli vpc DescribeSubnets` | 控制面子网不在集群 VPC 或不存在 | 用集群 VPC 内子网；CiliumOverlay 时 `SubnetId` 必传 |
| `InvalidParameterValue` | 查 `NetworkType` 拼写 | 非 `GR`/`VPC-CNI`/`CiliumOverlay` 三枚举值 | 用 `CiliumOverlay`（注意大小写） |
| `FailedOperation.Param`（`cluster os … not supported to use cilium overlay`） | 查 `ClusterOs` | OS 不在 CiliumOverlay 支持集（如 ubuntu20.04） | 改 `ClusterOs=tlinux3.1x86_64` |
| `FailedOperation.Param`（`CiliumMode … must not set when use CiliumOverlay`） | 查 AdvancedSettings | 与 Overlay 同传了 `CiliumMode`/`VpcCniType` 等 | 删 `CiliumMode`，只留 `NetworkType=CiliumOverlay` |
| `FailedOperation.Param`（`not supported to use dataplaneV2`） | 查 `DataPlaneV2` | Overlay 与 DataPlaneV2 互斥 | 勿传 `DataPlaneV2=true` |
| `FailedOperation.Param`（`only support kubeproxy with mode iptables`） | 查 `IPVS` | Overlay 仅 iptables | 勿传 `IPVS=true` |
| `UnsupportedOperation.ClusterNotSuitAddClusterCIDR`（`AddClusterCIDR`） | `DescribeClusters` → `Property` 解析 `NetworkType` | 集群为 CiliumOverlay（无 tke-bridge-agent 扩网段路径） | 勿对 Overlay 调 `AddClusterCIDR`；扩网段须 GR 集群 |

## 收尾确认

```bash
# 一次性核对：Running + Property.NetworkType=CiliumOverlay（勿只滤 state/name）
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "Clusters[0].{state:ClusterStatus,name:ClusterName,property:Property,subnet:ClusterNetworkSettings.SubnetId}"
# expected: state=Running；property 解析含 NetworkType=CiliumOverlay；subnet 为创建时 SubnetId

# 下一步前置：kubeconfig 可拉取（进创建节点池前须能连通集群）
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "Kubeconfig" --output text | head -1
# expected: apiVersion: v1
```

> 集群 Running + NetworkType=CiliumOverlay 定型 + kubeconfig 可拉取 = 创建闭环完成。但空集群(0 节点)无法运行 Pod（业务可用性边界），须创建节点池；CiliumOverlay 的 Pod 跨节点通信依赖 Cilium Overlay 隧道，节点池就绪后用 `kubectl get pods -o wide` <!-- kubectl验证CiliumOverlay Pod IP在独立网段，非tccli边界 --> 核 Pod IP 不在 VPC 子网段（Overlay 独立网段）端到端验证。

---

## 下一步

- [创建集群](../clusters/create.md) — 完整建集群流程（凭证/配额/VPC 检查/节点池）
- [配置 VPC-CNI](vpc-cni.md) — VPC-CNI 模型对比，若需固定 IP 改选此
- [网络管理](index.md) — 三模型总览与端点
- [配置集群](../clusters/configure.md) — 集群属性与扩容容器网段（CiliumOverlay 不支持）
