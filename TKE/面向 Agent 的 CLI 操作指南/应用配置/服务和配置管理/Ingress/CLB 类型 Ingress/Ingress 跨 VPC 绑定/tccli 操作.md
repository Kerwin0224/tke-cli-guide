# Ingress 跨 VPC 绑定（tccli）

> 对照官方：[Ingress 跨 VPC 绑定](https://cloud.tencent.com/document/product/457/59095) · page_id `59095`

## 概述

CLB 型 Ingress 默认在当前集群所在 VPC 的随机可用区生成 CLB。通过注解可指定 CLB 的可用区（包括其他地域），实现 CLB 跨 VPC/跨地域接入。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 通过 [云联网](https://cloud.tencent.com/document/product/877/18752) 打通当前集群 VPC 和目标 VPC
- 已 [在线咨询](https://cloud.tencent.com/online-service?from=doc_457) 申请开通跨 VPC 绑定功能

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作（Annotation） | 幂等 | 说明 |
|---|---|---|---|------|
| 选择当前 VPC 可用区 | `kubernetes.io/ingress.extensiveParameters: '{"ZoneId":"ap-guangzhou-1"}'` | 否(首次) | 指定本集群 VPC 内可用区 |
| 选择其他 VPC | `ingress.cloud.tencent.com/cross-vpc-id` | 否(首次) | 需配合 `cross-region-id` |
| 跨地域创建 CLB | `ingress.cloud.tencent.com/cross-region-id` | 否(首次) | 目标地域 ID |
| 选择其他 VPC 已有 CLB | `ingress.cloud.tencent.com/cross-region-id` + `kubernetes.io/ingress.existLbId` | 否(首次) | 异地接入已有 CLB |

## 操作步骤

> **注意：**
> - 需先通过云联网打通当前集群 VPC 和目标 VPC，并注意网段不能冲突
> - 集群所在 VPC 不能同时加入多个云联网中，否则路由不唯一
> - VPC 需与 TKE 集群在同一账号下
> - 地域 ID 参见 [地域和可用区](https://cloud.tencent.com/document/product/457/44787)

### 示例 1：指定本集群 VPC 的可用区

集群 VPC 在广州，指定广州一区的 CLB：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.extensiveParameters: '{"ZoneId":"ap-guangzhou-1"}'
  name: my-ingress
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

### 示例 2：创建异地接入的 CLB

跨地域（ap-guangzhou）跨 VPC（vpc-xxxxxxxx）创建新 CLB：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/cross-region-id: "ap-guangzhou"
    ingress.cloud.tencent.com/cross-vpc-id: "vpc-xxxxxxxx"
    ## 如需指定可用区：
    kubernetes.io/ingress.extensiveParameters: '{"ZoneId":"ap-guangzhou-1"}'
  name: my-ingress
spec:
  rules:
    - http:
        paths:
          - backend:
              service:
                name: <service-name>
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
```

### 示例 3：选择异地已有 CLB

跨地域（ap-guangzhou）使用已有 CLB（lb-xxxxxxxx）：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/cross-region-id: "ap-guangzhou"
    kubernetes.io/ingress.existLbId: "lb-xxxxxxxx"
  name: my-ingress
spec:
  rules:
    - http:
        paths:
          - backend:
              service:
                name: <service-name>
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
```

```bash
kubectl create -f cross-vpc-ingress.yaml
```

## 验证

### 数据面（kubectl）

```bash
kubectl get ingress my-ingress
kubectl describe ingress my-ingress
```

```text
NAME  STATUS  AGE
...
```

确认 Ingress 状态正常且 CLB 已创建或关联成功。

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress my-ingress
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| Ingress 创建失败 | `kubectl describe ingress` | 云联网已正确配置 | 确认云联网已正确配置，两端 VPC 网段无冲突 |
| 跨 VPC 不可用 | 检查云联网路由配置 | 需先通过在线咨询申请开通该功能 | 需先通过在线咨询申请开通该功能 |
| 路由不生效 | 检查云联网路由配置 | 确认集群 VPC 未加入多个云联网 | 确认集群 VPC 未加入多个云联网，冲突路由不生效 |
| 可用区资源售罄 | `kubectl describe <resource>` | 使用随机可用区而非指定可用区 | 使用随机可用区而非指定可用区 |

## 下一步

- [Ingress 基本功能](../Ingress%20基本功能/tccli 操作.md)
- [Ingress 使用已有 CLB](../Ingress%20使用已有%20CLB/tccli 操作.md)
- [Ingress Annotation 说明](../Ingress%20Annotation%20说明/tccli 操作.md)

## 控制台替代

在控制台 **新建 Ingress > VPC** 中选择"其他 VPC"（需先配置云联网），可选指定可用区。
