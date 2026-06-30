# ingress-nginx 应用部署指南（tccli）

> 对照官方：[ingress-nginx 应用部署指南](https://cloud.tencent.com/document/product/457/116122) · page_id `116122`

## 概述

在 TKE 集群中部署示例应用（nginx），通过 ingress-nginx Ingress Controller 暴露服务并配置 host/path 路由规则。ingress-nginx 由社区维护，通过 Ingress 资源驱动 CLB 将外部流量路由到集群内 Service。本文覆盖从 Deployment/Service 创建到 Ingress 规则下发与 CLB 验证的完整链路。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` 已在内网/VPN 环境就绪
- 集群已安装 ingress-nginx（参见 [自建 Nginx Ingress 实践教程/快速开始](../自建%20Nginx%20Ingress%20实践教程/快速开始/tccli%20操作.md)）
- 已准备对外解析到 CLB 的域名（如 `example.example.com`）
- 集群为 MANAGED_CLUSTER 类型，K8s 1.30.0，Region=ap-guangzhou

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"，ClusterType 为 MANAGED_CLUSTER
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看集群 Ingress | `kubectl get ingress -A`（需 VPN/IOA） | 是 |
| 创建 Deployment | `kubectl apply -f deployment.yaml`（需 VPN/IOA） | 是 |
| 创建 Service | `kubectl apply -f service.yaml`（需 VPN/IOA） | 是 |
| 创建 Ingress | `kubectl apply -f ingress.yaml`（需 VPN/IOA） | 是 |
| 查看 CLB | `tccli clb DescribeLoadBalancers --region <Region>` | 是 |
| 查看 CLB 后端健康 | `tccli clb DescribeTargetHealth --region <Region> --LoadBalancerId CLB_ID` | 是 |

## 操作步骤

### 步骤 1：创建 Deployment 和 Service（需 VPN/IOA）

#### 选择依据

- **Deployment**：无状态应用首选，支持滚动更新和副本自愈。nginx 示例用 `replicas: 2` 保证基本可用
- **Service 类型 ClusterIP**：ingress-nginx 通过 Service ClusterIP 转发到后端 Pod，无需 NodePort/LoadBalancer
- **selector**：必须与 Deployment `spec.template.metadata.labels` 完全一致，否则 Endpoints 为空

#### 最小配置

```bash
cat > app-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-app
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nginx-app
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
EOF

kubectl apply -f app-deploy.yaml
# expected: deployment.apps/nginx-app created
# expected: service/nginx-app created
```

#### 增强配置（生产环境）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: default
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: nginx
        image: nginx:1.27
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 30"]
```

### 步骤 2：创建 Ingress 路由规则（需 VPN/IOA）

#### 选择依据

- **ingressClassName: nginx**：ingress-nginx 默认监听的 IngressClass，确保流量由 ingress-nginx 处理
- **host + path**：host 用于多域名路由，path 用于同域名下多服务路由。示例用 `example.example.com` + `/api` 前缀
- **rewrite-target 注解**：`nginx.ingress.kubernetes.io/rewrite-target: /` 将 `/api` 前缀剥离后转发到后端，适用于后端不识别前缀的场景

```bash
cat > ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: nginx-app
            port:
              number: 80
EOF

kubectl apply -f ingress.yaml
# expected: ingress.networking.k8s.io/nginx-app-ingress created
```

### 步骤 3：确认 Ingress 获取 CLB 地址（需 VPN/IOA）

```bash
kubectl get ingress nginx-app-ingress -n default
# expected: CLASS=nginx，ADDRESS 列出现 CLB 外网 IP（需 30s-2min）
```

```text
NAME  STATUS  AGE
...
```

```output
NAME                CLASS   HOSTS                  ADDRESS         PORTS   AGE
nginx-app-ingress   nginx   example.example.com    203.0.113.10    80      2m
```

### 步骤 4：控制面确认 CLB 已创建（tccli）

```bash
tccli clb DescribeLoadBalancers --region <Region>
# expected: 返回 CLB 列表，含 ingress-nginx controller 创建的 CLB，LoadBalancerType=OPEN（公网）
```

```output
{
  "TotalCount": 1,
  "LoadBalancerSet": [
    {
      "LoadBalancerId": "lb-example",
      "LoadBalancerName": "k8s-ingress-nginx-controller",
      "LoadBalancerType": "OPEN",
      "Status": 1
    }
  ]
}
```

## 验证

### 控制面（tccli）

```bash
# 确认 CLB 后端健康
tccli clb DescribeTargetHealth --region <Region> --LoadBalancerId CLB_ID
# expected: Targets 中 HealthStatus 为 HealthCheckPassNormal（1）
```

```bash
# 确认集群状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \
    --filter 'Clusters[0].ClusterStatus'
# expected: "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 确认 Endpoints 已就绪
kubectl get endpoints nginx-app -n default
# expected: ENDPOINTS 列有 Pod IP:80
```

```text
NAME  STATUS  AGE
...
```

```output
NAME         ENDPOINTS                       AGE
nginx-app    10.0.1.10:80,10.0.1.11:80       3m
```

```bash
# 确认 Ingress 已分配 ADDRESS
kubectl get ingress nginx-app-ingress -n default
# expected: ADDRESS 列有 CLB 外网 IP

# 通过 host 头访问验证路由（需 VPN/IOA 环境或公网）
curl -H "Host: example.example.com" http://<CLB-IP>/api
# expected: 200，返回 nginx 默认页
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

> **副作用警告**：删除 Ingress 后，ingress-nginx controller 会自动解绑并释放其创建的 CLB（约 1-2 分钟）。手动指定 `service.kubernetes.io/loadbalance-id` 复用的 CLB 不会被删除。

```bash
kubectl delete ingress nginx-app-ingress -n default
# expected: ingress.networking.k8s.io "nginx-app-ingress" deleted

kubectl delete deployment nginx-app -n default
# expected: deployment.apps "nginx-app" deleted

kubectl delete svc nginx-app -n default
# expected: service "nginx-app" deleted
```

### 控制面（tccli）

```bash
# 确认 CLB 已被自动释放（等待 1-2 分钟）
tccli clb DescribeLoadBalancers --region <Region>
# expected: 不再返回 ingress-nginx 相关 CLB；如仍存在，需手动删除

# 如 CLB 未自动删除（手动复用的 CLB），手动释放
tccli clb DeleteLoadBalancer --region <Region> --LoadBalancerIds '["CLB_ID"]'
# expected: RequestId 返回，无 Error
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Ingress ADDRESS 长时间为空 | `kubectl describe ingress nginx-app-ingress -n default` 查看 Events | ingress-nginx controller 未运行或 IngressClass 未匹配 | `kubectl get pods -n ingress-nginx` 确认 controller Running；`kubectl get ingressclass` 确认存在 `nginx` |
| 访问返回 404 | `kubectl describe ingress nginx-app-ingress` 核对 rules | host 或 path 不匹配请求 | 核对 curl 的 Host 头与 Ingress `host` 一致；`pathType: Prefix` 时 `/api` 匹配 `/api/*` |
| 访问返回 502 Bad Gateway | `kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx` | 后端 Service 不存在或端口与 Ingress backend 不一致 | `kubectl get svc nginx-app -n default` 确认 Service 存在且 port=80；`kubectl get endpoints nginx-app -n default` 确认有 Endpoints |
| SSL/TLS 证书报错（https 404/超时） | `kubectl get secret -n default` 查看是否配置 TLS Secret | Ingress 配置了 tls 但未提供有效证书 | 创建 tls Secret：`kubectl create secret tls tls-secret --cert=xxx --key=xxx -n default`，并在 Ingress `spec.tls` 引用 |
| CLB 后端健康检查失败 | `tccli clb DescribeTargetHealth --region <Region> --LoadBalancerId CLB_ID` | CLB 健康检查端口/路径与 Pod readinessProbe 不一致 | 确认 CLB 监听器后端端口为 Service targetPort（80）；健康检查路径与 readinessProbe path 一致 |
| Endpoints 为空（无 ADDRESS 但 Ingress 已创建） | `kubectl get endpoints nginx-app -n default` | Service selector 与 Deployment label 不匹配 | 核对 Service `spec.selector.app` 与 Deployment `spec.template.metadata.labels.app` 完全一致 |

## 下一步

- [自建 Nginx Ingress 实践教程/快速开始](../自建%20Nginx%20Ingress%20实践教程/快速开始/tccli%20操作.md) -- page_id `104857`
- [Nginx Ingress 最佳实践](../Nginx%20Ingress%20最佳实践/tccli%20操作.md) -- page_id `51685`
- [核心应用流量完全无损升级](../核心应用流量完全无损升级/tccli%20操作.md) -- page_id `104996`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 服务与路由 → Ingress → 新建/查看 YAML。
