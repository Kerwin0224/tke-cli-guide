# Ingress 混合使用 HTTP 及 HTTPS 协议（tccli）

> 对照官方：[Ingress 混合使用 HTTP 及 HTTPS 协议](https://cloud.tencent.com/document/product/457/45693) · page_id `45693`

## 概述

默认场景下 Ingress 只能以 HTTP 或 HTTPS 单一协议暴露服务。TKE 通过注解 `kubernetes.io/ingress.rule-mix` 支持混合协议，使一个 Ingress 可同时暴露 HTTP 和 HTTPS 服务。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已有证书 Secret（如需 HTTPS）

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 启用混合协议 | `kubernetes.io/ingress.rule-mix: "true"` | 是(apply) | 开启混合规则 |
| 指定 HTTP 规则 | `kubernetes.io/ingress.http-rules` | 是(apply) | JSON Array 格式 |
| 指定 HTTPS 规则 | `kubernetes.io/ingress.https-rules` | 是(apply) | JSON Array 格式 |
| 混合协议全映射 | `kubernetes.io/ingress.rule-mix-both: "true"` | 是(apply) | 自动映射到 HTTP 和 HTTPS |

## 操作步骤

### 规则格式

`kubernetes.io/ingress.http-rules` 和 `kubernetes.io/ingress.https-rules` 的格式为 JSON Array：

```json
{
  "host": "<domain>",
  "path": "<path>",
  "backend": {
    "serviceName": "<service name>",
    "servicePort": "<service port>"
  }
}
```

### 混合规则配置步骤

1. 在 Ingress 中添加注解 `kubernetes.io/ingress.rule-mix: "true"` 开启混合规则
2. 将每条转发规则分别匹配到 `kubernetes.io/ingress.http-rules` 和 `kubernetes.io/ingress.https-rules`
3. 若注解中未找到对应规则，默认添加到 HTTPS 规则集中
4. 匹配时校验 Host、Path、ServiceName 及 ServicePort（Host 默认为 VIP，Path 默认为 `/`）

> **注意：** IPv6 的负载均衡没有 IPv4 地址，不具备提供默认域名的功能。

### YAML 示例

```yaml
# sample-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.http-rules: '[{"host":"<domain>","path":"/","backend":{"serviceName":"sample-service","servicePort":"80"}}]'
    kubernetes.io/ingress.https-rules: '[{"host":"<domain>","path":"/","backend":{"serviceName":"sample-service","servicePort":"80"}}]'
    kubernetes.io/ingress.rule-mix: "true"
  name: sample-ingress
  namespace: default
spec:
  rules:
    - host: <domain>
      http:
        paths:
          - backend:
              service:
                name: sample-service
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
  tls:
    - secretName: <cert-secret>
```

```bash
kubectl create -f sample-ingress.yaml
```

### 混合协议全映射（v2.8.0+）

如果希望 Ingress 的转发规则同时映射到 HTTP 和 HTTPS，无需手动配置 http-rules 和 https-rules 注解：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.rule-mix-both: "true"
  name: sample-ingress
spec:
  rules:
    - host: <domain>
      http:
        paths:
          - backend:
              service:
                name: sample-service
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
  tls:
    - secretName: <cert-secret>
```

> **注意：** 配置了 `rule-mix-both` 后，不能再配置 `rule-mix`、`rewrite-support` 或 `auto-rewrite` 注解。必须配置 tls 证书。

## 验证

### 数据面（kubectl）

```bash
kubectl get ingress sample-ingress
kubectl describe ingress sample-ingress
```

```text
NAME  STATUS  AGE
...
```

确认 CLB 上 HTTP 和 HTTPS 监听器均已创建转发规则。

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress sample-ingress
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| HTTP 规则未生效 | `rule-mix: "true"` | `rule-mix: "true"` 已设置 | 确认 `rule-mix: "true"` 已设置，`http-rules` JSON 格式正确 |
| HTTPS 规则未生效 | `kubectl get secret` | TLS Secret 存在且证书 ID 正确 | 确认 TLS Secret 存在且证书 ID 正确 |
| IPv6 CLB 无法使用默认域名 | `tccli clb DescribeLoadBalancers` | IPv6 负载均衡无 IPv4 地址 | IPv6 负载均衡无 IPv4 地址，需显式指定 host |

## 下一步

- [Ingress 重定向](../Ingress%20重定向/tccli 操作.md)
- [Ingress 证书配置](../Ingress%20证书配置/tccli 操作.md)
- [Ingress Annotation 说明](../Ingress%20Annotation%20说明/tccli 操作.md)

## 控制台替代

控制台不支持直接配置混合协议，需通过 YAML 方式添加注解。
