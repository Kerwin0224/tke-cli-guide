# Ingress 概述（tccli）

> 对照官方：[Ingress 概述](https://cloud.tencent.com/document/product/457/45685) · page_id `45685`

## 概述

Ingress 是 Kubernetes 中将外部 HTTP/HTTPS 流量路由到集群内 Service 的资源。TKE CLB 类型 Ingress 自动创建腾讯云七层 CLB，支持按域名、路径分发流量。

## 前置条件

- [环境准备](../../../../../环境准备.md)：`kubectl` ≥ v1.28
- 已获取集群 kubeconfig，集群可访问
- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-------------|:--:|
| 查看 Ingress 列表 | `kubectl get ingress` | 是 |
| 创建 Ingress | `kubectl apply -f ingress.yaml` | 否 |
| 查看 Ingress 详情 | `kubectl describe ingress NAME` | 是 |
| 删除 Ingress | `kubectl delete ingress NAME` | 是 |

## 操作步骤

本页面为概念性说明，无操作步骤。具体操作指引请参考子页面 [Ingress 基本功能](../Ingress 基本功能/tccli 操作.md)、[Ingress 使用已有 CLB](../Ingress 使用已有 CLB/tccli 操作.md) 等。

## 验证

不适用（概述页面）。

## 清理

不适用（概述页面）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| Ingress ADDRESS 为空 | `kubectl describe ingress NAME` 查看 Events | CLB 创建中或无可用 CLB | 等待 1-2 分钟，确认 Service 存在且 endpoints 正常 |

## 下一步

- [Ingress 基本功能](../Ingress 基本功能/tccli 操作.md) — CLB Ingress 实操
- [Service 基本功能](../../../Service/Service 基本功能/tccli 操作.md) — Service 暴露
- [Ingress Annotation 说明](../Ingress Annotation 说明/tccli 操作.md) — Annotation 参考

## 控制台替代

控制台详情参见 [Ingress 概述（控制台）](https://cloud.tencent.com/document/product/457/45685)
