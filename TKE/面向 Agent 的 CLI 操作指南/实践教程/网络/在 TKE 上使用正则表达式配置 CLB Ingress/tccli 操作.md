# 在 TKE 上使用正则表达式配置 CLB Ingress（tccli）

> 对照官方：[在 TKE 上使用正则表达式配置 CLB Ingress](https://cloud.tencent.com/document/product/457/122962) · page_id `122962`

## 概述

CLB 七层监听器支持基于 URL 路径的正则表达式路由规则，本文介绍在 TKE Ingress 中通过注解配置正则匹配，实现灵活的请求转发。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

tccli clb DescribeLoadBalancers --region <Region>
# expected: exit 0
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 创建 CLB Ingress | `kubectl apply -f ingress.yaml`（需 VPN/IOA） | 是 |
| 配置转发规则 | `tccli clb CreateRule` | 否 |
| 查看 CLB 监听器 | `tccli clb DescribeListeners` | 是 |
| 查看 Ingress | `kubectl get ingress -A`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：创建使用正则规则的 Ingress

#### 选择依据

- CLB 七层 URL 路径支持 `=~/regex/` 格式的正则表达式
- `nginx.ingress.kubernetes.io/use-regex: "true"` 启用正则匹配
- 适用场景：多版本 API 路由（`/api/v[12]/` ）、动态路径匹配（`/user/[0-9]+/`）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: regex-ingress
  namespace: NAMESPACE
  annotations:
    kubernetes.io/ingress.class: qcloud
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v[12]/users
        pathType: ImplementationSpecific
        backend:
          service:
            name: user-service-v1
            port:
              number: 8080
      - path: /api/v[3-9]/users
        pathType: ImplementationSpecific
        backend:
          service:
            name: user-service-v2
            port:
              number: 8080
      - path: /orders/[0-9]+
        pathType: ImplementationSpecific
        backend:
          service:
            name: order-service
            port:
              number: 8080
```

```bash
kubectl apply -f regex-ingress.yaml
# expected: ingress.networking.k8s.io/regex-ingress created
```

### 步骤 2：配置 CLB 七层转发规则

```bash
# 查看 CLB 监听器
tccli clb DescribeListeners --region <Region> --LoadBalancerId CLB_ID
# expected: 返回监听器列表

cat > create-regex-rule.json <<'EOF'
{
    "LoadBalancerId": "CLB_ID",
    "ListenerId": "LISTENER_ID",
    "Rules": [
        {
            "Domain": "api.example.com",
            "Url": "~/api/v[12]/users",
            "SessionExpireTime": 0,
            "HealthCheck": {
                "HealthSwitch": 1,
                "HttpCheckPath": "/health",
                "HttpCheckDomain": "api.example.com"
            }
        }
    ]
}
EOF
tccli clb CreateRule --region <Region> --cli-input-json file://create-regex-rule.json
# expected: exit 0
```

### 步骤 3：验证正则路由

```bash
kubectl get ingress regex-ingress -n NAMESPACE
# expected: ADDRESS 含 CLB VIP

# 验证规则已同步到 CLB
tccli clb DescribeListeners --region <Region> --LoadBalancerId CLB_ID
# expected: Rules 中含正则 URL 路径
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
tccli clb DescribeListeners --region <Region> --LoadBalancerId CLB_ID
# expected: 七层监听器含正则转发规则
```

### 数据面（需 VPN/IOA）

```bash
kubectl describe ingress regex-ingress -n NAMESPACE
# expected: Rules 含正则 path

kubectl get svc -n NAMESPACE user-service-v1 user-service-v2 order-service
# expected: 后端 Service Ready
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete ingress regex-ingress -n NAMESPACE
# expected: ingress deleted, CLB 转发规则自动移除

tccli clb DeleteRule --region <Region> --LoadBalancerId CLB_ID --ListenerId LISTENER_ID --LocationIds '["LOCATION_ID"]'
# expected: CLB 规则已删除（如手动创建）
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 正则路由不生效 | `kubectl describe ingress regex-ingress` | `use-regex: "true"` 注解缺失 | 添加注解并重建 Ingress |
| 路径匹配优先级异常 | 查看 CLB 规则顺序 | 精确匹配优先于正则，长路径优先 | 调整规则顺序，精确路径放前面 |
| 正则语法错误 | CLB CreateRule 返回 InvalidParameter | CLB 正则语法与标准 PCRE 有差异 | 使用 `~/pattern/` 格式，避免复杂断言 |
| Ingress 不创建 CLB 规则 | `kubectl describe ingress` 查看事件 | Ingress Controller 未运行或版本过旧 | `kubectl get pods -n kube-system` 确认 ingress 组件 |

## 下一步

- [Nginx Ingress 最佳实践](../Nginx%20Ingress%20最佳实践/tccli%20操作.md) -- page_id `51685`
- [ingress-nginx 应用部署指南](../ingress-nginx%20应用部署指南/tccli%20操作.md) -- page_id `116122`

## 控制台替代

[CLB 控制台](https://console.cloud.tencent.com/clb) -> 监听器 -> 七层转发规则。
