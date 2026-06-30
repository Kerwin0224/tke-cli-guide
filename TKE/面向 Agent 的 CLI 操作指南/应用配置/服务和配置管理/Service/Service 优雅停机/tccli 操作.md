# Service 优雅停机（tccli）

> 对照官方：[Service 优雅停机](https://cloud.tencent.com/document/product/457/60064) · page_id `60064`

## 概述

基于接入层直连 Pod 的场景，当后端进行滚动更新或后端 Pod 被删除时，如果直接将 Pod 从 CLB 后端摘除，则无法处理 Pod 已接收但还未处理的请求。特别对于长连接场景（如会议业务），直接更新或删除工作负载的 Pod 会导致连接直接中断。

优雅停机机制：在 Pod 的 CLB 后端权重置为 0 后，等待存量连接处理完毕再真正删除 Pod。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 仅针对 [直连场景](../使用%20LoadBalancer%20直连%20Pod%20模式%20Service/tccli%20操作.md) 生效
- 多 Service 复用相同 CLB 且同时更新多个 Service 后端时，优雅停机功能可能受影响

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群 | `tccli tke DescribeClusters` | 是 |
| 开启优雅停机 | annotation `service.cloud.tencent.com/enable-grace-shutdown: "true"` | 是(apply) |
| 开启直连模式 | annotation `service.cloud.tencent.com/direct-access: "true"` | 是(apply) |
| 配置 preStop Hook | YAML `spec.containers[].lifecycle.preStop` | 是(apply) |
| 配置优雅终止时间 | YAML `spec.terminationGracePeriodSeconds` | 是(apply) |

## 操作步骤

### 步骤1：使用 Annotation 标明优雅停机

```yaml
kind: Service
apiVersion: v1
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "true"          ## 开启直连 Pod 模式
    service.cloud.tencent.com/enable-grace-shutdown: "true"  ## 表示使用优雅停机
  name: my-service
spec:
  ports:
  - name: example
    port: 80
    targetPort: 80
  selector:
    app: anyserver
  type: LoadBalancer
```

```bash
kubectl apply -f my-service-grace-shutdown.yaml
```

### 步骤2：使用 preStop 和 terminationGracePeriodSeconds

#### 容器终止流程

1. Pod 被删除，此时 Pod 带有 DeletionTimestamp，状态置为 Terminating。CLB 到该 Pod 的权重调整为 0。
2. kube-proxy 更新转发规则，将 Pod 从 Service endpoint 列表中摘除，新流量不再转发到该 Pod。
3. 如果 Pod 配置了 preStop Hook，将会执行。
4. kubelet 对 Pod 中各容器发送 SIGTERM 信号，通知容器进程开始优雅停止。
5. 等待容器进程完全停止，如果在 `terminationGracePeriodSeconds` 内（默认 30s）还未完全停止，发送 SIGKILL 信号强制停止。
6. 所有容器进程终止，清理 Pod 资源。

#### 使用 preStop

务必在业务代码里处理 SIGTERM 信号：不接受新流量，继续处理存量流量，所有连接断开后再退出。

若无法在业务代码中处理 SIGTERM 信号，可通过 preStop 实现优雅终止：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      preStop:
        exec:
          command:
          - /clean.sh
```

在极端情况下，Pod 被删除的一小段时间内，kubelet 可能在 kube-proxy 同步完成规则前就停止容器。可利用 preStop 先 sleep 短暂时间等待 kube-proxy 规则同步：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      preStop:
        exec:
          command:
          - sleep
          - 5s
```

#### 使用 terminationGracePeriodSeconds 调整优雅时长

如果需要优雅终止时间较长（preStop + 业务进程停止可能超过 30s），可自定义 `terminationGracePeriodSeconds`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grace-demo
spec:
  terminationGracePeriodSeconds: 60  # 默认 30s，可设置更长时间
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      preStop:
        exec:
          command:
          - sleep
          - 5s
```

```bash
kubectl apply -f pod-grace.yaml
```

### 相关能力：优雅退出（Pod 不健康时权重置 0）

优雅停机仅在 Pod 删除时才把 CLB 后端权重置 0。若 Pod 在运行过程中出现不健康情况，可通过 Annotation 将 not-ready 的 CLB 后端权重置 0：

```yaml
metadata:
  annotations:
    service.cloud.tencent.com/enable-grace-shutdown-tkex: "true"
```

该注解根据 Endpoint 对象中 endpoints 是否 not-ready，将 not-ready 的 CLB 后端权重置为 0。

## 验证

### 数据面（kubectl）

```bash
# 查看 Service
kubectl get service my-service

# 查看 Pod 的 preStop 和 terminationGracePeriodSeconds 配置
kubectl get pod <pod-name> -o yaml | grep -A 10 terminationGracePeriodSeconds
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete service my-service --ignore-not-found
kubectl delete pod lifecycle-demo grace-demo --ignore-not-found
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 优雅停机不生效 | `direct-access: "true"` | 已开启直连模式（`direct-access: "true"`） | 确认已开启直连模式（`direct-access: "true"`）；仅直连场景生效 |
| 多 Service 复用 CLB 时优雅停机受影响 | `kubectl describe svc` | 避免同时更新复用同一 CLB 的多个 Service 后端 | 避免同时更新复用同一 CLB 的多个 Service 后端 |
| 部分请求在 Pod 删除时仍失败 | `kubectl describe <resource>` | 配置 preStop sleep 等待 kube-proxy 规则同步 | 配置 preStop sleep 等待 kube-proxy 规则同步 |
| 容器被 SIGKILL 强制终止 | `terminationGracePeriodSeconds` | 增加 `terminationGracePeriodSeconds` 值 | 增加 `terminationGracePeriodSeconds` 值 |
| 业务不处理 SIGTERM | `kubectl describe <resource>` | 在业务代码中处理 SIGTERM 信号 | 在业务代码中处理 SIGTERM 信号，或使用 preStop Hook |

## 下一步

- [使用 LoadBalancer 直连 Pod 模式 Service](../使用%20LoadBalancer%20直连%20Pod%20模式%20Service/tccli%20操作.md)
- [多 Service 复用 CLB](../多%20Service%20复用%20CLB/tccli%20操作.md)
- [Node 优雅下线](../Node%20优雅下线/tccli%20操作.md)
- [Pod 优雅删除](../Pod%20优雅删除/tccli%20操作.md)
- [Nginx Ingress Controller 后端解绑不优雅的问题](https://cloud.tencent.com/document/product/457/71216)

## 控制台替代

[控制台 → 集群 → 服务与路由 → Service](https://console.cloud.tencent.com/tke2/cluster) 编辑 Annotations，添加 `service.cloud.tencent.com/enable-grace-shutdown: "true"`。
