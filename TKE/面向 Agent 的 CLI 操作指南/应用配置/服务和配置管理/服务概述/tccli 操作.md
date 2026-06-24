# 服务概述（tccli）

> 对照官方：[服务概述](https://cloud.tencent.com/document/product/457/31700) · page_id `31700`

## 概述

Service 是 Kubernetes 中将一组 Pod 暴露为网络服务的抽象。TKE 支持四种 Service 类型：ClusterIP（集群内访问）、NodePort（节点端口）、LoadBalancer（CLB 负载均衡）和 ExternalName（外部 DNS 别名）。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置
- 熟悉 Kubernetes 基本概念（Pod、Service 等）
- CAM 权限：`tke:DescribeClusters`

## Service 类型

| 类型 | kubectl 配置 | 说明 |
|------|-------------|------|
| ClusterIP | `type: ClusterIP` | 默认类型，仅集群内可访问 |
| NodePort | `type: NodePort` | 在每个节点上开放端口 |
| LoadBalancer | `type: LoadBalancer` | 自动创建 CLB，公网或内网访问 |
| ExternalName | `type: ExternalName` | DNS 别名，无代理 |

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl/tccli 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看资源列表 | `kubectl get RESOURCE` | 是 |
| 创建资源 | `kubectl apply -f resource.yaml` | 否 |
| 删除资源 | `kubectl delete RESOURCE NAME` | 是 |

## 操作步骤

本页面为概念性说明，无操作步骤。具体操作指引请参考子页面 [Service 基本功能](../Service/Service 基本功能/tccli 操作.md)、[Service 负载均衡配置](../Service/Service 负载均衡配置/tccli 操作.md) 等。

## 验证

不适用（概述页面）。

## 清理

不适用（概述页面）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |

## 下一步

- [Namespaces](../Namespaces/tccli 操作.md) — 命名空间管理
- [Service 基本功能](../Service/Service 基本功能/tccli 操作.md) — Service 创建操作
- [Service 负载均衡配置](../Service/Service 负载均衡配置/tccli 操作.md) — CLB 配置
- [Service Annotation 说明](../Service/Service Annotation 说明/tccli 操作.md) — Annotation 参考

## 控制台替代

控制台详情参见 [服务概述（控制台）](https://cloud.tencent.com/document/product/457/31700)
