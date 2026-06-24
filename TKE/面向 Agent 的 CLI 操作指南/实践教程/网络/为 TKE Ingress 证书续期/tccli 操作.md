# 为 TKE Ingress 证书续期（tccli）

> 对照官方：[为 TKE Ingress 证书续期](https://cloud.tencent.com/document/product/457/49099) · page_id `49099`
## 概述

为 TKE CLB 类型 Ingress 的 TLS 证书进行续期操作。通过更新 Kubernetes TLS Secret 实现证书更新。

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

### 1. 准备新证书

新证书需包含 `tls.crt` 和 `tls.key` 文件。

### 2. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### 3. 数据面：查看当前 TLS Secret（需 VPN/IOA）

```bash
kubectl get secret <tls-secret-name> -o yaml
# expected: 显示当前证书 tls.crt 和 tls.key
```

```text
NAME  STATUS  AGE
...
```

### 4. 数据面：更新 TLS Secret

```bash
# 删除旧 Secret
kubectl delete secret <tls-secret-name>
# expected: secret "<tls-secret-name>" deleted

# 创建新 Secret
kubectl create secret tls <tls-secret-name> \
  --cert=new-server.crt \
  --key=new-server.key
# expected: secret/<tls-secret-name> created
```

### 5. 数据面：验证证书更新

```bash
kubectl describe ingress <ingress-name>
# expected: TLS 部分引用新 Secret

# 测试 HTTPS 访问
curl -vI https://<domain>
# expected: 新证书生效
```

```text
NAME  STATUS  AGE
...
```

### 6. 使用 cert-manager 自动续期

安装 cert-manager 组件后可实现证书自动续期：

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cert-manager
# expected: 组件安装中
```

参见 [使用 cert-manager 签发免费证书](../../安全/使用%20cert-manager%20签发免费证书/tccli%20操作.md)。

## 验证

### 控制面

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

### 数据面（需 VPN/IOA）

```bash
# 验证证书有效期
kubectl get secret <tls-secret-name> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
# expected: 显示新证书的 notBefore/notAfter
```

```text
NAME  STATUS  AGE
...
```

## 清理

<!-- 旧证书 Secret 需手动清理 -->

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 证书更新后不生效 | `kubectl get ingress <name>` 查看 TLS 状态 | CLB 缓存未刷新 | 等待几分钟或重启 Ingress Controller |
| HTTPS 访问证书错误 | `kubectl get secret <tls-secret> -o yaml` 检查键名 | Secret 格式不正确 | 确认 tls.crt/tls.key 键名正确 |
| 域名不匹配 | `openssl x509 -in cert.pem -text -noout \| grep -A1 "Subject Alternative Name"` | 新证书 SAN 不包含域名 | 重新申请包含正确域名的证书 |

## 下一步

- [使用 cert-manager 签发免费证书](../../安全/使用%20cert-manager%20签发免费证书/tccli%20操作.md) -- page_id `49368`
- [Ingress 基本功能](../../../应用配置/服务和配置管理/Ingress/Ingress%20基本功能/tccli%20操作.md) -- page_id `31711`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
