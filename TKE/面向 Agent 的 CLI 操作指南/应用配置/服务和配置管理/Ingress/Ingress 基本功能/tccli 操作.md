# Ingress 基本功能（tccli）

> 对照官方：[Ingress 基本功能](https://cloud.tencent.com/document/product/457/31711) · page_id `31711`
## 概述

通过 kubectl 创建和管理 CLB 类型 Ingress，实现基于域名和 URL 路径的 HTTP/HTTPS 七层路由转发。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。
## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已部署后端 Service
- 已获取集群 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: 生成 kubeconfig.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 创建 Ingress | `kubectl apply -f ingress.yaml` | 是 |
| 查看 Ingress | `kubectl get ingress` | 是 |
| 查看详情 | `kubectl describe ingress/<name>` | 是 |
| 更新 Ingress | `kubectl apply -f ingress-updated.yaml` | 是 |
| 删除 Ingress | `kubectl delete ingress/<name>` | 是 |

## 操作步骤

### 1. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

### 2. 数据面：创建 Ingress（需 VPN/IOA）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: qcloud
    ingress.cloud.tencent.com/direct-access: "true"
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
# expected: ingress.networking.k8s.io/basic-ingress created
```

### 3. 数据面：多路径 Ingress

```yaml
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

### 4. 数据面：查看 Ingress 状态

```bash
kubectl get ingress
# expected: ADDRESS 列为 CLB VIP
```

预期输出：

```text
NAME            CLASS    HOSTS         ADDRESS        PORTS   AGE
basic-ingress   <none>   example.com   1.2.3.6        80      2m
```

```bash
kubectl describe ingress basic-ingress
# expected: 显示 Rules、Backend、Annotations、Events
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

### 数据面（kubectl，需 VPN/IOA）

```bash
kubectl get ingress
# expected: ADDRESS 非空

curl -H "Host: example.com" http://<ADDRESS>
# expected: 返回后端服务响应
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete ingress basic-ingress
# expected: ingress.networking.k8s.io "basic-ingress" deleted
```

CLB 由 TKE 自动管理，删除 Ingress 后自动释放。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Ingress ADDRESS 为空 | `kubectl describe ingress` | CLB 创建中或后端 Service 不存在 | `kubectl describe ingress` 查看 Events |
| 404 Not Found | `kubectl describe ingress` | URL 路径不匹配 | 检查 `spec.rules[].http.paths[].path` |
| 503 Service Unavailable | `kubectl get endpoints <svc>` | 后端 Service 无就绪 Pod | `kubectl get endpoints <svc>` 检查 |

## 下一步

- [Ingress 概述](../概述/tccli%20操作.md) -- page_id `45685`
- [Service 基本功能](../../Service/Service%20基本功能/tccli%20操作.md) -- page_id `45489`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 服务与路由 -> Ingress -> 新建。
