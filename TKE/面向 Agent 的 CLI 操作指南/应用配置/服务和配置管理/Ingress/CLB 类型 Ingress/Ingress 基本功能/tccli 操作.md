# Ingress 基本功能（tccli）

> 对照官方：[Ingress 基本功能](https://cloud.tencent.com/document/product/457/31711) · page_id `31711`

## 概述

Ingress 是允许访问到集群内 Service 的规则的集合，通过配置转发规则，实现不同 URL 访问集群内不同的 Service。TKE 默认启用了基于腾讯云负载均衡器实现的 `l7-lb-controller`（应用型 CLB），支持 HTTP、HTTPS，同时也支持在集群内自建其他 Ingress 控制器。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 查看集群列表 | `tccli tke DescribeClusters` | 是 | 获取 ClusterId |
| 创建 Ingress（控制台） | `kubectl create -f <ingress>.yaml` | 否(同名报错) | 通过 YAML 创建 Ingress 资源 |
| 更新 Ingress（YAML） | `kubectl edit ingress/<name>` | 否 | 编辑 Ingress YAML |
| 更新转发配置 | `kubectl apply -f <ingress>.yaml` | 是 | 更新 Ingress 规则 |
| 删除 Ingress | `kubectl delete ingress/<name>` | 是(不存在不报错) | 删除 Ingress 资源 |
| 重建 Ingress | 删除后 `kubectl create -f` | 否 | YAML 中移除 status/managedFields 等自动字段后重建 |

## 操作步骤

### 创建 Ingress

```yaml
# my-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: qcloud
    ## kubernetes.io/ingress.existLbId: lb-xxxxxxxx  ## 指定已有CLB
    ## kubernetes.io/ingress.subnetId: subnet-xxxxxxxx ## 创建内网Ingress
  name: my-ingress
  namespace: default
spec:
  rules:
    - host: <domain>
      http:
        paths:
          - backend:
              service:
                name: <service-name>
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
```

> **注意：**
> - TKE Kubernetes 版本 ≥ 1.20 使用 `networking.k8s.io/v1`；版本 ≤ 1.20 使用 `extensions/v1beta1`
> - 确保容器业务不和 CVM 业务共用一个 CLB
> - 不支持在 CLB 控制台操作 TKE 管理的 CLB 的监听器、转发路径、证书和后端绑定的服务器
> - 使用已有 CLB 时：仅支持通过 CLB 控制台创建的负载均衡器，不支持复用 TKE 自动创建的 CLB；支持多个 Ingress 复用 CLB；支持 Ingress 和 Service 共用 CLB
> - 默认 CLB 转发规则限制为 50 个，超出需提工单提升配额
> - 删除 Ingress 后，复用 CLB 绑定的后端云服务器需自行解绑，保留的 `tag tke-clusterId: cls-xxxx` 需自行清理

#### IP 带宽包账号（需额外 annotations）

```yaml
metadata:
  annotations:
    kubernetes.io/ingress.internetChargeType: TRAFFIC_POSTPAID_BY_HOUR
    kubernetes.io/ingress.internetMaxBandwidthOut: "10"
```

| 参数 | 可选值 | 说明 |
|---|---|---|
| `kubernetes.io/ingress.internetChargeType` | `TRAFFIC_POSTPAID_BY_HOUR`, `BANDWIDTH_POSTPAID_BY_HOUR` | 公网带宽计费方式 |
| `kubernetes.io/ingress.internetMaxBandwidthOut` | [1, 2000] Mbps | 带宽上限 |

```bash
kubectl create -f my-ingress.yaml
```

### 更新 Ingress

```bash
kubectl edit ingress/<name>
```

或先删除旧的再重新创建：

```bash
kubectl delete ingress/<name>
kubectl create -f my-ingress.yaml
```

### 删除 Ingress

```bash
kubectl delete ingress/<name>
```

### 重建 Ingress

当 CLB 被误删或修改了某些需要更换 CLB 的注解时：

```bash
kubectl get ingress <name> -o yaml > ingress-backup.yaml
kubectl delete ingress <name>
# 编辑 ingress-backup.yaml，移除 status、managedFields、creationTimestamp、finalizers、generation、resourceVersion、uid 等字段
kubectl create -f ingress-backup.yaml
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 数据面（kubectl）

```bash
kubectl get ingress
```

返回类似以下信息即表示创建成功：

```
NAME         CLASS    HOSTS       ADDRESS       PORTS   AGE
my-ingress   <none>   <domain>    x.x.x.x       80      4s
```

```bash
kubectl describe ingress <name>
```

```text
Name:         ...
Status:       Running
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress/<name>
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| Ingress 创建后无 ADDRESS | `kubectl describe ingress` | CLB 是否创建成功 | 检查 CLB 是否创建成功，检查 Ingress Controller 是否正常运行 |
| 转发规则不生效 | `kubectl describe <resource>` | Service 存在且 endpoints 正常 | 确认 Service 存在且 endpoints 正常，确认域名解析正确 |
| 证书不生效 | `qcloud_cert_id` | Secret 存在且包含正确的 `qcloud_cert_id` | 确认 Secret 存在且包含正确的 `qcloud_cert_id` |
| Ingress 同步异常 | `LoadBalancerResource` | 勿对 `LoadBalancerResource` CRD 进行任何手动操作 | 勿对 `LoadBalancerResource` CRD 进行任何手动操作 |
| 公网 CLB 不显示 VIP | `tccli clb DescribeLoadBalancers` | 域名化 CLB 不再展示 VIP 地址 | 域名化 CLB 不再展示 VIP 地址，请使用域名访问 |

## 下一步

- [Ingress 使用已有 CLB](../Ingress%20使用已有%20CLB/tccli 操作.md)
- [Ingress 证书配置](../Ingress%20证书配置/tccli 操作.md)
- [Ingress Controllers 说明](../../Ingress%20Controllers%20说明/tccli 操作.md)

## 控制台替代

在控制台 **集群 > 服务与路由 > Ingress > 新建** 中创建和管理 Ingress。
