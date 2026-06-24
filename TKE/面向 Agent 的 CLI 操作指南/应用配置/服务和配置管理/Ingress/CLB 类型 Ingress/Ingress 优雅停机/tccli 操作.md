# Ingress 优雅停机（tccli）

> 对照官方：[Ingress 优雅停机](https://cloud.tencent.com/document/product/457/60065) · page_id `60065`

## 概述

基于 CLB 直连 Pod 场景，当后端滚动更新或删除 Pod 时，通过优雅停机机制先将 CLB 后端权重置为 0，让 Pod 处理完已接收的请求后再摘除，避免客户端感知抖动和错误。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 确认集群支持 [直连模式](https://cloud.tencent.com/document/product/457/41897)

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 开启优雅停机 | `ingress.cloud.tencent.com/enable-grace-shutdown: "true"` | 是(apply) | v2.2.0 起废弃，默认开启 |
| 开启直连 | `ingress.cloud.tencent.com/direct-access: "true"` | 是(apply) | 前置条件 |
| 配置 preStop | Pod spec `lifecycle.preStop` | 是(apply) | sleep 等待规则同步 |
| 配置优雅时长 | Pod spec `terminationGracePeriodSeconds` | 是(apply) | 默认 30s |
| 不健康后端摘除 | `ingress.cloud.tencent.com/enable-grace-shutdown-tkex: "true"` | 是(apply) | v2.2.0 起废弃，默认开启 |

## 操作步骤

> **注意：** 仅针对 [直连场景](https://cloud.tencent.com/document/product/457/41897) 生效。

### 步骤 1：Ingress 开启直连 + 优雅停机

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/direct-access: "true"
    ingress.cloud.tencent.com/enable-grace-shutdown: "true"
  name: my-ingress
spec:
  rules:
    - host: <domain>
      http:
        paths:
          - backend:
              service:
                name: <service-name>
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
```

### 步骤 2：Pod 配置 preStop + terminationGracePeriodSeconds

#### 容器终止流程

1. Pod 被删除，DeletionTimestamp 设置，状态变为 Terminating，CLB 权重置为 0
2. kube-proxy 更新转发规则，将 Pod 从 endpoint 列表中摘除
3. 执行 preStop Hook
4. kubelet 发送 SIGTERM 信号
5. 等待 terminationGracePeriodSeconds（默认 30s），超时则 SIGKILL
6. 清理 Pod 资源

#### 配置 preStop（处理存量请求）

业务代码应处理 SIGTERM 信号：不接受新连接，只处理存量连接，全部断开后退出。

如果不能修改业务代码，可配置 preStop 延迟等待：

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

#### 自定义 terminationGracePeriodSeconds

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grace-demo
spec:
  terminationGracePeriodSeconds: 60
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

### 优雅退出（不健康后端处理）

Pod 运行中变为不健康时，将 CLB 后端权重置为 0：

```yaml
annotations:
  ingress.cloud.tencent.com/enable-grace-shutdown-tkex: "true"
```

该注解根据 Endpoint 对象中 endpoints 是否 not-ready，将 not-ready 的 CLB 后端权重置为 0。

```bash
kubectl apply -f graceful-shutdown-ingress.yaml
```

```text
# command executed successfully
```

## 验证

### 数据面（kubectl）

```bash
kubectl get ingress my-ingress
kubectl get pods -o wide
```

```text
NAME  STATUS  AGE
...
```

更新部署时观察 Pod 是否正常退出且无请求失败。

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress my-ingress
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 更新时请求失败 | `kubectl describe <resource>` | preStop 中 sleep 时间足够 kube-proxy 同步规则 | 确保 preStop 中 sleep 时间足够 kube-proxy 同步规则 |
| 优雅停机不生效 | `direct-access: "true"` | 直连模式已开启（`direct-access: "true"`） | 确认直连模式已开启（`direct-access: "true"`） |
| Pod 被强制 kill | `terminationGracePeriodSeconds` | 增加 `terminationGracePeriodSeconds` 值 | 增加 `terminationGracePeriodSeconds` 值 |

## 下一步

- [Ingress 证书配置](../Ingress%20证书配置/tccli 操作.md)
- [Ingress Annotation 说明](../Ingress%20Annotation%20说明/tccli 操作.md)

## 控制台替代

控制台不支持直接配置优雅停机，需通过 YAML 添加 Annotation。
