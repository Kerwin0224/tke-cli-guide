---
doc_type: How-to
subtype: 6A
fused: true
---
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [注册节点价格和配额说明](https://cloud.tencent.com/document/product/457/79747)
>
> 配额：CLB 配额、LB 带宽限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：NodePort 暴露需安全组限制来源 IP；公网 LB 产生带宽费用；非健康后端致流量黑洞。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

# 流量接入

> 将注册节点的容器通过腾讯云负载均衡 CLB 对外暴露服务（四层 / 七层）。
> 控制台: [容器服务 - 节点池 - 注册节点](https://console.cloud.tencent.com/tke2/nodepool)

注册节点的容器对外暴露基于腾讯云 CLB，提供四层（Service 类型 LoadBalancer）与七层（Ingress）接入方案。

> 当前仅**注册节点（专线版）**支持接入 CLB；**公网版**暂时不支持 CLB。详见[创建注册节点（公网版）](public-network.md)。

## 触发条件

- 你已通过[专线版](dedicated-line.md)接入注册节点（CLB 仅专线版支持）。
- 你要把注册节点上运行的 Service / Ingress 通过 CLB 暴露到集群外。

## 四层接入（Service LoadBalancer）

为注册节点上的工作负载创建 `LoadBalancer` 类型 Service，集群自动为其创建 CLB 实例并绑定注册节点。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: registered-nodeport-svc
spec:
  type: LoadBalancer
  selector:
    app: registered-workload
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

> 注册节点的 Pod 需调度到注册节点池，CLB 后端才会包含对应节点。通过节点池 Labels / 工作负载 `nodeSelector` 约束调度目标。

## 七层接入（Ingress）

通过 Ingress 资源暴露 HTTP/HTTPS 路由，后端指向注册节点上的 Service。Ingress 控制器所用 CLB 同样仅专线版注册节点支持。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: registered-ingress
spec:
  rules:
    - host: "registered.example.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: registered-nodeport-svc
                port:
                  number: 80
```

## 验证

<!-- tccli管集群注册，kubectl管K8s原生Service/Ingress，非tccli边界 -->
> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
kubectl get svc registered-nodeport-svc -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# expected: 返回 CLB VIP，外部可访问
```

## 准备工作

- 已创建 TKE 标准集群（见[创建集群](../../clusters/create.md)）。
- 已配置 tccli 凭证（见[配置凭证](../../../getting-started/credentials.md)）。
- 已通过[专线版](dedicated-line.md)接入注册节点且节点在线（CLB 仅专线版支持，公网版不支持）。
- 已获取集群 kubeconfig 并可执行 `kubectl`（见[集群认证](../../security/auth.md)），本文操作步骤通过 `kubectl` 下发 Service / Ingress。
- 已确认流量接入层：四层（Service `LoadBalancer`）转发 TCP，七层（Ingress）转发 HTTP/HTTPS，按后端协议选择。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| CLB 未绑定注册节点 | Pod 未调度到注册节点池 | 检查 `nodeSelector` 与节点池 Labels |
| 公网版无法使用 CLB | 公网版不支持 CLB | 改用专线版接入 |

## 收尾确认

<!-- tccli管集群注册，kubectl管K8s原生Service/Ingress，非tccli边界 -->
> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
kubectl get svc registered-nodeport-svc -o wide
# expected: EXTERNAL-IP 为 CLB VIP，端点包含注册节点上的 Pod
```

## 下一步

- 专线版接入：[创建注册节点（专线版）](dedicated-line.md)
- 常见问题：[注册节点常见问题](faq.md)
