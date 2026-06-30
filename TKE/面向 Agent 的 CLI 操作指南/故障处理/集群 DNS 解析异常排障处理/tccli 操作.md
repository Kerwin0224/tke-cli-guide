# 集群 DNS 解析异常排障处理（tccli）

> 对照官方：[集群 DNS 解析异常排障处理](https://cloud.tencent.com/document/product/457/130406) · page_id `130406`

## 概述

排查 TKE 集群 DNS 解析异常问题，涵盖 CoreDNS 组件异常、配置错误、网络不可达等常见场景。通过 tccli 检查集群状态，配合 kubectl 检查 CoreDNS 组件和测试 DNS 解析。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)
- 已获取集群 kubeconfig（如需 kubectl 操作）

### 环境检查

```bash
tccli --version
# 预期输出: tccli version X.X.X
```

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER"
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / CLI | 幂等 |
|---|---|---|
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看节点实例 | `tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 查看 CoreDNS Pod | `kubectl get pods -n kube-system -l k8s-app=kube-dns`（需 VPN/IOA） | 是 |
| 查看 CoreDNS 日志 | `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50`（需 VPN/IOA） | 是 |
| 测试 DNS 解析 | `kubectl run test-dns --image=busybox --rm -it -- nslookup kubernetes.default`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤1：控制面检查集群

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0"
    }
  ]
}
```

### 步骤2：数据面检查 CoreDNS 组件（需 VPN/IOA）

```bash
# 检查 CoreDNS Pod 状态
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
# 预期: 所有 Pod STATUS 为 Running

# 检查 CoreDNS Service 和 Endpoints
kubectl get svc,endpoints -n kube-system kube-dns
# 预期: ClusterIP 和 Endpoints 正常

# 查看 CoreDNS 日志
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
# 预期: 无大量错误或超时日志
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：测试 DNS 解析（需 VPN/IOA）

```bash
# 集群内 DNS 测试
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default.svc.cluster.local
# 预期: 返回 kubernetes Service 的 ClusterIP

# 测试外网域名解析
kubectl run test-ext --image=busybox --rm -it --restart=Never -- nslookup www.qq.com
# 预期: 返回公网 IP 地址
```

### 步骤4：按症状诊断

**CoreDNS Pod 异常**

1. 检查 Pod Events：`kubectl describe pod -n kube-system <coredns-pod>`
2. 检查资源使用：`kubectl top pod -n kube-system -l k8s-app=kube-dns`
3. 若 CrashLoopBackOff，检查 CoreDNS ConfigMap 是否有语法错误

**集群内 DNS 解析超时**

1. 检查 kube-dns Service ClusterIP 是否可达
2. 检查 CoreDNS ConfigMap 中 upstream 配置
3. 检查节点 `/etc/resolv.conf` 中 DNS 配置是否与 CoreDNS 冲突

**外网域名无法解析**

1. 检查 CoreDNS ConfigMap 中 forward 配置的上游 DNS 地址
2. 确认节点可访问上游 DNS 服务器
3. 测试：`kubectl run test-net --image=busybox --rm -it -- nslookup www.qq.com 8.8.8.8`

### 步骤5：修复方法（需 VPN/IOA）

- **CoreDNS 副本不足**：`kubectl scale deploy -n kube-system coredns --replicas=3`
- **上游 DNS 不可达**：编辑 CoreDNS ConfigMap 修改 forward 地址
- **缓存问题**：`kubectl rollout restart deploy -n kube-system coredns`
- **内存不足 OOM**：增加 CoreDNS memory limits

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running"
    }
  ]
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
# 预期: 所有 CoreDNS Pod Running

kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
# 预期: 返回 ClusterIP
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。测试用 Pod 使用 `--rm` 会自动清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| CoreDNS CrashLoopBackOff | `kubectl logs` 显示配置解析错误 | CoreDNS ConfigMap 中 Corefile 语法错误 | 修正 ConfigMap 语法，注意 CoreDNS 特有的插件格式 |
| `nslookup` 返回 `server can't find` | CoreDNS 日志显示 `NXDOMAIN` | 域名不存在或上游 DNS 无法解析 | 确认域名拼写；检查上游 DNS 配置 |
| `nslookup` 返回 `connection timed out` | CoreDNS 日志无相关请求 | kube-dns Service 不可达或 kube-proxy 规则异常 | 检查 kube-dns Service ClusterIP 连通性；检查 kube-proxy |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 偶发 DNS 解析超时 | 监控 CoreDNS 延迟指标 | CoreDNS 副本数不足，高并发时排队 | 增加 CoreDNS 副本数；启用 CoreDNS autoscaler |
| 特定 Node 上 Pod 解析失败 | 在问题 Node 上运行测试 Pod | 节点 `/etc/resolv.conf` 配置异常或 conntrack 表满 | 检查节点 resolv.conf；调整 conntrack 表大小 |

## 下一步

- [集群 Kube-Proxy 异常排障处理](../集群%20Kube-Proxy%20异常排障处理/tccli%20操作.md) -- page_id `79797`
- [Pod 网络无法访问排查处理](../Pod%20网络无法访问排查处理/tccli%20操作.md) -- page_id `40332`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 组件管理 -> CoreDNS -> 查看日志和监控。
