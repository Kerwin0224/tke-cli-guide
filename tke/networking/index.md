---
doc_type: Overview
---
# 网络管理

> 集群的网络访问入口与 Pod 网络模型。决定如何连接集群、Pod 如何获取 IP。

## 是什么

TKE 集群网络分两层：**访问端点**（kubectl/API Server 如何连接）与 **Pod 网络模型**（Pod 如何获取 IP）。

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Cluster Endpoint | 集群 API Server 的入口（公网/内网） | 决定 kubectl 从哪连集群 |
| 公网端点 | 通过公网访问 API Server | 本地开发访问，需白名单 |
| 内网端点 | 通过 VPC 内网访问 API Server | 生产适用：安全、低延迟 |
| VPC-CNI | Pod 直接从 VPC 子网拿 IP | Pod 固定 IP、安全组直通 |
| Global Router | 容器网段独立于 VPC | 默认模式，IP 消耗少 |
| CiliumOverlay | Cilium Overlay 隧道承载 Pod 网络 | 要 Cilium 数据面但不占 VPC IP |

## 网络模型对比

> **控制台推荐序（按场景，非 api 默认）**：① **VPC-CNI**（时延优先 / 迁移 / 固定 Pod IP / 安全组直通）→ ② **Global Router（GR）**（普通业务 / 省 VPC IP / 可后期扩网段）→ ③ **CiliumOverlay**（分布式云 / Cilium 数据面且不占 VPC IP）。  
> api/`NetworkType` 未传时默认 **GR**——这是契约默认值，**不等于**上表场景推荐序；按上序场景选，再写入 `ClusterAdvancedSettings.NetworkType`。

| 模型 | Pod IP 来源 | IP 消耗 | 固定 IP | 安全组直通 | 后期扩网段 | 开启方式 |
|:-----|:-----------|:---------|:------:|:----------|:----------:|:---------|
| VPC-CNI | VPC 子网 | 多（每 Pod 一个） | 支持 | 支持 | 不支持 | 创建时 `NetworkType=VPC-CNI`，或创建后 `EnableVpcCniNetworkType` |
| Global Router（契约默认） | 容器网段（独立） | 少 | 不支持 | 不支持 | 支持 `AddClusterCIDR` | 创建时 `NetworkType=GR`（或不传） |
| CiliumOverlay | Overlay 隧道 | 少（不占 VPC IP） | 不支持 | 不支持 | 不支持 | 创建时 `NetworkType=CiliumOverlay` |

> 三模型互斥，`NetworkType` 创建时定型（仅 VPC-CNI 可创建后用 `EnableVpcCniNetworkType` 开启）。VPC-CNI 开启约束见 [配置 VPC-CNI](vpc-cni.md)；CiliumOverlay 创建时定型、不可切换见 [配置 CiliumOverlay](cilium-overlay.md)。

### 转发模式半常量（与 NetworkType 正交）

| 项 | 约束 |
|:---|:-----|
| **IPVS** | 仅**新建集群**时可开；开启后**不可关闭**；勿在集群内手动混用 IPVS 与 iptables |
| **iptables ↔ ipvs** | 一经选择不支持更改 |
| **Dataplane v2** | 开启后不再安装 kube-proxy，默认用 Cilium 转发；与 `KubeProxyMode` 语义互斥 |

创建时字段见 [创建集群 — 跨字段约束](../clusters/create.md#kube-proxy-转发模式-×-ipvs-×-kubeproxymode-互斥)。

## 端点类型对比

| 端点 | 访问方式 | 安全 | 延迟 | 适用 |
|:-----|:---------|:-----|:-----|:-----|
| 公网端点 | 公网 IP + 白名单 | 低（暴露公网） | 高 | 本地开发、CI/CD |
| 内网端点 | VPC 内网 IP | 高（VPC 隔离） | 低 | 生产、同 VPC 访问 |

> 生产环境用内网端点。公网端点需配 ACL 白名单，否则有安全风险。

## 不适用场景

- 不需要外部访问集群（仅控制台操作）→ 跳过端点配置
- 已用 Global Router 且不需固定 IP → 不需配置 VPC-CNI
- 不需要 Cilium 数据面 → 不需配置 CiliumOverlay
- 需要跨 VPC 访问 → 用对等连接/云联网，非端点问题

## 快速检查

```bash
# 查看集群端点状态
tccli tke DescribeClusterEndpointStatus --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: Status = "Creating"/"Running"/"NotFound"（未开启）
```

## 文档

- [管理访问端点](endpoints.md) — 开启/关闭公网/内网端点，ACL 白名单
- [配置 VPC-CNI](vpc-cni.md) — 开启 VPC-CNI 网络模型，子网与固定 IP
- [配置 CiliumOverlay](cilium-overlay.md) — 创建 CiliumOverlay 集群，控制面子网前置
- [查询集群](../clusters/query.md) — `DescribeClusterEndpoints` 看端点地址
