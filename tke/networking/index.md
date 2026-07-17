---
doc_type: Overview
---
# 网络管理

> 集群的网络访问入口与 Pod 网络模型。决定如何连接集群、Pod 如何获取 IP。
>
> 官方文档：[容器网络概述](https://cloud.tencent.com/document/product/457/50353) · [容器集群网络方案选型](https://cloud.tencent.com/document/product/457/106561) · [创建集群访问端口](https://cloud.tencent.com/document/api/457/39414)

## 是什么

TKE 集群网络分两层：**访问端点**（kubectl/API Server 如何连接）与 **Pod 网络模型**（Pod 如何获取 IP）。

## 触发条件

- 你要开启或关闭集群的公网/内网访问端点（kubectl 连接 API Server 的入口）— 去 [管理访问端点](endpoints.md)
- 你要给已建集群开启 VPC-CNI（Pod 固定 IP、安全组直通），或查 VPC-CNI 子网与固定 IP 约束 — 去 [配置 VPC-CNI](vpc-cni.md)
- 你要创建用于分布式云第三方节点或注册节点的 CiliumOverlay 集群，须确认控制面子网前置 — 去 [配置 CiliumOverlay](cilium-overlay.md)
- 你要查集群当前端点地址（公网/内网 IP）— 用 `DescribeClusterEndpoints`，见 [查询集群](../clusters/query.md)
- 你遇到三种网络模型（VPC-CNI / Global Router / CiliumOverlay）不知选哪个 — 看 [网络模型对比](#网络模型对比)

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Cluster Endpoint | 集群 API Server 的入口（公网/内网） | 决定 kubectl 从哪连集群 |
| 公网端点 | 通过公网访问 API Server | 本地开发访问，需白名单 |
| 内网端点 | 通过 VPC 内网访问 API Server | 生产适用：安全、低延迟 |
| VPC-CNI | Pod 直接从 VPC 子网拿 IP | Pod 固定 IP、安全组直通 |
| Global Router | 容器网段独立于 VPC | API `NetworkType` 的默认值，适合规模相对固定、无特殊 IP 和性能需求的简单业务 |
| CiliumOverlay | Cilium Overlay 隧道承载 Pod 网络 | 仅用于分布式云第三方节点或注册节点场景 |

## 网络模型对比

> **按场景选择**：公有云集群推荐 **VPC-CNI**；分布式云第三方节点或注册节点推荐 **CiliumOverlay**。**Global Router（GR）**适合规模相对固定、对 IP 分配和网络性能没有特殊需求的简单业务。  
> API 的 `NetworkType` 未传时默认 **GR**，这是契约默认值，**不等于产品推荐**；选定场景后将对应值写入 `ClusterAdvancedSettings.NetworkType`。

| 模型 | 适用范围 | Pod IP 来源 | 固定 IP | 安全组直通 | 后期扩网段 | 开启方式 |
|:-----|:---------|:-----------|:------:|:----------|:----------:|:---------|
| VPC-CNI | 公有云推荐；适合时延敏感、传统架构迁移、固定 Pod IP 或独立安全组场景 | VPC 子网 | 支持 | 支持 | 支持 | 创建时 `NetworkType=VPC-CNI`，或为 GR 集群调用 `EnableVpcCniNetworkType` 附加 |
| Global Router（API 默认） | 规模相对固定、无特殊 IP 和性能需求的简单业务 | 容器网段（独立） | 不支持 | 不支持 | 支持（暂未产品化，`AddClusterCIDR` 需开白） | 创建时 `NetworkType=GR`（或不传） |
| CiliumOverlay | 仅分布式云第三方节点或注册节点场景 | Overlay 隧道 | 不支持 | 不支持 | 不支持 | 创建时 `NetworkType=CiliumOverlay` |

> 创建 `ClusterAdvancedSettings.NetworkType` 时从 `GR`、`VPC-CNI`、`CiliumOverlay` 三选一，但不能据此断言运行态能力绝对互斥：GR 集群可通过 `EnableVpcCniNetworkType` 附加 VPC-CNI。当前公开 API 没有事后改为 CiliumOverlay 的开关或修改路径。VPC-CNI 开启约束见 [配置 VPC-CNI](vpc-cni.md)；CiliumOverlay 的创建约束见 [配置 CiliumOverlay](cilium-overlay.md)。

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
- 不属于分布式云第三方节点或注册节点场景 → 不选 CiliumOverlay
- 需要跨 VPC 访问 → 用对等连接/云联网，非端点问题

## 快速检查

```bash
# 查看集群端点状态
tccli tke DescribeClusterEndpointStatus --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: Status = "Created"/"Creating"/"NotFound"/"CreateFailed"（未开启为 NotFound；无 Running）
```

## 文档

- [管理访问端点](endpoints.md) — 开启/关闭公网/内网端点，ACL 白名单
- [配置 VPC-CNI](vpc-cni.md) — 开启 VPC-CNI 网络模型，子网与固定 IP
- [配置 CiliumOverlay](cilium-overlay.md) — 创建 CiliumOverlay 集群，控制面子网前置
- [查询集群](../clusters/query.md) — `DescribeClusterEndpoints` 看端点地址
