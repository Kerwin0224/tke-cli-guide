# Service 概述（tccli）

> 对照官方：[Service 概述](https://cloud.tencent.com/document/product/457/45487) · page_id `45487`

## 概述

Service 通过 Label Selector 将流量负载均衡到一组 Pod。TKE 深度集成腾讯云 CLB（Cloud Load Balancer），支持 TCP/UDP/HTTP/HTTPS 协议，内网/公网模式，以及丰富的 Annotation 配置。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置
- 熟悉 Kubernetes 基本概念（Pod、Service 等）
- CAM 权限：`tke:DescribeClusters`

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl/tccli 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看资源列表 | `kubectl get RESOURCE` | 是 |
| 创建资源 | `kubectl apply -f resource.yaml` | 否 |
| 删除资源 | `kubectl delete RESOURCE NAME` | 是 |

## 操作步骤

本页面为概念性说明，无操作步骤。具体操作指引请参考子页面 [Service 基本功能](../Service 基本功能/tccli 操作.md)、[Service 负载均衡配置](../Service 负载均衡配置/tccli 操作.md) 等。

## 验证

不适用（概述页面）。

## 清理

不适用（概述页面）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |

## 下一步

- [Service 基本功能](../Service 基本功能/tccli 操作.md) — 创建 Service
- [Service 负载均衡配置](../Service 负载均衡配置/tccli 操作.md) — CLB 高级参数
- [Service Annotation 说明](../Service Annotation 说明/tccli 操作.md) — Annotation 参考

## 控制台替代

控制台详情参见 [Service 概述（控制台）](https://cloud.tencent.com/document/product/457/45487)
