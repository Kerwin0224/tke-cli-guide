---
doc_type: How-to
subtype: 6B
fused: false
---
# 配置 CiliumOverlay 网络

> 控制台: [容器服务控制台 - 集群网络](https://console.cloud.tencent.com/tke2/cluster)
> CiliumOverlay 仅用于分布式云第三方节点或注册节点场景。创建集群时通过 `ClusterAdvancedSettings.NetworkType=CiliumOverlay` 指定；当前公开 API 无独立开关或事后修改路径。

## 触发条件

- 为分布式云第三方节点或注册节点新建集群，并选择不直接占用 VPC 子网 IP 的 CiliumOverlay 网络
- `DescribeClusters` → `ClusterStatus=Running`，但 `Property` 解析出的 `NetworkType` 非 `CiliumOverlay`；当前公开 API 没有事后改为 CiliumOverlay 的路径
- 规划 CiliumOverlay 集群容量；该模型不支持创建后扩大容器 CIDR


## 概述

CiliumOverlay 是 TKE 的三种容器网络类型之一（另两种：Global Router / VPC-CNI）。它以 Cilium 作为数据面，Pod 流量经 Overlay 隧道封装，不直接占用 VPC 子网 IP。

| 模型 | Pod IP 来源 | 占用 VPC IP | 固定 IP | 安全组直通 | 后期扩 Pod 网段 |
|:-----|:-----------|:----------:|:------:|:----------|:--------------:|
| Global Router（API 默认） | 容器网段（独立） | ❌ | ❌ | ❌ | 支持（暂未产品化，`AddClusterCIDR` 需开白） |
| VPC-CNI | VPC 子网 | ✅ | ✅ | ✅ | ❌ |
| CiliumOverlay | Overlay 隧道 | ❌ | ❌ | ❌ | ❌ |

> 三模型对比与选用见 [网络管理](index.md#网络模型对比)。创建时 `NetworkType` 从 `GR`、`VPC-CNI`、`CiliumOverlay` 三选一，但这不代表运行态能力绝对互斥：GR 集群可事后附加 VPC-CNI。当前公开 API 没有事后改为 CiliumOverlay 的开关或修改路径。

> 官方文档：[容器网络概述](https://cloud.tencent.com/document/product/457/50353) · [网络方案选型](https://cloud.tencent.com/document/product/457/106561) · [Cilium-Overlay 模式介绍](https://cloud.tencent.com/document/product/457/77964)
> 配额：集群 CIDR 在创建前规划；CiliumOverlay 不支持创建后扩大容器 CIDR。控制面子网须预留 ≥2 个可用 IP。[配额限制](https://cloud.tencent.com/document/product/457/9087)

## 决策依据

#### 什么时候选 CiliumOverlay

- **适用范围**：仅用于分布式云第三方节点或注册节点场景；全云上节点的公有云集群推荐 VPC-CNI
- **Overlay 地址平面**：Pod 流量经 VXLAN 隧道封装，不直接占用 VPC 子网 IP；官方说明 Cilium 与 kube-proxy 同时运行，不要把 CiliumOverlay 与 Dataplane V2 的“Cilium 替代 kube-proxy”混为一谈
- **不需要固定 IP / 安全组直通**：这两项是 VPC-CNI 的能力，CiliumOverlay 不支持。若需 Pod 固定 IP 或安全组直通，选 VPC-CNI（见 [配置 VPC-CNI](vpc-cni.md)）
- **能在创建后扩大容器 CIDR 吗？**：不能。须在创建前完成容量规划
- **能通过 TCCLI 事后切换吗？**：当前公开 API 没有修改 `NetworkType` 或启停 CiliumOverlay 的 Action；这只说明公开 API 边界，不扩展为“所有产品路径都只能重建”的绝对承诺

> API 的 `NetworkType` 未传时默认 `GR`，这是入参默认值，不是产品推荐。公有云推荐 VPC-CNI；注册节点推荐 CiliumOverlay。

## 配置项

| 字段 | 所属对象 | 类型 | 必填 | 作用 |
|:------|:---------|------|:--------:|------|
| NetworkType | ClusterAdvancedSettings | string | 否（API 默认 `GR`） | 容器网络类型枚举：`GR` / `VPC-CNI` / `CiliumOverlay`。选 CiliumOverlay 传 `CiliumOverlay` |
| SubnetId | ClusterBasicSettings | string | **CiliumOverlay 时必填** | 控制面子网。TKE 从该子网取 **2 个 IP** 创建内网负载均衡 |

> ⚠️ **`SubnetId` 的条件必填**：容器网络插件为 CiliumOverlay 时，TKE 会从指定子网获取 2 个 IP 创建内网负载均衡，故 `ClusterBasicSettings.SubnetId` 必传，且子网至少须有 2 个可用 IP。该字段 API 层 `required=false`，条件必填不体现在字段级标记中。

## 应用

### 创建 CiliumOverlay 集群

> CiliumOverlay 无独立开关 Action，通过 `CreateCluster` 的 `ClusterAdvancedSettings.NetworkType` 指定。完整创建流程（凭证/配额/VPC 检查）见 [创建集群](../clusters/create.md)，此处仅列 CiliumOverlay 相对于默认 GR 的差异。

```bash
tccli tke CreateCluster --region ap-guangzhou \
  --ClusterType MANAGED_CLUSTER \
  --ClusterBasicSettings '{"ClusterName":"<CLUSTER_NAME>","VpcId":"<VPC_ID>","SubnetId":"<SUBNET_ID>"}' \
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

### 用 --generate-cli-skeleton 取完整入参骨架

> `CreateCluster` 入参多且嵌套，上述命令仅列 CiliumOverlay 差异字段。实际创建前建议用 `--generate-cli-skeleton` 取得完整入参骨架，核对全部必填项。

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

> ⚠️ **`NetworkType` 在响应层的落点**：创建时 `NetworkType` 传在 `ClusterAdvancedSettings`（入参 object），但 `DescribeClusters` 响应的 `Cluster` object **无顶层 `NetworkType` 字段**，`ClusterNetworkSettings` 也不含 `NetworkType`——`NetworkType` 藏在 `Cluster.Property` 这个 **JSON 字符串**里（如 `"{\"NodeNameType\":\"lan-ip\",\"NetworkType\":\"GR\"}"`），须二次解析。`CiliumMode`/`DataPlaneV2`/控制面 `SubnetId` 则在 `ClusterNetworkSettings`（正常字段）。确认网络类型时不要查 `ClusterNetworkSettings.NetworkType`（不存在）。

## 回滚

> `NetworkType` 在创建时通过 `ClusterAdvancedSettings` 传入；`ModifyClusterAttribute` 不含该网络字段，当前公开 API 也没有 CiliumOverlay 的 Enable/Disable Action。因此，TCCLI 当前没有事后改为或改出 CiliumOverlay 的路径。若需变更方案，先通过腾讯云支持确认当前产品是否提供适用于目标集群的迁移能力，再制定变更计划。

---

## 限制与故障恢复

### CiliumOverlay 的固有限制

| 限制 | 影响 | 规避 |
|:-----|:-----|:-----|
| 不支持创建后扩大容器 CIDR | 容量不足时不能通过 `AddClusterCIDR` 扩大 | 创建前按业务规模规划容量 |
| 当前公开 API 无事后修改 `NetworkType` 的路径 | TCCLI 不能直接切换网络方案 | 变更前联系腾讯云支持确认可用迁移路径 |
| 控制面子网须预留 ≥2 个可用 IP | IP 不足会阻断内网负载均衡所需地址分配 | 用 `tccli vpc DescribeSubnets` 核对 `AvailableIpAddressCount` |

### 命令返回错误 (exit ≠ 0)

创建失败时先保留完整 `RequestId` 与服务端返回，再对照下表已知输入边界排查；不要仅凭未在下表列出的固定错误码或消息文本下结论。

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 子网相关错误 | `tccli vpc DescribeSubnets` 核对子网与 `AvailableIpAddressCount` | `SubnetId` 不属于目标 VPC、子网不存在或可用 IP 少于 2 | 改用目标 VPC 内且至少有 2 个可用 IP 的子网 |
| `NetworkType` 参数校验失败 | 核对 `ClusterAdvancedSettings.NetworkType` | 值不在 `GR` / `VPC-CNI` / `CiliumOverlay` 枚举内 | 使用大小写准确的 `CiliumOverlay` |
| 容器 CIDR 扩容失败 | 解析 `DescribeClusters` 返回的 `Property.NetworkType` | CiliumOverlay 不支持创建后扩大容器 CIDR | 创建前规划容量；已建集群联系腾讯云支持确认可行方案 |

## 收尾确认

```bash
# 一次性核对：Running + Property.NetworkType=CiliumOverlay（不要只滤 state/name）
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
