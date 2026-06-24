# Service 概述（tccli）

> 对照官方：[Service 概述](https://cloud.tencent.com/document/product/457/45487) · page_id `45487`
## 概述

Service 是 Kubernetes 中将一组 Pod 暴露为网络服务的抽象方式。TKE 支持 ClusterIP、NodePort、LoadBalancer、ExternalName 四种类型。

## 前置条件

- 熟悉 Kubernetes 基本概念（Pod、Service、Ingress 等）
- 已了解 TKE 集群基本架构

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群列表 | `tccli tke DescribeClusters --region <Region>` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |

## 操作步骤

### Service 类型

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| `ClusterIP` | 集群内部 IP，仅集群内访问 | 内部服务间通信 |
| `NodePort` | 在每个节点上开放端口 | 开发测试、简单外部访问 |
| `LoadBalancer` | 自动创建腾讯云 CLB | 生产环境外部访问 |
| `ExternalName` | DNS CNAME 映射 | 跨命名空间/外部服务引用 |

### Service 与 CLB 集成

TKE LoadBalancer 类型 Service 会自动创建腾讯云 CLB 实例，支持：
- 公网/内网 CLB
- 自动绑定后端 Pod
- 支持 Annotation 自定义 CLB 配置
- 支持使用已有 CLB


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
