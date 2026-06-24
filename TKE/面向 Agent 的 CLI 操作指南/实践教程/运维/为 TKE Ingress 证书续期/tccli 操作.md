# 为 TKE Ingress 证书续期（tccli）

> 对照官方：[为 TKE Ingress 证书续期](https://cloud.tencent.com/document/product/457/49099) · page_id `49099`

## 概述

TLS 证书到期后需更新 Secret 并触发 Ingress Controller 重载。cert-manager 可自动化此流程，手动方式则更新 Secret 后重启 Controller。

## 前置条件

- 已配置 HTTPS Ingress

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 更新证书 Secret | `kubectl create secret tls <name> --cert=cert.pem --key=key.pem --dry-run=client -o yaml \| kubectl apply -f -` | 是 |
| 查看证书过期 | `kubectl get secret <name> -o jsonpath='{.data.tls\.crt}' \| base64 -d \| openssl x509 -noout -dates` | 是 |
| 重启 Ingress | `kubectl rollout restart deploy nginx-ingress-controller -n kube-system` | 否 |

## 操作步骤

### 1. 检查证书过期时间

```bash
kubectl get secret example-tls -n default -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -enddate
```

```text
NAME  STATUS  AGE
...
```

### 2. 更新证书

```bash
kubectl create secret tls example-tls --cert=new-cert.pem --key=new-key.pem -n default --dry-run=client -o yaml | kubectl apply -f -
```

```text
# command executed successfully
```

### 3. 重启 Ingress Controller

```bash
kubectl rollout restart deployment nginx-ingress-controller -n kube-system
```

## 验证

```bash
curl -svo /dev/null https://app.example.com 2>&1 | grep "expire date"
```

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get secret` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl |
| 续期后 `curl` 仍显示旧过期时间 | `kubectl rollout status deploy nginx-ingress-controller -n kube-system`（VPN/IOA） | Ingress Controller 未重载新 Secret | `kubectl rollout restart deployment nginx-ingress-controller -n kube-system` |
| `kubectl get secret example-tls` 报 `NotFound` | `kubectl get secret -n default`（VPN/IOA） | Secret 名称/命名空间与 Ingress 引用不一致 | 核对 Ingress `secretName` 与实际 Secret 名称 |

## 下一步

- [使用 cert-manager 签发免费证书](../使用%20cert-manager%20签发免费证书/tccli%20操作.md)
- [Nginx Ingress 最佳实践](../../网络/Nginx%20Ingress%20最佳实践/tccli%20操作.md)

## 控制台替代

控制台：集群 → 配置管理 → Secret → 编辑。
