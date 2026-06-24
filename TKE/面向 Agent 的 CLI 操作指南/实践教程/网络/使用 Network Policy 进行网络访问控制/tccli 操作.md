# 使用 Network Policy 进行网络访问控制（tccli）

> 对照官方：[使用 Network Policy 进行网络访问控制](https://cloud.tencent.com/document/product/457/19793) · page_id `19793`

## 概述

Network Policy 是 Kubernetes 提供的 Pod 网络隔离机制。TKE 通过 kube-router 组件实现 Network Policy 功能，支持基于 PodSelector、NamespaceSelector、IPBlock 的入站/出站流量控制。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群已 [连接](../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 已确认集群状态：`tccli tke DescribeClusters --cli-input-json file://query.json`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 安装 NetworkPolicy 组件 | `tccli tke InstallAddon --cli-input-json file://install.json` | 是（已安装则忽略） |
| 查看组件状态 | `tccli tke DescribeAddon --cli-input-json file://describe-addon.json` | 是 |
| 创建 NetworkPolicy | `kubectl apply -f network-policy.yaml` | 是 |
| 查看 NetworkPolicy | `kubectl get networkpolicies -A` | 是 |
| 删除组件 | `tccli tke DeleteAddon --cli-input-json file://delete-addon.json` | 否 |

## 操作步骤

### 1. 安装 NetworkPolicy 扩展组件

```bash
tccli tke InstallAddon --region ap-guangzhou --cli-input-json file://install.json
```

`install.json`:

```json
{
    "ClusterId": "<ClusterId>",
    "AddonName": "kube-router",
    "AddonVersion": "<AddonVersion>"
}
```

```output
{"RequestId": "..."}
```

### 2. 验证组件安装

```bash
tccli tke DescribeAddon --region ap-guangzhou --cli-input-json file://describe-addon.json
```

```json
{"ClusterId": "<ClusterId>", "AddonName": "kube-router"}
```

```output
{"AddonName": "kube-router", "AddonVersion": "v1.x.x", "Status": "Running"}
```

### 3. 创建 NetworkPolicy

**本 namespace 内 Pod 互访，拒绝外部访问：**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-ns
  namespace: default
spec:
  ingress:
  - from:
    - podSelector: {}
  podSelector: {}
  policyTypes:
  - Ingress
```

```bash
kubectl apply -f allow-same-ns.yaml
```

```text
# command executed successfully
```

```output
networkpolicy.networking.k8s.io/allow-same-ns created
```

**允许特定 Namespace 的 Pod 访问特定端口：**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ns
  namespace: default
spec:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          app: trusted
    ports:
    - protocol: TCP
      port: 6379
  podSelector: {}
  policyTypes:
  - Ingress
```

**允许 Egress 到指定 CIDR：**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-cidr
  namespace: default
spec:
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 443
  podSelector: {}
  policyTypes:
  - Egress
```

### 4. 查看已创建的 NetworkPolicy

```bash
kubectl get networkpolicies -A
```

```text
NAME  STATUS  AGE
...
```

```output
NAMESPACE   NAME              POD-SELECTOR   AGE
default     allow-same-ns     <none>         30s
default     allow-from-ns     <none>         15s
```

## 验证

### Data plane (kubectl)

```bash
kubectl get networkpolicies -A --no-headers | wc -l
kubectl describe networkpolicy allow-same-ns -n default
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 组件安装失败 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId <ClusterId> --AddonName kube-router` | 集群 Kubernetes 版本 < 1.14 不支持 kube-router | 升级集群至 v1.14+ 后重试 `InstallAddon` |
| NetworkPolicy 不生效 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId <ClusterId> --AddonName kube-router` | kube-router 组件未安装或状态异常 | 确认 Addon `Status` 为 Running；若未安装则执行 `InstallAddon` |
| 大规模集群性能下降 | `kubectl get pods -n kube-system -l k8s-app=kube-router` | kube-router 在 8000+ Pod 场景存在性能瓶颈 | 减少 NetworkPolicy 规则数量，或使用 Cilium 替代方案 |

## 清理

### Data plane (kubectl)

```bash
kubectl delete networkpolicy --all -n default
```

### Control plane (tccli)

```bash
tccli tke DeleteAddon --region ap-guangzhou --cli-input-json file://delete-addon.json
```

## 下一步

- [Nginx Ingress 最佳实践](../Nginx%20Ingress%20最佳实践/tccli%20操作.md)
- [Nginx 升级最佳实践](../Nginx%20升级最佳实践/tccli%20操作.md)

## 控制台替代

控制台：集群 → 组件管理 → 安装 → 选择 kube-router → 完成安装。
