# CoreDNS ServiceAccount Token 过期问题解决方案（tccli）

> 对照官方：[CoreDNS ServiceAccount Token 过期问题解决方案](https://cloud.tencent.com/document/product/457/105020) · page_id `105020`

## 概述

CoreDNS ServiceAccount Token 过期会导致 CoreDNS 无法通过 APIServer 认证，造成 DNS 解析失败。本文介绍排查方法和自动续期方案。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看 CoreDNS 状态 | `tccli tke DescribeAddon --AddonName coredns` | 是 |
| 查看 SA Token | `kubectl get sa coredns -n kube-system -o yaml`（需 VPN/IOA） | 是 |
| 重建 SA Token | `kubectl delete sa coredns -n kube-system`（需 VPN/IOA） | 否 |
| 升级 CoreDNS | `tccli tke UpdateAddon` | 否 |

## 操作步骤

### 步骤 1：诊断 Token 过期

#### 判断依据

CoreDNS SA Token 过期时的典型症状：
- CoreDNS Pod 日志中出现 `Unauthorized` 或 `401` 错误
- `kubectl get pods -n kube-system` 显示 CoreDNS Pod 反复重启
- 集群内 DNS 解析间歇性失败

```bash
# 控制面检查
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns
# expected: Status "Running" 或异常状态值
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

数据面诊断（需 VPN/IOA）：

```bash
# 检查 CoreDNS Pod 日志
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50 | grep -i "unauthorized\|token\|401"
# expected: 如有 Token 过期，含 Unauthorized 错误

# 检查 SA Token 挂载
kubectl describe pod -n kube-system -l k8s-app=kube-dns | grep -A5 "Mounts:"
# expected: 含 /var/run/secrets/kubernetes.io/serviceaccount
```

```text
NAME  STATUS  AGE
...
```

### 步骤 2：修复 Token 过期

```bash
# 删除并重建 ServiceAccount（Kubernetes 自动生成新 Token）
kubectl delete secret -n kube-system -l kubernetes.io/service-account.name=coredns
kubectl rollout restart deployment coredns -n kube-system
# expected: CoreDNS Pod 重新创建并挂载新 Token

# 等待 Pod 就绪
kubectl wait --for=condition=ready pod -l k8s-app=kube-dns -n kube-system --timeout=120s
# expected: CoreDNS Pod Ready
```

### 步骤 3：验证修复

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20
# expected: 无 Unauthorized 错误

kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default
# expected: 解析成功
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns
# expected: Status "Running"
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
kubectl get pods -n kube-system -l k8s-app=kube-dns
# expected: Running，RESTARTS 为 0（新 Pod）

kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default.svc.cluster.local
# expected: 解析成功
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障页无需特殊清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| CoreDNS Pod 日志 Unauthorized | `kubectl logs -n kube-system -l k8s-app=kube-dns` 含 401 | SA Token 已过期，K8s < 1.22 Token 无自动续期 | `kubectl delete secret -n kube-system -l kubernetes.io/service-account.name=coredns` + restart |
| CoreDNS Pod CrashLoopBackOff | `kubectl describe pod -n kube-system -l k8s-app=kube-dns` | Token 过期导致健康检查失败，Pod 被 Kill | 按步骤 2 重建 Token |
| 修复后问题复现 | 观察 Token 创建时间 `kubectl get secret -n kube-system` | 集群版本 < 1.22 且未配置 Token 自动轮换 | 升级集群到 1.22+ 或配置 `--service-account-extend-token-expiration` |
| 无法重建 Token | `kubectl auth can-i delete secrets -n kube-system` | RBAC 权限不足 | 确认有 `secrets` 资源的 delete 权限 |

## 下一步

- [TKE DNS 最佳实践](../TKE%20DNS%20最佳实践/tccli%20操作.md) -- page_id `78005`
- [CoreDNS 升级常见错误和处理](../../../../故障处理/CoreDNS%20升级常见错误和处理/tccli%20操作.md) -- page_id `130406`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 组件管理 -> CoreDNS -> 重启。
