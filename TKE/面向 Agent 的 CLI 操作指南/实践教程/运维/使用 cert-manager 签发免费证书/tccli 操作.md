# 使用 cert-manager 签发免费证书（tccli）

> 对照官方：[使用 cert-manager 签发免费证书](https://cloud.tencent.com/document/product/457/49368) · page_id `49368`

## 概述

通过 cert-manager 自动化管理 TLS 证书，集成 Let's Encrypt 签发免费 SSL 证书。TKE 通过 `InstallAddon` 安装 cert-manager，配合 ClusterIssuer 和 Certificate CRD 实现自动签发和续期。

## 前置条件

- [环境准备](../../../环境准备.md)
- 域名 DNS 已指向 CLB IP

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 安装 cert-manager | `tccli tke InstallAddon --AddonName cert-manager` | 是 |
| 创建 ClusterIssuer | `kubectl apply -f issuer.yaml` | 是 |
| 创建 Certificate | `kubectl apply -f cert.yaml` | 是 |
| 查看证书状态 | `kubectl get certificate,certificaterequest,order` | 是 |

## 操作步骤

### 1. 安装

```bash
tccli tke InstallAddon --region ap-guangzhou --cli-input-json "{\"ClusterId\":\"<ClusterId>\",\"AddonName\":\"cert-manager\"}"
```

### 2. 创建 Let's Encrypt ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

```bash
kubectl apply -f cluster-issuer.yaml
```

### 3. 创建 Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
spec:
  secretName: example-tls-secret
  dnsNames:
  - app.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

### 4. Ingress 引用

```yaml
tls:
- hosts:
  - app.example.com
  secretName: example-tls-secret
```

## 验证

```bash
kubectl get certificate example-tls -o wide
```

```text
NAME  STATUS  AGE
...
```

```output
NAME          READY   SECRET              AGE
example-tls   True    example-tls-secret  2m
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get certificate` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl |
| `InstallAddon` 返回 `ClusterStatusError` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | 集群未处于 Running 状态 | 等待集群 Running 后重新安装 cert-manager |
| `Certificate READY False` | `kubectl describe certificaterequest`（VPN/IOA） | ACME http01 challenge 失败：域名未指向 CLB 或 ingress.class 不匹配 | 确认域名解析到 CLB IP；ClusterIssuer `ingress.class` 与实际 Ingress 一致 |

## 清理

```bash
kubectl delete certificate example-tls
```

## 下一步

- [为 TKE Ingress 证书续期](../为%20TKE%20Ingress%20证书续期/tccli%20操作.md)
- [Nginx Ingress 最佳实践](../../网络/Nginx%20Ingress%20最佳实践/tccli%20操作.md)

## 控制台替代

控制台：组件管理 → 安装 cert-manager。
