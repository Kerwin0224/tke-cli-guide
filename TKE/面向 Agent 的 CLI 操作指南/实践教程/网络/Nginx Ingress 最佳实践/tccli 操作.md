# Nginx Ingress 最佳实践（tccli）

> 对照官方：[Nginx Ingress 最佳实践](https://cloud.tencent.com/document/product/457/51685) · page_id `51685`

## 概述

TKE Nginx Ingress 配置最佳实践涵盖：日志采集配置、CLB 配置优化、Ingress Annotation 使用、健康检查、HTTPS 证书管理、会话保持等多个方面。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已安装 [NginxIngress 组件](../../../应用配置/组件和应用管理/组件管理/NginxIngress%20说明/tccli%20操作.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建 Ingress | `kubectl apply -f ingress.yaml` | 是 |
| 配置日志采集 | `kubectl apply -f` (annotation + CLS) | 是 |
| 配置 HTTPS | `kubectl create secret tls` + Ingress TLS | 是 |
| 查看 Ingress 状态 | `kubectl get ingress -A` | 是 |
| 查看 Ingress 日志 | `kubectl logs -n kube-system <nginx-pod>` | 是 |

## 操作步骤

### 1. 基础 Ingress 配置

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

```bash
kubectl apply -f app-ingress.yaml
```

```text
# command executed successfully
```

```output
ingress.networking.k8s.io/app-ingress created
```

### 2. HTTPS 配置

```bash
kubectl create secret tls app-tls --cert=cert.pem --key=key.pem -n default
```

```yaml
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http: ...
```

### 3. CLB 直连模式

```yaml
annotations:
  service.cloud.tencent.com/direct-access: "true"
```

### 4. 日志配置

通过 TkeServiceConfig 配置访问日志格式：

```yaml
apiVersion: cloud.tencent.com/v1alpha1
kind: TkeServiceConfig
metadata:
  name: ingress-log-config
spec:
  loadBalancer:
    l7Listeners:
    - protocol: HTTP
      port: 80
      keepaliveEnable: 1
      defaultServer: app.example.com
```

### 5. 常用 Annotation

| Annotation | 作用 | 示例值 |
|-----------|------|--------|
| `proxy-body-size` | 请求体大小限制 | `10m` |
| `proxy-connect-timeout` | 连接超时(秒) | `30` |
| `proxy-read-timeout` | 读取超时(秒) | `60` |
| `rewrite-target` | URL 重写 | `/` |
| `ssl-redirect` | HTTP→HTTPS | `true` |

## 验证

```bash
kubectl get ingress app-ingress -o wide
kubectl describe ingress app-ingress
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete ingress app-ingress
kubectl delete secret app-tls
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 502 Bad Gateway | `kubectl get svc -A \| grep <service-name>` 检查后端 Service | 后端 Service 不存在或端口不匹配 | 修正 Ingress backend.service.name 与 port；确认 Service 有 Ready Endpoints |
| 证书不生效 | `kubectl get secret <tls-secret> -o yaml` 检查 | `secretName` 与 `tls.hosts` 不匹配 | 确认 `secretName` 与 `tls.hosts` 域名一致 |
| CLB 创建失败 | `tccli vpc DescribeSubnets --region <Region>` 检查子网可用 IP | 子网 IP 不足 | 扩容子网或更换子网 |

## 下一步

- [Nginx Ingress 高并发实践](../Nginx%20Ingress%20高并发实践/tccli%20操作.md)
- [Nginx 升级最佳实践](../Nginx%20升级最佳实践/tccli%20操作.md)

## 控制台替代

控制台：集群 → 服务与路由 → Ingress → 新建。
