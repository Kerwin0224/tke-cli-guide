# 使用 cert-manager 签发免费证书（tccli）

> 对照官方：[使用 cert-manager 签发免费证书](https://cloud.tencent.com/document/product/457/49368) · page_id `49368`
## 概述

通过 cert-manager 组件自动签发和管理免费 Let's Encrypt 证书，实现 TKE Ingress TLS 证书自动化。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 相关 kubectl 操作 | 见 Procedure | -- |

## 操作步骤

### 1. 控制面：安装 cert-manager 组件

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cert-manager
# expected: RequestId 返回，组件安装中
```

### 2. 控制面：验证安装

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> \
    --filter "AddonSet[?AddonName=='cert-manager'].Status"
# expected: "Running"
```

### 3. 数据面：创建 Issuer（需 VPN/IOA）

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: default
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: qcloud
```

```bash
kubectl apply -f issuer.yaml
# expected: issuer.cert-manager.io/letsencrypt-prod created
```

### 4. 数据面：创建 Certificate 资源

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
  namespace: default
spec:
  secretName: example-tls-secret
  issuerRef:
    name: letsencrypt-prod
  dnsNames:
    - example.com
```

```bash
kubectl apply -f certificate.yaml
# expected: certificate.cert-manager.io/example-tls created
```

### 5. 数据面：Ingress 中引用证书

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    kubernetes.io/ingress.class: qcloud
spec:
  tls:
    - hosts:
        - example.com
      secretName: example-tls-secret
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
kubectl apply -f ingress-tls.yaml
# expected: ingress.networking.k8s.io/tls-ingress created
```

### 6. 数据面：验证证书签发

```bash
kubectl get certificate
# expected: READY True

kubectl describe certificate example-tls
# expected: 显示证书状态和事件
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> \
    --filter "AddonSet[?AddonName=='cert-manager'].Status"
# expected: "Running"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get certificate
# expected: READY True

kubectl get secret example-tls-secret
# expected: tls Secret 已创建
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| Certificate READY False | `kubectl describe certificate <name>` 查看 Events | ACME 验证失败（HTTP-01 或 DNS-01） | 确认域名 A 记录指向 Ingress CLB；检查 Issuer 配置 |
| DNS 验证失败 | `kubectl describe certificate <name>` 查看 Challenges | 域名未正确解析到 CLB VIP | 确保域名 A 记录指向 Ingress CLB 的 VIP |
| Let's Encrypt 频率限制 | `kubectl describe certificate <name>` 查看 Events | 短时间内请求同一域名过多 | 使用 Staging server（`acme-staging-v02.api.letsencrypt.org`）调试 |

## 清理

```bash
kubectl delete certificate example-tls
kubectl delete issuer letsencrypt-prod
kubectl delete ingress tls-ingress
# expected: 资源删除成功
```

cert-manager 组件需通过 tccli 卸载：

```bash
tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName cert-manager
```

## 下一步

- [为 TKE Ingress 证书续期](../../网络/为%20TKE%20Ingress%20证书续期/tccli%20操作.md) -- page_id `49099`
- [Ingress 基本功能](../../../应用配置/服务和配置管理/Ingress/Ingress%20基本功能/tccli%20操作.md) -- page_id `31711`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
