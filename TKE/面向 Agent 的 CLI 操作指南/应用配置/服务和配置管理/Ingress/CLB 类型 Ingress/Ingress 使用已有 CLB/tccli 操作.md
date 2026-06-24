# Ingress 使用已有 CLB（tccli）

> 对照官方：[Ingress 使用已有 CLB](https://cloud.tencent.com/document/product/457/45686) · page_id `45686`

## 概述

TKE 通过 `kubernetes.io/ingress.existLbId: <LoadBalanceId>` 注解支持 Ingress 使用已有负载均衡器（含包年包月类型），将负载均衡的生命周期管理从 Ingress Controller 中剥离，适用于需长期使用负载均衡且关注成本的场景。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已在 CLB 控制台创建好负载均衡器，且确认其空闲（无与 Ingress 冲突的端口，特别是 80 和 443）

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 选择已有 CLB | 添加 annotation `kubernetes.io/ingress.existLbId: lb-xxxxxxxx` | 否(首次) | 指定已有 CLB ID |
| 创建 Ingress | `kubectl create -f <ingress>.yaml` | 否(同名报错) | YAML 方式创建 |
| 查看关联 CLB | `kubectl describe ingress/<name>` | 是 | 查看 annotations 及 events |

## 操作步骤

> **注意：**
> - 容器业务不能与 CVM 业务共用一个 CLB
> - 不支持在 CLB 控制台操作 Ingress Controller 管理的监听器及后端服务器
> - 指定 CLB 的已有端口不能和 Ingress 定义的端口冲突（Ingress 通常会创建 80 和 443 端口）
> - 仅支持使用通过 CLB 控制台创建的负载均衡器，不支持 TKE 自动创建的 CLB
> - 删除 Ingress 不会自动删除 CLB

### 使用包年包月 CLB 创建 Ingress

```yaml
# nginx-ingress-exist-clb.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.existLbId: lb-xxxxxxxx
  name: nginx-ingress
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

```bash
kubectl create -f nginx-ingress-exist-clb.yaml
```

## 验证

### 数据面（kubectl）

```bash
kubectl get ingress nginx-ingress
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl describe ingress nginx-ingress
```

```text
Name:         ...
Status:       Running
...
```

查看 annotation 确认是否引用了正确的 CLB ID：

```bash
kubectl get ingress nginx-ingress -o jsonpath='{.metadata.annotations}'
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress nginx-ingress
```

> **注意：** 删除 Ingress 时 CLB 实例不会被删除。需手动在 CLB 控制台清理监听器和后端绑定，并清理 `tag tke-clusterId: cls-xxxx` 标签。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| CLB 端口冲突 | `tccli clb DescribeLoadBalancers` | 已有 CLB 的 80/443 端口未被占用 | 确认已有 CLB 的 80/443 端口未被占用 |
| 找不到已有 CLB | `tccli clb DescribeLoadBalancers` | CLB ID 正确 | 确认 CLB ID 正确，且是应用型（七层）CLB |
| 多 Ingress 复用 CLB | `ingress.cloud.tencent.com/enable-group: "true"` | 需使用 `ingress.cloud.tencent.com/enable-group: "true"` 注解 | 需使用 `ingress.cloud.tencent.com/enable-group: "true"` 注解 |
| 删除 Ingress 后 CLB 未释放 | `kubectl describe ingress` | 使用已有 CLB 时 Ingress Controller 不负责 CLB 删除 | 使用已有 CLB 时 Ingress Controller 不负责 CLB 删除，需手动处理 |

## 下一步

- [Ingress 基本功能](../Ingress%20基本功能/tccli 操作.md)
- [Ingress 使用 TkeServiceConfig 配置 CLB](../Ingress%20使用%20TkeServiceConfig%20配置%20CLB/tccli 操作.md)
- [多 Ingress 复用 CLB](../多%20Ingress%20复用%20CLB/tccli 操作.md)

## 控制台替代

在控制台 **新建 Ingress** 时，负载均衡器选择"使用已有"，从下拉列表中选择已创建的 CLB 实例。
