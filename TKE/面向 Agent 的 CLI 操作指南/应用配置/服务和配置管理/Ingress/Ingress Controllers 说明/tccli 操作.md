# Ingress Controllers 说明（tccli）

> 对照官方：[Ingress Controllers 说明](https://cloud.tencent.com/document/product/457/56844) · page_id `56844`

## 概述

TKE 支持两种类型的 Ingress Controller：应用型 CLB（基于腾讯云负载均衡器）和 Nginx Ingress Controller。前者由 TKE 托管管理，后者基于开源 Ingress-Nginx Controller。用户可根据对路由管理复杂度和 IP 收敛的需求选择合适的类型。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 选择 CLB 类型 | `kubernetes.io/ingress.class: "qcloud"` | 否(首次) | TKE 默认类型 |
| 选择 Nginx 类型 | `kubernetes.io/ingress.class: "nginx"` | 否(首次) | Nginx Ingress Controller |
| 选择 Others 类型 | 自定义 IngressClass + 自建 Controller | 否 | 自行安装 Ingress Controller |

## 各类型 Ingress Controllers 功能对比

| 模块 | 功能 | 应用型 CLB | Nginx Ingress Controller |
|---|---|---|---|
| 流量管理 | 支持协议 | HTTP, HTTPS | HTTP, HTTPS, HTTP2, gRPC, TCP, UDP |
| | IP 管理 | 一条 Ingress 对应一个 IP（CLB） | 多条 Ingress 对应一个 IP（CLB），IP 地址收敛 |
| | 特征路由 | host, URL | 更多特征支持：header, cookie 等 |
| | 流量行为 | 不支持 | 支持重定向、重写等 |
| | 地域感知负载均衡 | 不支持 | 不支持 |
| 应用访问寻址 | 服务发现 | 单 Kubernetes 集群 | 单 Kubernetes 集群 |
| 安全 | SSL 配置 | 支持 | 支持 |
| | 认证授权 | 不支持 | 支持 |
| 可观测性 | 监控指标 | 支持（需在 CLB 中查看） | 支持（云原生监控） |
| | 调用追踪 | 不支持 | 不支持 |
| 组件运维 | | 关联 CLB 已托管，仅需集群内运行 TKE Ingress Controller | 需集群内运行 Nginx Ingress Controller（控制面 + 数据面） |

> **注意：** NginxIngress 扩展组件已停止更新，但容器服务与开源 Ingress-Nginx Controller 的兼容性不受影响，可通过自建或 TKE 应用市场继续使用 Nginx Ingress。

## 操作步骤

### 选择 Ingress 类型

#### CLB 类型 Ingress

```yaml
metadata:
  annotations:
    kubernetes.io/ingress.class: "qcloud"
```

适合仅需简单路由管理、对 IP 收敛不敏感的场景。

#### Nginx 类型 Ingress

```yaml
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
```

或使用 IngressClassName：

```yaml
spec:
  ingressClassName: nginx
```

适合对接入层路由管理有更多诉求、需要 IP 地址收敛的场景。

#### Others 类型 Ingress

自行安装对应的 Ingress Controller 后，在 Ingress 中指定对应的 IngressClass。

## 验证

### 数据面（kubectl）

查看 Ingress 类型：

```bash
kubectl get ingress <name> -o jsonpath='{.metadata.annotations.kubernetes\.io/ingress\.class}'
```

```text
NAME  STATUS  AGE
...
```

列出所有 IngressClasses：

```bash
kubectl get ingressclass
```

```text
NAME  STATUS  AGE
...
```

## 清理

不适用（说明页面，不涉及资源创建）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| Ingress 类型选择不生效 | `kubectl get ingress <name> -o yaml` 确认 `ingress.class` 注解 | IngressClass 注解值与实际 Controller 不匹配 | 确认注解值为 `qcloud`（CLB）或 `nginx`（Nginx Ingress） |

## 下一步

- [Ingress 基本功能](../CLB%20类型%20Ingress/Ingress%20基本功能/tccli 操作.md)
- [安装 NginxIngress 实例](../Nginx%20类型%20Ingress（停止更新）/安装%20NginxIngress%20实例/tccli 操作.md)
- [Ingress Annotation 说明](../CLB%20类型%20Ingress/Ingress%20Annotation%20说明/tccli 操作.md)

## 控制台替代

在控制台 **新建 Ingress** 时，从 **Ingress 类型** 下拉菜单中选择 CLB 或 Nginx Ingress Controller。
