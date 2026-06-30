# 多 Ingress 复用 CLB（tccli）

> 对照官方：[多 Ingress 复用 CLB](https://cloud.tencent.com/document/product/457/127545) · page_id `127545`

## 概述

TKE 支持多个 Ingress 复用同一负载均衡器，通过 `ingress.cloud.tencent.com/enable-group: 'true'` 注解和 `kubernetes.io/ingress.existLbId` 注解实现流量聚合到单一 CLB 入口。支持跨命名空间复用，但每个"域名 + URL 路径"规则组合必须唯一。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 确保集群内 Service/Ingress Controller 版本 ≥ v2.10.0（如未满足请 [提交工单](https://console.cloud.tencent.com/workorder/category) 升级）
- 已在 CLB 购买页创建好负载均衡器

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作（Annotation） | 说明 |
|---|---|---|
| 启用多 Ingress 复用 | `ingress.cloud.tencent.com/enable-group: "true"` | 必须在创建时指定 |
| 指定已有 CLB | `kubernetes.io/ingress.existLbId: lb-xxxxxxxx` | 多个 Ingress 指向同一 CLB |
| 自定义监听端口 | `ingress.cloud.tencent.com/listen-ports` | 可各自使用不同端口 |
| 禁用手动重定向 | 不再需要 `rewrite-support` 注解 | 在复用场景下废弃 |

## 操作步骤

> **注意：**
> - 多个 Ingress 复用同一个 CLB 时，推荐复用数目不大于 100
> - 必须在创建 Ingress 时启用，已创建的 Ingress 不支持追加开启
> - 不支持复用 TKE CLB Ingress 自动创建的 CLB
> - 发生配置冲突时 Ingress Controller 将中断调谐并报错
> - 多个 Ingress 复用相同的监听器时，每个"域名 + URL 路径"必须唯一

### 查看组件版本

```bash
kubectl -n kube-system get cm tke-service-controller-config -o jsonpath='{.data.VERSION}'
kubectl -n kube-system get cm tke-ingress-controller-config -o jsonpath='{.data.VERSION}'
```

### 示例 1：跨命名空间，不同监听器

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ns1
---
apiVersion: v1
kind: Namespace
metadata:
  name: ns2
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/listen-ports: '[{"HTTP": 80}]'
    kubernetes.io/ingress.existLbId: lb-xxxxxxxx
    ingress.cloud.tencent.com/enable-group: "true"
  name: nginx-ingress-1
  namespace: ns1
spec:
  rules:
    - http:
        paths:
          - backend:
              service:
                name: nginx-service
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/listen-ports: '[{"HTTP": 81}]'
    kubernetes.io/ingress.existLbId: lb-xxxxxxxx
    ingress.cloud.tencent.com/enable-group: "true"
  name: nginx-ingress-2
  namespace: ns2
spec:
  rules:
    - http:
        paths:
          - backend:
              service:
                name: nginx-service
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
```

### 示例 2：不同 Ingress 复用相同监听器，各自使用不同域名

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ns1
---
apiVersion: v1
kind: Namespace
metadata:
  name: ns2
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/listen-ports: '[{"HTTP": 80}]'
    ingress.cloud.tencent.com/enable-group: "true"
    kubernetes.io/ingress.existLbId: lb-xxxxxxxx
  name: app1-ingress
  namespace: ns1
spec:
  rules:
  - host: app1.example.com
    http:
      paths:
      - backend:
          service:
            name: app1-server
            port:
              number: 80
        path: /
        pathType: Prefix
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/listen-ports: '[{"HTTP": 80}]'
    ingress.cloud.tencent.com/enable-group: "true"
    kubernetes.io/ingress.existLbId: lb-xxxxxxxx
  name: app2-ingress
  namespace: ns2
spec:
  rules:
  - host: app2.example.com
    http:
      paths:
      - backend:
          service:
            name: app2-server
            port:
              number: 80
        path: /
        pathType: Prefix
```

```bash
kubectl apply -f multi-ingress-reuse-clb.yaml
```

```text
# command executed successfully
```

## 支持进度（复用场景下）

| 注解 | 支持状态 |
|---|---|
| `ingress.cloud.tencent.com/listen-ports` | 支持 |
| `ingress.cloud.tencent.com/direct-access` | 支持 |
| `ingress.cloud.tencent.com/auto-rewrite` | 支持 |
| `kubernetes.io/ingress.rule-mix` | 支持 |
| `kubernetes.io/ingress.http-rules` | 支持 |
| `kubernetes.io/ingress.https-rules` | 支持 |
| `ingress.cloud.tencent.com/enable-grace-deletion` | 支持 |
| `ingress.cloud.tencent.com/lb-rs-weight` | 支持 |
| `ingress.cloud.tencent.com/tke-service-config` | 支持 |
| `ingress.cloud.tencent.com/pass-to-target` | 支持 |
| `ingress.cloud.tencent.com/tke-service-config-auto` | **不支持** |
| `ingress.cloud.tencent.com/rewrite-support` | **废弃** |

## 功能变更说明

### 手动重定向
`ingress.cloud.tencent.com/rewrite-support` 在复用场景下废弃。只需在 `kubernetes.io/ingress.http-rules` 和 `kubernetes.io/ingress.https-rules` 中配置重定向规则。

### 自动重定向
`ingress.cloud.tencent.com/auto-rewrite` 在复用场景下，删除注解与设为 `false` 均表示关闭自动重定向。

## 验证

### 数据面（kubectl）

```bash
kubectl get ingress -n ns1
kubectl get ingress -n ns2
kubectl describe ingress app1-ingress -n ns1
kubectl describe ingress app2-ingress -n ns2
```

```text
NAME  STATUS  AGE
...
```

检查注解中的 status conditions：
```bash
kubectl get ingress app1-ingress -n ns1 -o jsonpath='{.metadata.annotations}'
```

正常状态：
```
ingress.cloud.tencent.com/status.conditions: '[{"type":"Ready","status":"True",...}]'
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress app1-ingress -n ns1
kubectl delete ingress app2-ingress -n ns2
kubectl delete namespace ns1
kubectl delete namespace ns2
```

> **注意：** CLB 实例不会被自动删除，需手动在 CLB 控制台清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 端口冲突 | `kubectl get svc -o wide` | 不同 Ingress 的相同监听器下域名+路径必须唯一 | 不同 Ingress 的相同监听器下域名+路径必须唯一 |
| 配置冲突错误 | `status.conditions` | 注解 `status.conditions` 中的错误信息 | 检查注解 `status.conditions` 中的错误信息 |
| 存量 Ingress 无法开启 | `enable-group` | 不支持对已创建的 Ingress 追加 `enable-group` 注解 | 不支持对已创建的 Ingress 追加 `enable-group` 注解 |
| 与未指定 group 的 Ingress 共用 | `kubectl describe ingress` | 不支持 | 不支持，报错 ErrorCode: E4406 |
| 与 Service 共用 CLB | `kubectl describe svc` | ingress-controller < v2.12.0 不支持 | ingress-controller < v2.12.0 不支持 |
| 复用数过多导致配额问题 | `tccli clb DescribeQuota` | 联系负载均衡团队提交工单规划配额 | 联系负载均衡团队提交工单规划配额 |

## 下一步

- [Ingress Annotation 说明](../Ingress%20Annotation%20说明/tccli 操作.md)
- [Ingress 基本功能](../Ingress%20基本功能/tccli 操作.md)
- [Ingress 使用已有 CLB](../Ingress%20使用已有%20CLB/tccli 操作.md)

## 控制台替代

控制台不支持直接配置多 Ingress 复用 CLB，需通过 YAML 方式添加注解创建。
