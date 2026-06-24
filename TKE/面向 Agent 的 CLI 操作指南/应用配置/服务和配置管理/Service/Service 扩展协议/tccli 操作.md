# Service 扩展协议（tccli）

> 对照官方：[Service 扩展协议](https://cloud.tencent.com/document/product/457/51259) · page_id `51259`

## 概述

Kubernetes 原生的 Service 限制一个 Service 下暴露的多个端口协议必须相同。TKE 针对 LoadBalancer 模式扩展了更多协议支持，允许在一个 Service 上同时支持 TCP 和 UDP 混合，以及 TCP_SSL、HTTP、HTTPS、QUIC 等协议。

**扩展协议仅对 LoadBalancer 模式的 Service 生效**，通过注解 `service.cloud.tencent.com/specify-protocol` 描述协议与端口的关系。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群 | `tccli tke DescribeClusters` | 是 |
| 指定端口协议 | annotation `service.cloud.tencent.com/specify-protocol` | 是(apply) |
| 指定 TCP_SSL | annotation 值 `{"80":{"protocol":["TCP_SSL"],"tls":"cert-secret"}}` | 是(apply) |
| 指定 HTTP | annotation 值 `{"80":{"protocol":["HTTP"],"hosts":{"a.tencent.com":{}}}}` | 是(apply) |
| 指定 HTTPS | annotation 值 `{"443":{"protocol":["HTTPS"],"hosts":{"a.tencent.com":{"tls":"cert-secret"}}}}` | 是(apply) |
| 指定 QUIC | annotation 值 `{"80":{"protocol":["QUIC"],"tls":"cert-secret"}}` | 是(apply) |
| TCP/UDP 混合 | annotation 值 `{"80":{"protocol":["TCP","UDP"]}}`（仅直连模式支持） | 是(apply) |

## 操作步骤

### 前置说明

- 扩展协议仅对 LoadBalancer 模式的 Service 生效。
- **直连场景**下使用扩展协议没有限制，支持 TCP 和 UDP 协议混用。
- **非直连场景**下：
  - ClusterIP 和 NodePort 模式支持混用。
  - LoadBalancer 类型声明为 TCP 时，端口可将协议变更为 TCP_SSL、HTTP 或 HTTPS。
  - LoadBalancer 类型声明为 UDP 时，端口可将协议变更为 QUIC。

#### 注解与端口关系

| 规则 | 行为 |
|------|------|
| 扩展协议注解中未覆盖 Service Spec 端口 | 按 Service Spec 配置 |
| 扩展协议注解中描述的端口在 Service Spec 中不存在 | 忽略该配置 |
| 扩展协议注解中描述的端口在 Service Spec 中存在 | 覆盖 Service Spec 中声明的协议 |

### 扩展协议注解示例

#### TCP_SSL 示例

```json
{"80":{"protocol":["TCP_SSL"],"tls":"cert-secret"}}
```

#### HTTP 示例

```json
{"80":{"protocol":["HTTP"],"hosts":{"a.tencent.com":{},"b.tencent.com":{}}}}
```

#### HTTPS 示例

```json
{"443":{"protocol":["HTTPS"],"hosts":{"a.tencent.com":{"tls":"cert-secret-a"},"b.tencent.com":{"tls":"cert-secret-b"}}}}
```

#### TCP/UDP 混合示例（仅直连模式）

```json
{"80":{"protocol":["TCP","UDP"]}}
```

#### 混合示例（仅直连模式）

```json
{"80":{"protocol":["TCP_SSL","UDP"],"tls":"cert-secret"}}
```

#### QUIC 示例

```json
{"80":{"protocol":["QUIC"],"tls":"cert-secret"}}
```

> **注意**：TCP_SSL 和 HTTPS 中的字段 `cert-secret`，表示使用该协议需要指定一个证书（Opaque 类型的 Secret）。Secret 的 Key 为 `qcloud_cert_id`，Value 是证书 ID。详见 [Ingress 证书配置](https://cloud.tencent.com/document/product/457/45738)。

### 扩展协议使用说明

#### YAML 方式

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/specify-protocol: '{"80":{"protocol":["TCP_SSL"],"tls":"cert-secret"}}'
  name: test
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  type: LoadBalancer
```

```bash
kubectl apply -f test-service-extended.yaml
```

### 案例说明：TCP/UDP 混合协议（直连模式）

以下示例展示 80 端口使用 TCP 协议，8080 端口使用 UDP 协议。YAML 中仍使用相同的协议，但通过 Annotation 明确每个端口的协议类型。

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "true"  # 务必开启直连模式
    service.cloud.tencent.com/specify-protocol: '{"80":{"protocol":["TCP"]},"8080":{"protocol":["UDP"]}}'
  name: nginx
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: tcp-80-80
    nodePort: 32150
    port: 80
    protocol: TCP
    targetPort: 80
  - name: udp-8080-8080
    nodePort: 31082
    port: 8080
    protocol: TCP  # 注意：因 Kubernetes 限制，这里只能使用同类型协议，实际协议由 annotation 指定
    targetPort: 8080
  selector:
    k8s-app: nginx
    qcloud-app: nginx
  sessionAffinity: None
  type: LoadBalancer
```

```bash
kubectl apply -f nginx-tcp-udp.yaml
```

```text
# command executed successfully
```

## 验证

### 数据面（kubectl）

```bash
# 查看 Service
kubectl get service test

# 查看 Service 注解确认扩展协议
kubectl get service test -o yaml | grep specify-protocol

# 确认 Listeners
kubectl describe service test
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete service test nginx --ignore-not-found
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 扩展协议不生效 | `kubectl describe <resource>` | Service type 为 LoadBalancer | 确认 Service type 为 LoadBalancer |
| TCP/UDP 混合报错 | `kubectl describe <resource>` | 非直连模式下 LoadBalancer 不支持协议混用 | 非直连模式下 LoadBalancer 不支持协议混用，需开启直连模式 |
| TLS/HTTPS 证书不生效 | `qcloud_cert_id` | Secret 的 Key 为 `qcloud_cert_id` | 确认 Secret 的 Key 为 `qcloud_cert_id`，Value 为正确的证书 ID |
| 注解 JSON 格式错误 | `kubectl describe <resource>` | JSON 格式正确 | 确认 JSON 格式正确，键值对使用双引号 |
| 注解未覆盖的端口行为异常 | `kubectl get svc -o wide` | 扩展协议注解中未覆盖的端口按 Service Spec 原始配置生效 | 扩展协议注解中未覆盖的端口按 Service Spec 原始配置生效 |

## 下一步

- [多 Service 复用 CLB](../多%20Service%20复用%20CLB/tccli%20操作.md)
- [使用 LoadBalancer 直连 Pod 模式 Service](../使用%20LoadBalancer%20直连%20Pod%20模式%20Service/tccli%20操作.md)
- [CLB 目标组](../CLB%20目标组/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 服务与路由 → Service](https://console.cloud.tencent.com/tke2/cluster) 新建时选择"公网 LB"或"内网 LB"模式配置端口映射，或通过编辑 Annotation 添加 `specify-protocol`。
