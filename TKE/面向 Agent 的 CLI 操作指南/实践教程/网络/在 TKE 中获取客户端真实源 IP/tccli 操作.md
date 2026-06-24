# 在 TKE 中获取客户端真实源 IP（tccli）

> 对照官方：[在 TKE 中获取客户端真实源 IP](https://cloud.tencent.com/document/product/457/48949) · page_id `48949`

## 概述

在负载均衡场景下获取客户端真实 IP 的四种方式对比：externalTrafficPolicy: Local、CLB 直连 Pod、Proxy Protocol、X-Forwarded-For 头。根据场景选择最佳方案。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已有 CLB 类型 Service 或 Ingress

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 方式1: Local 模式 | `kubectl apply -f` (spec.externalTrafficPolicy: Local) | 是 |
| 方式2: CLB 直连 | `kubectl apply -f` (annotation direct-access: "true") | 是 |
| 方式3: Proxy Protocol | TkeServiceConfig `l4Listeners[].proxyProtocol` | 是 |
| 方式4: X-Forwarded-For | Nginx Ingress 默认支持 | 是 |

## 操作步骤

### 方式1：externalTrafficPolicy: Local

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-local
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

缺点：流量仅转发到本节点 Pod，节点间负载不均。

### 方式2：CLB 直连 Pod（推荐）

```yaml
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "true"
```

优点：无 SNAT，天然保留源 IP，无额外配置。

### 方式3：Proxy Protocol

```yaml
apiVersion: cloud.tencent.com/v1alpha1
kind: TkeServiceConfig
metadata:
  name: nginx-proxy-config
spec:
  loadBalancer:
    l4Listeners:
    - protocol: TCP
      port: 80
      proxyProtocol:
        enable: true
```

Service 注解引用：

```yaml
annotations:
  service.cloud.tencent.com/tke-service-config: nginx-proxy-config
```

Nginx 侧需配置 `proxy_protocol on`。

### 方式4：HTTP X-Forwarded-For

Nginx Ingress 默认通过 X-Forwarded-For 头传递客户端 IP：

```bash
curl -H "Host: example.com" http://<Ingress-IP>/
```

在应用层获取：

```python
# Python Flask
request.headers.get('X-Forwarded-For')
```

### 方式对比

| 方式 | 协议层 | 源 IP 保留 | 配置复杂度 | 性能影响 |
|------|--------|-----------|-----------|---------|
| Local | L4 | 是 | 低 | 负载不均 |
| CLB 直连 | L4 | 是 | 低 | 无 |
| Proxy Protocol | L4+L7 | 是 | 中 | 低 |
| X-Forwarded-For | L7 | 是 | 低 | 无 |

## 验证

```bash
kubectl logs <nginx-pod> | grep "client_ip"
```

```text
...log output...
```

## 清理

```bash
kubectl delete svc nginx-local
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| Local 模式源 IP 仍为节点 IP | `kubectl describe svc <svc> \| grep -i externalTrafficPolicy` | CLB 健康检查未通过，流量被转发到其他节点 | 确认 CLB 健康检查通过 |
| Proxy Protocol 不生效 | `kubectl logs <nginx-pod> \| grep proxy_protocol` | Nginx 未启用 proxy_protocol 监听 | Nginx 侧需配置 `listen 80 proxy_protocol` |

## 下一步

- [在 TKE 上使用负载均衡直连 Pod](../在%20TKE%20上使用负载均衡直连%20Pod/tccli%20操作.md)
- [Nginx Ingress 最佳实践](../Nginx%20Ingress%20最佳实践/tccli%20操作.md)

## 控制台替代

控制台：Service → 编辑 → 高级配置 → 外部流量策略选择 Local。
