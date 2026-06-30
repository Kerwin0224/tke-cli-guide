# TKE 集群 CoreDNS 使用废弃 API 问题的解决方案（tccli）

> 对照官方：[TKE 集群 CoreDNS 使用废弃 API 问题的解决方案](https://cloud.tencent.com/document/product/457/112097) · page_id `112097`

## 概述

Kubernetes 升级后，CoreDNS 可能引用已废弃的 API 版本（如 `extensions/v1beta1`），导致无法正常工作。本文提供检查和修复方案。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running", ClusterVersion >= 1.22
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
| 查看集群版本 | `tccli tke DescribeClusters` | 是 |
| 检查废弃 API | `kubectl api-resources --sort-by=name`（需 VPN/IOA） | 是 |
| 升级 CoreDNS | `tccli tke UpdateAddon` | 否 |
| 检查 CoreDNS 日志 | `kubectl logs -n kube-system -l k8s-app=kube-dns`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：识别废弃 API 使用

#### 判断依据

TKE 集群从较低版本升级后，CoreDNS 的 RBAC 资源可能仍使用旧版 API：
- `rbac.authorization.k8s.io/v1beta1` 在 K8s 1.22+ 已废弃
- `extensions/v1beta1` 在 K8s 1.16+ 已废弃
- 废弃 API 会导致 CoreDNS 无法创建/更新所需 RBAC 资源

数据面检查（需 VPN/IOA）：

```bash
# 1. 检查 CoreDNS RBAC 资源使用的 API 版本
kubectl get clusterrole,clusterrolebinding -n kube-system -l k8s-app=kube-dns -o yaml | grep "apiVersion"
# expected: 如出现 v1beta1 需升级

# 2. 检查 CoreDNS 日志中的废弃 API 警告
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100 | grep -i "deprecated\|unknown\|api"
# expected: 如有废弃 API，含 "the API version has been deprecated"
```

```text
NAME  STATUS  AGE
...
```

### 步骤 2：升级 CoreDNS 组件

```bash
# 通过 tccli 升级 CoreDNS 到支持新版 API 的版本
cat > update-coredns.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "AddonName": "coredns",
    "AddonVersion": "TARGET_VERSION"
}
EOF
tccli tke UpdateAddon --region <Region> --cli-input-json file://update-coredns.json
# expected: exit 0
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | cls-xxxxxxxx | `tccli tke DescribeClusters` |
| `TARGET_VERSION` | 目标 CoreDNS 版本 | >= 1.9.3 | `tccli tke DescribeAddon` 查看可用版本 |
| `REGION` | 地域 | ap-guangzhou | `tccli configure list` |

### 步骤 3：手动清理旧版 RBAC

如果升级 CoreDNS 未自动迁移 RBAC：

```bash
# 删除旧版 RBAC 资源
kubectl delete clusterrole system:coredns
kubectl delete clusterrolebinding system:coredns
# expected: 资源已删除

# CoreDNS 重启后自动重建新版 RBAC
kubectl rollout restart deployment coredns -n kube-system
# expected: CoreDNS 滚动更新，Pod Running
```

### 步骤 4：验证修复

```bash
kubectl get clusterrole system:coredns -o yaml | grep apiVersion
# expected: rbac.authorization.k8s.io/v1

kubectl get pods -n kube-system -l k8s-app=kube-dns
# expected: Running，无重启

kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default
# expected: 解析正常
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns
# expected: Status "Running", AddonVersion >= 1.9.3
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
kubectl get clusterrole,clusterrolebinding -l k8s-app=kube-dns -o yaml | grep "apiVersion: rbac.authorization.k8s.io/v1"
# expected: 所有 RBAC 使用 v1

kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default.svc.cluster.local
# expected: 正常解析
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障修复页，通常无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| CoreDNS 升级后 RBAC 异常 | `kubectl get clusterrole system:coredns -o yaml` | 旧版 CoreDNS 创建的 RBAC 使用 v1beta1 API | `kubectl delete clusterrole system:coredns` + restart |
| CoreDNS 日志含 deprecated API warning | `kubectl logs -n kube-system -l k8s-app=kube-dns` | CoreDNS 内部使用已废弃 API 版本 | 升级 CoreDNS 到 >= 1.9.3 |
| 集群升级后 DNS 完全不可用 | `kubectl get pods -n kube-system \| grep coredns` | CoreDNS 版本与 K8s 版本不兼容 | `tccli tke UpdateAddon` 升级到兼容版本 |
| RBAC 自动迁移失败 | 检查 `kubectl get events -n kube-system` | CoreDNS 升级时权限不足 | 确认 CoreDNS SA 有 cluster-admin 或足够 RBAC 权限 |

## 下一步

- [CoreDNS 升级常见错误和处理](../../../../故障处理/CoreDNS%20升级常见错误和处理/tccli%20操作.md) -- page_id `130406`
- [从 kube-dns 切换到 CoreDNS](../从%20kube-dns%20切换到%20CoreDNS/tccli%20操作.md) -- page_id `97873`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 组件管理 -> CoreDNS -> 升级。
