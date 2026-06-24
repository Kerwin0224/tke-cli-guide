# Ingress 概述（tccli）

> 对照官方：[Ingress 概述](https://cloud.tencent.com/document/product/457/45685) · page_id `45685`
## 概述

Ingress 是 Kubernetes 中管理集群外部 HTTP/HTTPS 路由的 API 对象。TKE 支持 CLB 类型 Ingress，自动创建腾讯云 CLB 七层负载均衡。

## 前置条件

- 熟悉 Kubernetes 基本概念（Pod、Service、Ingress 等）
- 已了解 TKE 集群基本架构

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群列表 | `tccli tke DescribeClusters --region <Region>` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |

## 操作步骤

### Ingress 工作原理

Ingress Controller 监听 Ingress 资源变化，动态配置七层负载均衡规则：

1. 创建 Ingress 资源时 TKE Ingress Controller 自动创建/配置 CLB
2. CLB 根据 Ingress rules 将流量路由到后端 Service
3. 支持基于域名和 URL 路径的转发规则

### Ingress 类型

| 类型 | 说明 |
|------|------|
| CLB 类型 Ingress | TKE 默认类型，自动创建公网/内网 CLB |
| Nginx 类型 Ingress | 基于 Nginx Ingress Controller（已停止更新） |

### Ingress 功能

- 基于域名的虚拟主机
- 基于 URL 路径的转发
- HTTPS/TLS 证书配置
- 自定义 Annotation（超时、重定向、CORS 等）
- 跨 VPC 绑定 CLB


## 验证

不适用（概述页面）。

## 清理

不适用（概述页面）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |

## 下一步

- 参考对应操作页面进行实践

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
