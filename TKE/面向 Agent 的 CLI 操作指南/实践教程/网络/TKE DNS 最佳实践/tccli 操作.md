# TKE DNS 最佳实践（tccli）

> 对照官方：[TKE DNS 最佳实践](https://cloud.tencent.com/document/product/457/78005) · page_id `78005`

## 概述

DNS 是 Kubernetes 服务发现的核心组件。本文涵盖 DNS 性能优化（ndots、缓存、TTL）、NodeLocal DNS Cache 部署、CoreDNS 副本数调优等最佳实践，确保集群 DNS 高可用与低延迟。

## 前置条件

- [环境准备](../../../环境准备.md)
- CoreDNS 为集群默认 DNS

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看 CoreDNS 配置 | `kubectl get configmap coredns -n kube-system -o yaml` | 是 |
| 调整 ndots | Pod `spec.dnsConfig.options` 或 CoreDNS ConfigMap | 是 |
| 调整 CoreDNS 副本数 | `kubectl scale deployment coredns -n kube-system --replicas=3` | 是 |
| 安装 NodeLocal DNS | `tccli tke InstallAddon --AddonName localdns` | 是 |
| 查看 DNS 性能 | `kubectl logs -n kube-system -l k8s-app=kube-dns` | 是 |

## 操作步骤

### 1. 优化 ndots 配置

默认 `ndots:5` 对短域名产生多次查询。调整为 `ndots:2`：

```yaml
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"
```

**集群级修改**（CoreDNS ConfigMap）：

```
kubernetes cluster.local in-addr.arpa ip6.arpa {
    pods insecure
    ndots 2
    fallthrough in-addr.arpa ip6.arpa
}
```

### 2. 启用 CoreDNS 缓存

```
cache 30 {
    success 9984 30
    denial 9984 5
}
```

### 3. 调整 CoreDNS 副本数

```bash
kubectl scale deployment coredns -n kube-system --replicas=3
```

```output
deployment.apps/coredns scaled
```

推荐：生产环境至少 3 副本，配合 PodAntiAffinity 分布在不同节点。

### 4. 启用 NodeLocal DNS Cache

参见 [NodeLocal DNS Cache](../DNS%20相关/在%20TKE%20集群中使用%20NodeLocal%20DNS%20Cache/tccli%20操作.md)。

### 5. 性能监控

```bash
kubectl get --raw /api/v1/namespaces/kube-system/services/kube-dns:9153/metrics | grep coredns_dns_request_duration
```

```text
NAME  STATUS  AGE
...
```

### 优化建议清单

| 配置项 | 建议值 | 说明 |
|--------|--------|------|
| ndots | 2-3 | 减少不必要的搜索域查询 |
| cache TTL | 30s | 平衡实时性与性能 |
| 副本数 | ≥3 | 高可用保障 |
| NodeLocal DNS | 启用 | 每节点缓存，降低 CoreDNS 压力 |

## 验证

```bash
kubectl run dns-perf --image=nginx:alpine --rm -it --restart=Never -- sh -c "time nslookup kubernetes.default 2>&1"
```

## 清理

```bash
kubectl scale deployment coredns -n kube-system --replicas=2
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| DNS 高延迟 | `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50` | CoreDNS 副本不足或负载高 | 启用 NodeLocal DNS Cache，增加 CoreDNS 副本数 |
| 解析失败 | `kubectl get pods -n kube-system -l k8s-app=kube-dns` | CoreDNS Pod 异常 | 检查 CoreDNS Pod 状态及日志 |
| 大量外部查询 | `kubectl logs -n kube-system -l k8s-app=kube-dns \| grep forward` | 外部域名直接走上游，无缓存 | 配置上游 DNS 缓存 |

## 下一步

- [在 TKE 中实现自定义域名解析](../DNS%20相关/在%20TKE%20中实现自定义域名解析/tccli%20操作.md)
- [从 kube-dns 切换到 CoreDNS](../DNS%20相关/从%20kube-dns%20切换到%20CoreDNS/tccli%20操作.md)

## 控制台替代

控制台：组件管理 → CoreDNS → 更新配置。
