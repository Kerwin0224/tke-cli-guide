# Ingress 证书配置（tccli）

> 对照官方：[Ingress 证书配置](https://cloud.tencent.com/document/product/457/45738) · page_id `45738`

## 概述

通过 Ingress 的 `spec.tls` 字段为 CLB HTTPS 监听器配置服务器证书，支持为不同域名绑定不同证书、配置泛域名证书及双向认证。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已在 [SSL 证书控制台](https://console.cloud.tencent.com/ssl) 准备好证书

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 新建证书 | SSL 控制台购买/上传/申请免费证书 | 否 | 获取证书 ID |
| 创建 Secret | `kubectl create -f <secret>.yaml` | 否(同名报错) | 存储证书 ID |
| 创建含证书的 Ingress | `kubectl create -f <ingress>.yaml` | 否(同名报错) | spec.tls 引用 Secret |
| 修改证书 | `kubectl edit secret <name>` | 否 | 修改 `qcloud_cert_id` |
| 更新转发配置 | `kubectl apply -f <ingress>.yaml` | 是 | 更新 Ingress |

## 操作步骤

> **注意：**
> - Secret 证书资源需和 Ingress 在同一 Namespace
> - 不支持直接在其他证书服务或 CLB 上修改证书，会被 Secret 内容还原
> - 为域名增加匹配证书后，CLB SNI 功能将同步开启（不支持关闭）
> - 传统型负载均衡不支持多证书配置

### 1. 创建证书 Secret

#### 方式一：通过 YAML（自动 BASE64 编码）

```yaml
# cert-secret.yaml
apiVersion: v1
stringData:
  qcloud_cert_id: <cert-id>
kind: Secret
metadata:
  name: tencent-com-cert
  namespace: default
type: Opaque
```

#### 方式二：配置双向证书（需额外 CA 证书）

```yaml
apiVersion: v1
stringData:
  qcloud_cert_id: <server-cert-id>
  qcloud_ca_cert_id: <ca-cert-id>
kind: Secret
metadata:
  name: tencent-com-cert
  namespace: default
type: Opaque
```

```bash
kubectl create -f cert-secret.yaml
```

### 2. 创建使用证书的 Ingress

```yaml
# ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
spec:
  rules:
  - host: <domain>
    http:
      paths:
      - backend:
          service:
            name: <service-name>
            port:
              number: 443
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - <domain>
    secretName: tencent-com-cert
```

```bash
kubectl create -f ingress-tls.yaml
```

### 3. 修改证书

```bash
kubectl edit secret tencent-com-cert
```

修改 `qcloud_cert_id` 值为新的证书 ID，保存后 TKE 自动同步到 CLB。

## Ingress 证书配置行为

| 配置方式 | 行为 |
|---|---|
| 仅配置 `secretName` 未配置 `hosts` | 为所有 HTTPS 转发规则配置该证书 |
| 配置一级泛域名 `*.abc.com` | 泛域名通配生效 |
| 同时配置证书与泛域名证书 | 精确匹配优先（如 `www.abc.com` 优先于 `*.abc.com`） |
| 更新 Ingress 修改 TLS | 仅校验 SecretName 存在性，不校验 Secret 内容 |
| 修改 Secret 证书 ID | 所有引用该 Secret 的 Ingress 的 CLB 证书同步变更 |

## 验证

### 数据面（kubectl）

```bash
kubectl get secret tencent-com-cert
kubectl describe ingress example
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress example
kubectl delete secret tencent-com-cert
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 创建不通过 | `kubectl describe <resource>` | 所有 HTTPS 域名已配置证书 | 确保所有 HTTPS 域名已配置证书 |
| 证书同步后被覆盖 | `kubectl get secret` | 请勿在 CLB 或证书平台直接修改 TKE Ingress 关联的证书 | 请勿在 CLB 或证书平台直接修改 TKE Ingress 关联的证书 |
| 多证书不生效 | `kubectl get secret` | 传统型 CLB 不支持基于域名和 URL 转发 | 传统型 CLB 不支持基于域名和 URL 转发，因此不支持多证书 |
| 域名匹配多个证书 | `kubectl get secret` | 随机选择一个 | 随机选择一个，不建议同一域名使用不同证书 |

## 下一步

- [Ingress 配置双算法证书](../Ingress%20配置双算法证书/tccli 操作.md)
- [Ingress 基本功能](../Ingress%20基本功能/tccli 操作.md)
- [Ingress 混合使用 HTTP 及 HTTPS 协议](../Ingress%20混合使用%20HTTP%20及%20HTTPS%20协议/tccli 操作.md)

## 控制台替代

在控制台 **新建 Ingress > 转发配置** 选择 **HTTPS:443** 并绑定服务器证书。
