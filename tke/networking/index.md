---
doc_type: Overview
---
# 网络管理

> 集群的网络访问入口与 Pod 网络模型。决定你怎么连接集群、Pod 怎么拿 IP。

## 这是什么

TKE 集群网络分两层：**访问端点**（kubectl/API Server 怎么连）与 **Pod 网络模型**（Pod 怎么拿 IP）。

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Cluster Endpoint | 集群 API Server 的入口（公网/内网） | 决定 kubectl 从哪连集群 |
| 公网端点 | 通过公网访问 API Server | 本地开发访问，需白名单 |
| 内网端点 | 通过 VPC 内网访问 API Server | 生产推荐，安全低延迟 |
| VPC-CNI | Pod 直接从 VPC 子网拿 IP | Pod 固定 IP、安全组直通 |
| Global Router | 容器网段独立于 VPC | 默认模式，IP 消耗少 |

## 网络模型对比

| 模型 | Pod IP 来源 | IP 消耗 | 固定 IP | 安全组直通 | 适用 |
|:-----|:-----------|:---------|:------:|:----------|:-----|
| Global Router（默认） | 容器网段（独立） | 少 | 不支持 | 不支持 | 大多数场景 |
| VPC-CNI | VPC 子网 | 多（每 Pod 一个） | 支持 | 支持 | 安全组直通、固定 IP |

> VPC-CNI 开启后不可关闭某些场景下有约束，见 [配置 VPC-CNI](vpc-cni.md)。两模式可共存（Global Router 为主 + VPC-CNI 子网补充）。

## 端点类型对比

| 端点 | 访问方式 | 安全 | 延迟 | 适用 |
|:-----|:---------|:-----|:-----|:-----|
| 公网端点 | 公网 IP + 白名单 | 低（暴露公网） | 高 | 本地开发、CI/CD |
| 内网端点 | VPC 内网 IP | 高（VPC 隔离） | 低 | 生产、同 VPC 访问 |

> 生产环境用内网端点。公网端点需配 ACL 白名单，否则有安全风险。

## 不适用场景

- 不需要外部访问集群（仅控制台操作）→ 跳过端点配置
- 已用 Global Router 且不需固定 IP → 不需配置 VPC-CNI
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
- [查询集群](../clusters/query.md) — `DescribeClusterEndpoints` 看端点地址
